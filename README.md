# Nestlé DOM Challenge — WISER 2026

## Overview

This repository contains a solution to the Nestlé Distributed Order
Management (DOM) challenge, part of the WISER 2026 Quantum Challenge. The
project designs, implements, and evaluates a solver that decides how
customer orders should be fulfilled across a network of distribution
centers, balancing inventory, dock capacity, shipping cost, and penalties
for unmet demand.

Three approaches were built and compared:
1. **Classical baselines** — a naive default-assignment approach and a
   greedy sequential-reassignment heuristic
2. **Exact optimization** — a Mixed Integer Linear Program (MILP), solved
   with PuLP, providing a true optimal benchmark
3. **Quantum-inspired optimization** — a QUBO/BQM formulation solved with
   Simulated Annealing and Tabu Search

## Repository Structure
```
nestle-dom/
├── notebooks/
│ └── 01_data_exploration.ipynb # All exploration, baselines, MILP,
│ QUBO code and results
├── report/
│ ├── 00_business_technical_summary.md # Task 1
│ ├── 01_data_understanding.md # Task 2
│ ├── 02_baselines.md # Task 2
│ ├── 03_formulation.md # Task 3 (MILP)
│ ├── 04_quantum_formulation.md # Task 4 (independent QUBO)
│ ├── 05_coupled_qubo_extension.md # Task 4 extension (coupled QUBO)
│ └── 06_scaling_analysis.md # Task 5
├── requirements.txt
└── README.md
```
**Note:** the Nestlé DOM data pack (`data/DOM-data/`) is **not included**
in this repository. See "Data Access" below.

## Setup

```bash
python3 -m venv venv
source venv/bin/activate      # or venv\Scripts\activate on Windows
pip install -r requirements.txt
```

## Data Access

This repository does not include the Nestlé DOM data pack, in line with
the challenge's data privacy guidelines (public repositories should
contain anonymized identifiers and aggregate metrics only, not the
underlying restricted dataset). The data pack is available to registered
participants via the WISER challenge workspace. Full documentation of the
data schema, file structure, and how each field maps to the model is in
`report/01_data_understanding.md`.

## How to Run

**With data pack access:** place the data pack at `data/DOM-data/`
(matching the structure referenced in `report/01_data_understanding.md`),
then open `notebooks/01_data_exploration.ipynb` and run all cells top to
bottom. The notebook follows the same order as the `report/` files above —
data exploration and baselines first, then the MILP formulation, then the
QUBO/quantum-inspired implementation and scaling tests.

**Without data pack access:** the notebook cannot be re-run end-to-end, but
all code is fully visible and documented inline, and every result —
including intermediate findings, bug diagnoses, and fixes — is written up
in the `report/` folder. No code execution is required to review the
methodology or verify the reported results.

## Key Results

| Method | Sample | Profit | Fill Rate |
|---|---|---|---|
| Default assignment (full dataset) | 1,109 orders | $75,927,920 | 95.82% |
| Greedy reassignment (full dataset) | 1,109 orders | $81,138,559 | 98.30% |
| **MILP (exact optimum, full dataset)** | 1,109 orders | **$84,666,068** | 97.17% |
| MILP (exact optimum, sample) | 20 orders | $2,188,900 | 97.80% |
| QUBO — independent orders (Simulated Annealing) | 20 orders | ~$2,047,886 | — |
| QUBO — coupled (constraint-aggregated) | 100 orders | $7,745,010 | 91.49% |

Full results, methodology, and honest discussion of limitations are in the
`report/` folder — each file corresponds to one task in the challenge
brief.

## AI Tool Disclosure

This project was built with substantial assistance from Claude (Anthropic),
used for: code generation and debugging (data cleaning, PuLP/dimod model
construction), mathematical formulation review, and drafting/editing of
this documentation. All code was run and verified by the author, and all
technical decisions (formulation choices, data assumptions, constraint
aggregation approach) were made and validated through iterative testing
documented in the notebook and report files. The author can explain and
defend all submitted work.

## Data & Privacy

This project uses only the anonymized challenge-approved Nestlé data pack,
which is not included in this repository (see "Data Access" above). No
raw operational data, customer identifiers, or confidential
distribution-center details are published here. Where external tools
(Claude, D-Wave Ocean SDK's local samplers) were used, only anonymized,
aggregated model coefficients were involved — no raw order-level data was
transmitted to any external service.

## Resources & Acknowledgments

- **Nestlé DOM Equations document** — Nestlé's own documented DOM
  proof-of-concept formulation, provided as part of the challenge data
  pack, used as a reference point for the MILP formulation in this project
  (see `report/03_formulation.md`).
- **PuLP documentation** — https://coin-or.github.io/pulp/ — used for
  MILP model construction (variables, constraints, objective).
- **Google OR-Tools documentation** — https://developers.google.com/optimization
  — referenced for assignment-problem formulation patterns.
- **Cornuéjols & Trick, "A Tutorial on Integer Programming"** — used as
  background reading on integer programming fundamentals (assignment
  problems, set covering).
- **D-Wave Ocean SDK / dimod documentation** — https://docs.ocean.dwavesys.com/
  — used for QUBO/BQM/CQM construction, `cqm_to_bqm` conversion, and
  Simulated Annealing / Tabu Search samplers.
- **Claude (Anthropic)** — used throughout for code generation, debugging
  assistance, and documentation drafting (see AI Tool Disclosure above).

All external resources were used for learning and implementation guidance;
all code was written, tested, and understood by the author.

## Author

Sishir Katepalli — B.Tech Computer Science and Engineering, Amrita School
of Engineering, Chennai

## Demo Video

A ~5-minute walkthrough of the project, covering the problem, the three
solution approaches compared, the full-scale MILP result, and the QUBO
scaling limitation we found and partially fixed via date-decomposition.

📺 [[Watch the demo]](https://drive.google.com/file/d/1JiIpjZomeQyepyelyJefOTLXmU1rA6Ep/view?usp=sharing)
