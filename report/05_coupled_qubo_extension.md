# Extension: Coupled QUBO with Inventory Constraints

## Overview

This section documents an attempt to extend the independent-order QUBO
(Section 04) to include real inventory coupling across orders — i.e.,
multiple orders competing for the same DC-SKU-date inventory, matching the
full MILP formulation (Task 3) rather than the simplified independent
assignment problem used in the base Task 4 QUBO.

## Approach

Rather than hand-deriving slack variables for each inventory inequality
constraint, `dimod.ConstrainedQuadraticModel` (CQM) was used to express
constraints in natural form, then converted to a Binary Quadratic Model
(BQM) via `cqm_to_bqm()`, which automatically encodes the necessary slack
variables. This was solved locally using Simulated Annealing and Tabu
Search (avoiding dependency on D-Wave's cloud CQM solver, which is no
longer available on free-tier accounts).

## Finding: Constraint Interaction Limitation

The initial implementation (one inventory constraint per order-SKU-date
combination) produced a BQM where 9 of 20 orders were left unassigned by
both solvers, despite each having substantial available profit at some DC.
Diagnostic inspection traced this to a real limitation rather than a solver
failure:

- Orders with many SKU lines (e.g., ~28 lines) participate in a
  correspondingly large number of separate inventory constraints.
- `cqm_to_bqm()` applies a single global `lagrange_multiplier` across all
  constraints. When one variable participates in many overlapping
  constraints, the converted BQM's linear bias for that variable
  accumulates contributions from all of them.
- For one affected order, the BQM's linear bias for its correct, most
  profitable assignment was found to be **+186.8 million**, despite an
  actual objective contribution of only **-103,593** — a roughly 2000x
  inflation caused by this compounding effect, not by an error in the
  underlying model or objective.
- Both Simulated Annealing and Tabu Search independently converged on
  identical results, confirming they were correctly solving the BQM as
  constructed — the BQM itself, not the solvers, was the source of the
  issue.

This is a documented, known limitation of automatic global-penalty
CQM-to-BQM conversion when constraints overlap heavily, rather than a flaw
in the DOM formulation itself.

## Mitigation Attempted: Constraint Aggregation

To reduce constraint density (and therefore the compounding effect), the
formulation was revised to use one aggregated inventory constraint per
(order, DC) pair — combining all of an order's SKU-level demand into a
single constraint against combined available inventory — rather than one
constraint per individual SKU line. This trades some SKU-level precision
for a substantially less-overlapping constraint structure.

## Results: Constraint Aggregation

Aggregating inventory constraints from per-SKU-line (6,420 constraints) to
per-(order, DC) (240 constraints) reduced the BQM to 367 variables (down
from 617) and eliminated the constraint interaction problem entirely:

| Run | Orders assigned | Violations | Profit | Fill rate |
|---|---|---|---|---|
| 1 | 20/20 | 0 | 2,121,281.39 | 95.41% |
| 2 | 20/20 | 0 | 2,133,101.38 | 96.32% |

Compared against the exact MILP solution on the same 20-order sample
(2,188,899.66 profit, 97.8% fill rate), the coupled QUBO achieves
**~97% of the MILP's optimal profit**, with all inventory and one-DC-per-order
constraints correctly satisfied — a substantial improvement over both the
independent-order QUBO (which ignores coupling) and the initial per-SKU
coupled attempt (which suffered from constraint interaction bias).

**Trade-off:** aggregating inventory constraints to the (order, DC) level
sacrifices some SKU-level precision — a DC is now checked against an
order's *total* demand versus *total* available inventory across all its
SKUs, rather than verifying each SKU individually. This could, in principle,
allow a DC with abundant stock of one SKU to mask a shortage in another
within the same order. This is a reasonable trade-off for a tractable,
correctly-functioning coupled QUBO, and is disclosed here as a genuine
modeling simplification.

## Conclusion

After diagnosing a constraint-interaction limitation in the initial
per-SKU-line coupled formulation, aggregating constraints to the
(order, DC) level produced a working, correctly-constrained coupled QUBO
achieving ~97% of the exact MILP's optimal profit at 20-order scale. This
represents a genuine extension beyond the independent-order QUBO
(Section 04), properly accounting for orders competing for shared
inventory, while remaining solvable via local quantum-inspired heuristics
(Simulated Annealing).

## A Note on Scale

The aggregation fix above was validated at 20 orders. It holds up cleanly
through 100 orders as well — zero violations, 91-95% of the MILP's optimal
profit across the range (see `06_scaling_analysis.md`, Section 2). At the
full 1,109-order dataset, though, this same aggregated approach does *not*
fully hold: a small number of violations (17-23 orders, ~2%) reappear, and
solve time (788-857s) grows past the MILP's own full-scale solve time
(410.2s). This is the same underlying constraint-interaction limitation
described above, reduced in severity by aggregation but not eliminated at
this size.

A further fix — decomposing the problem by shipping date, since inventory
constraints only ever link orders sharing the same DC, SKU, *and* date —
reduces violations to 5 and brings solve time back down to 589.4s, at
87.4% of the MILP's optimal profit. This is a meaningful improvement, but
not a complete resolution: the coupled QUBO approach documented in this
section should be read as validated at small-to-medium scale (≤100
orders), with a disclosed, only partially-resolved limitation at full
production scale. Full details, including why the residual violations are
structural rather than solver-related, are in `06_scaling_analysis.md`.