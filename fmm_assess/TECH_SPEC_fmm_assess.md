# `fmm_assess`: FAIRMODE Modelling-Assessment Evaluation Tool (Python)

**Technical Specification — v1.1**
**Target environment:** Python 3.10+, Spyder on a desktop workstation.
**Status:** specification ready for implementation.

**Changelog from v1.0:**
- Full coverage of every indicator defined in JRC146736 §3.3, §3.4,
  §3.7.1, §3.7.2 **and §3.7.3** (was: §3.3, §3.4, §3.7.1, §3.7.2 only).
- Adopted the `mqor` β convention: the stringency factor in every
  denominator is **`sqrt(1 + β²)`**, applied uniformly.

---

## 1. Purpose and scope

`fmm_assess` is a Python port (and trimming) of the FAIRMODE
**DeltaToolLight** (DTL) IDL code, restricted to the **assessment**
workflow defined in:

- **JRC146736** — Thunis, Tarrason, Pisoni, Janssen, 2026. *FAIRMODE
  Guidance Document on Modelling Quality Objectives and Benchmarking*
  (hereafter *the Guidance Document*, v4.0 / 2026-05-06).
- **DTL User Manual v1.4** (Cuvelier, 2025-01-24) — bundled `DTLmanual.pdf`.
- The **`mqor` R package** (`mqor-main.zip`) — the reference numerical
  implementation. *Where the IDL DTL and `mqor` disagree on a formula,
  `mqor` wins.*
- **Directive (EU) 2024/2881** — revised Ambient Air Quality Directive.

It computes, per modelling application (one year, one pollutant, one
model, one domain), **every indicator listed in the Guidance Document**:

| Group              | Indicators                                                                   | Spec § |
|--------------------|------------------------------------------------------------------------------|--------|
| Modelling Quality  | `MQI_short`, `MQI_long`                                                      | §7.2   |
| Temporal MPIs      | `TI_B`, `TI_R`, `TI_σ`                                                       | §7.4   |
| Spatial MPIs       | `SI_RMSE`, `SI_B`, `SI_R`, `SI_σ`                                            | §7.5   |
| Spatial gradients  | `MPI_UB-RB`, `MPI_UT-UB`                                                     | §7.6   |
| Temporal coherence | `MPI_seasonal`, `MPI_week`, `MPI_day` — each × {industry, traffic, background} | §7.7 |

with the associated MQO / MPC pass-fail flags at the 90th-percentile.

It uses the AAQD measurement-uncertainty curve:

- `uncertainty_def: aaqd` — Directive 2024/2881 piecewise function
  (Guidance Document §3.3, Table 1).

The β stringency multiplier is applied as `sqrt(1 + β²)` in every
indicator denominator.

It also produces eight diagnostic diagrams (radar, target, temporal–
spatial report, scatter, Taylor, scatter–dynamic-evaluation, bar, time
series).

### Explicitly out of scope

- **Forecast mode** — already covered by `fmf_eval`.
- The DTL "composite mapping" platform (NUTS-level inter-model comparison).
- IDL-style interactive sliders (`Beta_Sliders` in DTL §10). The Python
  equivalent is re-running with a different config or override CSV.

---

## 2. Package layout

```
fmm_assess/
├── __init__.py
├── cli.py                       # F5-runnable entry point for Spyder
├── config.py                    # YAML run config (pydantic v2)
├── uncertainty.py               # U(O): AAQD piecewise (§5)
├── stats_core.py                # bias, rmse, crmse, corr, sd, rmsu (paired)
├── indicators.py                # ALL §3 indicators, mqor-style sqrt(1+β²)
├── aggregations.py              # hourly→required temporal aggregation
├── io_startup.py                # DeltaTool startup.ini reader
├── io_obs.py                    # per-station observation CSV reader
├── io_model.py                  # NetCDF / per-station CSV model reader
├── filters.py                   # 75 %-rule, station-type/area filtering
├── pipeline.py                  # end-to-end orchestration
├── plots/
│   ├── __init__.py
│   ├── radar.py
│   ├── target.py
│   ├── ts_report.py             # Temporal–Spatial bar report
│   ├── scatter.py
│   ├── taylor.py
│   ├── scatter_dyneval.py
│   ├── bars.py
│   └── timeseries.py
├── data/
│   └── uncertainty_aaqd.csv     # Table 1 of the Guidance Document
└── tests/
    ├── test_uncertainty.py
    ├── test_stats_core.py
    ├── test_indicators.py
    ├── test_aggregations.py
    └── test_end_to_end.py
```

---

## 3. Dependencies

```bash
pip install numpy pandas scipy matplotlib netCDF4 xarray PyYAML pydantic
```

Versions: `numpy >= 1.24`, `pandas >= 2.0`, `matplotlib >= 3.7`, `pydantic >= 2.0`.
All available in conda-forge / stock Anaconda → Spyder picks them up.

---

## 4. Inputs

### 4.1 `startup.ini`

Identical to the DeltaTool / `fmf_eval` format; parser reused verbatim
from `fmf_eval.io_startup`. Three sections, semicolon-separated.
`[MONITORING]` rows have 12 mandatory fields plus 1 optional field:
`Station Code; Station Name; Abbrev; Altitude; Lon; Lat; GMTLag;
Region; Station Type; Area Type; Siting; listOfvariables`
`[; MeasurementType]`.

Station-Type values are normalised to lower-case
(`background` | `traffic` | `industrial`). Area-Type values likewise
(`urban` | `suburban` | `rural`). Only stations whose `listOfvariables`
contains the active pollutant are loaded.

The optional 13th column `MeasurementType` is `fixed` or `indicative`
(case-insensitive). It records this station's measurement category
per AAQD Annex VI / FAIRMODE Guidance §3.3 and drives per-station
dispatch through the uncertainty params CSV (see §5 — `fixed` and
`indicative` are separate rows with their own `u_rv`, `u_phi`, and
`β`). When the column is absent for a row, or is blank, or holds an
unrecognised value, the run-level default `RunConfig.measurement_type`
(YAML key `measurement_type`) is applied to that station. This makes
legacy 12-column startup files behave identically to the previous
single-type behaviour: every station inherits the YAML default, the
all-fixed code path runs unchanged, and the pipeline output is
bit-identical to the pre-mixed-network releases.

### 4.2 Observations

Two equivalent input formats, auto-detected by extension (same as for
model results — see §4.3):

- **Directory of per-station CSVs** under `obs_dir/<StationName>.csv`.
  Header: `year;month;day;hour;<pollutant>;...;`. Missing value `-999`.
- **NetCDF** (`.cdf` / `.nc` / `.nc4`), DeltaTool layout: one variable
  per station, dims `(T, V)` with `T = 8760` (or `8784` leap year).

Times in CSV inputs are local; the GMT lag from `startup.ini` is
applied at load to convert to UTC and re-index onto a continuous hourly
UTC timeline for the calendar year of the run. NetCDF inputs are
assumed already on the UTC hourly grid (matching DeltaTool convention).

### 4.3 Model results

Two equivalent input formats, auto-detected by extension:

- **NetCDF**, DeltaTool layout: `<YEAR>_<MODEL>_TIME.cdf`, one variable
  per station, dims `(T, V)` with `T = 8760` (or `8784` leap year).
- **Directory of per-station CSVs** with the same layout as observations.

Any combination of obs and model formats is valid (CSV obs + NetCDF
model, NetCDF obs + CSV model, both CSV, both NetCDF); results are
bit-identical across formats.

### 4.4 `run.yaml` — single user-facing config

```yaml
run_id: milan_no2_2021
pollutant: NO2
year: 2021

# Which MQI(s) to compute. Always computes both unless restricted.
aggregations:
  - short          # hourly for NO2; daily for PM10/PM25; daily-8h-max for O3
  - long           # annual mean

# Uncertainty definition.
uncertainty_def: aaqd       # aaqd
measurement_type: fixed     # fixed | indicative — RUN-LEVEL DEFAULT applied
                            # to any station whose 13th [MONITORING] column is
                            # blank/absent. Stations that supply an explicit
                            # MeasurementType in startup.ini override this on
                            # a per-station basis (mixed networks).

# Filters
min_data_capture: 0.75      # 75 % rule on the active aggregation
min_stations: 10            # below this, the MQO 90th-percentile is
                            # unreliable (Guidance §5.2); logged as a warning
include_station_types: [background, traffic, industrial]
include_area_types:    [urban, suburban, rural]

# Optional override for the U(O) parameter table.
inputs:
  startup_ini: ./resource/startup.ini
  obs_dir:    ./data/monitoring/po_valley
  model_file: ./data/modeling/po_valley/2021_AENSa_TIME.cdf
  uncertainty: null         # null = use bundled defaults

outputs:
  dir: ./out/milan_no2_2021
  formats: [png, pdf]
  dpi: 200

timeseries:
  stations: [ES0041A, ES0094A]   # explicit list, or
  nuts:     []                   # NUTS prefix(es), mean over matching stns
  show_obs: true
```

`measurement_type` selects between the "fixed" and "indicative" rows of
the AAQD parameter table (which carry **different β** — see §5.2;
AAQD Annex V).

---

## 5. Measurement uncertainty `U(O)`

The formulation returns the **absolute** measurement uncertainty in
µg/m³ at the observation concentration `O` (or at the time-series mean
`Ō` for long-term). Uniform Python signature:

```python
def U_of_O(O: np.ndarray, params: UncertaintyParams) -> np.ndarray: ...
```

so that everything downstream (`rmsu`, every indicator) is curve-agnostic.

### 5.1 `uncertainty_def: aaqd` — piecewise (Guidance §3.3)

```
U_O,fix(O) = U_φ + O · (U_O,fix(LV) − U_φ) / LV     if O ≤ LV
U_O,fix(O) = O · U_O,fix(LV) / LV                   if O >  LV
```

with `U_O,fix(LV) = U_Ofix_LV_rel · LV`. Parameters live in
[`data/uncertainty_aaqd.csv`](../data/uncertainty_aaqd.csv) (Guidance
Document Table 1, plus β values for short and long aggregation; β is
**not** applied inside `U(O)` — it appears only in the indicator
denominator, §7.1).

**Full combination — every `(term, pollutant, time resolution, type)`
shipped in the bundled parameter table.** CSV columns: `term`,
`pollutant`, `time`, `type`, `rv_unit`, `rv` = `LV`, `u_rv` =
`U_Ofix_LV_rel`, `u_phi` = `U_φ`, `b` = `β`.

| term  | pollutant | time res.            | type       | LV   | unit   | U_Ofix_LV_rel | U_φ   | β     |
|-------|-----------|----------------------|------------|------|--------|---------------|-------|-------|
| short | NO2       | hourly               | fixed      | 200  | µg/m³  | 0.15          | 3     | 3.2   |
| short | NO2       | hourly               | indicative | 200  | µg/m³  | 0.25          | 3     | 1.92  |
| short | NO2       | daily                | fixed      | 50   | µg/m³  | 0.15          | 3     | 3.2   |
| short | NO2       | daily                | indicative | 50   | µg/m³  | 0.25          | 3     | 1.92  |
| long  | NO2       | annual               | fixed      | 20   | µg/m³  | 0.30          | 6.0   | 1.4   |
| long  | NO2       | annual               | indicative | 20   | µg/m³  | 0.40          | 8.0   | 1.05  |
| short | O3        | max daily 8hr mean   | fixed      | 120  | µg/m³  | 0.15          | 10    | 2.2   |
| short | O3        | max daily 8hr mean   | indicative | 120  | µg/m³  | 0.25          | 10    | 1.32  |
| long  | O3        | seasonal             | fixed      | 60   | µg/m³  | 0.15          | 9.0   | 1.7   |
| long  | O3        | seasonal             | indicative | 60   | µg/m³  | 0.25          | 15.0  | 1.02  |
| short | PM10      | daily                | fixed      | 45   | µg/m³  | 0.25          | 4     | 2.2   |
| short | PM10      | daily                | indicative | 45   | µg/m³  | 0.50          | 4     | 1.1   |
| long  | PM10      | annual               | fixed      | 20   | µg/m³  | 0.20          | 4.0   | 1.3   |
| long  | PM10      | annual               | indicative | 20   | µg/m³  | 0.30          | 6.0   | 0.87  |
| short | PM2.5     | daily                | fixed      | 25   | µg/m³  | 0.25          | 3     | 2.5   |
| short | PM2.5     | daily                | indicative | 25   | µg/m³  | 0.35          | 3     | 1.79  |
| long  | PM2.5     | annual               | fixed      | 10   | µg/m³  | 0.30          | 3.0   | 1.7   |
| long  | PM2.5     | annual               | indicative | 10   | µg/m³  | 0.40          | 4.0   | 1.28  |
| short | SO2       | hourly               | fixed      | 350  | µg/m³  | 0.15          | 5     | 3.2   |
| short | SO2       | hourly               | indicative | 350  | µg/m³  | 0.25          | 5     | 1.92  |
| short | SO2       | daily                | fixed      | 50   | µg/m³  | 0.15          | 5     | 3.2   |
| short | SO2       | daily                | indicative | 50   | µg/m³  | 0.25          | 5     | 1.92  |
| long  | SO2       | annual               | fixed      | 20   | µg/m³  | 0.30          | 6.0   | 1.4   |
| long  | SO2       | annual               | indicative | 20   | µg/m³  | 0.40          | 8.0   | 1.05  |
| short | CO        | daily                | fixed      | 4    | mg/m³  | 0.15          | 0.5   | 3.2   |
| short | CO        | daily                | indicative | 4    | mg/m³  | 0.25          | 0.5   | 1.92  |
| short | CO        | max daily 8hr mean   | fixed      | 10   | mg/m³  | 0.10          | 0.5   | 4.9   |
| short | CO        | max daily 8hr mean   | indicative | 10   | mg/m³  | 0.20          | 0.5   | 2.45  |
| long  | benzene   | annual               | fixed      | 3.4  | µg/m³  | 0.25          | 0.85  | 1.7   |
| long  | benzene   | annual               | indicative | 3.4  | µg/m³  | 0.35          | 1.19  | 1.21  |
| long  | lead      | annual               | fixed      | 0.5  | µg/m³  | 0.25          | 0.125 | 1.7   |
| long  | lead      | annual               | indicative | 0.5  | µg/m³  | 0.35          | 0.175 | —     |
| long  | arsenic   | annual               | fixed      | 6    | ng/m³  | 0.40          | 2.4   | 1.1   |
| long  | arsenic   | annual               | indicative | 6    | ng/m³  | 0.50          | 3.0   | —     |
| long  | cadmium   | annual               | fixed      | 5    | ng/m³  | 0.40          | 2.0   | 1.1   |
| long  | cadmium   | annual               | indicative | 5    | ng/m³  | 0.50          | 2.5   | —     |
| long  | nickel    | annual               | fixed      | 20   | ng/m³  | 0.40          | 8.0   | 1.1   |
| long  | nickel    | annual               | indicative | 20   | ng/m³  | 0.50          | 10.0  | —     |
| long  | B(a)P     | annual               | fixed      | 1    | ng/m³  | 0.50          | 0.5   | 1.1   |
| long  | B(a)P     | annual               | indicative | 1    | ng/m³  | 0.60          | 0.6   | —     |

For long-term rows the Guidance Document's simplification
`U_φ = U_O,fix(LV)` is encoded at load time (`U_φ = U_Ofix_LV_rel · LV`).
The short-term `U_φ` exceptions are stored literally. A dash (`—`) in
the β column means the AAQD does not prescribe a β value for that
(pollutant, type) pair; in that case `fmm_assess` does not compute an
MQI/MPI for it (`load_uncertainty` raises a clear error).

### 5.2 Root-mean-square uncertainty `RMSU_O`

For a time series of length N, aggregate point-wise:

`RMSU_O = sqrt( mean( U_of_O(O_t)² ) )`

For long-term, `RMSU_Ō = U_of_O(Ō)` (single value).

---

## 6. Aggregations (`aggregations.py`)

Hourly model and observation series are aggregated to the time resolution
required by the MQI for the active pollutant:

| pollutant | short-term                | long-term                |
|-----------|---------------------------|--------------------------|
| NO2       | hourly  (no aggregation)  | annual mean              |
| O3        | daily 8h-max              | seasonal mean (Apr–Sep)  |
| PM10      | daily mean                | annual mean              |
| PM25      | daily mean                | annual mean              |

### 6.1 Daily 8h-max (O3)

Per local-time day d, compute all overlapping 8-hour running means
starting at hours 17(d−1) through 16(d); take the maximum. A daily value
is valid if ≥ 18 of the 24 daily 8h means are valid; an 8h mean is
valid if ≥ 6 of its 8 hourly values are present. Per Annex IV §6 of
Directive 2024/2881.

### 6.2 Seasonal mean (long-term O3)

Reference period Apr 1 – Sep 30 inclusive. Used with the long-term
O3 row (RV=60, β=1.7).

### 6.3 75 % rule (`filters.rule_75`)

A station is dropped for the active aggregation if its valid-data
fraction is below `min_data_capture` (default 0.75). The fraction is
computed on the **paired** model+obs series of the active aggregation;
a NaN in either side disqualifies that timestep.

---

## 7. Indicators (`indicators.py`)

> **Master rule (mqor convention).** Every MQI / MPI denominator has the
> form `sqrt(1 + β²) · RMSU_O`. The factor `sqrt(1 + β²)` is *the*
> stringency adjustment.

All formulas below come from §3.3, §3.4, §3.7.1, §3.7.2 and §3.7.3 of
the Guidance Document, cross-checked against `mqor::statistics.R`.

Notation: `O`, `M` are aligned 1-D arrays of paired observation and
model values after aggregation and 75 %-rule filtering. `Ō`, `M̄` denote
means; `σ_O`, `σ_M` standard deviations; `R` Pearson correlation;
`RMSE`, `BIAS`, `CRMSE` the usual paired statistics.

### 7.1 The universal denominator helper `D`

```python
# Short-term, per station:
D_short = sqrt(1 + β_short²) · rmsu(O, params_short)

# Long-term, per station (single annual value Ō):
D_long  = sqrt(1 + β_long²)  · rmsu([Ō], params_long)
```

`rmsu(...)` computes `sqrt(mean(U_of_O(O_t)²))` over the paired
hourly/daily observation series (collapses to `U_of_O(Ō)` for the
long-term single-value case).

This `D` appears in every formula in §7.2–§7.7.

### 7.2 Modelling Quality Indicators — §3.3, Eqs. 1, 3

```
MQI_short(station) = RMSE(M, O) / D_short                     (Eq. 1)
MQI_long(station)  = |BIAS_annual|(M̄, Ō) / D_long             (Eq. 3)
```

This matches `mqor::vec_mqi()`.

### 7.3 MQO — §3.4, Eqs. 4, 5

```
MQO_short fulfilled  ⇔  percentile_90(MQI_short across stations) ≤ 1
MQO_long  fulfilled  ⇔  percentile_90(MQI_long  across stations) ≤ 1
```

The 90th-percentile uses linear interpolation per Eq. 5:
`N_90 = floor(0.9·N_s)`, `D_interp = 0.9·N_s − N_90`,
`MQI_90 = MQI[N_90] + D_interp · (MQI[N_90+1] − MQI[N_90])`,
applied on ascending-sorted finite values.

### 7.4 Temporal MPIs (`TI_*`) — §3.7.1, Table 2

Applied per station on the short-term aggregated time series.
`mqor` form (`vec_pi_bias2`, `vec_pi_cor2`, `vec_pi_sd2`):

```
TI_B(station)     = BIAS(M, O)                        / D_short
TI_R(station)     = sqrt( 2 · σ_O · σ_M · (1 − R) )   / D_short
TI_σ(station)     = (σ_M − σ_O)                       / D_short
```

MPCs (necessary, not sufficient): `|TI_B|_90 ≤ 1`, `|TI_R|_90 ≤ 1`,
`|TI_σ|_90 ≤ 1`. The square-root form of `TI_R` is exactly
`mqor::vec_pi_cor2()` — this is the implementation followed (per spec
priority: mqor wins over IDL DTL).

Together with Eq. 10 of the Guidance Document
`MQI² ≈ TI_B² + TI_R² + TI_σ²` (decomposition into bias, correlation,
standard-deviation error terms).

### 7.5 Spatial MPIs (`SI_*`) — §3.7.2, Table 3

First reduce each station to a single value by time-averaging
(annual mean). Then apply paired statistics across the **station
population** (the array index is the station, not time). This matches
`mqor::vec_si_rmse()` and the analogous `vec_pi_*` calls with `term =
"long"` and the station-mean vectors:

```
RMSE_sp  = RMSE(M̄_i, Ō_i)             across i ∈ stations
BIAS_sp  = mean_i(M̄_i) − mean_i(Ō_i)
σ_O,sp   = sd_i(Ō_i)
σ_M,sp   = sd_i(M̄_i)
R_sp     = cor(M̄_i, Ō_i)

D_long_sp = sqrt(1 + β_long²) · RMSU_Ō,sp
            with RMSU_Ō,sp = sqrt( mean_i( U_of_O(Ō_i)² ) )

SI_RMSE  = RMSE_sp                              / D_long_sp     (MPC: SI_RMSE ≤ 1)
SI_B     = BIAS_sp                              / D_long_sp     (MPC: |SI_B|   ≤ 1)
SI_R     = sqrt( 2 · σ_O,sp · σ_M,sp · (1−R_sp))/ D_long_sp     (MPC: SI_R    ≤ 1)
SI_σ     = (σ_M,sp − σ_O,sp)                    / D_long_sp     (MPC: |SI_σ|  ≤ 1)
```

Each SI is a single number per run (not one per station).

### 7.6 Spatial-gradient MPIs — §3.7.3, Table 4

Evaluate how well the model reproduces concentration *increments*
between station classes. For a pair of station classes (A, B):

```
INC_A−B(model)    = mean_i∈A(M̄_i) − mean_i∈B(M̄_i)
INC_A−B(observed) = mean_i∈A(Ō_i) − mean_i∈B(Ō_i)

RMSU_Ō,A   = sqrt( mean_i∈A( U_of_O(Ō_i)² ) )
RMSU_Ō,B   = sqrt( mean_i∈B( U_of_O(Ō_i)² ) )
D_gradient = sqrt(1 + β_long²) · 0.5 · (RMSU_Ō,A + RMSU_Ō,B)

MPI_A−B = | INC_A−B(model) − INC_A−B(observed) | / D_gradient   (MPC: ≤ 1)
```

Two specific pairs are evaluated:

- `MPI_UB-RB` — Urban Background minus Rural Background (Eq. 13).
- `MPI_UT-UB` — Urban Traffic minus Urban Background (Eq. 14).

Each returns NaN (with a logged warning) when fewer than 2 stations of
either class exist in the filtered set. Station classes are derived
from the `startup.ini` Station-Type × Area-Type combination:

- Urban Background (UB):   `station_type=background, area_type=urban`
- Rural Background (RB):   `station_type=background, area_type=rural`
- Urban Traffic (UT):      `station_type=traffic,    area_type=urban`

### 7.7 Temporal-coherence MPIs — §3.7.3, Table 5

Evaluate how well the model reproduces seasonal, weekly and daily
variability. Three sub-period contrasts:

- **Seasonal:** Winter (DJF) minus Summer (JJA), in local time.
- **Week / weekend:** Mon–Fri minus Sat–Sun.
- **Day / night:** 10:00–18:00 minus 22:00–06:00 local. Only meaningful
  when the active short-term aggregation is hourly (NO2). For daily
  pollutants (O3 d8h-max, PM10/PM25 daily), day/night returns NaN with
  a warning.

For each contrast `Δ_obs(station)`, `Δ_mod(station)` is computed per
station, then **averaged within each station-type class**:

```
Δ_class(obs) = mean over stations in class of Δ_obs
Δ_class(mod) = mean over stations in class of Δ_mod

RMSU_Ō,class = sqrt( mean_i_in_class( U_of_O(Ō_i)² ) )
D_class      = sqrt(1 + β_long²) · RMSU_Ō,class

MPI_class = | Δ_class(mod) − Δ_class(obs) | / D_class           (MPC: ≤ 1)
```

This produces a 3 × 3 grid of values:

|              | Industry        | Traffic        | Background        |
|--------------|-----------------|----------------|-------------------|
| Seasonal     | MPI_seas_I      | MPI_seas_T     | MPI_seas_B        |
| Week–WeekEnd | MPI_week_I      | MPI_week_T     | MPI_week_B        |
| Day–Night    | MPI_day_I*      | MPI_day_T*     | MPI_day_B*        |

\* NaN if pollutant aggregation is daily.

Classes derived from `Station Type` in `startup.ini`:
`industrial → Industry`, `traffic → Traffic`, `background → Background`.
A cell is NaN (with a logged warning) when its class contains fewer than
2 stations after filtering.

---

## 8. Pipeline (`pipeline.py`)

Single public entry point:

```python
def run_assessment(config_path: str | Path) -> Results: ...
```

Execution order:

1. Load and validate `run.yaml` → typed `RunConfig`.
2. Load `startup.ini` → station catalogue.
3. Apply station-type / area-type filters and `listOfvariables` filter.
4. Load hourly observations (UTC, full-year reindex, NaN where missing).
5. Load hourly model (same shape).
6. Compute short and long aggregations on both series.
7. Apply `rule_75` per station; below `min_stations` log a warning per
   Guidance §5.2.
8. Build `UncertaintyParams` for short and long aggregations.
9. `compute_per_station(...)`:
   - For each station compute every per-station value used by the
     indicators (the paired stats, denominators, MQI, TI and the
     dynamic-evaluation Δ quantities).
   - This includes the `Δ` quantities needed by §7.7.
10. `aggregate(...)`:
    - 90th-percentiles of MQI_short, MQI_long, TI_B, TI_R, TI_σ.
    - Spatial MPIs (§7.5) — one call each.
    - Spatial-gradient MPIs (§7.6) — needs station type/area classes.
    - Temporal-coherence MPIs (§7.7) — uses the per-station Δs.
11. Write CSVs (§11).
12. Render diagrams (§9). Close all figures at the end.
13. Return a `Results` dataclass containing both `per_station` and
    `aggregate` so the user can keep exploring in Spyder.

A per-run `run.log` captures every warning, in particular: dropped
stations (low capture), low N, NaN cells in the temporal-coherence
grid, MPI_UB-RB / MPI_UT-UB with too-few stations in a class.

---

## 9. Diagrams (`plots/`)

Eight plot modules, one per file, each exposing `plot_<name>(...) ->
matplotlib.figure.Figure`. The pipeline saves them in every requested
output format (`png`, `pdf`).

### 9.1 Radar plot (`plots/radar.py`) — DTL §RADAR PLOT

Polar plot with one axis per indicator. Order matches the DTL convention,
extended with every Guidance Document MPI:

`MQI_short, MQI_long, TI_B, TI_R, TI_σ, T(W-S), T(Wk-We), T(D-N),
B(W-S), B(Wk-We), B(D-N), I(W-S), I(Wk-We), I(D-N), SI_R, SI_σ,
MPI_UT-UB, MPI_UB-RB`

(`T(...)` = traffic-class temporal coherence MPI, `B(...)` = background-
class, `I(...)` = industry-class — DTL ordering.)

For temporal MPIs (TI_*), the radius is the 90th-percentile value.
For spatial / gradient / coherence MPIs (SI_*, MPI_*), the radius is the
single number from the run. Radius 1 = MQO/MPC-fulfilment boundary
(green disc). Polygon filled with the model colour.

Annotations: pollutant, year, station filter, station counts per class
(UT/UI/UB/RT/RI/RB/ST/SI/SB), `uncertainty_def`, β values for short
and long.

### 9.2 Target plot (`plots/target.py`) — §3.6.2, Fig. 3

One dot per station:

- y = `BIAS / D_short`
- x = `±CRMSE / D_short`, sign set by which error term dominates
  (Guidance Eq. 9): `σ`-dominated → right, `R`-dominated → left.

Grey / green disc of radius 1 = MQO-fulfilment zone. Annotations:
`MQI_short,90th` (pass/fail), `MQI_long,90th` (pass/fail), β,
`U_O,fix(LV)`, LV. Optional colouring by station type.

### 9.3 Temporal–Spatial Report (`plots/ts_report.py`) — DTL §TS REPORT

Horizontal-bar diagram. One bar per indicator:

- TI_B, TI_R, TI_σ: one dot per station; ≥ 15 stations → switch to a
  box-and-whisker showing 25/50/75 plus min/max. Vertical tick on each
  bar = 90th-percentile; rightmost column prints the value (green if
  MPC fulfilled, red otherwise). MQO-zone shaded green for `|·| ≤ 1`.
- SI_RMSE, SI_B, SI_R, SI_σ: a single dot.
- MPI_UB-RB, MPI_UT-UB: a single dot.
- Temporal-coherence MPIs (Table 5): a 3×3 mini-grid attached to the
  bottom of the diagram, one bar per (contrast × class). MPCs (`≤ 1`)
  shaded green.

### 9.4 Scatter plot (`plots/scatter.py`) — §3.6.1, Fig. 2

`Ō` (annual obs mean) vs `M̄` (annual model mean), one dot per station.
1:1 line solid. MQO-fulfilment wedge `|M̄ − Ō| ≤ D_long_per_station`
shaded grey. Annotations: `MQI_long,90th`, β, LV, `U_O,fix(LV)` or
`U_r(RV)`+`RV`.

### 9.5 Taylor diagram (`plots/taylor.py`) — DTL §TAYLOR PLOT

Quarter-circle polar:

- radial = `σ_M / σ_O` (normalised model SD),
- azimuthal = `arccos(R)`,
- iso-CRMSE arcs centred on `(1, 0)`.

One dot per station; reference at `(1, 0)`. `MQI_short,90th` and
`MQI_long,90th` annotated in the corners.

### 9.6 Scatter dynamic-evaluation (`plots/scatter_dyneval.py`) — DTL §SDE PLOT

2 × 2 panel of `Δ_mod` vs `Δ_obs` scatters:

- Top-left: Winter − Summer (`ΔWS_mod` vs `ΔWS_obs`).
- Top-right: WeekDays − WeekEnd (`ΔWkWe_mod` vs `ΔWkWe_obs`).
- Bottom-left: Day − Night (`ΔDN_mod` vs `ΔDN_obs`). Hidden when
  aggregation is daily.
- Bottom-right: legend + per-panel Pearson R across stations.

Each panel draws the 1:1 line. Dots optionally coloured by station
type (consistent with §7.7's grouping into Industry / Traffic /
Background).

### 9.7 Bar plot (`plots/bars.py`) — DTL §BAR PLOT

Vertical bars: observation means. Coloured dots overlaid: model means.
Grouped by (station-type × area-type) class (UT, UI, UB, RT, RI, RB,
ST, SI, SB; DTL order). Weighted means printed on the right.

### 9.8 Time series (`plots/timeseries.py`) — DTL §TIME SERIES

One panel per station listed in `cfg.timeseries.stations`. Hourly obs
in black, model in colour. Means annotated on the right. NUTS-level
aggregation supported when the startup file carries a NUTS column.

---

## 10. CLI / Spyder workflow

```python
# fmm_assess/cli.py
from pathlib import Path
from fmm_assess.pipeline import run_assessment

CONFIG_PATH = Path("./run.yaml")        # ← edit this path

if __name__ == "__main__":
    results = run_assessment(CONFIG_PATH)
    a = results.aggregate
    print(f"MQI_short(90th) = {a.MQI_short_p90:.3f} ({'PASS' if a.MQO_short_pass else 'FAIL'})")
    print(f"MQI_long (90th) = {a.MQI_long_p90:.3f}  ({'PASS' if a.MQO_long_pass  else 'FAIL'})")
    print(f"SI_RMSE = {a.SI_RMSE:.3f}  SI_B = {a.SI_B:.3f}  "
          f"SI_R = {a.SI_R:.3f}  SI_σ = {a.SI_sigma:.3f}")
    print(f"MPI_UB-RB = {a.MPI_UB_RB:.3f}   MPI_UT-UB = {a.MPI_UT_UB:.3f}")
    print("Temporal coherence MPIs:")
    print(a.MPI_coherence.round(3))
```

In Spyder: open `cli.py`, edit `CONFIG_PATH`, press **F5**. Figures
appear in the Plots pane and on disk under `outputs.dir`.

Equivalently from the IPython console:

```python
from fmm_assess.pipeline import run_assessment
res = run_assessment("./run.yaml")
res.per_station[0]
res.aggregate.MPI_coherence
```

---

## 11. Outputs

Files written to `cfg.outputs.dir/`:

| file                              | content                                                                                  |
|-----------------------------------|------------------------------------------------------------------------------------------|
| `per_station_<run_id>.csv`        | one row per station: type/area, **measurement_type**, BIAS, RMSE, CRMSE, σ_O, σ_M, R, D_short, D_long, MQI_short, MQI_long, TI_B, TI_R, TI_σ, plus ΔWS_obs/mod, ΔWkWe_obs/mod, ΔDN_obs/mod |
| `summary_<run_id>.csv`            | one row (POOLED — all stations contribute regardless of measurement_type): MQI_short_p90, MQI_long_p90, TI_*_p90, SI_*, MPI_UB-RB, MPI_UT-UB, all pass flags |
| `summary_by_type_<run_id>.csv`    | mqor `by_type` slot: one row per measurement_type present in the network, same columns as `summary_<run_id>.csv` prefixed by `measurement_type` |
| `summary_temporal_coherence_<run_id>.csv`          | 3 × 3 grid (seasonal/week/day × industry/traffic/background) with MPI value + pass flag per cell |
| `radar.<png\|pdf>`                | §9.1                                                                                     |
| `target.<png\|pdf>`               | §9.2                                                                                     |
| `ts_report.<png\|pdf>`            | §9.3                                                                                     |
| `scatter.<png\|pdf>`              | §9.4                                                                                     |
| `taylor.<png\|pdf>`               | §9.5                                                                                     |
| `scatter_dyneval.<png\|pdf>`      | §9.6                                                                                     |
| `bars.<png\|pdf>`                 | §9.7                                                                                     |
| `timeseries_<station>.<png\|pdf>` | §9.8 (one per requested station/NUTS)                                                    |
| `run.log`                         | INFO-level pipeline log (dropped stations, low N, NaN cells)                             |

CSV separator `,`; missing values blank; floats 6 sig.fig.

---

## 12. Tests (`tests/`)

`pytest`-based. Per-module unit tests plus one `test_end_to_end.py`
which builds a tiny synthetic dataset (3 stations × 8760 h hourly NO2,
two crafted to yield MQI ≈ 0.5 and one MQI ≈ 1.5) and runs the full
pipeline. Assertions:

- `MQI_short_p90` matches analytical expectation to 1e-6.
- Every figure file exists, non-empty, opens via PIL.
- `MQO_short_pass` flips correctly when test data are shifted upward.
- `MPI_UB-RB` reproduces a hand-computed value on a 4-station fixture
  (2 UB, 2 RB).
- A coherence-grid fixture covers each contrast × class cell, including
  the "fewer than 2 stations → NaN" edge case.

**Cross-validation:** one fixture station is bundled with reference
values pre-computed against an analytically known denominator.
`MQI`, `TI_B`, `TI_R`, `TI_σ` and `SI_RMSE` are checked to within
1e-4 relative tolerance for `uncertainty_def: aaqd`,
`measurement_type: fixed`.

---

## 13. Open issues / deferred work

Tracked in `TODO.md`, derived from Guidance §5 and informal DTL feedback:

1. **PM2.5 stringency** — Guidance §5.1: AAQD β for PM2.5 may be too
   lax. No code change needed; user can override β via a per-run CSV.
2. **Minimum number of stations < 10** — currently a warning, not a
   hard stop. Guidance §5.2 recommends competent-authority review;
   consider rendering p90 in orange in figures when N < 10.
3. **Data-fusion / cross-validation (leave-one-out)** — Guidance §5.3.
   Not implemented. Users run the pipeline N times with N station
   subsets and aggregate externally.
4. **NUTS lookup from lon/lat** — currently expects a NUTS column in
   `startup.ini`. Could be added via `geopandas` + GISCO shapefile if
   demand arises (compare DTL Annex 3's R utility).

---

## 14. References

- **JRC146736** — Thunis, P., Tarrason, L., Pisoni, E., Janssen, S., 2026.
  *FAIRMODE Guidance Document on Modelling Quality Objectives and
  Benchmarking.* Publications Office of the European Union, Ispra.
  Primary methodological reference; all indicator formulas come from
  §3.3, §3.4, §3.7.1, §3.7.2 and §3.7.3.
- **`mqor` R package** — Tarrason et al., 2025. Reference numerical
  implementation. Where IDL DTL and `mqor` disagree, `mqor` wins. The
  `sqrt(1+β²)` stringency convention used throughout this spec is the
  `mqor` convention (`vec_mqi2`, `vec_pi_bias2`, `vec_pi_cor2`,
  `vec_pi_sd2`, `vec_si_rmse`).
- **Cuvelier, K., 2025.** *DeltaToolLight (DTL) User Manual v1.4.*
  Reference for plot layouts (radar, target, TS report, scatter, Taylor,
  SDE, bars, time series).
- **Directive (EU) 2024/2881** — revised Ambient Air Quality Directive
  (AAQD). Annex V defines the piecewise measurement-uncertainty curve
  bundled in `data/uncertainty_aaqd.csv`.
- **De Meij, A. et al., 2025.** *A new set of indicators for model
  evaluation complementing FAIRMODE's MQO.* Geosci. Model Dev. 18,
  4231–4245. Underlies the §3.7.3 complementary indicators implemented
  here (UB–RB, UT–UB, temporal-coherence grid).
