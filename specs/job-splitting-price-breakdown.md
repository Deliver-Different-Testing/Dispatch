# Job Splitting — Parent Price Breakdown UI

**Author:** George
**For:** Jacob
**Date:** 2026-09-16
**Related work:** Jacob's fix this week that locks the split amount so it no longer
changes after a job has been split on the parent (customer charging) job.

---

## 1. Summary

Three things to solve:

1. **The parent job should keep its original price breakdown.** Today, after a split,
   the parent's breakdown is replaced by every child line item (5 items becomes 10:
   `Base Part A`, `Base Part B`, …). The parent is the job that charges the customer —
   it should still read like the job we quoted.
2. **Editing the parent headline price breaks archiving.** The headline price and the
   child job prices are independently editable, so an edit to one puts it out of sync
   with the sum of the parts, and the job then won't archive.
3. **There is no usable way to retrospectively edit prices once a job is split.** The
   flat 10-row list doesn't say which leg a row belongs to or which driver is running
   it, and there's no safe place to adjust what a driver gets paid for their leg.

The proposed fix is one idea applied consistently: **the parent job is the single
editable source of truth, and the child jobs are fully derived from it.** The UI that
expresses this is a three-column parent breakdown — Revenue, Part A cost, Part B cost —
which makes the out-of-sync state structurally impossible rather than something we
validate for at archive time. The same modal serves at the moment of splitting, so the
operator can get it right up front rather than splitting and then fixing (§8).

Structurally that rests on one change: **splitting stops deleting the parent's price items.**
Today it destroys them and writes per-leg rows whose only link back is their name. Keeping the
parent's rows and giving each leg row a foreign key to its parent item is what makes the grid
groupable, and it removes the name-parsing the current design would otherwise need (§5).

---

## 2. Worked example

All figures below are the real staging job, so they can be used as a test fixture.
Pre-split breakdown and the split screen are from **KT4070V**; the post-split parent
and children are from **KT4071V** / **KT4071VA** / **KT4071VB**. Same rate, same numbers.

*(The negative fuel cost — Base Fuel costs more than it earns — is the known staging
rating quirk. It's carried through the tables below unchanged because it's a useful
edge case for the margin display, but it is not in scope to fix here.)*

### Stage 1 — Parent before the split (KT4070V)

Total Revenue **US$114.00** · Total Cost **US$69.00** · Gross Profit **US$45.00** · 39.5% margin

| Item | Revenue | Cost | Profit | Margin |
|---|---:|---:|---:|---:|
| Base | 64.00 | 32.00 | 32.00 | 50% |
| Base Fuel | 16.00 | 24.00 | −8.00 | −50% |
| Items | 20.00 | 4.00 | 16.00 | 80% |
| Items Fuel | 5.00 | 3.00 | 2.00 | 40% |
| Congestion | 9.00 | 6.00 | 3.00 | 33% |
| **Total** | **114.00** | **69.00** | **45.00** | **39.5%** |

### Stage 2 — Confirm Split Pricing

Split by road distance per leg: **Leg A 8 mi → 80%**, **Leg B 2 mi → 20%**.
The banner already states the important thing: *"The job total stays US$114.00 —
splitting does not change what the client is invoiced."*

Each line item gets its own **Share %** input, defaulted to the overall share, so a
single charge can be given its own share if only one leg incurred it.

| Item | Share % | Leg A | Leg B |
|---|---:|---:|---:|
| Base | 80 | 51.20 (cost 25.60) | 12.80 (cost 6.40) |
| Base Fuel | 80 | 12.80 (cost 19.20) | 3.20 (cost 4.80) |
| Items | 80 | 16.00 (cost 3.20) | 4.00 (cost 0.80) |
| Items Fuel | 80 | 4.00 (cost 2.40) | 1.00 (cost 0.60) |
| Congestion | 80 | 7.20 (cost 4.80) | 1.80 (cost 1.20) |
| **Leg total** | | **91.20 (cost 55.20)** | **22.80 (cost 13.80)** |

**Confirmed behaviour:** the share is applied to *both* revenue and cost at the same
rate. Every figure above is exactly `parent value × share`, and margin is therefore
identical on the parent and on both legs (39.5%). This is what makes everything in
§4 reconcilable from the parent alone.

### Stage 3 — Parent after the split (KT4071V) — the problem screen

Totals are still correct (114.00 / 69.00 / 45.00) but Price Items is now **10 items**:

| Item | Revenue | Cost |
|---|---:|---:|
| Base Part A | 51.20 | 25.60 |
| Base Part B | 12.80 | 6.40 |
| Base Fuel Part A | 12.80 | 19.20 |
| Base Fuel Part B | 3.20 | 4.80 |
| Items Part A | 16.00 | 3.20 |
| Items Part B | 4.00 | 0.80 |
| Items Fuel Part A | 4.00 | 2.40 |
| Items Fuel Part B | 1.00 | 0.60 |
| Congestion Part A | 7.20 | 4.80 |
| Congestion Part B | 1.80 | 1.20 |

What's wrong with it:

- The customer-charging job no longer shows the shape of what we charged. A
  three-leg split would make this 15 rows, a four-leg split 20.
- `Part A` / `Part B` are opaque. Nothing names the driver on each leg.
- Legs are interleaved by item, so no leg can be read as a unit.
- Every row is individually editable, including revenue — which is how the parent
  drifts out of sync with the children.

### Stage 3b — The children (KT4071VA / KT4071VB)

| | Revenue | Cost | Profit | Margin |
|---|---:|---:|---:|---:|
| KT4071VA (Leg A, 80%) | 91.20 | 55.20 | 36.00 | 39.5% |
| KT4071VB (Leg B, 20%) | 22.80 | 13.80 | 9.00 | 39.5% |
| **Sum** | **114.00** | **69.00** | **45.00** | **39.5%** |

Both children currently expose the full editing surface — per-item edit, delete, and
**Add Item**. That is the second way the parent and children drift apart.

---

## 3. The proposed parent Price Breakdown

Restore the parent to **its original 5 rows** and break the *cost* column out per leg.
Revenue stays a single column, because there is only ever one customer being charged.

### Where this UI applies

| Job | Price Breakdown shown |
|---|---|
| Not split | The current modal, unchanged. |
| **Split parent** | **The new modal described below.** |
| Split child (Part A / Part B) | The current modal, but **read-only** — see §6. |

### The header stays

The three summary cards — **Total Revenue**, **Total Cost**, **Gross Profit** with the
margin badge — remain exactly as they are today (114.00 / 69.00 / 45.00 / 39.5% for the
worked example). The job's overall profitability must stay legible at the top of the
modal; the per-leg breakdown is added below it, not in place of it.

### Leg summary strip

Answers "which driver is doing which bit", which nothing on the current screen does.

| Leg | Job | Driver | Share | Revenue | Cost | Margin |
|---|---|---|---:|---:|---:|---:|
| A | KT4071VA | *driver name* | 80% | 91.20 | 55.20 | 39.5% |
| B | KT4071VB | *driver name* | 20% | 22.80 | 13.80 | 39.5% |

An unassigned leg must read **Unassigned** explicitly, not blank.

### Price items grid

| Item | Revenue | Part A cost | Part B cost | Total cost | Profit | Margin |
|---|---:|---:|---:|---:|---:|---:|
| Base | 64.00 | 25.60 | 6.40 | 32.00 | 32.00 | 50% |
| Base Fuel | 16.00 | 19.20 | 4.80 | 24.00 | −8.00 | −50% |
| Items | 20.00 | 3.20 | 0.80 | 4.00 | 16.00 | 80% |
| Items Fuel | 5.00 | 2.40 | 0.60 | 3.00 | 2.00 | 40% |
| Congestion | 9.00 | 4.80 | 1.20 | 6.00 | 3.00 | 33% |
| **Total** | **114.00** | **55.20** | **13.80** | **69.00** | **45.00** | **39.5%** |

The two leg-cost column totals (55.20 / 13.80) are exactly the two child jobs' Total
Cost. The Revenue total is exactly the headline price. Reconciliation is visible on
the face of the screen rather than being a rule enforced somewhere else.

`Total cost`, `Profit` and `Margin` are derived columns.

### Leg cell contents

Each leg cell shows that leg's **cost** as the primary, editable figure, with that
leg's derived **revenue** as a subline:

```
Part A cost
  25.60
  rev 51.20
```

This mirrors how the Confirm Split Pricing screen already stacks the two figures
(`US$51.20` / `cost US$25.60`), just inverted to put cost first. It matters because
the three-column layout otherwise drops per-item leg revenue entirely, which is the
one thing the current split screen shows that the new grid would lose — and §8 makes
these the same component, so it has to work at split time too. Make the subline
toggleable if it reads as noise.

### Column widths at 3+ legs

Grow the modal width first — the current modal is narrow relative to the viewport and
has room to expand. Only once the modal is at its maximum width do the leg columns
scroll horizontally. When they do, **freeze the Item and Revenue columns** so the
operator keeps their anchor while scrolling through legs.

### What is editable

| Cell | Editable | Effect |
|---|---|---|
| **Revenue** | Yes | Re-derives each leg's revenue for that item by its share. Changes what the customer is invoiced. |
| **Part A / Part B cost** | Yes | Sets that leg's cost for that item directly. Does **not** touch revenue. Zero is valid. |
| **Share %** per item | Yes, secondary | Re-allocates revenue and non-overridden cost across the legs. |
| Total cost / Profit / Margin | No | Derived. |
| Child job headline price | No | Derived — see §6. |
| Parent headline price | No | Derived — it is the Revenue column total. See §6. |

An overridden leg cost is flagged visually and shows the derived value it replaced,
with a **reset** affordance, so it's obvious a human changed it.

---

## 4. The rules

1. **The parent is the only editable surface.** Child job prices are derived and
   read-only. This is what makes the archive bug unreachable rather than validated-for.
2. **The headline price is not separately editable on a split parent.** It *is* the
   sum of the Revenue column.
3. **Editing parent revenue re-derives child revenue by share.** Leg revenue for an
   item is always `item revenue × leg share`, and shares sum to 100%, so the legs
   always re-sum to the parent.
4. **Cost overrides are sticky.** Once a leg's cost for an item is overridden it
   survives later share changes until explicitly reset.
5. **Editing cost never changes revenue**, and vice versa.
6. **Shares always sum to 100%, per item.** Pick and implement a rounding remainder
   rule so leg amounts always re-sum to the parent to the cent — recommend allocating
   the remainder to the largest leg. 80/20 on these figures is exact, but a 3-way
   split will drift without a rule.
7. **Splitting never changes what the client is invoiced** — as the split screen
   banner already promises.

---

## 5. Data model — how each leg row keys back to its parent item

> Jacob's question: today the split deletes the parent's price-item rows and writes new
> per-leg rows named `Base Part A` / `Base Part B`, so the only link back to "this is the
> same conceptual item" is the name minus the ` Part X` suffix. How should the new grid
> key the grouping?

**It shouldn't have to.** Name matching is only necessary because the split destroys the
rows it derived from. Once the parent keeps its own items — the whole point of §3 — there
is a durable row to point at, and the grouping is a foreign key, not a string parse.

### The change

1. **Splitting stops deleting the parent's price items.** It becomes additive. The parent's
   rows stay exactly as they were and remain what the customer is charged on.
2. **Each child job gets one derived row per parent item**, carrying a key back to it:

| Column on the child price item | Meaning |
|---|---|
| `parent_price_item_id` | FK to the parent job's price item row. **This is the grouping key.** |
| `job_id` | the child (leg) job — which column the row lands in |
| `share_pct` | that leg's share of this item, so per-item shares (§3) work |
| `cost_override` | set when an operator edits that leg's cost directly (§4.4); null means derived |

3. **Exactly one child row per `(parent_price_item_id, job_id)`**, enforced by a unique
   constraint. Every item exists on every leg — at zero cost where that driver isn't paid
   for it (§7.1) — so the grid is a clean pivot and the column totals reconcile by
   construction rather than by convention.
4. **Deleting a parent item cascades** to its child rows (§7.1).
5. **The ` Part A` / ` Part B` suffix becomes unnecessary.** The child job number already
   identifies the leg, so child jobs can show the item's real name. Check first for anything
   downstream that parses those names today.

### Where the values are written

Downstream consumers — Accounts, settlement, invoicing — read the child job's rows, so the
derived figures need to exist as real rows rather than being computed on read. Keep them
stored, but make the parent the **only write path**: one transactional operation recalculates
and rewrites every affected child row whenever a parent item, a share, or a leg cost changes.
That keeps the single-source-of-truth property of §4 while leaving downstream reads working
exactly as they do now.

⚠️ **Unconfirmed, but it bears on this decision.** George has seen a courier payment edited in
the Dispatch Web price breakdown not appear in the Accounts courier payment section. It may be
unrelated. But if Accounts reads a snapshot taken at archive or settlement rather than the live
child rows, the rewrite path above has to reach that snapshot too — otherwise the editability
promised in §7.5 silently does nothing. Worth confirming before Phase 2 starts.

### Existing split jobs — not in scope

No backfill, no migration. The fix applies **going forward only**.

This needs no extra work, because a job split before the change has had its original parent
price items deleted already — they aren't recoverable, so the new grid could not render one
whatever key it used. Those jobs keep today's flat breakdown until they age out, which means
the existing display path stays until then. Everything in this spec applies to jobs split
after the change.

---

## 6. Locking the two editable surfaces that cause the drift

Both need doing, or the parent and children can still diverge from the other side:

- **Parent job headline price field** (`PRICING US$114.00` on the job detail panel):
  on a split parent, make it read-only with a link into the breakdown.
- **Child job Price Breakdown modal** (KT4071VA / KT4071VB): on a job that is a split
  child, make Price Items read-only — no per-row edit, no delete, no **Add Item** —
  with a line explaining that pricing is managed on the parent, and a link to it.

---

## 7. Decisions

### 7.1 Adding and removing price items after the split — allowed

Both are allowed on the parent, and both cascade to every leg:

- **Adding** an item creates a corresponding cost item on **every** leg job. Each leg's
  cost cell is then editable, so the operator can pay some drivers for it, pay them
  different amounts, or set a leg to **zero** to not pay that driver at all.
- **Removing** an item removes it from every leg, which removes that element of the
  drivers' pay.

The item's **revenue** still needs a share in order to divide across the legs.
Recommend defaulting a new item to the overall leg share, editable inline — the cost
cells are independent of it and can be zeroed regardless.

Adding or removing is subject to the locks in §7.5: an item can't be added or removed
in a way that changes revenue after invoicing, or changes a settled leg's cost.

### 7.2 More than two legs — widen, then scroll

Grow the modal before introducing horizontal scroll; freeze the Item and Revenue
columns once scrolling starts. See §3.

### 7.3 Existing out-of-sync jobs — no migration needed

Jobs that had drifted have already been corrected manually, so there is **no history
to remediate**. No migration, no repair tool. The structural fix only needs to prevent
it recurring.

### 7.4 Reassigning a leg to a different driver — cost is unchanged

Cost stays as allocated. It does **not** re-derive against the new driver's pay setup.

### 7.5 When editing locks

Delivery does **not** lock anything — prices remain editable after POD. The two locks
have different triggers, and the cost lock is **per leg**, because each leg has its own
driver and its own settlement:

| What | Editable until | After that |
|---|---|---|
| **Revenue** (whole column) | the parent job is **invoiced to the customer** | read-only |
| **Part X cost** (one leg's column) | **that leg's driver settlement** has been run | read-only *for that leg only*; other legs stay editable |
| **Share %** | invoiced, or any affected leg settled — whichever comes first | read-only |

So Leg A can be settled and frozen while Leg B is still fully editable. The UI must
show the lock per column, with the reason ("settled 12 Sep" / "invoiced"), not disable
the whole grid.

Share % locks on the stricter of the two triggers because it moves revenue *and* cost;
this is the conservative choice and worth confirming.

Editing a leg's cost **after** settlement is handled by the existing retrospective
function in Accounts, which raises a deduction or an extra payment. That is explicitly
**out of scope** for this UI — the Job Search modal just needs to stop offering the
edit and say where it's done.

---

## 8. One modal, two modes

The Confirm Split Pricing screen and the new parent edit modal should be **the same
component**, so the operator can do this editing at the time of splitting rather than
splitting first and fixing it afterwards.

| | Split mode (pre-split) | Edit mode (post-split) |
|---|---|---|
| Legs identified by | Leg A / Leg B + mileage | child job number + driver name |
| Overall share control | Yes — seeded from road distance | Yes |
| Per-item share | Yes | Yes |
| Per-leg cost cells | Yes, editable | Yes, editable |
| Header summary cards | Yes | Yes |
| Locks (§7.5) | N/A | Applied per column |
| Primary action | **Confirm & Split** | **Save & Close** |

The banner already on the split screen — *"The job total stays US$114.00 — splitting
does not change what the client is invoiced"* — is worth keeping in both modes.

---

## 9. Suggested build order

**Phase 1 — stop the bleeding.** Lock the parent headline price field and make the
child breakdowns read-only (§6). This alone kills the archive bug with no new UI, and
needs no migration (§7.3).

**Phase 2 — stop deleting, then build the grid.** The schema change in §5 comes first:
splitting stops destroying the parent's price items, and each child row carries a foreign key
back to its parent item. Everything else depends on it. Then the grid itself — the parent's
original rows, the per-leg cost columns, leg summary strip and per-column locks (§3, §7.5),
shown for split jobs only.

**Phase 3 — unify.** Re-point Confirm Split Pricing at the same component (§8).

---

## 10. Acceptance criteria

- [ ] Splitting no longer deletes the parent's price items.
- [ ] The new grid applies to jobs split after the change; jobs split before it continue to
      render as they do today, with no migration.
- [ ] Each child price item carries a foreign key to the parent item it derives from; no
      code path groups rows by parsing names.
- [ ] Exactly one child row exists per (parent item, leg), enforced by a unique constraint.
- [ ] A parent-side change rewrites every affected child row in one transaction.
- [ ] The new modal is shown **only** for split parent jobs; unsplit jobs are unchanged.
- [ ] The Total Revenue / Total Cost / Gross Profit header cards are retained.
- [ ] After a split, the parent Price Breakdown shows the job's **original** items
      (5 for this example), not the Part A / Part B expansion.
- [ ] The grid has one cost column per leg, with a total row per column.
- [ ] The leg cost column totals equal the corresponding child jobs' Total Cost
      (55.20 and 13.80 in the worked example).
- [ ] The Revenue column total equals the parent headline price (114.00) and the sum
      of the children's revenue (91.20 + 22.80).
- [ ] A leg summary strip names the driver on each leg, or **Unassigned**.
- [ ] The parent headline price field is read-only on a split parent.
- [ ] Child job Price Breakdowns are read-only — no edit, delete, or Add Item.
- [ ] Editing an item's Revenue updates both children's revenue by share, and the
      parent total, in one action.
- [ ] Editing a leg's cost cell changes no revenue figure anywhere, and accepts zero.
- [ ] Adding an item on the parent creates a cost item on every leg; removing it
      removes it from every leg.
- [ ] Changing a share leaves overridden costs untouched and re-derives the rest.
- [ ] Reassigning a leg's driver leaves that leg's cost unchanged.
- [ ] Revenue is read-only once the parent is invoiced.
- [ ] A settled leg's cost column is read-only while unsettled legs stay editable.
- [ ] Prices remain editable after POD.
- [ ] At 3+ legs the modal widens before scrolling; Item and Revenue stay frozen.
- [ ] A split job can be archived after any sequence of the above edits.

---

## 11. Out of scope

- The negative fuel line (staging rating quirk).
- Changing how the initial mileage-based allocation is calculated.
- Retrospective cost edits after settlement — handled by the existing Accounts
  function that raises a deduction or extra payment (§7.5).
- Invoicing and driver-payment-run behaviour downstream of the split.

---

## 12. Screenshots

Source of record — Google Drive (Urgent Couriers account):
<https://drive.google.com/drive/folders/1uCU_e608q818EzbDLIqrzHDAcRDgkRBp>

| File in Drive | Shows | Spec section |
|---|---|---|
| `PreSplit Price Breakdown.png` | KT4070V before splitting, 5 items, US$114.00 | §2 Stage 1 |
| `Split screen1.png` | Confirm Split Pricing, 80/20 by road distance | §2 Stage 2 |
| `ParentPrice BreakDown.png` | KT4071V after splitting, 10 items — the problem screen | §2 Stage 3 |
| `PriceBreakdown PartA.png` | KT4071VA, Leg A child, US$91.20 / US$55.20 | §2 Stage 3b |
| `Price Breakdown PartB.png` | KT4071VB, Leg B child, US$22.80 / US$13.80 | §2 Stage 3b |

Every figure from all five screens is transcribed into the tables above, so this
spec is complete without them. The images are corroboration, not a dependency.
