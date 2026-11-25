# E-Commerce Integration — Build This Last

The existing e-commerce app (Go + Next.js + Postgres, already live) is a **separate
database**, kept intentionally decoupled. Do not start this until Steps 1–7 in
`02-build-order.md` are stable — retrofitting serial-correct behavior into e-commerce
orders after the fact is much harder than building it in from the start.

## Ownership

- **ERP is the source of truth** for stock quantity, serials, and price tiers.
- **E-commerce app is the source of truth** for orders, customers, content, and
  cart/checkout state. It holds only a cached copy of stock/price for display.

## Sync direction 1 — stock & price (ERP → e-commerce)

Polling pull, not webhooks, as the default (simpler, self-healing, acceptable latency
for this business):

- `GET /sync/stock-prices?since=<timestamp>` — ERP returns changed SKUs/prices since
  last sync, backed by a changelog/`updated_at` table in the ERP.

## Sync direction 2 — orders (e-commerce → ERP)

This is the higher-risk direction — handle carefully:

- `POST /orders/reserve` — must be **idempotent** (accepts an external order ID; a retry
  of the same ID returns the same result rather than double-deducting stock). For
  serialized items, reserve a specific serial; for bulk items, use an atomic
  counter/row-lock decrement to prevent overselling on simultaneous orders.
- `POST /orders/confirm`
- `POST /orders/cancel-reservation`
- Order confirmation to the customer must wait for the ERP's reservation to succeed —
  never confirm an order the ERP hasn't actually reserved stock for.
- Decide branch assignment for web orders: a dedicated "online" virtual branch, or a
  manual allocation queue for a person to assign. Manual allocation is the simpler
  starting point; automatic nearest-branch routing can come later.

## Sync direction 3 — returns (e-commerce → ERP)

- `POST /rma` — web-originated RMA intake, writes into the **same** `rma_tickets` table
  as counter RMAs. Do not build a separate returns path for web orders — warranty
  lookup must work identically regardless of sales channel.

## Non-negotiable

Web orders must go through the same serial-deduction and invoice-generation path as
POS sales. If a web order bypasses this "for simplicity," warranty lookup and inventory
accuracy break for every item sold online.
