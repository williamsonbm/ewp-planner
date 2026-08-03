# EWP Purchase Planner

Standalone offline tool. Drop in one or more MiTek "Material Summary" CSVs, say how many
distinct stock lengths you're willing to order, and get back the length sets that minimise
what you have to buy.

```
npm install && npm run planner    # → http://127.0.0.1:5178
npm test                          # → 73 tests, node:test, no framework
```

## Hard constraints — do not break these

- **No database.** No `pg`, no `dotenv`, no reaching into the parent web app.
  `src/planner/server.js` stays DB-free: this is a laptop tool, not a service.
- **One runtime dependency (`express`).** Do not add packages. If something seems to need a
  parser or a spreadsheet reader, write it or reuse `src/ewp/parseCsv.js`.
- **Localhost only.** The server binds `127.0.0.1` deliberately — there is no auth layer.

## Shared engine — edit with care

`src/ewp/optimizeCuts.js` and its helpers are ported from the hanger web app and kept
byte-identical on purpose. The packing behaviour is regression-locked over there. Change
them only deliberately; silent drift between the two copies causes real bugs.

## This repo is PUBLIC

- **No identifiable company information in anything committed** — company names, staff
  names, customer businesses, street addresses, phone numbers.
- New job material CSVs take the **`-scrubbed.csv`** suffix; wholly fabricated files take
  **`-synthetic.csv`**. Real-but-cleaned and invented are different things.
- **Never commit a raw MiTek export.** `.gitignore` guards `*-raw.csv`, `*-unscrubbed.csv`
  and `samples/` — a safety net, not a substitute for checking.
- Fixtures belong in `test/ewp-fixtures/` (pinned to LF), never `samples/` (gitignored).

## Working on on-hand stock / inventory?

Read [`docs/inventory-feature-brief.md`](docs/inventory-feature-brief.md) first.

Short version: the engine **already** accepts on-hand inventory items and is switched off by
`greenfieldStubs()` in `src/ewp/selectStockLengths.js`. Don't write a solver — feed the one
that's there, and read the brief's landmine section before touching those stubs.
