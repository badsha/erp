# Hardware Trading ERP — Overview

A standalone ERP for a computer hardware trading business: POS/billing, inventory
(serialized + bulk), warranty/RMA, procurement, accounting, and RBAC. An existing
e-commerce app (Go + Next.js + Postgres, already live) is integrated **last**, via the
API contract in `03-ecommerce-integration-deferred.md` — do not build it in this pass.

## Stack

- Backend: Go, REST API, Postgres
- Frontend: Next.js (backoffice screens + POS screen)
- Auth: JWT with role claims (see RBAC in `02-build-order.md` Step 7)

## Resolved architecture decisions

- **POS is online-only.** Direct calls to the cloud Postgres-backed API. No local-first
  database, no offline sync engine, no conflict resolution. A short client-side retry on
  a failed request is sufficient resilience — see `.cursor/rules/20-frontend-web.mdc`.
- **Branches are optional.** `branch_id` is nullable on every table that references it.
  A single-location business can run the whole system with zero branch rows. Multi-branch
  features (transfers, per-branch stock) activate naturally once branches exist, but
  nothing requires them.
- **Two separate databases for ERP and e-commerce.** They sync via API, not a shared
  schema. See `03-ecommerce-integration-deferred.md`.

## Repo structure

```
/erp
  /server
    /internal
      /db
      /products
      /inventory
      /pos
      /rma
      /procurement
      /accounting
      /auth
    /cmd/api/main.go
  /web
  /migrations
```

## Build order

Build in the order given in `02-build-order.md` — each step depends on the schema and,
in most cases, the step before it. Don't start POS before inventory, don't start RMA
before POS, etc.

## Cross-cutting rules

Every module must follow `.cursor/rules/00-global.mdc` (production-grade, no
hardcoding, mandatory test coverage) regardless of which step it belongs to.
