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

The formulation was tested at increasing scale to confirm robustness before
selecting a final tractable subset size:

| Orders | Order-SKU lines | x vars | f vars | Solve time | Status | Fill rate |
|---|---|---|---|---|---|---|
| 20 | 553 | 240 | 6,636 | <1s | Optimal | 97.8% |
| 100 | 2,460 | 1,200 | 29,520 | 9.4s | Optimal | 97.42% |
| 300 | 6,801 | 3,600 | 81,612 | 51.8s | Optimal | 96.99% |

Solve time grows faster than linearly with order count, consistent with
typical MILP behavior. 300 orders (~27% of the full 1,109-order dataset) was
selected as the final tractable subset for detailed baseline comparison,
balancing solution scale against solve time and remaining project timeline.

## Path to Quantum-Inspired Solving (Task 4 preview)

This formulation is linear with no bilinear (variable × variable) terms,
making it directly solvable via classical MILP solvers (PuLP/OR-Tools).
For the quantum-inspired implementation, constraints 1–5 will be reformulated
as penalty terms added to the objective, converting this into a Quadratic
Unconstrained Binary Optimization (QUBO) problem suitable for solving via
simulated annealing (`dimod`/`neal`).

