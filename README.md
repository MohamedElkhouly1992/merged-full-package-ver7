# HVAC ROM-Degradation Suite — Journal-Hourly Version

A deployable Streamlit research software package for reduced-order HVAC energy modelling with degradation, maintenance strategies, climate scenarios, validation sheets, early sensitivity ranking, robustness analysis, reporting, CatBoost surrogate modelling, and native hourly/sub-daily weather processing.

## Main design principle

`hvac_v3_engine.py` remains the numerical authority. The Streamlit interface collects inputs, calls the engine, and post-processes outputs. The UI does not duplicate the HVAC energy, degradation, COP, fan, pump, auxiliary, maintenance, or KPI equations.

## What is included

- `hvac_v3_engine.py` — core reduced-order HVAC degradation engine
- `streamlit_app.py` — Streamlit user interface
- `report_addons.py` — upload handling, validation, detailed sheets, zone tables, ZIP export
- `run_example.py` — fast local smoke test
- `requirements.txt` — dependencies
- `examples/sample_daily_weather.csv` — sample daily weather file
- `examples/sample_hourly_weather.csv` — generated during testing/example use when needed
- `docs/flowchart.png` and `docs/flowchart.svg` — journal-ready flowchart


## Pump and auxiliary energy inclusion

This modified version includes pump and auxiliary electrical energy in the total HVAC energy balance. The engine now calculates:

```text
total energy = thermal HVAC/compressor-equivalent energy
             + fan energy
             + pump energy
             + auxiliary energy
```

Pump power uses an area-normalized input `PUMP_SPECIFIC_W_M2` and is scaled by operating/occupancy factor and degradation:

```text
P_pump = Area × Pump specific power / 1000 × operating factor × (1 + 0.30 × degradation index)
```

Auxiliary power uses `AUXILIARY_W_M2` and represents controls, standby loads, valves, small motors, and other HVAC auxiliary equipment:

```text
P_aux = Area × Auxiliary power density / 1000 × operating factor
```

Both are controlled in the **Parameter Switches** tab using:

- HVAC pump energy
- HVAC auxiliary energy

They are also configured in **Building Identity & Setup → HVAC sizing and component** through:

- Pump specific power (W/m²)
- Auxiliary power density (W/m²)

The detailed output includes component columns such as `thermal_hvac_kwh_period`, `fan_kwh_period`, `pump_kwh_period`, and `auxiliary_kwh_period`.

## New journal-strength feature: native hourly/sub-daily weather

The earlier time-step selector scaled daily weather to sub-daily periods. This version upgrades that behavior:

- Uploaded or path-based EPW files are parsed as hourly weather.
- Timestamped CSV weather files are preserved as timestamped records.
- Hourly weather can be resampled to Hourly, 3-hour, 6-hour, 12-hour, or Daily simulation steps.
- Daily weather files still work; they are expanded to the selected time-step with a transparent diurnal profile.
- The output `weather_timeseries.csv` records the final time-step weather actually used by the engine.

The setup tab includes a selector for:

- Daily, 24 h
- 12-hour
- 6-hour
- 3-hour
- Hourly

The original daily model is recovered when Daily is selected.

For sub-daily modes, the engine computes each period using the selected period's weather record:

```text
energy per period = HVAC power × selected period hours
CO2 per period = energy × emission factor
cost per period = energy × price + maintenance cost
dust/fouling/degradation growth = daily rate × time-step/24
```

## Hourly occupancy schedule

For sub-daily simulation, the annual/semester occupancy factor is multiplied by an intra-day educational occupancy profile:

- low occupancy overnight
- rising occupancy from morning
- peak occupancy around midday/afternoon
- reduced occupancy after evening
- lower weekend factor

This makes hourly and sub-daily runs more realistic than simply repeating the same daily occupancy value.

## Solar handling

For native sub-daily weather, the model uses the raw/resampled GHI profile from the weather data. To avoid double seasonal weighting, the extra annual solar-season multiplier is disabled in sub-daily mode and retained only for daily reduced-order mode.

## Early benchmark sensitivity analysis

The **Sensitivity & Robustness** tab runs a fast one-at-a-time screening analysis before or beside the full scenario matrix.

For each selected input parameter, the model runs:

- baseline case
- low case: parameter × (1 - perturbation)
- high case: parameter × (1 + perturbation)

The ranking metric is central elasticity:

```text
elasticity = (% KPI change) / (% input change)
```

KPIs evaluated:

- Total Energy MWh
- Total CO2 tonne
- Mean Degradation Index
- Mean Comfort Deviation C
- Total Cost USD

Outputs:

- `early_sensitivity_ranking.csv`
- `early_sensitivity_details.csv`
- `sensitivity_base_summary.csv`
- `figures/early_sensitivity_ranking.png`
- `early_sensitivity_metadata.json`

## Robustness analysis

The robustness tool performs bounded Monte-Carlo perturbation of selected inputs and repeats the selected scenario.

Outputs:

- `robustness_samples.csv`
- `robustness_summary.csv`
- `figures/robustness_kpi_boxplot.png`
- `robustness_metadata.json`

The summary includes:

- mean
- standard deviation
- coefficient of variation
- 5th percentile
- median
- 95th percentile
- minimum
- maximum

## Main UI structure

Tabs are ordered for research workflow:

1. Building Identity & Setup
2. Parameter Switches
3. Scenario Modeling
4. Sensitivity & Robustness
5. Extra UI Tools
6. KPI Charts
7. Surrogate Train / Predict
8. Exports
9. Guide

## Run locally

```bash
pip install -r requirements.txt
streamlit run streamlit_app.py
```

If Streamlit is not recognized:

```bash
python -m streamlit run streamlit_app.py
```

## Smoke test

```bash
python run_example.py
```

## Recommended journal workflow

1. Run Daily first to verify baseline logic.
2. Upload EPW or hourly CSV and run 6-hour or Hourly for the final analysis.
3. Run early sensitivity on 1–3 analysis years to identify dominant parameters.
4. Run robustness analysis on the shortlisted parameters.
5. Run the final full scenario matrix.
6. Train the surrogate model on the final dataset.

## Honest limitation

This remains a reduced-order model, not a full heat-balance simulator like EnergyPlus. The hourly upgrade improves weather and occupancy temporal resolution, but the thermal-load equations are still reduced-order equations designed for fast scenario, degradation, maintenance, and surrogate-model research.

---

## Publication Plus upgrade

This package version adds separate publication-strength diagnostic tabs while keeping `hvac_v3_engine.py` as the main calculation authority.

### New tabs

1. **Model Validation**
   - Upload measured, EnergyPlus, DesignBuilder, or published reference CSV data.
   - Calculate MBE, NMBE, RMSE, CVRMSE, MAE, MAPE, and R².

2. **Heat Exchanger Diagnostics**
   - Calculates air-side and water-side pressure drops.
   - Estimates air and water inlet/outlet temperatures.
   - Calculates degraded UA, LMTD, effectiveness ratio, and detailed pump power implication.
   - Exports `heat_exchanger_diagnostics.csv`.

3. **Part-Load COP Curves**
   - Adds linear, quadratic, and cubic part-load-ratio COP correction curves.
   - Exports `part_load_cop_analysis.csv`.

4. **Latent Cooling Load**
   - Estimates humidity-ratio-based latent cooling load from outdoor temperature/RH, indoor RH target, ventilation, and infiltration.
   - Exports `latent_cooling_analysis.csv`.

5. **Zone-Level Load Analysis**
   - Builds a reduced-order zone-level load/energy table using zone area, occupancy density, and time-step results.
   - Exports `native_zone_loads.csv`.

6. **Global Sensitivity**
   - Reads robustness samples and computes Pearson/Spearman input-KPI screening indices.
   - Exports `global_sensitivity_screening.csv`.

7. **Advanced Plot Studio**
   - Supports line, scatter, bar, heatmap, multi-axis line, and combined line+bar charts.
   - Can plot any CSV output or uploaded CSV.
   - Can download interactive HTML charts.

### Deployment

```bash
pip install -r requirements.txt
streamlit run streamlit_app.py
```

For heavy hourly simulations, use a local workstation, Render, Docker/VPS, or university server rather than free notebook runtimes.

## EMS, Operation Scheduling, and Multi-Objective Optimization Upgrade

This version adds three new publication-oriented tabs:

### 1. EMS Control Strategies
Adds transparent Energy Management System overlays that are applied inside the engine simulation loop, not as separate UI-only post-processing. Supported EMS controls include:

- occupancy-based setpoint and airflow reset
- night setback
- demand-response event periods
- economizer / free-cooling logic
- optimum start / pre-cooling
- smart hybrid EMS
- custom scheduled EMS from the Operation Scheduling tab

EMS actions are exported in the time-step dataset using columns such as `ems_active`, `ems_occ_control`, `ems_night_setback`, `ems_demand_response`, `ems_economizer`, `ems_custom_schedule`, and `ems_optimum_start`.

### 2. Schedule of Operations
Provides an editable schedule table with weekday/weekend windows, occupancy multipliers, setpoint shifts, airflow factors, and demand-response flags. The schedule can be saved as `operation_schedule.csv` and can be passed directly into the engine run when custom scheduled EMS is enabled.

### 3. Multi-Objective Optimization
Screens EMS/control candidates against multiple KPIs:

- total energy
- mean degradation index
- mean comfort deviation
- total carbon emissions

Built-in options include weighted random search, grid search, and NSGA-II-style screening labels. The optimizer tab exports:

- `multi_objective_candidates.csv`
- `multi_objective_pareto.csv`
- `multi_objective_metadata.json`

This module is intended for reproducible publication screening and Pareto-style comparison. Heavy external optimizers can be added later by connecting them to the same candidate-evaluation function.
