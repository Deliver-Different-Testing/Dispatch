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
validate for at archive time.

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

### Leg summary strip (above the grid)

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

`Total cost`, `Profit` and `Margin` are derived columns. They're kept because the
current modal shows Profit and Margin on every row and dropping them would be a
regression; hide them behind a toggle if the grid gets tight.

### What is editable

| Cell | Editable | Effect |
|---|---|---|
| **Revenue** | Yes | Re-derives each leg's revenue for that item by its share. Changes what the customer is invoiced. |
| **Part A / Part B cost** | Yes | Sets that leg's cost for that item directly (an override of the rate-derived value). Does **not** touch revenue. |
| **Share %** per item | Yes, secondary | Re-allocates revenue and non-overridden cost across the legs. |
| Total cost / Profit / Margin | No | Derived. |
| Child job headline price | No | Derived — see §5. |
| Parent headline price | No | Derived — it is the Revenue column total. See §5. |

An overridden leg cost is flagged visually and shows the derived value it replaced,
with a **reset** affordance, so it's obvious a human changed it.

---

## 4. The rules

1. **The parent is the only editable surface.** Child job prices are derived and
   read-only. This is what makes the archive bug unreachable rather than validated-for.
2. **The headline price is not separately editable on a split parent.** It *is* the
   sum of the Revenue column. Editing revenue happens per item, in the breakdown.
3. **Editing parent revenue re-derives child revenue by share** — the behaviour asked
   for directly. Leg revenue for an item is always `item revenue × leg share`, and
   shares sum to 100%, so the legs always re-sum to the parent.
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

## 5. Locking the two editable surfaces that cause the drift

Both need doing, or the parent and children can still diverge from the other side:

- **Parent job headline price field** (`PRICING US$114.00` on the job detail panel):
  on a split parent, make it read-only with a link into the breakdown.
- **Child job Price Breakdown modal** (KT4071VA / KT4071VB): on a job that is a split
  child, make Price Items read-only — no per-row edit, no delete, no **Add Item** —
  with a line explaining that pricing is managed on the parent, and a link to it.

---

## 6. Edge cases to decide

These are real gaps in the proposal, not hypotheticals:

1. **Adding a price item after the split.** The parent's **Add Item** button still
   exists. A new item has no share yet — does it default to the overall leg share, go
   100% to one leg, or force the operator to choose? *Recommend: default to the overall
   share, editable inline.*
2. **Deleting a price item after the split** must remove it from every child.
3. **More than two legs.** Three columns becomes N+1. Decide the behaviour at 4+ legs —
   horizontal scroll, or collapse to a per-leg grouped view.
4. **Jobs already out of sync.** There will be existing split jobs in this state that
   still won't archive. They need a remediation path — recommend a "re-derive children
   from parent" action rather than a silent migration, so the operator sees what moved.
5. **Reassigning a leg to a different driver** after the split — does cost re-derive
   against the new driver's pay setup, or stay as allocated?
6. **Locking after POD / invoice.** Can prices still be edited once a leg is delivered
   or the parent invoiced, and what freezes at that point?

---

## 7. Suggested build order

**Phase 1 — stop the bleeding.** Lock the parent headline price field and make the
child breakdowns read-only (§5), and add the "re-derive children from parent" repair
action (§6.4). This alone kills the archive bug without any new UI.

**Phase 2 — the new grid.** Restore the parent's original rows and add the per-leg
cost columns and leg summary strip (§3).

---

## 8. Acceptance criteria

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
- [ ] Editing an item's Revenue on the parent updates both children's revenue by share,
      and the parent total, in one action.
- [ ] Editing a leg's cost cell changes no revenue figure anywhere.
- [ ] Changing a share leaves overridden costs untouched and re-derives the rest.
- [ ] A split job can be archived after any sequence of the above edits.
- [ ] Existing out-of-sync split jobs can be repaired and then archived.

---

## 9. Out of scope

- The negative fuel line (staging rating quirk).
- Changing how the initial mileage-based allocation is calculated.
- Invoicing and driver-payment-run behaviour downstream of the split.

---

## 10. Screenshots

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
