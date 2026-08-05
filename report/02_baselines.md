# Classical Baselines

## Baseline 1: Default-Assignment Baseline

**Definition:** Every order is fulfilled entirely from its default distribution
center (`Plant`). No reassignment to alternate DCs occurs, regardless of whether
the default DC has sufficient inventory.

**Method:**
- For each order-SKU line, filled quantity = min(ordered quantity, available
  inventory at the default DC on the order's transportation planning date).
- Revenue = filled quantity × per-case price (`Order_SKU_Revenue /
  OrderedQty_converted`).
- Penalty cost = unfilled quantity × SKU revenue × penalty rate
  (`Penaltyforpotentialcuts`).
- Shipping cost = per-lane shipping cost (`Shipping_Cost`) from
  `input_shipping_cost_data.csv`, joined on default `Plant` and order `ZipCode`,
  charged **once per (order, DC) pair actually used** — not once per
  order-SKU line (see Data Correction Note below).
- Profit = revenue − penalty cost − shipping cost.

**Results:**

| Metric | Value |
|---|---|
| Total ordered quantity | 2,345,613 |
| Total filled quantity | 2,247,512 |
| Overall fill rate | 95.82% |
| Total revenue | 84,915,014.38 |
| Total penalty cost | 7,600,586.25 |
| Total shipping cost | 1,386,508.00 |
| **Total profit (revenue − penalty − shipping)** | **75,927,920.13** |
| Number of reassignments | 0 (by definition) |

**Interpretation:** This baseline represents the "do nothing" scenario — no
optimization or reassignment logic applied. It serves as the floor that all
other approaches (greedy, exact optimization, quantum-inspired) must improve
upon. A 95.82% fill rate indicates most demand is satisfiable locally, but the
remaining ~4.18% unfilled demand drives a meaningful penalty cost of ~$7.6M.
Combined with ~$1.39M in shipping cost, this yields a baseline profit of
~$75.9M — the number every other method in this report is measured against.

## Baseline 2: Greedy / Sequential-Reassignment Baseline

**Definition:** For each order-SKU line that could not be fully filled at its
default DC, search all other DCs for spare inventory of the same SKU on the
same date. Assign the shortfall to the DC with the most available inventory,
depleting that DC's inventory pool as assignments are made (sequential/greedy —
no global optimization, first-come-first-served by row order).

**Method:**
- Only order-SKU lines with unfilled demand from Baseline 1 are considered
  (1,340 lines, 98,101 units of shortfall).
- For each, the alternate DC with the greatest available inventory is chosen;
  as much of the shortfall as possible is filled from it.
- Revenue = default-DC revenue (from Baseline 1) plus the additional revenue
  from quantity recovered via reassignment.
- Shipping cost for reassigned lines uses the new (reassigned) DC; unaffected
  lines keep their Baseline 1 shipping cost — again charged once per
  (order, DC) pair actually used, not once per line.
- Profit = revenue − penalty cost − shipping cost.

**Results:**

| Metric | Baseline 1 (Default) | Baseline 2 (Greedy) | Change |
|---|---|---|---|
| Total filled quantity | 2,247,512 | 2,305,622 | +58,110 |
| Overall fill rate | 95.82% | 98.30% | +2.48 pts |
| Total revenue | 84,915,014.38 | 87,092,157.33 | +2,177,142.95 |
| Total penalty cost | 7,600,586.25 | 2,731,905.98 | -4,868,680.27 |
| Total shipping cost | 1,386,508.00 | 3,221,692.00 | +1,835,184.00 |
| **Total profit** | **75,927,920.13** | **81,138,559.36** | **+5,210,639.23** |
| Number of reassignments | 0 | 849 | — |

**Interpretation:** Greedy reassignment recovers a meaningful amount of unfilled
demand (58,110 of 98,101 units, or ~59% of the original shortfall) by diverting
orders to DCs with spare inventory. This comes at a real cost: shipping expense
rises by ~$1.84M, since 849 reassignments mean many orders now ship from a
second (or, for split orders, an additional) DC beyond their default. However,
the penalty cost reduction (~$4.87M) far outweighs the shipping cost increase,
producing a net profit gain of ~$5.2M over the default baseline. This baseline
still has 39,991 units of unrecoverable shortfall, where no alternate DC had
any spare inventory for that SKU/date at all — a hard constraint no
reassignment strategy can overcome without addressing inventory positioning
itself.

**Limitation of this baseline:** greedy assignment is order-of-processing
dependent and picks only the single best alternate DC per line without
considering global trade-offs (e.g., an earlier greedy choice may consume
inventory another, higher-priority order needed more). It can also split a
single order's SKU lines across multiple DCs (228 of 419 reassignment-affected
orders were split this way), unlike the default baseline and the MILP, both of
which enforce exactly one DC per order. This motivates the need for the formal
mathematical optimization approach in the next section, which considers all
orders and DCs jointly rather than sequentially, and enforces single-DC
assignment by constraint.

## Data Correction Note

An earlier version of the shipping-cost calculation for both baselines summed
`Shipping_Cost` once per order-*SKU-line* rather than once per (order, DC)
pair actually used. Since a typical order spans ~22.7 SKU lines
(25,193 lines / 1,109 orders), this multiply-counted the same shipping charge
once per line on any multi-line order — inflating default shipping to
$32,210,085.00 and greedy shipping to $33,743,662.00, both roughly 20-25x too
high. Both were corrected by charging shipping once per distinct (order, DC)
pair each baseline actually uses, matching how the MILP (Section 3) charges
shipping via its binary `x_{o,d}` assignment variable. All figures in this
document reflect the corrected calculation.