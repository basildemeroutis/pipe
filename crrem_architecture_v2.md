# PIPE CRREM: Revised Architecture — Declining Building Line

## The Core Correction

The building line is **not flat**. A "do nothing" building still gets cleaner over time because the UK electricity grid is decarbonising. The CRREM methodology models this explicitly — the Back-end of the CRREM V2.07 tool contains year-by-year UK grid emission factors declining from **0.233 kgCO₂e/kWh (2020) to 0.007 (2050)**.

A flat line massively overstates stranding risk for electrically-heated buildings and understates it for gas-dependent ones.

---

## What This Changes

### Before (Flat Line — Wrong)
```
Building CO₂ = epc_co2_emissions (constant, every year 2024-2050)
```

### After (Grid-Adjusted Decline — Correct)
```
Building CO₂(year) = (elec_share × EUI × grid_EF(year)) + (gas_share × EUI × gas_EF)
```

Where:
- **EUI** = total energy use intensity (kWh/m²/yr), held constant ("do nothing")
- **elec_share** / **gas_share** = energy mix split for the building type (from defaults)
- **grid_EF(year)** = UK grid emission factor for that year (declining)
- **gas_EF** = natural gas emission factor (constant: 0.18316 kgCO₂e/kWh)

The EUI is back-calculated from the EPC CO₂ figure:
```
EUI = epc_co2_emissions / weighted_EF(baseline_year)
weighted_EF = (elec_share × grid_EF(baseline)) + (gas_share × gas_EF)
```

---

## Impact by Building Type

| Type                | Elec % | Gas %  | 2024  | 2030  | 2040  | 2050  | Decline |
|---------------------|--------|--------|-------|-------|-------|-------|---------|
| Office              | 44%    | 56%    | 52.3  | 38.9  | 34.0  | 32.8  | -37%    |
| Retail High Street  | 100%   | 0%     | 77.0  | 26.7  | 7.9   | 3.6   | -95%    |
| Retail Warehouse    | 29%    | 71%    | 70.8  | 59.2  | 54.9  | 53.9  | -24%    |
| Hotel               | 33%    | 67%    | 55.0  | 44.7  | 40.8  | 39.9  | -27%    |
| Industrial W/house  | 20%    | 80%    | 45.0  | 40.0  | 38.2  | 37.8  | -16%    |

An all-electric high street retail unit de-strands almost entirely from grid decarbonisation alone. A gas-heavy warehouse barely moves. **The energy mix is the decisive variable.**

---

## Three New Reference Data Files (for GitHub repo)

### 1. `uk_grid_emission_factors.csv`
UK electricity grid emission factors 2020–2050 (kgCO₂e/kWh).
Source: CRREM V2.07 Risk Assessment Tool, Back-end sheet, Row 53.
31 rows. Excludes T&D losses (aligned with SBTi).

### 2. `fuel_emission_factors.csv`
Static fuel emission factors (kgCO₂e/kWh).
Source: CRREM V2.07.
- Natural gas: 0.18316
- Fuel oil: 0.24677
- District heating: 0.20431
- Coal: 0.34473
- LPG: 0.21419

### 3. `uk_energy_mix_defaults.csv`
Default electricity/gas energy split by CRREM building type.
Source: CIBSE TM46:2008 energy benchmarks, mapped to CRREM types.
11 rows (one per CRREM building type).

These are the "hard-coded numbers in a CSV" — reference data that doesn't change per session. Val loads them once and applies the heuristics.

---

## Revised Calculation Flow

```
pipe-crrem Val receives trigger from GPT
    │
    ├── 1. Read from pipe_fields (canonical):
    │       - epc_co2_emissions (kgCO₂/m²/yr)
    │       - use_class (UK planning class)
    │       - gia_sqm
    │
    ├── 2. Deterministic mapping:
    │       use_class → crrem_type (lookup table in Val code)
    │
    ├── 3. Load reference data (from GitHub CSVs or hardcoded in Val):
    │       - CRREM pathways (crrem_uk_pathways_1.5C.csv — existing)
    │       - UK grid EFs (uk_grid_emission_factors.csv — NEW)
    │       - Fuel EFs (fuel_emission_factors.csv — NEW)
    │       - Energy mix defaults (uk_energy_mix_defaults.csv — NEW)
    │
    ├── 4. Back-calculate EUI:
    │       weighted_ef = (elec_share × grid_ef_2024) + (gas_share × 0.18316)
    │       eui = epc_co2_emissions / weighted_ef
    │
    ├── 5. Project "do nothing" building line (2024-2050):
    │       For each year:
    │         building_co2[year] = (elec_share × eui × grid_ef[year])
    │                            + (gas_share × eui × gas_ef)
    │
    ├── 6. Calculate stranding years:
    │       strand_1_5 = first year where building_co2[year] > pathway_1_5[year]
    │       strand_2_0 = first year where building_co2[year] > pathway_2_0[year]
    │       (NOTE: building may ALREADY be stranded if building_co2[2024] > pathway[2024])
    │       (NOTE: building may UN-strand if its decline crosses back below pathway)
    │
    ├── 7. Calculate excess emissions:
    │       For each year where building_co2[year] > pathway[year]:
    │         excess += (building_co2[year] - pathway[year]) × gia_sqm / 1000
    │       (Cumulative overshoot in tonnes CO₂)
    │
    ├── 8. Calculate Carbon Value at Risk:
    │       VAR = excess_tonnes × £75/tCO₂ (blended UK ETS)
    │
    ├── 9. Call Flask /crrem-chart with:
    │       - building_line[] (declining array, 2024-2050)
    │       - pathway_1_5[] 
    │       - pathway_2_0[]
    │       - strand_years
    │       - building_type label
    │       → receives base64 PNG
    │
    ├── 10. Store to SQL:
    │       pipe_fields: strand_year, delta, VAR, crrem_type
    │       pipe_images: crrem_chart (base64)
    │
    ├── 11. Upload to Dropbox:
    │       /PIPE_Charts/{session_id}/crrem.png → share link
    │
    └── 12. Return to GPT:
            { chart_url, strand_year_1_5, strand_year_2_0, delta, var, crrem_type }
```

---

## Chart Specification (Updated)

Three lines, **but the building line now curves downward**:

1. **Building "Do Nothing" line** — starts at epc_co2, declines based on grid decarbonisation
   - Color: #C62828 (FORE red), 2.5pt, **dashed** (to signal it's a projection, not measured)
2. **CRREM 1.5°C pathway** — from crrem_uk_pathways
   - Color: #2E7D32 (FORE green), 2.5pt, solid
3. **CRREM 2.0°C pathway** — from crrem_uk_pathways
   - Color: #F57F17 (amber), 2.0pt, solid

Stranding zone: shaded red (15% opacity) between building line and pathway where building > pathway.

**New chart element**: A horizontal dotted grey line at the EPC value showing what the "flat line" assumption would have been — makes the grid decarbonisation benefit visually obvious.

---

## Critical Nuances

### 1. "Already Stranded" Detection
If `building_co2[2024] > pathway[2024]`, the building is **already stranded**. The stranding year should be reported as "Already stranded (2024)" not a future year. The previous GPT-generated report showed 2029 for a building that was already stranded — this was the confabulation that triggered the whole redesign.

### 2. Possible De-Stranding
With a declining building line, there's a scenario where a building is currently above the pathway but its declining line eventually crosses back below. This means it "un-strands." The chart should show this — the shaded excess zone ends where the building line drops below the pathway. The narrative should flag this: *"While currently exceeding the 1.5°C pathway, projected grid decarbonisation suggests the building may re-align by [year] without physical intervention."*

### 3. Energy Mix Uncertainty
The default energy mix is the weakest link. A building assumed to be 44% electric (office default) might actually be 80% electric (if it has a heat pump) or 90% gas (if it's an old gas-heated building with minimal electrical load). The report must caveat: *"Energy mix assumed from CIBSE TM46 typical benchmarks for [building type]. Actual energy mix may differ — a building-specific assessment would refine this projection."*

### 4. Baseline Year for Grid EF
The EPC's CO₂ figure was calculated using SAP methodology with a specific grid EF at the time of assessment. We use 2024 as the baseline year for the grid EF in the back-calculation. This introduces a small error if the EPC was assessed in a different year — acceptable for screening, flagged in caveats.

### 5. Alignment with CRREM Methodology
From the methodology document (Section 5): CRREM uses **location-based emission factors** and **whole-building approach** (tenant + landlord). EPCs are broadly consistent with this. Our approach is CRREM-aligned but simplified — we use default energy mixes rather than measured consumption data. This should be labelled: *"CRREM-aligned screening assessment using EPC data and default energy mix assumptions."*

---

## Report Caveats (Mandatory Text)

> This analysis uses a CRREM-aligned screening methodology. The building's future carbon trajectory assumes constant energy consumption with declining grid emission factors (UK CRREM V2.07 projections). Energy mix is estimated from CIBSE TM46 typical benchmarks for the building type. A full CRREM assessment would require measured energy consumption data broken down by source. This projection does not account for planned retrofit interventions, occupancy changes, or climate-adjusted heating/cooling demand.

---

## What's NOT in Your Repo Yet

Your repo has:
- ✅ `crrem_uk_pathways_1.5C.csv` — target pathway data (CO₂ and kWh by building type)

Your repo needs:
- ❌ `uk_grid_emission_factors.csv` — extracted from CRREM V2.07 Back-end (31 rows)
- ❌ `fuel_emission_factors.csv` — static fuel EFs (5 rows)
- ❌ `uk_energy_mix_defaults.csv` — CIBSE TM46 mapped to CRREM types (11 rows)

All three CSVs are ready — created during this session.

---

## Deliberate Simplifications (Documented)

| Full CRREM | PIPE Implementation | Impact |
|---|---|---|
| Measured energy consumption by source | EPC CO₂ + default energy mix | Medium — energy mix is estimated |
| Weather normalisation (HDD/CDD) | Not applied | Low — UK climate relatively stable |
| Occupancy correction | Not applied | Low — screening context |
| Climate change projections (RCP4.5) | Not applied | Low — minor effect to 2050 |
| F-gas emissions | Not included | Low-Medium — significant for retail with refrigeration |
| District heating EF projection | Static default | Low — minimal DH in UK commercial |
| Energy source decomposition per building | Type-level defaults | Medium — see energy mix caveat |
