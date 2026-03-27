# Claude Code Prompt — Comentra Otto Integration (Listing Upload + Order Sync)

Baue eine vollständige Otto Partner API Integration für die Comentra SaaS-Plattform.
Analog zur bestehenden eBay Integration — gleiche Struktur, gleiche Patterns.
Deployment: docker compose up --build -d auf Hostinger VPS. Kein Vercel.

---

## Architektur

Browser (React) → Next.js API Routes → Otto Partner API v4
                        ↓
                  Supabase (Produkte + Orders)

Credentials pro Tenant in channels Tabelle (type = 'otto'):
credentials: {
  username,         ← Otto API-Nutzer (aus OPC Portal)
  password,         ← Otto API-Passwort
  access_token,     ← Bearer Token (kurzlebig)
  token_expires_at  ← Timestamp für Auto-Refresh
}

---

## Otto Partner API Basis

Base URL Live:    https://api.otto.market
Base URL Sandbox: https://sandbox.api.otto.market

Auth: OAuth2 Client Credentials
Token URL: POST https://api.otto.market/v1/token
  Body (form-urlencoded): grant_type=client_credentials&username={user}&password={pass}
  Response: { access_token, token_type, expires_in }
  Header: Authorization: Bearer {access_token}

Token-Refresh: automatisch wenn token_expires_at < now() + 5min
Neuen Token in channels.credentials speichern (service key).

---

## Teil 1 — Produkte hochladen (Otto Products API v4)

### Vorbereitungsschritte (PFLICHT vor Upload):

**Schritt 1: Marken abfragen**
GET https://api.otto.market/v4/products/brands
→ Liste aller bei Otto bekannten Marken
→ Markenname des Produkts muss exakt mit Otto-Markenname übereinstimmen

**Schritt 2: Kategorien abfragen**
GET https://api.otto.market/v4/products/categories
→ paginiert, limit=2000
→ Kategoriename muss exakt mit Otto-Kategorie übereinstimmen

**Schritt 3: Attribute pro Kategorie abfragen**
GET https://api.otto.market/v4/products/categories/{categoryName}/attributes
→ Gibt Pflicht-Attribute (featureRelevance=LEGAL → IMMER Pflicht)
→ Gibt empfohlene Attribute (relevance=HIGH → für bessere Sichtbarkeit)
→ Gibt variationThemes → welche Attribute Varianten unterscheiden

### Produkt hochladen:
POST https://api.otto.market/v4/products
Content-Type: application/json
[
  {
    "sku": "EC-SANDBOX-001-NAT",
    "productReference": "EC-SANDBOX-001",  ← max 50 Zeichen, gruppiert Varianten
    "category": "Sandkästen",               ← exakter Otto-Kategoriename
    "brand": "EmsCraft24",                  ← exakter Otto-Markenname
    "productLine": "Sandkasten Holz 120x120",
    "attributes": [
      { "name": "Material", "values": ["Holz"] },
      { "name": "Farbe", "values": ["Natur"] },
      { "name": "Breite (in cm)", "values": ["120"] },
      { "name": "Tiefe (in cm)", "values": ["120"] }
    ],
    "mediaAssets": [
      { "type": "IMAGE", "url": "https://..." }  ← mind. 1 Bild Pflicht
    ],
    "pricing": {
      "standardPrice": { "amount": 89.99, "currency": "EUR" }
    },
    "availability": {
      "quantity": 10
    },
    "shipping": {
      "type": "PARCEL",
      "weight": { "value": 5.0, "unit": "kg" },
      "dimensions": {
        "length": { "value": 120, "unit": "cm" },
        "width": { "value": 120, "unit": "cm" },
        "height": { "value": 30, "unit": "cm" }
      }
    },
    "description": "...",
    "bulletPoints": ["...", "...", "..."],
    "eans": ["4251234567890"]
  }
]

Response:
{
  "state": "pending",
  "links": [
    { "rel": "self", "href": "/v4/products/update-tasks/{taskId}" }
  ]
}

### Upload-Status pollen:
GET https://api.otto.market/v4/products/update-tasks/{taskId}
→ state: PENDING → warten
→ state: DONE → prüfe succeeded/failed Anzahl
→ Bei failed: GET /v4/products/update-tasks/{taskId}/failed → Fehlerdetails

### Marketplace-Status prüfen (2. Validierungsstufe):
GET https://api.otto.market/v4/products/marketplace-status?sku={sku}
→ Gibt finalen Status auf otto.de
→ Link zur Produktseite wenn erfolgreich

### Varianten (Multi-Variation):
- Gleiche productReference = werden automatisch als Varianten zusammengefasst
- Bsp: productReference = "EC-RAHMEN-001"
  Variante 1: sku "EC-RAHMEN-001-10x15", attribute "Bildformat": ["10x15 cm"]
  Variante 2: sku "EC-RAHMEN-001-13x18", attribute "Bildformat": ["13x18 cm"]
- variationThemes aus Kategorie-Attributen bestimmen was Varianten unterscheidet

### Preise/Bestand aktualisieren:
PATCH https://api.otto.market/v4/products/{sku}/prices
PATCH https://api.otto.market/v4/products/{sku}/quantities

### API Routes (Next.js):
POST /api/channels/[id]/otto/upload
  → prüft Brands + Categories in Supabase Cache (24h)
  → baut Otto-Payload aus Supabase-Produktdaten
  → POST zu Otto API → taskId
  → pollt taskId alle 5s bis DONE
  → speichert external_id (Otto SKU) in products_marketplace

GET /api/channels/[id]/otto/categories
  → cached in Supabase 24h

GET /api/channels/[id]/otto/brands
  → cached in Supabase 24h

---

## Teil 2 — Orders Sync (Otto Orders API v4)

### Cron (alle 5 Minuten):
GET https://api.otto.market/v4/orders?fulfillmentStatus=PROCESSABLE
→ Pagination via nextcursor:
  while (response.links.next) {
    GET https://api.otto.market/v4/orders?nextcursor={cursor}
  }

### Otto Order Status → Comentra Status Mapping:
ANNOUNCED             → "pending"      ← noch nicht bearbeitbar, Adresse fehlt
PROCESSABLE           → "confirmed"    ← bereit zur Bearbeitung
SENT                  → "shipped"
RETURNED              → "returned"
CANCELLED_BY_PARTNER      → "cancelled"
CANCELLED_BY_MARKETPLACE  → "cancelled"

### Pro Order → Supabase upsert:
orders:
  channel_order_id = order.salesOrderId
  status = mapping oben
  payment_status = "paid"              ← Otto zahlt immer vor Versand
  currency = "EUR"
  subtotal = summe positionItems * itemValueGrossPrice.amount
  shipping_cost = 0                    ← Otto übernimmt Versand
  total = subtotal
  shipping_address = order.deliveryAddress (jsonb)
  ordered_at = order.orderDate
  metadata = { otto_order_id: order.salesOrderId, positionItems: [...ids] }

order_items (pro positionItem):
  name = positionItem.product.title
  sku = positionItem.sku
  qty = positionItem.quantity (meist 1)
  unit_price = positionItem.itemValueGrossPrice.amount
  total = unit_price * qty
  tax_rate = 19

### Tracking zurückschreiben (PFLICHT bei Otto!):
POST https://api.otto.market/v4/shipments
{
  "trackingKey": {
    "carrier": "DHL",                   ← DHL, HERMES, DPD, GLS, UPS etc.
    "trackingNumber": "1234567890"
  },
  "shipDate": "2026-03-27T10:00:00+01:00",
  "positionItems": [
    {
      "positionItemId": "...",
      "salesOrderId": "..."
    }
  ]
}
→ KRITISCH: Ohne Tracking bleibt Otto-Auftrag offen → schlechte Lieferzeit-KPI

### Sandbox Test-Orders generieren:
POST https://sandbox.api.otto.market/v4/orders/testorders
→ Erzeugt sofort 8 vordefinierte Test-Szenarien:
  6x PROCESSABLE, 1x ANNOUNCED (Vorauszahlung), 1x CANCELLED_BY_MARKETPLACE

---

## Dateistruktur (analog zu eBay)

src/
  lib/
    otto/
      auth.ts           ← Token-Refresh Wrapper
      products.ts       ← Products API v4 (upload, brands, categories, status)
      orders.ts         ← Orders API v4 (sync mit nextcursor Pagination)
      shipments.ts      ← Tracking zurückschreiben
  app/api/channels/[id]/otto/
    upload/route.ts         ← POST: Produkt hochladen
    upload-status/route.ts  ← GET: taskId Status pollen
    orders/route.ts         ← GET: Orders sync triggern
    categories/route.ts     ← GET: Kategorien (24h Cache)
    brands/route.ts         ← GET: Marken (24h Cache)

comentra-connector/connectors/
  otto.js               ← Cron Sync (bereits vorhanden, erweitern)

---

## Supabase

products_marketplace:
  marketplace = 'otto'
  external_id = Otto SKU (nach DONE + marketplace-status OK)
  status: 'pending' → 'active' nach erfolgreichem Upload
  error_message = Otto Fehlermeldung bei failed

schema_cache (bereits vorhanden):
  Brands + Categories 24h cachen
  product_type = 'otto_brands' / 'otto_categories_{page}'

---

## .env.example Ergänzungen
OTTO_API_BASE_URL=https://api.otto.market
OTTO_SANDBOX_URL=https://sandbox.api.otto.market
OTTO_USE_SANDBOX=false    ← true für Tests

---

## Wichtige Otto-spezifische Hinweise
- Otto erstellt Rechnungen selbst → KEINE Rechnungen für Otto-Orders in Comentra
- Tracking ist PFLICHT — fehlende Sendungsnummern → schlechte Seller-KPI
- Produktupload ist ASYNCHRON — taskId immer bis DONE pollen
- ANNOUNCED Orders haben noch keine Lieferadresse → erst bei PROCESSABLE verarbeiten
- Brands + Categories müssen exakt mit Otto-Namen übereinstimmen → immer erst abfragen
- Attribute mit featureRelevance=LEGAL sind absolut Pflicht
- Sandbox zurücksetzen: jeden ersten Sonntag im Monat
- Pagination: nextcursor statt page/offset
- tenant_id bei allen Supabase-Queries Pflicht
- Kommentare auf Deutsch
- Kein TypeScript-Fehler (npm run build)
- Deployment: docker compose up --build -d auf Hostinger VPS
- Am Ende: Git commit + push auf GitHub
