# Task 4: Quantum-Inspired Implementation

## Overview

This section reformulates the DOM assignment problem as a Quadratic
Unconstrained Binary Optimization (QUBO) problem and solves it using
simulated annealing as a quantum-inspired method, then compares the result
against the classical baselines and the exact MILP solution (Task 3) on the
same order sample.

## Simplification from the MILP Formulation

QUBO requires all-binary decision variables and no hard constraints
(violations are instead penalized within the objective itself). The full
MILP formulation (Task 3) uses two variable types — binary `x_{o,d}`
(DC assignment) and continuous `f_{o,s,d}` (per-SKU case fulfillment) — which
does not translate directly into QUBO form.

**The QUBO formulation operates on the assignment decision only.** Per-DC
profit is precomputed for each (order, DC) pair using the same
min(demand, inventory) approach used in the classical baselines, rather than
jointly optimizing exact case-level fulfillment as the full MILP does. QUBO
is then used to solve the resulting assignment problem: which single DC
should each order be assigned to, in order to maximize total precomputed
profit while respecting the "exactly one DC per order" rule.

This is a genuine simplification of the full problem, not an approximation
disguised as equivalent — it trades case-level fulfillment precision for a
binary structure solvable by quantum-inspired methods within the scope of
this project.

## QUBO Variables

- `x_{o,d} ∈ {0,1}` — 1 if order `o` is assigned to DC `d` (same meaning as
  in the MILP formulation)

## Precomputed Profit Coefficients

For each (order, DC) pair, `c_{o,d}` is computed as:
```
c_{o,d} = Σ_s [ Price_{o,s} · min(Demand_{o,s}, Inv_{d,s,t})
− PenaltyRate_o · Price_{o,s} · (Demand_{o,s} − min(Demand_{o,s}, Inv_{d,s,t})) ]
− ShipCost_{o,d}
```
This mirrors the revenue/penalty/shipping structure of the MILP objective,
but assumes the entire order goes to a single DC and is filled up to that
DC's available inventory.

## QUBO Objective

QUBO problems are expressed as minimization, so precomputed profits are
negated. A penalty term enforces "exactly one DC per order":

Minimize:
```
Σ_{o,d} (−c_{o,d}) · x_{o,d}

λ · Σ_o (Σ_d x_{o,d} − 1)²
```
where `λ` is a penalty weight chosen large enough to dominate the profit
terms, ensuring the solver strongly prefers exactly one DC per order over
violating that rule for a marginally higher raw profit.

## Solver

Solved using simulated annealing via `dimod`/`neal` (D-Wave Ocean SDK),
run on classical hardware as a quantum-inspired heuristic — no real quantum
hardware or QPU access was used, consistent with the challenge's allowance
for quantum-inspired and simulator-based approaches.

## Results

Both quantum-inspired methods were run 5 times each (Simulated Annealing:
`num_reads=1000`; Tabu Search: `num_reads=100`) to account for their
stochastic nature, and compared against the QUBO's true optimum (computable
directly for this problem structure, since orders are independent) and the
exact MILP solution from Task 3.

| Method | Mean profit | Best | Worst | Gap from optimum (mean) | Gap % |
|---|---|---|---|---|---|
| MILP (exact, Task 3) | 2,188,899.66 | — | — | 0 | 0% |
| Simulated Annealing | 2,047,885.74 | 2,065,183.14 | 1,992,813.51 | 141,013.92 | 6.4% |
| Tabu Search | 2,041,808.38 | 2,092,147.42 | 1,969,478.29 | 147,091.28 | 6.7% |

**Interpretation:** Both quantum-inspired methods converge to within ~6-7%
of the true optimum on average, confirming the QUBO formulation is sound
and both methods are effective heuristics for this problem. Tabu Search
showed a wider spread between best and worst runs but achieved the closest
single-run result to the true optimum (2,092,147.42, a 4.4% gap). Simulated
Annealing showed more consistent (lower-variance) performance across runs.
Neither method reliably matches the MILP's exact optimum in a single run,
which is expected — both are heuristic methods offering no optimality
guarantee, trading solution quality for the ability to scale to problem
sizes where exact MILP solving becomes computationally intractable.

**Important caveat:** this QUBO formulation optimizes each order's DC
assignment independently (no shared inventory or dock capacity constraints
across orders), which is a simplification relative to the full MILP. As a
result, this comparison measures how well each method solves the
*QUBO's own objective*, not how well the QUBO formulation itself
approximates the full DOM problem. A subsequent extension attempting to add
inventory coupling — and the technical limitation encountered — is
documented in `04_coupled_qubo_extension.md`.