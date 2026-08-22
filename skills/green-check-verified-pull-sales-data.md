---
name: green-check-verified-pull-sales-data
description: Pull normalized sales, product, inventory and customer data for a connected cannabis-related business from Green Check Access, paginating correctly across POS systems.
api: Green Check Access
generated: '2026-08-22'
method: generated
source: openapi/green-check-verified-access-openapi.yaml + https://developer.greencheckverified.com/tutorials/api-integration-workflow + https://developer.greencheckverified.com/guides/insights-quickstart
operations:
  - get-token
  - get-service-provider-crbs
  - get-crb-sales
  - get-crb-products
  - get-crb-inventory
  - get-crb-inventory-locations
  - get-crb-inventory-by-location
  - get-crb-customers
---

# Pull operational data for a CRB

Read-only. This is the "Insights" half of Green Check Access: one contract across 20+ point-of-sale
and seed-to-sale systems.

## Step 1 — Token (`get-token`)

`POST /auth/token` with client credentials. Bearer token, 3600 seconds. You need
`point-of-sale:read` in the returned `scope[]` for everything below.

## Step 2 — Find CRBs that actually have data (`get-service-provider-crbs`)

`GET /service-providers/{sp_id}/crbs`

**Check `pos_integration_status` on each CRB before querying.** Sales, product, inventory and customer
data exist only for CRBs with a connected POS or invoice-tracking system. A CRB onboarded through the
connect-integration path with no POS will return empty collections, not an error — so an empty result
is ambiguous unless you checked this field first.

Coverage is also per-POS and per-data-type: several integrations (Biotrack Wholesale, Brytemap,
Growflow) expose sales only, and no products, inventory or customers. Check
<https://developer.greencheckverified.com/tools/integration-status> before promising a customer a
data point.

## Step 3 — Sales (`get-crb-sales`)

`GET /service-providers/{sp_id}/crbs/{crb_id}/sales?start_date=YYYY-MM-DD&end_date=YYYY-MM-DD&limit=1000&offset=0`

Response:

```json
{ "data": [ { "id": "...", "crb_id": "...", "date": "...", "local_date": "...",
              "subtotal": 4500, "total_discounts": 500, "tax_paid": 315, "total_paid": 4315,
              "transaction_type": "sale", "pos_name": "Dutchie",
              "line_items": [ { "product_name": "...", "num_units": 1, "price_per_unit": 4500,
                                "grams": 1, "product_type": "flower", "cannabis_product": true } ] } ],
  "metadata": { "total": 1250, "limit": 100, "offset": 0 } }
```

- **Amounts are integers in minor units.** `4500` is $45.00. Do not render them as dollars.
- `date` is UTC; `local_date` is the CRB's local time. Use `local_date` for anything a dispensary
  operator will read, and `date` for reconciliation.
- `cannabis_product` on each line item is the flag that separates plant-touching revenue from
  accessories — the distinction the whole compliance product rests on.

## Step 4 — Paginate

`limit` defaults to 1000 and maxes at 1000. Loop: while `offset + limit < metadata.total`, increment
`offset` by `limit` and repeat. Every collection endpoint uses this same shape.

## Step 5 — The rest

| Data | Operation | Path |
|---|---|---|
| Products | `get-crb-products` | `/service-providers/{sp_id}/crbs/{crb_id}/products` |
| Inventory (snapshot for a date) | `get-crb-inventory` | `.../inventory` |
| Locations | `get-crb-inventory-locations` | `.../inventory-locations` |
| Inventory at one location | `get-crb-inventory-by-location` | `.../inventory-locations/{location_id}/inventory` |
| Customers | `get-crb-customers` | `.../customers` |

Inventory is a **snapshot per date**, not a running balance — request the date you want.

Every record carries both Green Check's `id` and the upstream `pos_*` id (`pos_sale_id`,
`pos_product_id`, `pos_customer_id`, `pos_location_id`). Store both; the `pos_*` id is how you
reconcile against the CRB's own POS reports when a number is disputed.

## Handle with care

Customer records are described in the docs as hashed, but the schema carries `medical_id`,
`drivers_license_id` and `other_id`. Confirm what is actually returned before you persist any of it.
This is regulated consumer and patient identity data.

## Rate limits and errors

Watch `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset` and back off when remaining
hits zero. Green Check publishes no numeric limit and no status code for exhaustion, so size backfills
conservatively and be ready for a 429 you were not told about.

`422` on this surface is almost always a malformed `start_date` / `end_date` — the format is
`YYYY-MM-DD`. Read the `details` object; it names the field.
