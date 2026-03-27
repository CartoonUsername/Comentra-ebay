# Claude Code Prompt — Comentra eBay Integration (Listing Upload + Order Sync)

Baue eine vollständige eBay-Integration für die Comentra SaaS-Plattform.
Zwei Teile: (1) Produkte von Comentra auf eBay hochladen, (2) eBay Orders in Supabase einspielen.
Deployment: docker compose up --build -d auf Hostinger VPS. Kein Vercel.

---

## Architektur

Browser (React) → Comentra Backend API (Next.js API Routes) → eBay REST API
                                    ↓
                              Supabase (Produkte + Orders)

Credentials pro Tenant in channels Tabelle (type = 'ebay'):
credentials: {
  client_id,        ← App ID aus eBay Developer Portal
  client_secret,    ← Cert ID aus eBay Developer Portal
  refresh_token,    ← OAuth User Token
  access_token,     ← kurzlebig, wird automatisch refreshed
  token_expires_at  ← Timestamp für Auto-Refresh
}

---

## Teil 1 — Produkte hochladen (eBay Inventory API)

### Flow (3 Schritte pro Produkt):

**Schritt 1: Inventory Item erstellen**
PUT https://api.ebay.com/sell/inventory/v1/inventory_item/{sku}
Content-Language: de-DE
{
  product: {
    title,
    description,
    imageUrls: [...],
    aspects: { "Marke": ["EmsCraft24"], "Material": ["Holz"] }
  },
  condition: "NEW",
  availability: {
    shipToLocationAvailability: { quantity: N }
  },
  packageWeightAndSize: {
    weight: { value, unit: "KILOGRAM" },
    dimensions: { length, width, height, unit: "CENTIMETER" }
  }
}

**Schritt 2: Offer erstellen**
POST https://api.ebay.com/sell/inventory/v1/offer
{
  sku,
  marketplaceId: "EBAY_DE",
  format: "FIXED_PRICE",
  availableQuantity: N,
  categoryId: "...",   ← eBay Kategorie-ID
  listingDescription: "...",
  pricingSummary: {
    price: { value: "39.99", currency: "EUR" }
  },
  fulfillmentPolicyId: "...",   ← aus eBay Account Policies
  paymentPolicyId: "...",
  returnPolicyId: "..."
}

**Schritt 3: Offer publishen**
POST https://api.ebay.com/sell/inventory/v1/offer/{offerId}/publish
→ Response: listingId (eBay Item ID)
→ Speichern in products_marketplace.external_id

### Varianten (Multi-Variation Listing):
- Für jede Variante: eigenes inventory_item (SKU der Variante)
- POST /sell/inventory/v1/inventory_item_group (Parent-Gruppe)
  {
    inventoryItemGroupKey: "PARENT_SKU",
    title, description, imageUrls,
    variantSKUs: ["SKU-ROT-S", "SKU-ROT-M", ...],
    aspects: { "Größe": ["S", "M", "L"], "Farbe": ["Rot", "Blau"] },
    variesBy: { aspectsImageVariesBy: ["Farbe"], specifications: [...] }
  }
- POST /sell/inventory/v1/offer/{offerId}/publish (pro Variante)
  ODER: publishOfferByInventoryItemGroup für alle auf einmal

### API Routes (Next.js):
POST /api/channels/[id]/ebay/upload
  → nimmt product_id + variant_ids aus Supabase
  → führt 3-Schritt-Flow aus
  → speichert external_id (eBay Item ID) in products_marketplace

GET /api/channels/[id]/ebay/categories?q=sandkasten
  → GET https://api.ebay.com/commerce/taxonomy/v1/category_tree/186/get_category_suggestions?q=...
  → gibt passende eBay Kategorie-IDs zurück

GET /api/channels/[id]/ebay/policies
  → GET https://api.ebay.com/sell/account/v1/fulfillment_policy?marketplace_id=EBAY_DE
  → gibt Versand/Rückgabe/Zahlungs-Profile zurück

---

## Teil 2 — Orders sync (eBay Fulfillment API)

### Cron (alle 5 Minuten, VPS crontab):
GET https://api.ebay.com/sell/fulfillment/v1/order
  ?filter=lastmodifieddate:[{last_sync_at}Z..]
  &limit=200

### Pro Order → Supabase upsert:
orders:
  channel_order_id = order.orderId
  status:
    AWAITING_PAYMENT  → "pending"
    ALL_FULFILLED     → "shipped"
    IN_PROGRESS       → "confirmed"
    CANCELLED         → "cancelled"
  payment_status:
    PAID     → "paid"
    PENDING  → "pending"
  currency = order.pricingSummary.total.currency
  subtotal = order.pricingSummary.priceSubtotal.value
  shipping_cost = order.pricingSummary.deliveryCost.value
  tax_amount = order.pricingSummary.tax.value
  total = order.pricingSummary.total.value
  shipping_address = order.fulfillmentStartInstructions[0].shippingStep.shipTo
  ordered_at = order.creationDate
  paid_at = order.paymentSummary.payments[0].paymentDate

order_items (pro lineItem):
  name = lineItem.title
  sku = lineItem.sku
  qty = lineItem.quantity
  unit_price = lineItem.lineItemCost.value
  total = lineItem.total.value
  tax_rate = 19  ← Default für DE

### Tracking zurückschreiben (wenn Comentra versendet):
POST https://api.ebay.com/sell/fulfillment/v1/order/{orderId}/shipping_fulfillment
{
  lineItems: [{ lineItemId, quantity }],
  shippedDate: "2026-03-26T10:00:00Z",
  shippingCarrierCode: "DHL",
  trackingNumber: "1234567890"
}

---

## OAuth Token Management

Token-Refresh Logik (in services/ebay/auth.js):
1. Prüfe channels.credentials.token_expires_at
2. Falls abgelaufen (oder < 5 Min): refresh via:
   POST https://api.ebay.com/identity/v1/oauth2/token
   grant_type=refresh_token
   refresh_token={token}
   Basic Auth: client_id:client_secret (Base64)
3. Neuen access_token + expires_in in channels.credentials speichern
4. Alle API-Calls gehen durch diesen Wrapper

---

## Dateistruktur

src/
  lib/
    ebay/
      auth.js           ← Token-Refresh Wrapper
      inventory.js      ← Inventory API (upload, varianten)
      fulfillment.js    ← Orders abrufen, Tracking schreiben
      taxonomy.js       ← Kategorie-Suche
      account.js        ← Policies abrufen
  app/api/channels/[id]/ebay/
    upload/route.js     ← POST: Produkt hochladen
    orders/route.js     ← GET: Orders sync triggern
    categories/route.js ← GET: Kategorien suchen
    policies/route.js   ← GET: Policies laden

comentra-connector/connectors/
  ebay.js               ← Cron Sync (bereits vorhanden, erweitern)

---

## Supabase Updates

products_marketplace:
  marketplace = 'ebay'
  external_id = eBay Item ID (nach publishOffer)
  status: 'pending' → 'active' nach erfolgreichem Publish
  error_message = eBay Fehlermeldung bei Misserfolg

channels:
  credentials.access_token    ← wird automatisch refreshed
  credentials.token_expires_at ← Timestamp

---

## .env.example Ergänzungen
EBAY_CLIENT_ID=           ← App ID aus developer.ebay.com
EBAY_CLIENT_SECRET=       ← Cert ID
EBAY_MARKETPLACE_ID=EBAY_DE
EBAY_SANDBOX=false        ← true für Tests

---

## Wichtige Hinweise
- tenant_id bei allen Supabase-Queries Pflicht
- Token-Refresh transparent im Wrapper — kein manueller Eingriff nötig
- eBay Sandbox für Tests: api.sandbox.ebay.com statt api.ebay.com
- Rate Limits: 5000 Calls/Tag für Inventory API
- Kommentare auf Deutsch
- Kein TypeScript-Fehler (npm run build)
- Deployment: docker compose up --build -d auf Hostinger VPS
- Am Ende: Git commit + push auf GitHub
