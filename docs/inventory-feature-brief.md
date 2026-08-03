# Handoff brief — stock inventory vs. job demand

Written 2026-08-03, from the `ewp-optimizer` session that built this tool. Read this before
starting the inventory feature; it will save you an afternoon.

**The feature:** the user uploads a CSV of on-hand stock alongside the job material
summaries, and the planner reports what the jobs need *against what is already on the yard*
— so the purchase list covers only the shortfall.

**The headline:** you do not need to write a solver. The engine in this repo is already
fully inventory-aware. It is deliberately switched off. Your job is to feed it.

---

## 1. What this tool is

A standalone, database-free purchase planner. `npm install && npm run planner`, then
`http://127.0.0.1:5178`. One runtime dependency (`express`), binds localhost only, runs
disconnected on any machine with Node 18+.

The hard constraint in [`src/planner/server.js`](../src/planner/server.js) is worth keeping:
no `pg`, no `dotenv`, no reaching into the parent web app. It shares the **engine** with the
hanger web app (one copy of `optimizeCuts.js`, no drift) and nothing else.

Uploads are read in the browser with `FileReader` and POSTed as JSON text, which avoids a
multipart parser entirely. Your inventory CSV should arrive the same way.

## 2. The engine already understands on-hand stock

[`src/ewp/optimizeCuts.js`](../src/ewp/optimizeCuts.js) is byte-identical to the web app's
copy, and that copy runs against a real inventory database. It already:

- accepts on-hand items shaped `{ source: "inventory", depth, item, span, qty, threshold? }`
- matches stock to demand on `normalizeSize(item) + span`
- prefers on-hand over purchase, shortest-sufficient board first
- **depletes across jobs** — job 2 sees what job 1 consumed (`inventoryBySize` is mutated
  per cut group as the batch runs)
- tags every board `cutFrom: "on-hand" | "purchase" | "unfulfillable"`
- computes threshold breaches after the batch, for reorder signals
- supports remnants for free — a remnant is just a board at an odd span, already keyed
  `(item, span)`, and the engine prefers it because it is on-hand and shortest-sufficient

`depth` is informational only. The solver matches on `normalizeSize(item)` plus `span`,
never on depth. Getting `depth` wrong is cosmetic; getting `item` wrong silently means
"no stock".

## 3. The seam — where real stock goes in

[`src/ewp/selectStockLengths.js:239`](../src/ewp/selectStockLengths.js#L239):

```js
function greenfieldStubs(cutItems) { ... }   // zero-qty stub per distinct size
```

Called in exactly two places:

| Line | Call site | Covers |
|---|---|---|
| 266 | `packConstant()` | LVL + RimBoard |
| 380 | `selectStockLengths()` | I-Joist |

Every stub is `{ source: "inventory", item, span: 0, qty: 0 }`. Span 0 is never drawable
(`fillPack` requires `qty > 0` **and** `span >= length`), so every piece routes to
`purchase`. That is the "greenfield" assumption the whole tool currently runs on.

Those two call sites are the entire integration surface.

## 4. ⚠️ The landmine — stubs are load-bearing, do not just replace them

The stubs are not only an off-switch. They also suppress a pre-flight check.

If a size in the job has **no inventory row at all**, `optimizeCuts` raises
`no_inventory_match`, and `detectWarnings` escalates that to **blocking the entire batch**.
The stubs exist partly so a stock-free run doesn't trip it for every size at once.

> **Merge real stock *with* stubs for sizes the CSV doesn't mention. Never substitute.**

Get this wrong and a batch containing one product you don't stock fails completely, with an
error that points at inventory rather than at your merge. The natural shape is: build the
stub list as today, then overlay real rows on top, so unmatched sizes keep their zero-qty
key.

The same trick is documented in [`src/ewp/dbAdapters.js`](../src/ewp/dbAdapters.js) under
`specialOrderInventoryStubs()`, which solves the identical problem for special-order series
in the web app. Read that comment — it explains the reasoning better than this paragraph.

## 5. Recommended design — greenfield search, then net against stock

**Do not feed stock into the length-set search.** Apply it after.

The search ranks candidate length sets by I-Joist true waste, and
[`selectStockLengths.js:51-53`](../src/ewp/selectStockLengths.js#L51-L53) records that waste
and feet-purchased are equivalent orderings *only under greenfield*. Feed stock into the
search and:

- that equivalence breaks
- reported waste becomes stock-dependent, so two runs a week apart aren't comparable
- every existing ranking test is now asserting against a moving target

Netting afterwards instead — "here is the best set of lengths to buy, and here is what you
already have that covers part of it" — preserves all 73 tests, matches how a buyer actually
reasons, and is mostly just `inventoryImpact.js` (below).

**The road not taken:** a genuinely stock-aware search would pick lengths knowing what's on
the yard, which is the more *correct* answer and could beat this on real yields. It is also
a different product and a much bigger change. Worth doing later, deliberately, not as a
side effect of adding a CSV upload.

## 6. What was copied in, and what wasn't

### Copied — [`src/ewp/inventoryImpact.js`](../src/ewp/inventoryImpact.js)

This is precisely "need vs. stock". Given inventory items and committed rows it returns:

- `depletion[]` — `{ depth, item, stockLength, startQty, used, remaining, threshold,
  belowThreshold }`, one row per stock line actually consumed
- `purchases[]` — `{ category, size, stockLength, qty }`, the material not drawn from stock

Its only dependency is `normalizeSize.js`, already present here and byte-identical, so it
loads clean. Nothing imports it yet.

Two things to know before you trust it:

1. **It has no tests in this repo.** It is regression-locked in `ewp-optimizer` via
   `test/ewp-golden.test.js`, but that test drives the database pipeline and does not
   travel. **Write tests for it here.**
2. **It de-dupes to distinct boards before tallying** (lines 46–58). One board can yield
   several cuts, so several committed rows share a `stockPieceNumber` while removing
   exactly *one* piece from stock. Counting rows overstates depletion. This is the subtlety
   most likely to be reintroduced if anyone rewrites it.

### Not copied — `readInventory.js`

The web app's stock reader lives at `ewp-optimizer/src/ewp/readInventory.js`. It is **not**
here on purpose: it reads XLSX via SheetJS, and adding `xlsx` would double this tool's
dependency count for a feature specified as CSV.

Use it as a **shape reference**. Its useful ideas:

- a sheet counts as stock only if its header row carries `Span`, `Item` **and** `Qty`
  (grab `Threshold` when present) — everything else is skipped rather than erroring
- **do not de-duplicate `(item, span)` pairs.** Emit every row and let the consumer sum
  them with `+=`. De-duplicating was a real undercount bug.
- skip rows with a blank item or a non-numeric span/qty — those are subtotal and spacer
  rows, not data

Also copy the clamp from `dbAdapters.js` `inventoryItemsFromRows()`: **negative quantities
clamp to 0**. A negative would corrupt the `startQty` baseline in the impact report even
though `fillPack` would never draw it.

## 7. The inventory CSV

Sample: [`test/ewp-fixtures/stock-inventory-sample-synthetic.csv`](../test/ewp-fixtures/stock-inventory-sample-synthetic.csv)

```
item,span,qty,threshold
"11 7/8"" PJI-40",48,12,4
"2.1 RigidLam DF LVL 1-3/4 x 11-7/8",22,3,
```

`item`, `span`, `qty` are required; `threshold` is optional and may be blank.

**It is not naive-splittable.** Real EWP item strings contain double quotes (`11 7/8" PJI-40`),
so fields are RFC 4180 quoted with doubled inner quotes. A `line.split(',')` parser will
corrupt every row. Handle quoting, or reuse the quote-aware splitting already in
[`src/ewp/parseCsv.js`](../src/ewp/parseCsv.js).

The sample is deliberately built to exercise the edges, and its keys all match the existing
job fixtures — you can run it end-to-end on day one:

- a zero-qty row (`TJI® 210`) — a known item with none on hand
- an odd-span row (LVL at 22 ft) — a remnant
- a blank threshold
- all 7 product keys the four job fixtures actually demand

## 8. Data handling rules — this repo is PUBLIC

`github.com/williamsonbm/ewp-planner` is public. Everything below is a hard rule.

- **No identifiable company information in anything committed.** No company names, staff
  names, customer businesses, street addresses or phone numbers. The existing fixtures
  already read `Example Truss Co`; keep it that way.
- **New job material CSVs take the `-scrubbed.csv` suffix** — e.g.
  `34500J-materials-scrubbed.csv`. It marks a file as reviewed and safe to commit.
- **Fabricated files take `-synthetic.csv`**, as the sample above does. Real-but-cleaned and
  wholly-invented are different things and the names should say which.
- **Never commit a raw MiTek export.** `.gitignore` guards `*-raw.csv` and `*-unscrubbed.csv`
  and ignores `samples/` wholesale — that is a safety net, not a substitute for looking.
- Put fixtures in `test/ewp-fixtures/`, not `samples/` (which is gitignored).
  `.gitattributes` pins `test/ewp-fixtures/*.csv` to LF so the suite behaves the same on
  Windows.

**Known and accepted:** the four existing job fixtures carry real job numbers
(`33591J`, `33844J`, `34120J`, `34182J`) and real delivery dates, because the tests assert
on them. These are readable by anyone. That is a deliberate, recorded decision — not an
oversight — and it is why the rules above matter for everything added from here.

## 9. Open items carried over

**Speed on wide pools.** Four jobs against the supplier set runs ~40s. Ticking "all" runs to
minutes and crosses `GREEDY_ABOVE_SETS` (400), switching from the exhaustive sweep to greedy
forward-selection — which can give up a few feet. Parallelising across length counts was
estimated at 3–4× and is the largest remaining win.

**Fixture coverage is thin on mixed products.** Every fixture is single-product 11-7/8
except `33591J`, which carries two products at one depth. The per-product pool feature —
the reason pools are keyed per product rather than per depth — is barely exercised. A real
mixed-product job would test it better than anything currently in the suite.
