# Claude Code Prompt — Comentra eBay Integration (Korrigiert)

Baue/überarbeite die eBay-Integration in der Comentra SaaS-Plattform.
Zwei Teile: (1) Produkte hochladen, (2) Orders sync.
Deployment: docker compose up --build -d auf Hostinger VPS. Kein Vercel.

WICHTIG: Die bisherige eBay-Integration (src/lib/ebay/) ist unvollständig.
Der Upload würde beim ersten echten publishOffer scheitern weil
Business Policies und Inventory Location fehlen.
Diese müssen als Setup-Schritt pro Tenant einmalig angelegt werden.

---

## Architektur

Browser (React) → Next.js API Routes → eBay REST API
                        ↓
                  Supabase (Produkte + Orders)

Credentials pro Tenant in channels Tabelle (type = 'ebay'):
credentials: {
  client_id,          ← App ID aus developer.ebay.com
  client_secret,      ← Cert ID
  refresh_token,      ← OAuth User Token
  access_token,       ← kurzlebig, automatisch refreshed
  token_expires_at,   ← Timestamp
  merchant_location_key,  ← nach Setup einmalig gespeichert
  fulfillment_policy_id,  ← nach Setup einmalig gespeichert
  payment_policy_id,      ← nach Setup einmalig gespeichert
  return_policy_id        ← nach Setup einmalig gespeichert
}

---

## NEUER SCHRITT 0 — Einmaliges Setup pro Tenant (PFLICHT vor erstem Upload)

Diese Schritte müssen EINMALIG pro eBay-Channel ausgeführt werden.
Danach werden die IDs in channels.credentials gespeichert und wiederverwendet.

### 0a. Business Policies aktivieren
POST https://api.ebay.com/sell/account/v1/program/opt_in
{ "programType": "SELLING_POLICY_MANAGEMENT" }
→ Nur nötig wenn noch nicht aktiviert (HTTP 204 = OK, HTTP 409 = bereits aktiv)

### 0b. Fulfillment Policy erstellen (Versand)
POST https://api.ebay.com/sell/account/v1/fulfillment_policy
{
  "name": "Comentra Standard Versand DE",
  "marketplaceId": "EBAY_DE",
  "categoryTypes": [{ "name": "ALL_EXCLUDING_MOTORS_VEHICLES" }],
  "handlingTime": { "value": 2, "unit": "DAY" },
  "shippingOptions": [{
    "optionType": "DOMESTIC",
    "costType": "FLAT_RATE",
    "shippingServices": [{
      "shippingServiceCode": "DE_DHLPaket",
      "shippingCost": { "value": "0.00", "currency": "EUR" },
      "freeShipping": true,
      "buyerResponsibleForShipping": false,
      "sortOrder": 1
    }]
  }]
}
→ Speichere fulfillmentPolicyId in channels.credentials

### 0c. Return Policy erstellen (Rückgabe)
POST https://api.ebay.com/sell/account/v1/return_policy
{
  "name": "Comentra 30 Tage Rückgabe DE",
  "marketplaceId": "EBAY_DE",
  "categoryTypes": [{ "name": "ALL_EXCLUDING_MOTORS_VEHICLES" }],
  "returnsAccepted": true,
  "returnPeriod": { "value": 30, "unit": "DAY" },
  "returnMethod": "REPLACEMENT_OR_EXCHANGE",
  "returnShippingCostPayer": "SELLER"
}
→ Speichere returnPolicyId in channels.credentials

### 0d. Payment Policy erstellen
POST https://api.ebay.com/sell/account/v1/payment_policy
{
  "name": "Comentra PayPal DE",
  "marketplaceId": "EBAY_DE",
  "categoryTypes": [{ "name": "ALL_EXCLUDING_MOTORS_VEHICLES" }],
  "paymentMethods": [{ "paymentMethodType": "PAYPAL" }],
  "immediatePay": true
}
→ Speichere paymentPolicyId in channels.credentials

### 0e. Inventory Location erstellen (Lager)
POST https://api.ebay.com/sell/inventory/v1/location/{merchantLocationKey}
merchantLocationKey = "comentra-{tenantId-first-8-chars}"  (max 50 Zeichen)
{
  "name": "Hauptlager",
  "locationTypes": ["WAREHOUSE"],
  "location": {
    "address": {
      "city": "Werlte",
      "stateOrProvince": "Niedersachsen",
      "postalCode": "49716",
      "country": "DE"
    }
  },
  "merchantLocationStatus": "ENABLED"
}
→ Speichere merchantLocationKey in channels.credentials

### API Route für Setup:
POST /api/channels/[id]/ebay/setup
→ Führt alle 5 Schritte durch (idempotent — prüft ob bereits vorhanden)
→ Speichert alle IDs in channels.credentials
→ Frontend zeigt Setup-Status pro Channel an

---

## Teil 1 — Produkte hochladen (eBay Inventory API v1)

### Flow (4 Schritte):

**Schritt 1: Inventory Item erstellen**
PUT https://api.ebay.com/sell/inventory/v1/inventory_item/{sku}
Headers:
  Authorization: Bearer {access_token}
  Content-Language: de-DE   ← PFLICHT
  Content-Type: application/json
{
  "product": {
    "title": "...",
    "description": "...",
    "imageUrls": ["https://..."],
    "aspects": {
      "Marke": ["EmsCraft24"],
      "Material": ["Holz"],
      "Farbe": ["Natur"]
    },
    "ean": ["4251234567890"]
  },
  "condition": "NEW",
  "availability": {
    "shipToLocationAvailability": { "quantity": 10 }
  },
  "packageWeightAndSize": {
    "weight": { "value": 5.0, "unit": "KILOGRAM" },
    "dimensions": {
      "length": 120, "width": 120, "height": 30, "unit": "CENTIMETER"
    }
  },
  "regulatory": {
    "economicOperator": {
      "companyName": "EmsCraft24 GmbH & Co. KG",
      "street1": "...",
      "city": "Werlte",
      "postalCode": "49716",
      "country": "DE",
      "email": "info@emscraft24.de",
      "phone": "+49..."
    }
  }
}

**Schritt 2: Offer erstellen**
POST https://api.ebay.com/sell/inventory/v1/offer
Headers: Content-Language: de-DE
{
  "sku": "...",
  "marketplaceId": "EBAY_DE",
  "format": "FIXED_PRICE",
  "availableQuantity": 10,
  "categoryId": "...",       ← eBay Kategorie ID
  "listingDescription": "...",
  "pricingSummary": {
    "price": { "value": "39.99", "currency": "EUR" }
  },
  "merchantLocationKey": "{aus credentials}",
  "fulfillmentPolicyId": "{aus credentials}",
  "paymentPolicyId": "{aus credentials}",
  "returnPolicyId": "{aus credentials}"
}
→ Response: offerId

**Schritt 3: Offer publishen**
POST https://api.ebay.com/sell/inventory/v1/offer/{offerId}/publish
→ Response: listingId (eBay Item ID)
→ Speichern in products_marketplace.external_id

### Varianten (Multi-Variation Listing):
Schritt 1 (pro Variante):
PUT /sell/inventory/v1/inventory_item/{sku-variante}
→ Gleiche Struktur wie oben

Schritt 2 (Gruppe):
PUT /sell/inventory/v1/inventory_item_group/{inventoryItemGroupKey}
{
  "title": "...",
  "description": "...",
  "imageUrls": ["..."],
  "variantSKUs": ["SKU-NAT", "SKU-BRAUN"],
  "variesBy": {
    "aspectsImageVariesBy": ["Farbe"],
    "specifications": [
      { "name": "Farbe", "values": ["Natur", "Braun"] }
    ]
  }
}

Schritt 3 (pro Variante):
POST /sell/inventory/v1/offer (wie oben, pro SKU)

Schritt 4 (alle publishen):
POST /sell/inventory/v1/offer/{offerId}/publish PER VARIANTE
ODER: POST /sell/inventory/v1/inventory_item_group/{groupKey}/publish_offer

### Kategorie-Suche:
GET https://api.ebay.com/commerce/taxonomy/v1/category_tree/186/get_category_suggestions
  ?q={suchbegriff}
→ 186 = eBay DE Kategorie-Baum ID
→ Gibt passende categoryId Werte zurück

### API Routes:
POST /api/channels/[id]/ebay/setup           ← Einmaliges Setup
POST /api/channels/[id]/ebay/upload          ← Produkt hochladen
GET  /api/channels/[id]/ebay/categories      ← Kategorien suchen
GET  /api/channels/[id]/ebay/policies        ← Policies abrufen

---

## Teil 2 — Orders sync (eBay Fulfillment API v1)

### Cron (alle 5 Minuten):
GET https://api.ebay.com/sell/fulfillment/v1/order
  ?filter=lastmodifieddate:[{last_sync_at}Z..]
  &limit=200
→ Pagination: offset + limit

### Status Mapping:
AWAITING_PAYMENT   → "pending"
IN_PROGRESS        → "confirmed"
ALL_FULFILLED      → "shipped"
CANCELLED          → "cancelled"

### Payment Status Mapping:
PAID     → "paid"
PENDING  → "pending"
FAILED   → "failed"

### Tracking zurückschreiben:
POST https://api.ebay.com/sell/fulfillment/v1/order/{orderId}/shipping_fulfillment
{
  "lineItems": [{ "lineItemId": "...", "quantity": 1 }],
  "shippedDate": "2026-03-27T10:00:00.000Z",
  "shippingCarrierCode": "DHL",
  "trackingNumber": "1234567890"
}

---

## Dateistruktur

src/
  lib/
    ebay/
      auth.ts         ← Token-Refresh (bereits vorhanden, prüfen)
      setup.ts        ← NEU: Business Policies + Location Setup
      inventory.ts    ← ERWEITERN: GPSR + Content-Language Header
      fulfillment.ts  ← Orders sync (bereits vorhanden, prüfen)
      taxonomy.ts     ← Kategorie-Suche
  app/api/channels/[id]/ebay/
    setup/route.ts    ← NEU
    upload/route.ts   ← ERWEITERN
    categories/route.ts
    policies/route.ts

---

## .env.example
EBAY_CLIENT_ID=
EBAY_CLIENT_SECRET=
EBAY_MARKETPLACE_ID=EBAY_DE
EBAY_SANDBOX=false
EBAY_CATEGORY_TREE_ID=186   ← eBay DE

---

## Wichtige Hinweise
- Setup (Schritt 0) MUSS vor erstem Upload laufen — ohne Policies/Location schlägt publishOffer fehl
- Setup ist idempotent — prüfe ob Policies/Location bereits vorhanden bevor neu erstellt
- Content-Language: de-DE ist Pflicht-Header bei inventory_item Calls
- GPSR Economic Operator ist EU-Pflicht — immer mitgeben
- Sandbox: api.sandbox.ebay.com statt api.ebay.com
- tenant_id bei allen Supabase-Queries Pflicht
- Kommentare auf Deutsch
- Kein TypeScript-Fehler (npm run build)
- Deployment: docker compose up --build -d auf Hostinger VPS
- Am Ende: Git commit + push auf GitHub
