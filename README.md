# Hardware Trading ERP — Cursor Rules & Spec Kit

Drop the contents of this zip into the root of your (new or existing) repo. It contains:

```
.cursor/rules/           Cursor rule files (always-on + path-scoped)
  00-global.mdc           Production-grade code, no hardcoding, mandatory tests — always applies
  10-backend-go.mdc       Go backend conventions (applies to server/**/*.go)
  20-frontend-web.mdc     Next.js conventions (applies to web/**/*.{ts,tsx})
  30-database.mdc         Migration/schema conventions (applies to migrations/**/*.sql)
  40-testing.mdc          Coverage requirements per module (applies to test files)
.cursorrules              Legacy fallback pointing to the files above
spec/
  00-overview.md           Stack, resolved architecture decisions, repo layout
  01-schema.md             Core database schema
  02-build-order.md        Step-by-step module specs with workflows and edge cases
  03-ecommerce-integration-deferred.md   Integration contract — build last
```

## How to use this with Cursor

1. Unzip into your repo root so `.cursor/`, `.cursorrules`, and `spec/` sit alongside
   your code.
2. Cursor will automatically load `.cursor/rules/00-global.mdc` on every request (it's
   marked `alwaysApply: true`), and the others when you're editing matching files.
3. Work through `spec/02-build-order.md` one step at a time. For each step, point Cursor
   at the relevant section and let it read `spec/00-overview.md` + `spec/01-schema.md`
   first if it's a fresh session.
4. Don't start `03-ecommerce-integration-deferred.md` until Steps 1–7 are stable.

## What's already decided (don't re-litigate these with the agent)

- POS is a plain online web app — no offline/local-first sync.
- Branches are optional — `branch_id` is nullable everywhere; single-location use needs
  zero branch rows.
- ERP and the existing e-commerce app use two separate databases, synced via API.

## Open item flagged in the spec

`spec/02-build-order.md` Step 5 calls out that the bulk-item costing method (FIFO vs.
weighted average) needs to be picked before that step is built — it changes the schema.
