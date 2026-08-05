# Mathematical Formulation

## Overview

This formulation is adapted from Nestlé's documented DOM proof-of-concept
approach, simplified to align with the fields available in the anonymized
challenge data pack. It omits case-pick/pallet-pick logistics and the
two-tier penalty structure used internally by Nestlé, since the data pack
does not provide the granularity needed to support them. It is formulated
as a Mixed-Integer Linear Program (MILP).

## Sets

- `o ∈ O` — orders (`Group_Flag`)
- `s ∈ S(o)` — SKUs within order `o` (`MaterialNumber`)
- `d ∈ D(o)` — candidate distribution centers for order `o` (default `Plant`,
  plus any DC holding inventory for the order's SKUs)
- `t(o)` — the order's fixed planning date (`transportationplanningdate`)

## Decision Variables

- `x_{o,d} ∈ {0,1}` — 1 if order `o` is assigned entirely to DC `d`
- `f_{o,s,d} ≥ 0` — cases of SKU `s` in order `o` fulfilled from DC `d`

## Parameters

- `Price_{o,s}` — revenue per case, computed as `Order_SKU_Revenue / OrderedQty_converted`
  (see Data Correction note below; the raw field is total line revenue, not a per-case rate)
- `Demand_{o,s}` — ordered quantity (`OrderedQty_converted`)
- `PenaltyRate_o` — penalty rate (`Penaltyforpotentialcuts`)
- `ShipCost_{o,d}` — shipping cost from DC `d` to order `o`'s zip
  (`Shipping_Cost`, joined via `ZipCode`)
- `Inv_{d,s,t}` — available inventory at DC `d` for SKU `s` on date `t`
  (`Available_inventory`)
- `DockCapacity_{d,t}` — dock capacity at DC `d` on date `t` (`Dock_Capacity`)

## Objective Function

Maximize total revenue, minus penalty for unmet demand, minus shipping cost:
```
Maximize:
  Σ_{o,s,d} Price_{o,s} · f_{o,s,d}
  − Σ_{o,s} PenaltyRate_o · Price_{o,s} · (Demand_{o,s} − Σ_d f_{o,s,d})
  − Σ_{o,d} ShipCost_{o,d} · x_{o,d}
```
## Constraints

1. **Single DC per order:**
   `Σ_d x_{o,d} = 1` ∀ o

2. **Fulfillment only at the assigned DC:**
   `f_{o,s,d} ≤ Demand_{o,s} · x_{o,d}` ∀ o,s,d

3. **Cannot exceed ordered quantity:**
   `Σ_d f_{o,s,d} ≤ Demand_{o,s}` ∀ o,s

4. **Inventory constraint:**
   `Σ_o f_{o,s,d} ≤ Inv_{d,s,t}` ∀ d,s,t

5. **Dock capacity constraint:**
   `Σ_o x_{o,d} ≤ DockCapacity_{d,t}` ∀ d,t

## Relationship to the Baselines

- The **default-assignment baseline** is the special case where `x_{o,d}=1`
  only for `d = default Plant`, for every order.
- The **greedy baseline** is a fast, sequential heuristic approximation of
  this same problem — it makes locally good choices but does not guarantee
  global optimality, since inventory consumed by an earlier order in
  processing order may have been better allocated elsewhere.
- This MILP formulation, solved exactly, provides the true optimal benchmark
  that both baselines are compared against in later sections.
- The default and greedy baselines were computed across the full 1,109-order
  dataset from the start. The MILP was validated on progressively larger
  subsets (20 / 100 / 300 orders) before being solved on the full dataset
  (see Scaling Validation below). With the full-dataset MILP result now
  available ($84,666,068.27 profit, 97.17% fill rate), all three methods
  can be compared directly on identical, dataset-wide numbers for the
  first time — the MILP result serves as the true optimal benchmark
  against which the default and greedy baselines' full-dataset profit
  and fill rate gaps can now be reported.

## Assumptions

- **Missing penalty rate (`Penaltyforpotentialcuts`):** Rows with a missing
  penalty rate correspond exclusively to non-top customers (`IsTopCust = 'N'`),
  confirmed by cross-tabulation. This indicates these customers have no
  contractual penalty clause, rather than missing/unrecorded data. Missing
  values are treated as 0 (no penalty), reflecting this business distinction.

## Sample Validation (20 orders)

The formulation was implemented in PuLP and solved on a tractable sample of 20
orders (553 order-SKU lines, 12 candidate DCs) to validate correctness before
scaling to the full dataset.

**Result:**
- Status: Optimal
- Objective value (revenue − penalty − shipping): $2,188,899.66
- Total cases filled: 53,544 / 54,744 demanded (97.8% fill rate)
- Every order assigned to exactly one DC (constraint 1 satisfied)

This confirms the formulation is correct and solvable. Two data issues were
identified and resolved during implementation:
1. Six DC-SKU-date inventory records had negative `Available_inventory`
   (as low as -400), which made the model infeasible. These were clipped
   to 0, treating oversold/backordered stock as zero available inventory.
2. `Order_SKU_Revenue` represents total line revenue, not per-case price.
   A corrected per-case price was derived as
   `Order_SKU_Revenue / OrderedQty_converted` for use in the objective
   function.

## Scaling Validation

The formulation was tested at increasing scale to confirm robustness, then
solved on the full dataset:

| Orders | Order-SKU lines | x vars | f vars | Solve time | Status | Fill rate |
|---|---|---|---|---|---|---|
| 20 | 553 | 240 | 6,636 | <1s | Optimal | 97.8% |
| 100 | 2,460 | 1,200 | 29,520 | 9.4s | Optimal | 97.42% |
| 300 | 6,801 | 3,600 | 81,612 | 51.8s | Optimal | 96.99% |
| 1,109 (full) | 25,193 | 13,308 | 302,316 | 410.2s | Optimal | 97.17% |

Solve time grows faster than linearly with order count, consistent with
typical MILP behavior. 300 orders (~27% of the full 1,109-order dataset) was
validated as a tractable subset early in the project; the full 1,109-order
dataset was subsequently solved directly (row above) to genuine optimality
(not a gap-limited cutoff), providing an exact optimal benchmark at full
scale rather than a subset extrapolation.

**Full-dataset result:**
- Status: Optimal (solved exactly, not gap-terminated)
- Objective value (revenue − penalty − shipping): $84,666,068.27
- Total cases filled: 2,279,299 / 2,345,613 demanded (97.17% fill rate)
- Fill rate is consistent with the 20/100/300-order runs (97.8%, 97.42%,
  96.99%), confirming inventory and dock capacity are not a materially
  tighter bottleneck at full scale. Average profit per order ($76,344) is
  lower than in the 20-order sample ($109,445), which reflects the full
  dataset's broader mix of orders (including lower-margin and higher
  penalty-exposure orders) rather than any degradation in solution quality.

## Baseline Comparison (Full Dataset, 1,109 Orders)

With the full-dataset MILP solved and both baselines corrected for a
shipping-cost double-counting bug (shipping is a per-order cost, not a
per-SKU-line cost — see note below), all three methods can now be compared
directly on identical dataset-wide numbers:

| Method | Revenue | Penalty | Shipping | Profit | Fill rate | Gap vs. MILP |
|---|---|---|---|---|---|---|
| Default assignment | $84,915,014.38 | $7,600,586.25 | $1,386,508.00 | $75,927,920.13 | 95.82% | −$8,738,148.14 (−10.32%) |
| Greedy reassignment | $87,092,157.33 | $2,731,905.98 | $3,221,692.00 | $81,138,559.36 | 98.30% | −$3,527,508.91 (−4.17%) |
| **MILP (exact)** | — | — | — | **$84,666,068.27** | 97.17% | 0 (optimal) |

**Interpretation:** The greedy heuristic (849 reassignments across 1,109
orders) recovers most of the profit gap left by naive default assignment,
landing within ~4.2% of the true optimum — a reasonable result for a fast,
local heuristic. Notably, greedy's fill rate (98.30%) is *higher* than the
MILP's (97.17%), even though its profit is lower: the MILP maximizes profit,
not fill rate, and can rationally leave lower-value demand unfilled when the
penalty cost of doing so is cheaper than the shipping cost required to fill
it. This is expected behavior, not an error.

**Modeling note — greedy vs. MILP order-splitting:** the greedy heuristic
reassigns at the individual SKU-line level, so an order can end up split
across multiple DCs (228 of 419 reassignment-affected orders were split
across more than one DC). The MILP and default baseline both enforce
"exactly one DC per order" by constraint. This is a genuine structural
difference between the greedy heuristic and the other two methods, not a
bug — it is disclosed here rather than forced into single-DC form, since
constraining greedy to match would change what heuristic is actually being
evaluated.

**Data correction note:** an earlier version of the baseline shipping-cost
calculation summed `Shipping_Cost` once per order-*SKU-line* rather than
once per order actually shipped from a given DC, which multiply-counted
shipping for any order with more than one SKU line (inflating default and
greedy shipping to $32.2M and $33.7M respectively). Both were corrected to
charge shipping once per (order, DC) pair actually used.


This formulation is linear with no bilinear (variable × variable) terms,
making it directly solvable via classical MILP solvers (PuLP/OR-Tools).
For the quantum-inspired implementation, constraints 1–5 will be reformulated
as penalty terms added to the objective, converting this into a Quadratic
Unconstrained Binary Optimization (QUBO) problem suitable for solving via
simulated annealing (`dimod`/`neal`).