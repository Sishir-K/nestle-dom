# Data Understanding

This document describes the following for each file in the anonymized Nestlé
DOM data pack: its grain, key columns, and how it maps to the mathematical
formulation. *(Column definitions cross-referenced against the challenge's
data dictionary.)*

For each file, **"Used in current model"** lists only the columns that
actually appear as parameters in `03_formulation.md`.
**"Present but not used"** lists columns that exist in the data pack but were
deliberately excluded — either because the model's scope doesn't need them,
or (for case-pick/pallet-pick throughput and the two-tier penalty structure)
because the data pack lacks the granularity to support them, as noted in the
Math Formulation doc's Overview.

## 1. input_order_data.csv
- **Grain:** One row per order-SKU line (order × SKU)
- **Rows:** 25,193 | **Unique orders:** 1,109 (avg ~22.7 SKUs/order)
- **Key columns:** `Group_Flag` (order ID), `MaterialNumber` (SKU ID)
- **Other identifiers:** `Plant` (default DC), `LoadNumber` (shipment grouping —
  multiple orders can share a LoadNumber, so it is NOT a unique order key)
- **Used in current model:** ordered quantity (`OrderedQty_converted`),
  order revenue (`Order_SKU_Revenue`), penalty rate
  (`Penaltyforpotentialcuts`), delivery zip (`ZipCode`)
- **Present but not used:** `FillRateThreshold` — not referenced anywhere in
  the current MILP/QUBO formulations; flagged here in case a future
  extension incorporates a fill-rate constraint or objective term

## 2. input_capacity_planning.csv
- **Grain:** One row per DC-SKU-date
- **Key columns:** `LocationID` (DC), `MaterialID` (SKU), `DATE`
- **Note:** column names differ from input_order_data.csv (`LocationID` vs
  `Plant`, `MaterialID` vs `MaterialNumber`) — must rename before joining
- **Used in current model:** available inventory (`Available_inventory`)
- **Present but not used:** opening/closing stock and other demand figures —
  present in the file but not referenced in the current formulation, which
  only needs the point-in-time available inventory figure

## 3. input_dock_capacity.csv
- **Grain:** One row per DC-date
- **Key columns:** `Plant`, `Date`
- **Used in current model:** dock capacity (`Dock_Capacity`)
- **Present but not used:** `Dock_Remaining`, `InboundAppointments` — not
  referenced in the current formulation, which models dock capacity as a
  single per-(DC, date) ceiling rather than tracking real-time remaining
  capacity or scheduled appointments

## 4. input_shipping_cost_data.csv
- **Grain:** One row per DC-origin/destination zip lane
- **Key columns:** `Plant`, `OrigZip3`, `TargetZip`
- **Used in current model:** shipping cost per lane (`Shipping_Cost`)
- **Join note:** links to orders via the order's `ZipCode` field (matched
  against `TargetZip`)

## 5. input_throughput_capacity.csv
- **Grain:** One row per DC-date
- **Key columns:** `Plant`, `transportationplanningdate`
- **Present but not used:** case-pick / pallet-pick throughput
  (`util_case_picks`, `util_pallets`) — **explicitly excluded from the
  current formulation.** As noted in `01_mathematical_formulation.md`, the
  model omits case-pick/pallet-pick logistics because the data pack does
  not provide the granularity needed to support them. This file is
  documented here for completeness and as a candidate extension, not
  because it feeds into the current MILP or QUBO objective/constraints.