# Core Schema

Build this before any feature module — everything depends on it. Give this directly to
the agent as a migration-writing task.

> `branch_id` is nullable everywhere it appears below. Multi-branch is optional — a
> single-location shop should work with zero branch records, `branch_id` left null
> throughout. See `.cursor/rules/30-database.mdc`.

- `branches (id, name, address, is_active)` — optional table; may remain empty
- `categories (id, parent_id nullable, name)` — self-referencing for hierarchy
- `brands (id, name)`
- `products (id, category_id, brand_id, name, mpn, is_serialized boolean, base_unit)`
- `product_attributes (id, product_id, key, value)` — flexible spec fields (socket type,
  wattage, RAM speed, etc.)
- `price_tiers (id, product_id, tier_type enum['retail','cash_discount','corporate','dealer'], price)`
- `serials (id, product_id, serial_number unique, branch_id nullable, status enum['in_stock','sold','in_transit','rma','loaner'], purchase_batch_id, warranty_months, warranty_start_date)`
- `stock_bulk (id, product_id, branch_id nullable, quantity)` — non-serialized items
- `suppliers (id, name, contact_info)`
- `purchase_orders (id, supplier_id, branch_id nullable, status, created_by)`
- `purchase_order_items (id, po_id, product_id, quantity, unit_cost)`
- `goods_receipts (id, po_id, received_by, received_at)`
- `goods_receipt_serials (id, grn_id, serial_id)` — links received serials to a GRN
- `customers (id, name, phone, type enum['walk_in','corporate'], bin_tin nullable, credit_limit nullable)`
- `invoices (id, branch_id nullable, customer_id nullable, invoice_number, status enum['quotation','invoice','voided'], subtotal, vat_amount, total, created_by)`
- `invoice_items (id, invoice_id, product_id, serial_id nullable, quantity, unit_price)`
- `payments (id, invoice_id, method enum['cash','bkash','nagad','card','bank','emi'], amount, transaction_ref)`
- `stock_transfers (id, from_branch_id, to_branch_id, status enum['requested','dispatched','received'], requested_by)`
- `stock_transfer_items (id, transfer_id, serial_id nullable, product_id, quantity)`
- `rma_tickets (id, invoice_item_id, status enum['received','sent_to_distributor','under_service','replacement_approved','ready','handed_over'], condition_notes, original_serial_id, replacement_serial_id)`
- `audit_logs (id, user_id, action, entity_type, entity_id, old_value, new_value, created_at)`
- `users (id, name, phone/email, password_hash, role)`
- `roles (id, name, permissions jsonb)`

## Conventions

- UUID primary keys, `created_at`/`updated_at` on every table.
- Status fields as Postgres `ENUM` types (or `CHECK` constraints), mirrored as typed
  constants in application code — never retype the value list by hand in two places.
- Index every foreign key and every hot lookup column (serial number, invoice number,
  barcode).
- Seed script: one admin user. No branch rows required — the system must work with none.
