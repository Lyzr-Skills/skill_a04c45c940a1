---
name: supplier-performance-review
description: >
  Analyse supplier performance against contracted SLAs in the Anaplan PRM model —
  On-Time Delivery %, Delivery Fill Rate %, Lead Time, Supplier Tier, To Review flags,
  Total Spend and AOV — and produce a ranked underperformer action list for a
  Procurement or Sourcing Manager. Use this skill whenever the user mentions suppliers,
  vendors, SLAs, OTD, DFR, fill rate, lead time, supplier tiers, supplier scorecards,
  spend at risk, late deliveries, short shipments, "who should I chase", "which
  suppliers are underperforming", supplier reviews, QBRs, dual-sourcing decisions, or
  asks about performance at a named DC, plant, location or region — even if they never
  say the words "supplier performance" or name a module. Also use it when the user asks
  a narrower follow-up about one supplier, one product or one site, because the same
  cached coordinates and the same data traps apply.
compatibility: >
  Requires the Anaplan MCP tool set (set_model_context, catalog_modules,
  catalog_line_items, read_cells, sql_query, catalog_lists) and a model bound to the
  PRM supplier performance model.
---

# Supplier Performance Review

## Why this skill exists

Answering "which suppliers are underperforming?" is not hard analysis — it is one read
of one module. What makes it slow is **rediscovery**: hunting for the right model, the
right module among dozens, the exact line item names, the dimension names, and then
relearning the half-dozen quirks of this particular module by hitting them one at a
time.

That rediscovery is identical every single run. This skill caches it, so the work
collapses to *read once, analyse, write*. The target is **2–3 tool calls and under a
minute**, against roughly 15 calls unassisted.

The second reason this skill exists: the obvious answer to this question is wrong in
three specific, repeatable ways (documented in *Known traps* below). A fast answer that
is also wrong is worse than a slow one. The traps are encoded here so you avoid them by
default rather than by luck.

---

## The fast path

### Step 0 — Bind the model (skip if already bound)

Call `get_model_context` first. If a model is already bound, do not rebind — rebinding
costs a model-selection slot and clears any time anchor.

If nothing is bound:

```
set_model_context(workspace_id=<ws>, model_id=<model>)
```

The working model is **`PRM - PR Management 1.0`**. If `catalog_models` returns several
PRM variants, show the user the list and let them pick — do not guess, because tier
thresholds and SLA baselines differ between model copies.

### Step 1 — One read, at leaf grain

Everything needed for a full review lives in a single module:

**`KPI220 - Supplier Performance Reporting`**

```
read_cells(
  module = "KPI220 - Supplier Performance Reporting",
  line_items = [
    "Final OTD %", "Final DFR %", "Final Lead Time",
    "OTD Variance", "DFR Variance", "Lead Time Variance",
    "Contract SLA OTD %", "Contract SLA DFR %", "Contract Lead Time",
    "Final Supplier Tier", "To Review?",
    "Total Spend", "AOV", "Priority (Compound Alerts)"
  ],
  slice = { "<location dimension>": "<the location the user named>" },
  leaf_dimensions = ["<supplier dimension>", "<product dimension>"],
  limit = 200
)
```

Two leaf dimensions is the practical ceiling — a third times out and tells you nothing
a follow-up query wouldn't.

**Verify names once per session, then stop.** Line item and dimension spellings drift
between model versions. Run `catalog_line_items` on the module **once** at the start of
a session, cache the exact strings, and reuse them for every follow-up question in that
conversation. Do not re-catalog between questions — that is the single biggest avoidable
cost in a multi-question session.

If `read_cells` rejects a dimension as unaccounted for, it names the dimension in the
error. Add it to `slice` (pinned to the roll-up member) and re-run. Do not fall back to
exploratory `sql_schema` calls unless two `read_cells` attempts have failed.

### Step 2 — Scope confirmation (only when genuinely ambiguous)

If the user named a location ("Florida DC", "our Dallas site"), use it. If they named
nothing, do **not** stop and ask — run at the top-level roll-up, state the scope you
used in the first line of the answer, and offer to narrow. A clarifying question costs
the user a full round trip; a stated assumption costs them one sentence.

Only interrupt when the request is genuinely ambiguous between two real options — for
example, a name that matches both a region and a DC inside it.

---

## Known traps

These are properties of this module. Each one has produced a wrong answer before.

**1. Ratio measures do not roll up.**
`Final OTD %`, `Final DFR %` and the variance line items return **blank** at the
`All Products` / `All Suppliers` roll-up. They are only populated at leaf grain. If you
read at roll-up and see blanks, the suppliers are not failing — the measure simply
doesn't aggregate. `Total Spend` *does* roll up correctly and can be used to validate
your leaf sums. **All performance analysis must happen at Supplier × Product leaf.**

**2. `0% / 0%` with zero variance means missing data, not zero performance.**
Lines reporting 0% OTD and 0% DFR with variance of exactly 0 have no delivery records
flowing — they are measurement gaps, not catastrophic suppliers. They are also **not
flagged To Review**, so they are invisible in the exception queue. Report them in a
separate "blind spots" section with their spend attached, and never rank them as
underperformers. A line with spend recorded and no delivery data is a reporting defect
worth raising on its own.

**3. The `To Review?` flag produces false positives.**
The flag fires on *category change* (including improvement) as well as on breach. Expect
a handful of flagged lines that are beating SLA on all three metrics. Call these out
explicitly and recommend `Clear Review`, because an exception queue the manager stops
trusting is worse than no queue.

Also check `Priority (Compound Alerts)` — in several model instances it is unpopulated
across every row, which means the queue is firing on category-change logic alone with no
business-impact weighting. Worth flagging as a configuration gap.

**4. Negative Lead Time variance is good.**
LT variance is actual minus contract, so negative means delivering *faster* than
contracted. Never list it as a breach. In practice LT breaches are rare in this model —
if you find zero, say so prominently, because "every failure is OTD/DFR, not lead time"
changes the supplier conversation from planning horizons to reliability and capacity.

**5. No Time, no Versions dimension.**
`KPI220` carries neither. Values reflect whatever window is configured in
**`KPI100 - Supplier KPI Time Settings`**. Do not try to filter by period and do not ask
the user which version they want. If they ask about a different horizon, point them at
`KPI100` rather than attempting a time filter here.

---

## The analysis recipe

Run all of these on the single result set. They are cheap, and together they are what
turns a data dump into a review.

1. **Spend at risk per supplier** — sum `Total Spend` across that supplier's lines with
   negative OTD variance. Express as both an absolute figure and a % of that supplier's
   total. Rank suppliers by absolute spend at risk, not by tier and not by worst
   percentage. A 50% OTD line worth $700 is not a priority; a 71% line worth $65,000 is.

2. **Breadth vs depth** — count breaching lines as a fraction of the supplier's active
   lines. A supplier failing 4 of 5 lines moderately has a systemic constraint; one
   failing 1 of 6 severely has an isolated SKU problem. These need completely different
   conversations, so say which one you're looking at.

3. **The internal-inconsistency lever** — for each underperformer, find lines where they
   run 100%. This is the strongest negotiating position available: it removes "we can't"
   and leaves only "we aren't prioritising you". Always surface it when present.

4. **DFR-tracks-OTD test** — if fill rate degrades in lockstep with on-time rate across a
   supplier's lines, the constraint is upstream supply, not logistics. If OTD falls but
   DFR holds, it is a scheduling or transport problem. This distinction tells the manager
   what to actually ask for.

5. **Cross-supplier product scan** — pivot the breaches by product. If three or more
   independent suppliers miss on the same SKU, that is almost certainly an internal root
   cause (spec change, demand volatility, late order placement, or volume spread too thin
   to command priority anywhere). **Surface this above the individual supplier chases** —
   it can resolve more exposure with one internal fix than four phone calls.

6. **Tier-masking check** — look for high-tier suppliers carrying material breaching
   spend. Tiers are weighted averages, so a Gold supplier with two large 100% lines can
   hide a line running at 50%. Flag these explicitly, because they will never appear on a
   tier-based exception report and the manager will otherwise never chase them.

7. **Reallocation arbitrage** — find the same SKU where one supplier runs 100% and
   another runs poorly on comparable or greater spend. This is the highest-value single
   recommendation you can make and it falls straight out of the data.

---

## Output structure

Use this template. It front-loads the decision and keeps the evidence underneath, which
is what a procurement manager with ten minutes actually needs.

```markdown
# [Location] — Supplier Performance Review

## The headline
[One number: spend at risk, as $ and % of total. Then the single most
important structural insight — usually what is NOT broken, because that
reframes the conversation.]

## Chase these first — $X at risk
### 1. [Supplier] — $spend, N% at risk
[Breaching lines with OTD/DFR/variance/spend. The leverage point.
One-line diagnosis.]
[Repeat for top 3, ranked by absolute spend at risk]

## Two things you'd miss on a standard report
[Tier-masked suppliers. Cross-supplier product patterns.]

## Data gaps worth closing
[Blind spot lines with spend. False-positive To Review flags.
Unpopulated alert weightings.]

## Your biggest single opportunity
[The reallocation arbitrage, with both sides' numbers.]
```

Then offer two concrete follow-ups — typically a drill-down to PO level on one supplier,
and the same analysis at other locations to build a cross-site negotiating position.

**Always save the full detail as an artifact** (complete scorecard, every flagged line
with every metric, per-supplier talk tracks, and an action plan split into this week /
this month / structural). Keep the conversational reply to the decisions; put the
evidence in the artifact. A second artifact — a spend-at-risk chart, compliant vs
breaching by supplier — is worth creating whenever there are four or more suppliers.

### Talk tracks

For each priority supplier, write a quotable line the manager can take into a call. Not
a summary of the data — the actual argument. The pattern that works: *state their own
best performance, state their worst, name the gap, name the consequence.*

> "You run 100% on two of our lines and 50% on the two largest. We need to understand
> why this DC's volume is being deprioritised, and we need a recovery plan on
> [SKU] — it's $171k of our spend running at 62.5% OTD."

---

## Method note to include

Close the artifact with a short caveats section so the numbers survive scrutiny:

- Source module, grain, and the location slice used
- Variances read **as stored in the model**, not recomputed
- Spend-at-risk sums are the only arithmetic performed outside the model
- The ratio-roll-up limitation, and that `Total Spend` was validated against the roll-up
- That the module has no Time/Versions dimension, and where the horizon is configured

This takes thirty seconds and is what makes the analysis defensible when a supplier
pushes back on the numbers in the room.

---

## Follow-up questions in the same session

Most reviews generate three or four follow-ups. Treat them as reads against the data you
already have:

- **"Tell me more about Henkel"** — answer from the result set already in context. No
  new tool call.
- **"What about our other DCs?"** — one more `read_cells` with a different slice. Reuse
  the cached line item names.
- **"Show me this by product instead"** — re-pivot in your head; the data is the same.
- **"Who changed this number?"** — `cell_change_history` with the slice copied verbatim
  from the read. Remember calculated cells have no history of their own; use
  `explain_cell` first to find the input that feeds it.

The cached coordinates are the whole point. A five-question session should cost three
tool calls, not fifteen.
