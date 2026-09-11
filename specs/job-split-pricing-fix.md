# Fix: Job splitting must not alter the parent job's customer-facing price

**Reported by:** Victor Trinidad (Operations Manager, On The Go Cargo), via email "Splits" forwarded 2026‑09‑11
**Raised by:** Steve Bonnici
**Screen affected:** Job Details screen, Dispatch web app (`despatchweb`)

## Background

When a job booked by a customer is being run by two couriers, dispatch splits
it in the Job Details screen into:

- **Parent job** — the customer-facing job. This is what the customer is
  charged (the "headline price").
- **Part A** and **Part B** — two courier-facing jobs, one per leg, used to
  pay the drivers.

Once the job is split, each leg (Part A / Part B) covers only part of the
original route, so the two legs' combined mileage is not guaranteed to equal
the original single-route mileage the parent job was priced on.

## The bug

Today, splitting a job recalculates total mileage from the two new legs, and
that recalculated total is being used to update the **parent job's headline
price** (and its mileage). Splitting is a purely internal/dispatch operation
— it must never change what the customer is charged. Victor's report showed
a ticket ("Nelson") whose price and mileage changed after the split it
produced ("Mark Z"), which should not happen.

## Required behavior

### 1. Parent job is immutable on split

The parent job's headline price and mileage must be frozen at whatever they
were immediately before the split. No recalculation triggered by splitting,
re-splitting, or editing a leg's mileage may ever write to the parent job's
own price or mileage fields.

### 2. Mileage percentage is still calculated — but only feeds the split, not the parent

This part of the existing logic is correct and should be kept, just properly
scoped:

- `total_split_miles = Part A miles + Part B miles`
- `Part A percentage = Part A miles / total_split_miles`
- `Part B percentage = Part B miles / total_split_miles`

`total_split_miles` is a value used only for this percentage calculation. It
must never be written back to the parent job's mileage field.

### 3. Each split leg's headline price is a percentage share of the parent's headline price

- `Part A headline price = Part A percentage × Parent headline price`
- `Part B headline price = Part B percentage × Parent headline price`

`Part A headline price + Part B headline price` must always sum exactly to
the parent job's headline price (handle rounding so the two parts reconcile
to the cent — e.g. compute one part by percentage and derive the other as
the remainder, rather than rounding both independently).

### 4. Split legs keep the standard job price/cost format

Part A and Part B are jobs like any other, so they keep the normal two-field
structure:

- **Headline price** — as computed in (3) above.
- **Cost allocation** — calculated the same way cost allocation is normally
  derived against a job's headline price. No new/special cost logic is
  needed here; it should just operate on the leg's own (now-correct)
  headline price. This cost allocation is what the driver is paid for that
  leg.

## Out of scope / follow-up

Victor also flagged the "unassigned" ticket state produced mid-split as
visually confusing ("an eyesore"). That's a separate, cosmetic issue and is
not addressed by this spec — worth logging separately if it needs a UI
cleanup.

## Acceptance criteria

- [ ] Splitting a job never changes the parent job's headline price or
      mileage, regardless of how the two legs' mileage compares to the
      original route mileage.
- [ ] Editing a leg's mileage after a split recalculates that leg's (and its
      sibling's) percentage and headline price, but still never touches the
      parent.
- [ ] Part A headline price + Part B headline price == Parent headline price,
      exactly, after any split or re-split.
- [ ] Each leg's cost allocation (driver pay) is derived from that leg's own
      headline price using the existing cost-allocation calculation.
