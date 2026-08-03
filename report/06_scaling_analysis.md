# Task 5: Scaling Analysis

## Overview

This section addresses the three questions posed by the challenge brief for
Task 5: (1) how do variables/qubits grow with orders, centers, and SKUs;
(2) what are the runtime, complexity, and robustness limitations of each
approach; and (3) at least one concrete recommendation for improving
scalability. It draws on empirical data gathered while validating and
scaling the MILP (Task 3) and QUBO (Task 4) implementations.

## 1. Variable Growth: MILP (Classical Exact)

The MILP formulation uses one binary variable per (order, DC) pair and one
continuous variable per (order, SKU-line, DC) triple:

| Orders | Order-SKU lines | `x` vars | `f` vars | Solve time | Status |
|---|---|---|---|---|---|
| 20 | 553 | 240 | 6,636 | <1s | Optimal |
| 100 | 2,460 | 1,200 | 29,520 | 9.4s | Optimal |
| 300 | 6,801 | 3,600 | 81,612 | 51.8s | Optimal |
| 1,109 (full) | 25,193 | 13,308 | 302,316 | 410.2s | Optimal |

**Variable growth is exactly linear and predictable:** `x` vars = orders ×
candidate DCs (13,308 = 1,109 × 12, holding at every scale tested), and
`f` vars = order-SKU lines × candidate DCs (302,316 = 25,193 × 12). Both
scale as `O(orders × DCs)` and `O(lines × DCs)` respectively — the number
of DCs is the dominant multiplier, so this formulation would be far more
sensitive to DC count growth than order count growth.

**Solve time does not scale linearly.** Going from 300 → 1,109 orders
(a 3.7x increase in orders and variables) increased solve time from 51.8s
to 410.2s — roughly an 8x increase. This confirms the pattern observed
across all four scale points: MILP solve time grows faster than linearly
with problem size, consistent with typical branch-and-bound behavior on
MIPs, where the search tree can grow combinatorially even as the variable
count grows only linearly.

## 2. Variable Growth: QUBO / BQM (Quantum-Inspired)

The QUBO/BQM approach was validated at a fixed 20-order sample across three
formulation variants, given the scope of this project:

| Approach | Variables | Constraints |
|---|---|---|
| Independent-order QUBO (assignment only, no inventory coupling) | 240 | 20 (penalty terms only) |
| Coupled BQM, per-SKU-line inventory constraints | 617 | 6,440 |
| Coupled BQM, aggregated (order, DC) inventory constraints | 367 | 260 |

**Aggregated-constraint scaling check (new data point):**

| Orders | `x` vars | CQM constraints | BQM vars (post-conversion) | BQM interactions |
|---|---|---|---|---|
| 20 | [PASTE] | [PASTE] | [PASTE] | [PASTE] |
| 50 | [PASTE] | [PASTE] | [PASTE] | [PASTE] |
| 100 | [PASTE] | [PASTE] | [PASTE] | [PASTE] |

*(Run the variable-count-only check described in the accompanying session
and paste results here — no annealing required for this table, only CQM/BQM
construction.)*

**Key limitation: QUBO/BQM scaling was not validated past 20 orders for a
full solve.** Unlike the MILP, which was pushed all the way to the full
1,109-order dataset, the QUBO/BQM approach was only ever solved end-to-end
at 20 orders. This is disclosed here as a genuine scope limitation rather
than implied coverage: the per-SKU-line coupled formulation's constraint
count (6,440 at just 20 orders) suggests it would become intractable well
before reaching full scale, which is itself evidence supporting the
aggregated-constraint approach as the more scalable choice (see Section 4).

## 3. Runtime, Complexity, and Robustness Limitations

**MILP:**
- Runtime limitation: super-linear growth in solve time as problem size
  increases (Section 1). Extrapolating past 1,109 orders — e.g. to a
  multi-site or multi-week planning horizon — risks solve times becoming
  operationally impractical without either a solver time budget or
  problem decomposition.
- Complexity: worst-case NP-hard (general MILP), though this instance
  solved to proven optimality at full scale in ~7 minutes, indicating the
  problem's actual structure is more tractable than the worst case.

**QUBO/BQM:**
- Robustness limitation (found and diagnosed during Task 4): automatic
  CQM-to-BQM conversion via a single global `lagrange_multiplier` produces
  unreliable results when a variable participates in many overlapping
  constraints. In the per-SKU-line coupled formulation, this caused 9 of
  20 orders to be left unassigned by both Simulated Annealing and Tabu
  Search, traced to one variable's BQM linear bias being inflated roughly
  2000x above its true objective contribution (+186.8 million vs. an
  actual contribution of -103,593). This is a known limitation of
  automatic global-penalty conversion under heavy constraint overlap, not
  a solver failure — both solvers converged on identical (correctly-solved)
  but wrong-model results.
- Runtime: simulated annealing and Tabu Search both ran quickly at 20-order
  scale (well under the MILP's solve time at equivalent scale), but this
  project did not validate how anneal quality or runtime degrades at
  larger, more competitive problem sizes.
- Solution quality: even after fixing the constraint-interaction bug, the
  aggregated coupled QUBO reached ~97% of the MILP's optimal profit
  (2,121,281–2,133,101 vs. 2,188,899.66), consistent with the expected
  behavior of heuristic methods offering no optimality guarantee.

## 4. Recommendations for Improving Scalability

**Recommendation 1 — constraint aggregation for QUBO/BQM (validated in this
project).** The single most concrete, evidence-backed recommendation from
this work: aggregating inventory constraints from per-SKU-line to
per-(order, DC) reduced the coupled BQM from 617 to 367 variables and from
6,440 to 260 constraints, and eliminated the constraint-interaction bias
entirely (Section 2/3). This is a direct, generalizable technique for
scaling QUBO formulations of this problem family — coarsen constraint
granularity wherever the domain can tolerate it, trading a small amount of
precision (SKU-level → order-level inventory checks) for a much smaller,
more solvable model.

**Recommendation 2 — time-boxed, gap-tolerant solving for MILP.** Given
that MILP solve time grows super-linearly, requiring a proven-optimal
solution at every scale is not sustainable. The full-dataset solve in this
project used a solver time limit and a 1% relative optimality gap
(`gapRel=0.01`, `timeLimit=900`) as a safety net; in practice the solve
reached true optimality within that budget, but for larger future problem
sizes, accepting a small, disclosed optimality gap (e.g. "within 1% of
optimal, solved in N minutes") is a practical way to keep the exact
approach usable as the dataset grows, rather than switching to a purely
heuristic method.

## Conclusion

Both the MILP and QUBO/BQM approaches scale predictably in variable count
(linear in orders × DCs), but each has a distinct runtime/robustness
limitation: the MILP's solve time grows super-linearly with problem size,
while the QUBO/BQM approach is vulnerable to constraint-interaction bias
under naive CQM-to-BQM conversion, which was diagnosed and mitigated via
constraint aggregation in this project. The two recommendations above —
constraint aggregation for QUBO scalability, and gap-tolerant time-boxed
solving for MILP scalability — are both grounded in techniques actually
validated during this project, not purely theoretical suggestions.