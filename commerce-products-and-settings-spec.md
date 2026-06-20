# Commerce: Products Hub & Settings — Scoped Spec

> Status: Draft for review
> Author: design pass, 2026-06-12
> Scope: Voxys Connect. Backend largely exists; this spec is mostly the
> missing UI + a few model fields + a settings layer over the existing commerce engine.

## 0. Context — what already exists (do NOT rebuild)

The commerce backend is ~70% there. Confirmed in the codebase:

- **Products / catalog**
  - `CatalogProduct` model (name, price, currency, availability, sku, category, image_url, additional_data).
  - `catalogs_controller` actions: `sync` (pull from Meta), `push_to_meta` (push out), `import_csv` (bulk create without a store), `products` (list).
  - `CatalogProducts.vue` — read-only products grid/list.
  - Create-Meta-catalog via Graph API (from the editor).
- **Cart / orders / recovery**
  - `Commerce::CartBuilder` — in-chat cart stored on the conversation.
  - `Commerce::{Salla,Zid,WooCommerce,Magento}::Adapter` — `create_order`, abandoned-cart listing, webhook registration.
  - `Commerce::AbandonedCartPollJob` (scheduled) — polls adapters for abandoned carts.
  - Order mapper supports **COD vs online** payment.

**Implication:** this is an *evolution*, not a new system. The work is UI + a few fields + a config layer.

---

## PART A — Products Hub

### A1. Problem
Today "products" only exist by syncing from a connected store (Salla/Zid/Woo/Magento → Meta → us).
Non-retail businesses with no store — clinics, salons, restaurants, tutoring, services — are locked
out, even though they want to list packages/services and send them on WhatsApp.

Also: the **Commerce** integration page (connect a store) duplicates what Integrations already does.

### A2. Decisions
1. **Rename / repurpose** the sidebar entry to **Products** (source-neutral, user's mental model).
2. **Kill the redundant "Commerce" connect-page**; store connection stays in Settings → Integrations.
3. **One owner per product** (avoids two-way-sync conflict hell):
   - `source: store`  → read-only here (the store is the source of truth; we only display).
   - `source: manual` → editable here, pushed out to Meta via `push_to_meta`.
   - The *catalog* is still effectively two-way (some flow in, some flow out); each *product* has a single owner.
4. **Product type**: `physical` (has quantity/stock) vs `service` (price only — packages, appointments, classes).

### A3. Model changes (`catalog_products`)
- `source:string` (default `manual`; `store` for synced) — set `store` on `sync`, `manual` on create/CSV.
- `quantity:integer` (nullable) — only meaningful for `physical`; ignored for `service`.
- `product_type:string` (default `physical`; `service`).
- (Keep existing `availability` enum; for services default to `in_stock`/N/A.)

### A4. Backend (mostly wiring existing actions)
- `POST /catalogs/:id/products` — create one product (manual). New, small.
- `PATCH /catalogs/:id/products/:pid`, `DELETE …` — edit/delete **manual** products only (guard `source == manual`).
- Reuse `import_csv` (bulk manual) and `push_to_meta` (publish manual → WhatsApp catalog) as-is.
- `sync` stamps `source: store` and leaves manual products untouched (don't let a store sync delete manual rows).

### A5. UI — "Products" page
- List all products with: image, name, type badge (physical/service), price, quantity (physical only),
  availability, source badge (Store / Manual), SKU.
- **Create/Edit form** (manual only): name, type, price+currency, quantity (if physical), description, image, SKU, category.
- Store-sourced rows: read-only, with a "managed by {store}" note.
- Actions: "Publish to WhatsApp catalog" (push_to_meta), CSV import, sync (for store catalogs).
- Filters: by source, type, availability; server-side search (the products endpoint already paginates/searches).

### A6. Meta-catalog reality (call out in UI)
Native WhatsApp **product messages** require a Meta catalog. So manual products must be `push_to_meta`'d
into a Meta catalog (which we can create via Graph API) before they can be sent as product cards.
Without a Meta catalog, products can still be sent as plain formatted/template messages — not native cards.

### A7. Gating
Same single switch as the rest of commerce: `feature_enabled?('whatsapp_catalog')` (or `ecommerce`).
However the switch is set (add-on, n8n Platform API, manual grant), the Products page responds.

---

## PART B — Commerce Settings

### B1. Problem
The commerce engine (cart, abandoned-cart polling, COD/online orders) runs on defaults with no
per-account control. Businesses need to tune recovery timing, messages, and COD behavior.

### B2. A "Commerce Settings" page (Settings → Commerce)
Stored per account (jsonb, e.g. `custom_attributes['commerce_settings']` or a dedicated table).
All behaviors gated by the same commerce feature switch.

**Abandoned-cart recovery**
- Enable / disable.
- Delay before first nudge (e.g. 1h / 3h / 24h) — feeds `AbandonedCartPollJob`.
- Recovery message / template (WhatsApp template picker), with cart summary + checkout link.
- Max nudges + spacing (e.g. up to 2, 24h apart) — avoid spam / Meta quality hits.
- Quiet hours (respect timezone) — important in MENA.

**COD (cash on delivery) confirmation**
- Enable / disable COD as a payment option.
- Confirmation flow: auto-send a confirm prompt ("Reply YES to confirm your COD order") before
  creating the order in the store adapter.
- COD order threshold / max amount (fraud guard).
- Optional: require address capture before COD confirm.

**Checkout / payment**
- Default payment mode (online via store checkout link vs COD).
- Online checkout: which adapter generates the link (already in `OrderMapper`).

**General**
- Currency display, order-status update messages (order.create / order.status.update webhooks already exist).
- Auto-reply when a catalog order message arrives (the incoming-order path exists).

### B3. Backend
- A `Commerce::Settings` reader/writer (jsonb) with sane defaults so nothing breaks if unset.
- `AbandonedCartPollJob` + recovery sender read delay/template/quiet-hours/caps from settings.
- COD path: order adapter consults COD enable + threshold + confirmation-required before `create_order`.

### B4. UI
- Settings → Commerce: grouped toggles + inputs (Recovery, COD, Checkout, General).
- Reuse existing template picker + components-next form controls.

---

## Sequencing (suggested)
1. **Products model fields** (`source`, `quantity`, `product_type`) + stamp `source` on sync/create.
2. **Products page** (list + manual create/edit + publish), repurpose sidebar to "Products", drop Commerce connect-page.
3. **Commerce Settings storage + defaults**, wire `AbandonedCartPollJob` to read them.
4. **Commerce Settings page** (Recovery, COD, Checkout, General).

## Non-goals / guardrails
- No true bidirectional per-product editing (one owner per product).
- No new feature flags (column is full) — gate via existing `ecommerce`/`whatsapp_catalog` switch.
- COD confirmation must gate order creation to avoid fake orders.
- Recovery must respect Meta messaging policy (template + quiet hours + caps).
