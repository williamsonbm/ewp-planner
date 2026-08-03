# EWP Purchase Planner

Decides **which stock lengths to order** for a batch of EWP jobs, and **how many of each**.

```bash
npm install          # one dependency: express
npm run planner      # → http://127.0.0.1:5178
```

Drop in one or more MiTek "Material Summary" CSVs, set how many distinct lengths you're
willing to put on a PO, and it reports the lengths to buy, the quantities, and a per-board
cut list.

**No database, no `.env`, no network at runtime.** It binds `127.0.0.1` only. Once
`npm install` has run, it works fully disconnected.

---

## The question it answers

The cut optimizer in the web app asks *"given these allowed stock lengths, how do I cut
them?"*. This asks the inverse:

> *"I'll order at most N distinct stock lengths for these jobs — which N do I buy?"*

It also answers the question behind that one — **how many lengths are worth buying at all**
— by running N = 1…max and showing where the waste curve flattens:

| Lengths allowed | Best set | I-Joist waste |
|---|---|---|
| 1 | 36′ | 68 ft |
| **2** | **36′, 32′** | **0 ft** |
| 3–5 | 36′, 32′, 28′… | 0 ft |

For that job (`33844J`), lengths 3 through 5 buy you nothing. Two lengths is the answer.

## Waste vs drops — read this before judging a number

- **True waste** — material you lose.
- **Recoverable drops** — LVL offcuts at or above the drop threshold (default 8′) that go
  back on the rack. The engine has always costed these at **zero**.

This distinction is not academic. Job `33844J` shows **144 ft of remainder** — every foot of
it LVL drops, with the I-Joist plan an exact fit at **0 ft of waste**. Reporting raw
remainder made a perfect plan look broken. If your shop does *not* reuse drops, raise the
threshold in the UI and they reclassify as true waste.

## Pools are per PRODUCT, not per depth

`11 7/8" PJI-40` and `11 7/8" TJI 560` share a depth but are different SKUs that can be
stocked in different lengths. So purchasable lengths are set **per product**, each product
gets its own length budget, and each gets its own recommendation. Drop your CSVs in and the
pool editor fills itself with exactly the products in the batch.

This works because the problem **separates**: the engine builds one cut group per
product and never shares a board between groups (a 16″ board cannot hold an 11-7/8 cut).
Measured — two depths optimized together produce 26 + 8 = **34 ft**, exactly their separate
totals, with byte-identical boards.

## Speed

Runtime scales with jobs × products × pool size × length cap. Four jobs on the supplier set
(8 lengths) at a cap of 4 is about **40 seconds**.

Ticking **all** takes the pool from 8 lengths to 19, which is 16,663 combinations instead of
218. Above 400 combinations the search switches from exhaustive to **greedy
forward-selection** — grow the set one length at a time — which is ~95 evaluations instead
of 16,663. Greedy is a heuristic and can give up a little waste, so the UI says when it was
used and how to get an exact search.

Use the supplier set for real ordering. Use "all" as a diagnostic: *"would a 38-footer save
enough to be worth asking the supplier for?"*

## Layout

```
src/planner/     server.js, planner.html   — the tool
src/ewp/         optimizeCuts.js           — the packing engine
                 selectStockLengths.js     — the search over length sets
                 cutListModel.js           — board grouping (shared with the browser)
                 inventoryImpact.js        — stock depletion report (not wired up yet)
                 parseCsv.js, extractDepth.js, normalizeSize.js, dbAdapters.js
public/          cutList.js, cutList.css   — per-board cut diagrams
test/            three suites; npm test
docs/            inventory-feature-brief.md
```

`optimizeCuts.js` and its helpers are ported from the hanger-web-app EWP engine. Keep them
in step deliberately — the packing behaviour is regression-locked over there.

Planning the on-hand-stock feature? Start with
[`docs/inventory-feature-brief.md`](docs/inventory-feature-brief.md) — the engine is already
inventory-aware and deliberately switched off, and the brief says where the seam is.

## Test fixtures are scrubbed

`test/ewp-fixtures/*.csv` are real job structures with **all identifying information
removed** — company name, staff names, customer businesses, street addresses, phone numbers,
and job names are replaced or blanked. Only the material rows (product, quantity, length,
label), job numbers and dates are real, because those are what the tests assert on.

**Never commit a raw MiTek export.** `.gitignore` guards `*-raw.csv` and `*-unscrubbed.csv`
and ignores `samples/` wholesale, but that is a safety net, not a substitute for checking.

**Naming, for anything added from here on.** New job material CSVs take the
**`-scrubbed.csv`** suffix (`34500J-materials-scrubbed.csv`) to mark them as reviewed and
safe to commit. Wholly fabricated files take **`-synthetic.csv`** instead — real-but-cleaned
and invented are different things and the filename should say which. The four fixtures above
predate the convention and keep their names.
