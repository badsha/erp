# Build Order & Module Specs

Build sequentially — each step depends on the ones before it. Test requirements listed
per step are the minimum bar, not a ceiling; see `.cursor/rules/40-testing.mdc`.

## Step 1 — Products & Inventory

- CRUD APIs for products, categories, brands, product_attributes.
- Serial intake: register a new serial against a product (+ branch, if set), status
  `in_stock`.
- Bulk stock endpoints: add/deduct quantity for non-serialized items.
- Serial lookup endpoint: given a serial number, return product, branch (if any),
  status, purchase batch, and computed warranty remaining
  (`warranty_start_date + warranty_months - now()`).

**Tests:** duplicate serial registration rejected; lookup of a nonexistent serial
returns a clear 404, not a crash; warranty calculation edge cases (already expired,
expires today).

## Step 2 — POS / Billing

POS is a standard online web app — direct calls to the backend API, no offline mode, no
local database, no sync queue. See `.cursor/rules/20-frontend-web.mdc`.

1. Create invoice (`quotation` or `invoice` directly for walk-ins).
2. Add invoice_items by scanning product barcode, then serial barcode for serialized
   items.
3. On adding a serialized item: verify serial status is `in_stock` (and, if `branch_id`
   is set on both the serial and the invoice, that they match); reject if already
   sold/in_transit/rma; on success mark serial `sold` and link to this invoice line —
   only commit this on final invoice confirmation, not on quotation.
4. On adding bulk items: deduct from `stock_bulk` atomically (DB transaction + row lock
   to prevent overselling at the same till in a race condition).
5. Support quotation → invoice conversion (same invoice row, status change, re-validate
   serial availability at conversion time since time may have passed).
6. Support split payments: multiple payment rows per invoice summing to total.
7. Support "hold cart" — persist an in-progress invoice as `quotation` so it can be
   resumed.
8. Calculate VAT per category-level VAT rule, stored on invoice (rate itself lives in
   config/DB, never hardcoded — see `.cursor/rules/00-global.mdc`).

**Tests:** selling a serial twice in a race — exactly one attempt succeeds; partial
payment totals not matching invoice total rejected or flagged; holding and resuming a
cart round-trips correctly.

## Step 3 — Stock Transfers

- Requisition → dispatch → receive workflow.
- On dispatch: mark serials `in_transit`, remove from origin branch's sellable stock.
- On receive: verify each serial by scanning before marking `in_stock` at destination.
- Flag any discrepancy (dispatched-but-not-received, or received serial not on the
  dispatch list) rather than silently accepting it.

**Tests:** a dispatched-not-yet-received serial is excluded from both branches'
sellable stock; a scan mismatch on receive is flagged, not silently accepted.

## Step 4 — RMA

- Create RMA ticket against an invoice_item (requires the sold serial).
- Status transitions, in order only: `received → sent_to_distributor → under_service →
  replacement_approved → ready → handed_over`. Reject out-of-order transitions.
- Replacement serial swap: on `replacement_approved`, link a new serial as
  `replacement_serial_id`, set its status to `sold`, and **copy the original
  `warranty_start_date` and `warranty_months` onto the replacement** — warranty expiry
  must not reset.
- Loaner tracking: optional serial marked `loaner`, configurable reminder window if not
  returned (window itself in config, not hardcoded).

**Tests:** out-of-order status transition rejected; replacement swap preserves the
*original* warranty window, not a fresh one.

## Step 5 — Procurement

- Purchase order creation and goods receipt (GRN).
- On GRN: accept serial numbers (manual entry or CSV import) for serialized products,
  create serial records `in_stock`, link to GRN and purchase batch.
- Landed cost allocation: distribute shipping/customs costs proportionally across
  received items to compute true unit cost. Decide and document the costing method for
  bulk items (FIFO vs. weighted average) before building this — it changes the schema.

**Tests:** landed cost allocation sums back to the total within a defined rounding
tolerance; CSV import of serials rejects duplicates cleanly.

## Step 6 — Accounting

- Double-entry ledger: every invoice confirmation, payment, purchase receipt, and stock
  adjustment writes matching debit/credit entries against a chart of accounts.
- Shift-based cash drawer: open (starting cash), close (reconcile counted cash/MFS/card
  totals against expected from the day's invoices + payments), flag variances.
- Mushak 6.3 / NBR VAT formatting: flag as needing verification against current NBR
  requirements before go-live — VAT compliance rules change and should not be assumed
  from memory.

**Tests:** every ledger-writing operation produces balanced debits/credits; shift close
correctly flags a variance when counted cash doesn't match expected.

## Step 7 — RBAC & Audit

- Role-based permissions: `roles.permissions` as a capability list (e.g.,
  `void_invoice`, `edit_purchase_price`, `approve_discount_above_X`).
- Enforce permission checks at the service/middleware layer, not just by hiding UI
  buttons — a user without a permission must be rejected even if they reach the
  endpoint directly.
- Audit logging as write-through: price override, void, stock adjustment, or RMA status
  change automatically writes an `audit_logs` row (before/after values, user,
  timestamp) — implement this at a shared service-function or DB-trigger level so a
  future new endpoint can't accidentally bypass it.

**Tests:** a user without a given permission is rejected server-side even if the UI
path is bypassed; every flagged action type produces exactly one audit log row.
