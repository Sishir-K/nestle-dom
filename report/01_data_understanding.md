# Data Understanding

This document describes the following for each file in the anonymized Nestlé DOM data pack: 
its grain,key columns, and how it maps to the mathematical formulation.

# Purely taken from the Example.xlsx file given

## 1. input_order_data.csv
- **Grain:** One row per order-SKU line (order × SKU)
- **Rows:** 25,193 | **Unique orders:** 1,109 (avg ~22.7 SKUs/order)
- **Key columns:** `Group_Flag` (order ID), `MaterialNumber` (SKU ID)
- **Other identifiers:** `Plant` (default DC), `LoadNumber` (shipment grouping —
  multiple orders can share a LoadNumber, so it is NOT a unique order key)
- **Model parameters covered:** ordered quantity (`OrderedQty_converted`), 
  order revenue (`Order_SKU_Revenue`), penalty rate (`Penaltyforpotentialcuts`),
  fill-rate threshold (`FillRateThreshold`), delivery zip (`ZipCode`)

## 2. input_capacity_planning.csv
- **Grain:** One row per DC-SKU-date
- **Key columns:** `LocationID` (DC), `MaterialID` (SKU), `DATE`
- **Note:** column names differ from input_order_data.csv (`LocationID` vs `Plant`,
  `MaterialID` vs `MaterialNumber`) — must rename before joining
- **Model parameters covered:** available inventory (`Available_inventory`),
  opening/closing stock, demand figures

## 3. input_dock_capacity.csv
- **Grain:** One row per DC-date
- **Key columns:** `Plant`, `Date`
- **Model parameters covered:** dock capacity (`Dock_Capacity`, `Dock_Remaining`,
  `InboundAppointments`)

## 4. input_shipping_cost_data.csv
- **Grain:** One row per DC-origin/destination zip lane
- **Key columns:** `Plant`, `OrigZip3`, `TargetZip`
- **Model parameters covered:** shipping cost per lane (`Shipping_Cost`)
- **Join note:** links to orders via the order's `ZipCode` field (matched against
  `TargetZip`)

## 5. input_throughput_capacity.csv
- **Grain:** One row per DC-date
- **Key columns:** `Plant`, `transportationplanningdate`
- **Model parameters covered:** case-pick / pallet-pick throughput
  (`util_case_picks`, `util_pallets`)
