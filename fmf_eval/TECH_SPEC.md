# `fmf_eval`: FAIRMODE Forecast-Evaluation Tool (Python)

**Technical Specification — v1.0**
**Target environment:** Python 3.10+, Spyder on a desktop workstation.
**Status:** specification ready for implementation; 32 tests pass at
HEAD.

---

## 1. Purpose and scope

`fmf_eval` is a Python port of the **forecast** subset of the FAIRMODE
**DeltaTool** (DTL) IDL code, restricted to the evaluation workflow
defined in:

- **JRC146007** — Vitali, L. and contributors, 2026. *FAIRMODE Guidance
  Document on Quality control indicators for AQ forecasts* (hereafter
  *the Guidance Document*, v2.1).
- **JRC146736** — Thunis, Tarrason, Pisoni, Janssen, 2026. *FAIRMODE
  Guidance Document on Modelling Quality Objectives and Benchmarking*.
  Source of the AAQD short-term `U(O)` parameter table.
- **JRC129254** — Janssen & Thunis, 2022. Source of the legacy FAIRMODE
  `U(O)` parameter table.
- **DeltaTool installer (IDL)** — `elaboration_routines.pro` is the
  inspectable ground truth. The compiled `app/delta.sav` is **not**
  inspected (EULA forbids reverse engineering).
- **Directive (EU) 2024/2881** — revised Ambient Air Quality Directive.

It computes, per modelling application (one year, one pollutant, one
model, one or more forecast horizons), **every indicator listed in
the Guidance Document**:

| Group              | Indicators                                                       | Spec § |
|--------------------|------------------------------------------------------------------|--------|
| Continuous quality | `MQIf` (Eq. 1), `MPI1` (Eq. 6), `MPI2` (Eq. 7)                   | §7.2   |
| Continuous summary | `MQIf_p90_worst`, `MQOf_fulfilled_fraction`, `MPC_both_fulfilled_fraction` | §7.3 |
| Categorical        | `GA+`, `GA−`, `FA`, `MA` (contingency table)                     | §7.4   |
| Categorical scores | `ACC`, `SR`, `POD`, `FBIAS`, `TS`, `GSS` (forecast + persistence) | §7.5  |
| Threshold sensitivity | `POD_minus`, `POD_plus`, `SR_minus`, `SR_plus`                 | §7.6   |
| AQI                | per-station observed/forecast day counts per category            | §7.7   |

with the associated MQOf / MPC pass-fail flags at the 90th-percentile.

It supports **two interchangeable measurement-uncertainty curves**,
toggled by a single config flag:

- `uncertainty_def: fairmode` — Janssen & Thunis 2022 smooth quadratic
  (the original DeltaTool form).
- `uncertainty_def: aaqd`     — Directive 2024/2881 piecewise function
  (Table 1 of JRC146736).

**Both formulations plug into the same downstream math.** The choice
changes (a) the shape of `U(O)`, (b) which parameter row is loaded,
and (c) the absolute values of every continuous indicator — but **not**
the formula structure.

It also produces **eight diagnostic diagrams** (Forecast Target,
Forecast MPI, Forecast Threshold Performance, Forecast Threshold
Normalised Performance, POD/SR sensitivity, Summary Report,
Summary P-Normalised Report, AQI), each in two backends (matplotlib
PNG/PDF + plotly interactive HTML).

### Explicitly out of scope

- **Assessment mode** — already covered by `fmm_assess`.
- The full DeltaTool assessment functionality (assessment Target plot,
  Taylor, time-series reports, source-apportionment / planning
  modules).
- Probabilistic / ensemble forecast scoring (CRPS, Brier, ROC) —
  flagged as open issue in JRC146007 §3 but not implemented.
- IDL-style interactive sliders. The Python equivalent is re-running
  with a different config or override CSV.
- Map plotting (no station geolocation rendering required).
- Reading raw model NetCDF / GRIB files at gridded resolution — the
  tool consumes pre-extracted, station-paired time series (same
  convention as DeltaTool).

---

## 2. Package layout

```
fmf_eval/
├── __init__.py
├── cli.py                       # F5-runnable entry point for Spyder
├── config.py                    # YAML run config (pydantic v2)
├── uncertainty.py               # U(O): fairmode quadratic + aaqd piecewise (§5)
├── stats_core.py                # bias, rmse, crmse, mfe, mfe_ou, rmse_ou, mfu, _aligned
├── indicators.py                # MQIf, MPI1, MPI2 + p90/p10/fulfilment helpers (§7)
├── categorical.py               # ContingencyTable + 6 derived scores (§7.5)
├── aqi.py                       # AQI binning across 4 schemes (§7.7)
├── aggregations.py              # NO2 daily-max, O3 daily-max-8h, PM daily-mean, 75% rule (§6)
├── persistence.py               # Eq. 2 with sign-preserving Crit envelope (§7)
├── io_startup.py                # DeltaTool startup.ini reader
├── io_obs.py                    # dispatcher: CSV-dir or NetCDF observation reader
├── io_model.py                  # NetCDF / per-station CSV model reader
├── pipeline.py                  # end-to-end orchestration (§8)
├── plots/
│   ├── __init__.py              # PLOTLY_AXIS_STYLE + apply_plotly_axes()
│   ├── forecast_target.py       # §9.1
│   ├── forecast_mpi.py          # §9.2
│   ├── forecast_threshold_perf.py    # §9.3 (Figs. 4 + 5)
│   ├── forecast_pod_sr_sensitivity.py # §9.4
│   ├── forecast_summary.py      # §9.5 (Figs. 7 + 8)
│   └── forecast_aqi.py          # §9.6
├── data/
│   ├── uncertainty_default.csv  # FAIRMODE / Janssen-Thunis 2022 parameters
│   ├── uncertainty_aaqd.csv     # AAQD 2024/2881 short-term parameters (Table 1)
│   └── aqi_scales_default.csv   # 4 AQI schemes × pollutants
└── tests/
    ├── test_aggregations.py
    ├── test_aqi.py
    ├── test_categorical.py
    ├── test_end_to_end.py
    ├── test_indicators.py
    ├── test_persistence.py
    └── test_uncertainty.py
```

---

## 3. Dependencies

```bash
pip install numpy pandas scipy matplotlib plotly netCDF4 xarray PyYAML pydantic
```

Versions: `numpy >= 1.24`, `pandas >= 2.0`, `matplotlib >= 3.7`,
`plotly >= 5.18`, `pydantic >= 2.0`. All available in conda-forge /
stock Anaconda → Spyder picks them up. Optional: `kaleido` to export
plotly figures as static PNG/PDF/SVG.

---

## 4. Inputs

### 4.1 `startup.ini`

Three sections, semicolon-separated, parsed in order; blank lines and
lines starting with `;` are ignored. UTF-8 with fallback to
Windows-1252.

**`[MODEL]`** — exactly three lines: 4-digit year; data frequency
(`hour` / `day` / `year`); spatial scale (`urban` / `rural` /
`regional`).

**`[PARAMETERS]`** — one line per pollutant, `code;type;unit`:

- `code` — `NO2 | O3 | PM10 | PM25` (matches the YAML `pollutant`).
- `type` — `GAS | PM`.
- `unit` — typically `ugm-3`.

**`[MONITORING]`** — header line (skipped) + one row per station with
12 semicolon-separated fields:

| # | Field                | Notes                                                  |
|---|----------------------|--------------------------------------------------------|
| 1 | Station Code         | Unique key — matches the obs/forecast CSV file naming  |
| 2 | Station Name         | Free-form; used as the NetCDF variable name            |
| 3 | Station abbreviation | Free-form short label                                  |
| 4 | Altitude (m)         | Float                                                  |
| 5 | Lon, Lat             | Decimal degrees                                        |
| 7 | GMTLag               | Signed hours offset from UTC; `GMT` ≡ `GMT0`           |
| 8 | Region               | Free-form                                              |
| 9 | Station Type         | `background` / `traffic` / `industrial`                |
|10 | Area Type            | `urban` / `suburban` / `rural`                         |
|11 | Siting               | Free-form                                              |
|12 | listOfvariables      | `*`-separated; only stations listing the run pollutant are used |

A trailing `;` per row is tolerated.

### 4.2 Observations

Two equivalent input formats, auto-detected by extension (same as for
model results — see §4.3):

- **Directory of per-station CSVs** under `obs_dir/<StationName>.csv`.
  Header: `year;month;day;hour;<pollutant>;...;`. Missing value `-999`.
- **NetCDF** (`.cdf` / `.nc` / `.nc4`), DeltaTool layout: one variable
  per station, dims `(T, V)` with `T = 8760` (or `8784` leap year),
  parameter names in the `Parameters` global attribute.

Times in CSV inputs are local; the GMT lag from `startup.ini` is
applied at load to convert to UTC and re-index onto a continuous
hourly UTC timeline for the calendar year of the run. NetCDF inputs
are assumed already on the UTC hourly grid (matching DeltaTool
convention).

> **Note.** The YAML field carrying the observation path is
> `inputs.obs_dir` regardless of whether the path points at a
> directory or a single `.cdf` file. The name is kept for backwards
> compatibility — see
> [`CONVENTIONS.md` — Documented divergences](CONVENTIONS.md#documented-idl--python-divergences).

### 4.3 Model results (forecasts)

One file **per forecast horizon**, declared in `inputs.model_files` as
`{0: ..., 1: ..., 2: ...}`. Two equivalent input formats, auto-detected
by extension:

- **NetCDF**, DeltaTool layout: `<YEAR>_<MODEL>_TIME.cdf`, one variable
  per station, dims `(T, V)`.
- **Directory of per-station CSVs** with the same layout as
  observations.

Any combination of obs and forecast formats is valid; results are
bit-identical across formats.

### 4.4 `run.yaml` — single user-facing config

```yaml
run_id: milan_pm25_2025
pollutant: PM25
forecast_horizons: [0, 1, 2]
threshold_ugm3: 50              # categorical exceedance threshold (µg/m³)
threshold_sensitivity: 1        # ±1 µg/m³ for Fig. 6
aqi_scheme: EEA_AQI             # EEA_AQI | UK4_AQI | UK10_AQI | USEPA_AQI

# Filters
min_data_capture: 0.75          # 75 % rule per station
min_stations: 4                 # below this, p90 is logged as a warning

# Uncertainty toggle — the FAIRMODE vs AAQD switch.
uncertainty_def: fairmode       # fairmode | aaqd

# Optional override for the U(O) parameter table.
inputs:
  startup_ini: ./resource/startup.ini
  obs_dir:    ./data/monitoring/po_valley       # directory OR a .cdf file
  model_files:
    0: ./data/modeling/po_valley/2025_AENSa_TIME_FH0.cdf
    1: ./data/modeling/po_valley/2025_AENSa_TIME_FH1.cdf
    2: ./data/modeling/po_valley/2025_AENSa_TIME_FH2.cdf
  uncertainty: null             # null = use bundled defaults
  aqi_scales:  null             # null = use bundled defaults

outputs:
  dir: ./out/milan_pm25_2025
  formats: [png, pdf]
  dpi: 200
```

`uncertainty_def` selects which parameter table is loaded
(`uncertainty_default.csv` for `fairmode`, `uncertainty_aaqd.csv` for
`aaqd`). `aqi_scheme` selects which row block of
`aqi_scales_default.csv` is used.

---

## 5. Measurement uncertainty `U(O)`

Both formulations return the **absolute** measurement uncertainty in
µg/m³ at the observation concentration `O`. They share a uniform
Python signature:

```python
def U_of_O(O: np.ndarray, params: UncertaintyParams) -> np.ndarray: ...
```

so that everything downstream (`persistence.persistence_uncertainty`,
`MFU`, every continuous indicator) is curve-agnostic.

### 5.1 `uncertainty_def: fairmode` — smooth quadratic (default)

Janssen & Thunis (2022) / DeltaTool `goals_criteria_oc.dat`
formulation:

```
U(O) = U_rv95r · sqrt( (1 − α²) · O² + α² · RV² )
```

Parameters per pollutant (`data/uncertainty_default.csv`):

| pollutant | Urv95r | RV (µg/m³) | α    | Np  | Nnp | limit_value | k |
|-----------|--------|------------|------|-----|-----|-------------|---|
| NO₂       | 0.24   | 200        | 0.20 | 5.5 | 5.2 | 200         | 2 |
| O₃        | 0.18   | 120        | 0.79 | 3   | 11  | 120         | 2 |
| PM10      | 0.28   | 50         | 0.25 | 1.5 | 20  | 50          | 2 |
| PM2.5     | 0.36   | 25         | 0.50 | 1.5 | 20  | 25          | 2 |

This is the original behaviour of DeltaTool — existing `run.yaml`
files without the `uncertainty_def` key continue to use this
formulation.

### 5.2 `uncertainty_def: aaqd` — piecewise

Short-term form from JRC146736 Table 1, reflecting AAQD 2024/2881:

```
U_O,fix(O) = U_φ + O · (U_O,fix(LV) − U_φ) / LV     if O ≤ LV
U_O,fix(O) = O · U_O,fix(LV) / LV                   if O >  LV
```

with `U_O,fix(LV) = U_Ofix_LV_rel · LV`. Parameters per pollutant
(`data/uncertainty_aaqd.csv`):

| pollutant | time res.    | LV (µg/m³) | U_Ofix(LV) | U_φ (µg/m³) | β_short |
|-----------|--------------|------------|------------|-------------|---------|
| NO₂       | hour         | 200        | 15 %       | 3           | 3.2     |
| O₃        | daily 8h max | 120        | 15 %       | 10          | 2.2     |
| PM10      | day          | 45         | 25 %       | 4           | 2.2     |
| PM2.5     | day          | 25         | 25 %       | 3           | 2.5     |

`beta_short` is carried on the params object for downstream use but
**not** baked into `U_of_O` itself.

### 5.3 Where `U(O)` is evaluated

Two use sites take **different arguments**, matching the IDL:

| Use site                 | `U(O)` argument | IDL `elaboration_routines.pro` line |
|--------------------------|-----------------|-------------------------------------|
| Persistence `Crit` (Eq. 2) | `obs2`        | 680                                 |
| `MFU` (Eq. 4)            | `obs`           | 1471                                |

Both call the same `U_of_O` function — only the argument differs.

---

## 6. Aggregations (`aggregations.py`)

All continuous and categorical indicators operate on **daily values**.
The daily metric depends on the pollutant:

| Pollutant | Daily statistic                              | Function           |
|-----------|----------------------------------------------|--------------------|
| NO₂       | daily maximum of hourly values               | `daily_max`        |
| O₃        | daily maximum of 8-hour moving averages      | `daily_max_8h`     |
| PM10      | daily mean of hourly values                  | `daily_mean`       |
| PM2.5     | daily mean of hourly values                  | `daily_mean`       |

### 6.1 The 75 %-completeness rule

At every aggregation step, days where fewer than 75 % of the
constituent hourly values are present give NaN (e.g. < 18 of 24 for
daily-mean; < 75 % of 17 sliding-window values for O₃ max-8h). The
rule is applied in `aggregations.py` and re-checked at the
station-level in `pipeline.py` against `min_data_capture`.

### 6.2 Local-time aggregation

Daily aggregation is performed in **local time** (UTC + station
`GMTLag`) so that a "daily" value corresponds to the calendar day at
the station. UTC alignment happens at load time in `io_obs` / `io_model`.

### 6.3 Persistence shift

After daily aggregation, the persistence series at forecast horizon FH
is `obs2_t = O_{t − 1 − FH}` for `t ≥ 1 + FH`. The first `1 + FH` days
are dropped from the paired set. Implementation:
`persistence.build_persistence` (IDL line 652).

---

## 7. Indicators (`indicators.py`, `categorical.py`, `aqi.py`)

> **Master rule.** Every continuous indicator is a **ratio**: forecast
> error in the numerator, persistence or measurement-uncertainty
> baseline in the denominator. The pass criterion is uniformly
> `indicator ≤ 1`.

All formulas below come from §1.1, §1.2, §1.3 and §1.4 of the
Guidance Document, cross-checked line-for-line against the IDL
`elaboration_routines.pro` source.

Notation: `O`, `M` are aligned 1-D arrays of paired daily observation
and forecast values after aggregation and 75 %-rule filtering.
`obs2 = O` shifted by `1 + FH` days. `obstempDiff` is the
sign-preserving inflated persistence residual.

### 7.1 The persistence ingredients

```python
# Built once per station, fed to every continuous indicator:
obs2          = shift(O, 1 + FH)                                  # IDL 652
obstempDiff   = O - obs2
Crit          = U_rv95r * sqrt((1 - α²) * obs2² + α² * LV²)       # IDL 680
obstempDiff   = where(obstempDiff > 0, obstempDiff + Crit,        # IDL 681-686
                                       obstempDiff - Crit)
```

`Crit` is added in the **same direction as the raw residual** — not
in quadrature. This is the IDL form, not the simpler
`sqrt(mean((P − O)² + U²))`.

### 7.2 Modelling Quality Indicator for forecasts (MQIf) — §1.1, Eq. 1

```
RMSE   = sqrt( mean( (M − O)² ) )                                 # IDL 1252
RMSE_p = sqrt( mean( obstempDiff² ) )
MQIf   = RMSE / RMSE_p
```

This matches IDL `elaboration_routines.pro` line 1252 to machine
precision on every synthetic test fixture.

### 7.3 MQOf — §1.1

The Modelling Quality Objective for forecasts:

```
MQIf_p90_worst              = percentile_worst(MQIf_per_station, 90, method='idl')
MQOf_fulfilled_fraction     = mean(MQIf ≤ 1)
MQOf fulfilled              ⇔ MQIf_p90_worst ≤ 1
```

Equivalent statement: at least 90 % of stations have `MQIf ≤ 1`.

The Python port reproduces the **IDL DeltaTool** interpolation by
default (`method='idl'`): `n = 0.9·N`, `nf = floor(n)`,
`pos = nf − 1 + (n − nf)`, linearly interpolated on the
ascending-sorted values. This is bit-identical to the IDL. A
`method='numpy'` opt-in (`numpy.percentile`, default `linear`) is
retained for backwards compatibility; it differs by at most ~0.1 in
rank position and can flip green/red at the boundary. Both are
documented in
[`CONVENTIONS.md`](CONVENTIONS.md#90th-percentile-rule).

### 7.4 Modelling Performance Indicators MPI1, MPI2 — §1.2, Eqs. 6, 7

```
MFE_f = mean( 2 · |M − O| / (M + O) )                             # forecast MFE
MFE_p = mean( 2 · |obstempDiff| / (O + obs2) )                    # IDL 1473
MFU   = mean( 2 · U(O) / O )                                       # IDL 1471 — U on O, not obs2

MPI1  = MFE_f / MFE_p
MPI2  = MFE_f / MFU
```

The `MFE_p` denominator is the **plain sum** `O + obs2`, with no
absolute value and no extra `U`. Tied to IDL line 1473.

### 7.5 MPC — §1.2

The per-station Modelling Performance Criterion is `MPI1 ≤ 1 AND
MPI2 ≤ 1`. The network-level fulfilment fraction is:

```
MPC_both_fulfilled_fraction = mean(MPI1 ≤ 1 AND MPI2 ≤ 1)
```

The two ratios are reported per-station in
`indicators_per_station_FH<n>.csv` and visualised in Fig. 3
(Forecast MPI Plot).

### 7.6 Categorical contingency table — §1.3

For threshold `T` (set via `threshold_ugm3`):

```
GA+ = count( M > T  AND  O > T )                                  # IDL 1214-1218
FA  = count( M > T  AND  O ≤ T )
MA  = count( M ≤ T  AND  O > T )
GA− = count( M ≤ T  AND  O ≤ T )
```

**Strict `>`** is used: a day where `O == T` is a non-exceedance.

### 7.7 The six derived categorical scores — §1.3, Table 2

```
ACC   = (GA+ + GA−) / N
SR    = GA+ / (GA+ + FA)              # NaN if denominator = 0
POD   = GA+ / (GA+ + MA)              # NaN if denominator = 0
FBIAS = (GA+ + FA) / (GA+ + MA)       # NaN if denominator = 0; optimum at 1
TS    = GA+ / (GA+ + FA + MA)         # NaN if denominator = 0
H     = (GA+ + MA) * (GA+ + FA) / N
GSS   = (GA+ − H) / (GA+ + FA + MA − H)
```

The six same formulas are also computed against persistence
(`obs2` substituted for `M`), giving `ACCp`, `SRp`, `PODp`, `FBIASp`,
`TSp`, `GSSp` (IDL lines 1438–1441).

JRC146007 §3 leaves the pass criterion for these scores as an **open
issue** — `fmf_eval` reports them as bare numbers.

### 7.8 Threshold sensitivity — §2.4

For each station, POD and SR are recomputed at thresholds shifted by
`±threshold_sensitivity` µg/m³:

```
POD_minus = POD at threshold = T − Δ
POD_plus  = POD at threshold = T + Δ
SR_minus  = SR  at threshold = T − Δ
SR_plus   = SR  at threshold = T + Δ
```

> **Divergence from IDL.** The IDL evaluates POD/SR at 5 points
> (`LV ± 1`, `LV ± 0.5`, `LV`) and reports the *slope*. The Python
> port evaluates at 2 points and stores absolute scores. Same
> underlying data, different metric.

### 7.9 Persistence-normalised summaries — §2.3

```
POD_over_PODp_p10 = percentile_worst(POD/PODp, 10, method='idl')
SR_over_SRp_p10   = percentile_worst(SR/SRp,   10, method='idl')
```

These 10th-percentile-worst-station ratios are the headline numbers
on Fig. 5 (Forecast Threshold Normalised Performance plot).

### 7.10 AQI — §1.4

For each station, classify daily `O` and `M` values into the bins of
the active `aqi_scheme` (`EEA_AQI`, `UK4_AQI`, `UK10_AQI`,
`USEPA_AQI`); output two count vectors per station (observation,
forecast). Bin convention: `[lower, upper)` for all categories
except the last, which is `[lower, upper]`. No fitness criterion is
computed — JRC146007 §3 flags this as an open issue.

Reference: `fmf_eval/aqi.py`. The IDL `aqindex` routine lives in a
separate IDL file and was not inspected; equivalence is unverified.

---

## 8. Pipeline (`pipeline.py`)

Single public entry point:

```python
def run_evaluation(config_path: str | Path) -> Results: ...
```

Execution order, per requested forecast horizon:

1. Load and validate `run.yaml` → typed `RunConfig`.
2. Load `startup.ini` → station catalogue.
3. Apply `listOfvariables` filter (only stations measuring the active
   pollutant are kept).
4. Load hourly observations (UTC, full-year reindex, NaN where missing).
5. Load hourly forecast for FH = n.
6. Daily-aggregate both series per pollutant rule (§6); apply 75 %
   rule.
7. Build `obs2` per Eq. 2; apply station-level `min_data_capture`
   filter.
8. `_per_station_indicators(...)`:
   - For each station compute the persistence ingredients (`obstempDiff`,
     `Crit`), the continuous indicators (`MQIf`, `MFE_f/p`, `MFU`,
     `MPI1`, `MPI2`), and the per-station categorical contingency
     table for both forecast (M vs O) and persistence (obs2 vs O).
   - Compute threshold-sensitivity POD/SR at `T ± Δ`.
   - Compute AQI day-counts for both M and O.
9. Network aggregation:
   - `MQIf_p90_worst`, `MQOf_fulfilled_fraction`,
     `MPC_both_fulfilled_fraction`.
   - `POD_over_PODp_p10`, `SR_over_SRp_p10`.
10. Write CSVs (§11).
11. Render diagrams (§9) in both backends. Close all figures at the
    end of each FH.
12. Return a `Results` dataclass containing the per-station table, the
    summary row, AQI counts, and figure paths so the user can keep
    exploring in Spyder.

A per-run `run.log` captures every warning: dropped stations (low
capture), low N (below `min_stations`), NaN categorical scores,
threshold-sensitivity NaN cases.

---

## 9. Diagrams (`plots/`)

Six plot modules covering eight figures (Fig. 4 + Fig. 5 share
`forecast_threshold_perf.py`; Fig. 7 + Fig. 8 share
`forecast_summary.py`). Each module exposes
`plot_<name>(...) -> matplotlib.figure.Figure` and a plotly
counterpart `plot_<name>_plotly(...) -> plotly.graph_objects.Figure`.
The pipeline saves them in every requested matplotlib format (`png`,
optionally `pdf`) and always writes the plotly HTML.

### 9.1 Forecast Target plot (`plots/forecast_target.py`) — §2.1, Fig. 2

One dot per station:

- **x = `±CRMSE / RMSE_p`**, sign set by `FA > MA` (strict) → positive
  (right) means FA-dominated.
- **y = `BIAS / RMSE_p`**, positive upward (forecast over-predicts).

**Green disc of radius 1** = MQOf-fulfilment zone (any station inside
has `MQIf < 1`). Dotted inner half-circle of radius 0.5 = tighter
"very good" guide. Annotations: `MQIf_p90_worst` (green if `< 1`, red
if `≥ 1`), considered threshold, valid / selected stations.

### 9.2 Forecast MPI plot (`plots/forecast_mpi.py`) — §2.2, Fig. 3

One dot per station with coordinates (`MPI2`, `MPI1`). Plane split
into three zones:

- **Green** lower-left `[0,1] × [0,1]` — both ratios `< 1` (MPC
  fulfilled).
- **Orange** upper-left + lower-right — only one ratio `< 1`.
- **White** upper-right — neither.

Annotation: network MPC fulfilment percentage.

### 9.3 Forecast Threshold Performance plots (`plots/forecast_threshold_perf.py`) — §2.3, Figs. 4 + 5

**Fig. 4 (raw)** — `(SR, POD)` plane with two families of reference
curves:

- **Dashed grey FBIAS isolines** `POD = FBIAS · SR` (lines `#888888`,
  labels `#555555`).
- **Solid green TS isolines** `1/POD = 1/SR + 1/TS − 1` at
  `TS = 0.1, 0.25, 0.5, 0.75` (line `#2ca02c`, labels `#1f6f1f`).

Gold star at (1, 1). Subtitle: threshold, valid/selected stations.

**Fig. 5 (persistence-normalised)** — `(SR/SR_p, POD/POD_p)` plane,
four-quadrant colouring (green / orange / white per the same rule as
Fig. 3). Annotation: `POD_over_PODp_p10` and `SR_over_SRp_p10`
top-right.

In the plotly backend, FB edge-labels use `xref="paper"` /
`yref="paper"` anchoring so they don't clip.

### 9.4 POD / SR sensitivity bar plot (`plots/forecast_pod_sr_sensitivity.py`) — §2.4, Fig. 6

Per station: a group of two parent bars (POD light grey, SR dark grey)
plus two thin overlay bars per parent — **red** (`T − Δ`), **yellow**
(`T + Δ`).

Matplotlib: three `bar()` calls at the same x with shifted widths.
Plotly: three `go.Bar` traces with explicit `width=0.127` and
`offset=parent_offset + bar_width/2 ± (sens_width + gap/2)` so the
overlays land inside the parent bar.

### 9.5 Forecast Summary Reports (`plots/forecast_summary.py`) — §2.5, Figs. 7 + 8

Vertically stacked strips, one per indicator (ten strips total):

- **Count strips** (`GA+`, `GA−`, `FA`, `MA`): x-axis 0 → 365 days.
- **Rate strips** (`ACC`, `SR`, `POD`, `FBIAS`, `TS`, `GSS`):
  x-axis 0 → 3 (unitless).

Layout switches automatically:

- `N < 15` → small dots (markersize 2) at `y = 0.5`.
- `N ≥ 15` → horizontal boxplot at `y = 0.5`, `showfliers=False`,
  boxes pinned to `positions=[0.5]`.

**Fig. 7 (raw)** — per-station indicator value.

**Fig. 8 (persistence-normalised)** — rate strips show
`value / persistence_value` instead, with a green "adequate" band
per indicator: `(1, 3)` for ACC/SR/POD/TS/GSS and `(0.75, 1.25)` for
FBIAS (two-sided, optimum at 1). Count strips are kept in raw-value
form.

Y-labels for the rate rows in Fig. 8 are rewritten via the
`RATE_PNORM_LABEL` dict (`ACC/ACCp`, `SR/SRp`, …).

Indicator labels are placed horizontally on the left:
matplotlib `set_ylabel(label, rotation=0)`; plotly uses
`add_annotation` with `xref="paper"` (plotly axis titles cannot be
set to angle 0).

### 9.6 Forecast AQI plot (`plots/forecast_aqi.py`) — §2.6, Fig. 9

Per station: a stacked bar pair (left = Model day counts per AQI
category, right = Observation day counts). Categories coloured per
the active scheme (`EEA_AQI`, `UK4_AQI`, `UK10_AQI`, `USEPA_AQI`).

---

## 10. CLI / Spyder workflow

```python
# fmf_eval/cli.py
from pathlib import Path
from fmf_eval.pipeline import run_evaluation

CONFIG_PATH = Path("./run.yaml")        # ← edit this path

if __name__ == "__main__":
    res = run_evaluation(CONFIG_PATH)
    s = res.summary
    print(f"MQIf(90th worst) = {s['MQIf_p90_worst']:.3f}  "
          f"({'PASS' if s['MQIf_p90_worst'] <= 1.0 else 'FAIL'})")
    print(f"MQOf fulfilled at {s['MQOf_fulfilled_fraction']*100:.0f}% of stations")
    print(f"MPC  fulfilled at {s['MPC_both_fulfilled_fraction']*100:.0f}% of stations")
    print(f"POD/PODp (10th worst) = {s['POD_over_PODp_p10']:.3f}")
    print(f"SR /SRp  (10th worst) = {s['SR_over_SRp_p10']:.3f}")
```

In Spyder: open `cli.py`, edit `CONFIG_PATH`, press **F5**. Matplotlib
figures appear in the Plots pane and on disk under `outputs.dir`;
plotly HTMLs open in the user's default browser when double-clicked.

Equivalently from the IPython console:

```python
from fmf_eval.pipeline import run_evaluation
res = run_evaluation("./run.yaml")
res.per_station                  # one row per (station, FH)
res.summary                      # one row per FH
```

---

## 11. Outputs

Files written to `cfg.outputs.dir/`:

| File | Content |
|---|---|
| `indicators_per_station_FH<n>.csv` | One row per station: `station`, `N_valid`; continuous indicators `BIAS`, `RMSE`, `RMSEp`, `CRMSE`, `MQIf`, `MFEf`, `MFEp`, `MFU`, `MPI1`, `MPI2`; categorical counts `GA+`, `GA-`, `FA`, `MA`; forecast rates `ACC`, `SR`, `POD`, `FBIAS`, `TS`, `GSS`; persistence rates `ACCp`, `SRp`, `PODp`, `FBIASp`, `TSp`, `GSSp`; threshold sensitivity `POD_minus`, `POD_plus`, `SR_minus`, `SR_plus`. |
| `indicators_summary_FH<n>.csv` | One row per FH: `MQIf_p90_worst`, `MQOf_fulfilled_fraction`, `MPC_both_fulfilled_fraction`, `POD_over_PODp_p10`, `SR_over_SRp_p10`, `n_stations`, `FH`. |
| `aqi_counts_FH<n>.csv` | One row per station × AQI category: `station`, `category_index`, `obs_days`, `fcst_days`. |
| `mplotl/forecast_<topic>_FH<n>.png` | §9.1–§9.6 — matplotlib PNG (and PDF if requested in `outputs.formats`). |
| `pltly/forecast_<topic>_FH<n>.html` | §9.1–§9.6 — plotly interactive HTML. |
| `run.log` | INFO-level pipeline log (dropped stations, low N, NaN categorical cells). |

The eight `<topic>` strings are: `target`, `mpi`, `threshold_perf`,
`threshold_perf_norm`, `pod_sr_sensitivity`, `summary`,
`summary_pnorm`, `aqi`.

CSV separator `,`; missing values blank; floats 6 sig.fig.

---

## 12. Tests (`tests/`)

`pytest`-based. **32 tests pass at HEAD.** Per-module unit tests plus
one `test_end_to_end.py` which builds a synthetic 4-station dataset
on disk and runs the full pipeline. Test coverage:

- **`test_uncertainty.py`** — `U_of_O` for both `fairmode` and `aaqd`
  formulations against hand-computed reference values; parameter CSV
  loading.
- **`test_aggregations.py`** — NO₂ daily-max, O₃ daily-max-8h, PM
  daily-mean; the 75 %-completeness rule at the boundary; local-time
  vs UTC alignment.
- **`test_persistence.py`** — `obs2` shift correctness at FH = 0, 1,
  2; sign-preserving `Crit` inflation (positive vs negative
  residuals); cross-check against the IDL `obstempDiff` formula at
  lines 681–686.
- **`test_indicators.py`** — `MQIf`, `MPI1`, `MPI2` formulas; the
  `MFE_p` plain-sum denominator (the IDL-aligned form, not the
  earlier `|M+O| + U` bug); `percentile_worst` and `fraction_meeting`
  helpers.
- **`test_categorical.py`** — contingency-table edge cases (zero
  denominators → NaN, not 0); strict `>` exceedance; GSS chance-hits
  correction; persistence categorical with `obs2`.
- **`test_aqi.py`** — bin classification for all four schemes (EEA,
  UK4, UK10, USEPA); `[lower, upper)` vs final `[lower, upper]`
  convention; user override schema validation.
- **`test_end_to_end.py`** — full pipeline on a synthetic 4-station
  fixture: asserts every per-station indicator within `1e-6` of the
  hand-computed reference, asserts all 8 figures
  (`mplotl/*.png` + `pltly/*.html`) plus the 3 CSVs exist.

---

## 13. Open issues / deferred work

1. **JRC146007 §3 open issues** — no fitness-for-purpose criterion is
   defined for the six categorical scores or for AQI distributional
   matching. The tool reports them as bare numbers; the choice of
   pass/fail thresholds is left to the user / the FAIRMODE community.
2. **AAQD 2024/2881 Annex V full uncertainty table** — only Table 1
   short-term values are currently bundled in
   `data/uncertainty_aaqd.csv`. The remainder of Annex V (long-term
   parameters, additional pollutants) is not yet confirmed and not
   shipped.
3. **IDL `aqindex` semantics** — the AQI binning IDL routine lives in
   a separate IDL file and was not inspected. The Python `[lower,
   upper)` convention is plausible but unverified; only matters at
   the exact bin edges.
4. **Probabilistic / ensemble forecast scoring** (CRPS, Brier, ROC) —
   out of scope for v1.0. Architecture leaves room for a future
   `probabilistic.py` sibling without refactor.
5. **Percentile interpolation** — *resolved.* `percentile_worst` now
   defaults to `method='idl'`, reproducing the IDL DeltaTool
   `pos = nf − 1 + (n − nf)` interpolation bit-for-bit. The previous
   `numpy.percentile` (`linear`) behaviour is retained as a
   `method='numpy'` opt-in for backwards comparison.

---

## 14. References

- **JRC146007** — Vitali, L. and contributors, 2026. *FAIRMODE
  Guidance Document on Quality control indicators for AQ forecasts*
  (v2.1). Publications Office of the European Union, Ispra. Primary
  methodological reference; every continuous and categorical indicator
  formula comes from §1.1, §1.2, §1.3 and §1.4.
- **JRC146736** — Thunis, P., Tarrason, L., Pisoni, E., Janssen, S.,
  2026. *FAIRMODE Guidance Document on Modelling Quality Objectives
  and Benchmarking* (v4.0). Source of the AAQD short-term `U(O)`
  parameter table bundled in `data/uncertainty_aaqd.csv` (Table 1).
- **JRC129254** — Janssen, S. & Thunis, P., 2022. *FAIRMODE Guidance
  Document on Modelling Quality Objectives and Benchmarking* (v3.3).
  Source of the FAIRMODE smooth-quadratic `U(O)` parameters bundled in
  `data/uncertainty_default.csv`.
- **DeltaTool installer (IDL)** — `elaboration_routines.pro` is the
  inspectable IDL source used as ground truth for every indicator
  formula. The compiled `app/delta.sav` is not inspected (EULA forbids
  reverse engineering).
- **DeltaTool User Guide v7.3.1** — bundled in the IDL installer.
  Reference layout for every plot.
- **Vitali et al., 2023.** *A standardized methodology for the
  validation of air quality forecast applications (F-MQO).* Geosci.
  Model Dev. 16, 6029–6047. Background paper on the F-MQO methodology
  formalised in JRC146007.
- **Directive (EU) 2024/2881** — revised Ambient Air Quality Directive.
  Source of the limit values and Annex V uncertainty parameters used
  in `aaqd` mode.
- **Boylan & Russell, 2006; Chemel et al., 2010** — performance
  criteria for MFE on PM and O₃; background to the MPI1 / MPI2
  formulation.
