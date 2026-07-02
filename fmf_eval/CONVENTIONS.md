# Conventions cheat-sheet

A short reference card for the decisions baked into `fmf_eval`. Use
this document when verifying that `fmf_eval` matches a paper formula
or another tool (in particular the IDL DeltaTool). For full
derivations see [`INDICATORS.md`](INDICATORS.md).

## Contents

- [Master denominator](#master-denominator)
- [Which `U(O)` formulation?](#which-uo-formulation)
- [Statistical conventions](#statistical-conventions)
- [Sign conventions](#sign-conventions)
- [90th-percentile rule](#90th-percentile-rule)
- [MPC and MQOf criteria](#mpc-and-mqof-criteria)
- [Time conventions](#time-conventions)
- [Reference-implementation alignment](#reference-implementation-alignment)

---

## Master denominator

Every continuous indicator in `fmf_eval` is built around the same
**persistence baseline** from JRC146007 Eq. 2: the model's residuals
are compared against those of a trivial "yesterday's observation"
predictor, with the measurement uncertainty applied as a
sign-preserving envelope on the persistence residuals.

| Denominator | Used for | Formula |
|---|---|---|
| `RMSE_p` | `MQIf` | `√mean(obstempDiff²)` over paired daily obs |
| `MFE_p` | `MPI1` | `mean(2·\|obstempDiff\| / (O + obs2))` |
| `MFU` | `MPI2` | `mean(2·U(O) / O)` |

The three "p" denominators share one building block, `obstempDiff`,
constructed from the obs series shifted by the forecast horizon and
inflated **in the same direction** as the residual:

```
obs2_i        = obs shifted by (1 + FH) days       (IDL line 652)
obstempDiff_i = obs_i − obs2_i
Crit_i        = U_rv95r · sqrt( (1 − α²)·obs2_i² + α²·LV² )
if obstempDiff_i > 0:  obstempDiff_i += Crit_i      (IDL lines 681–686)
if obstempDiff_i ≤ 0:  obstempDiff_i −= Crit_i
```

`Crit` is added in the **same direction as the residual**, not in
quadrature. Small persistence errors are therefore not penalised when
they fall within measurement noise. This is **not** the simpler form
`sqrt(mean((P−O)² + U²))`; the IDL formula carries an extra
cross-term `2·|P−O|·U` inside the mean and is what DeltaTool actually
uses.

The pass/fail thresholds are `MQIf ≤ 1`, `MPI1 ≤ 1` and `MPI2 ≤ 1` —
i.e. "forecast no worse than the appropriate baseline". See
[`INDICATORS.md` — Continuous indicators](INDICATORS.md#continuous-indicators-mqif-mpi1-mpi2)
for the per-indicator algebra.

### Which `U(O)` argument?

The choice of *which argument* the uncertainty function is evaluated
on differs between use sites:

| Use site                 | `U(O)` evaluated on   | Why                                                  |
|--------------------------|-----------------------|------------------------------------------------------|
| Persistence `Crit` (Eq. 2) | `obs2`              | The envelope inflates the *persistence prediction's* residual; that prediction is `obs2`. |
| `MFU` (Eq. 4)            | `obs` (today's obs)   | `MFU` measures the noise floor of the observation itself, not of the shifted obs. |

This split is **not cosmetic** — it matches the IDL exactly
(`elaboration_routines.pro` lines 680 vs 1471) and the two values are
different at the MQOf boundary.

---

## Which `U(O)` formulation?

`U(O)` is selected at run time via `uncertainty_def` in `run.yaml`:

| `uncertainty_def` | Formula | Parameter table | Reference |
|---|---|---|---|
| `fairmode` (default) | `U_rv95r · √((1−α²)·O² + α²·RV²)` (smooth) | [`uncertainty_default.csv`](../data/uncertainty_default.csv) | Janssen & Thunis 2022 (JRC129254) |
| `aaqd` | `U_φ + O·(U_LV − U_φ)/LV` for `O ≤ LV`; `O·U_LV/LV` for `O > LV` (piecewise) | [`uncertainty_aaqd.csv`](../data/uncertainty_aaqd.csv) | AAQD 2024/2881, Table 1 of JRC146736 |

Both formulations return the **absolute** observation uncertainty in
µg/m³. The stringency factor `β` is **not** baked into `U_of_O` itself
— `beta_short` is carried on the params object for downstream use only.

A user-supplied override CSV (`inputs.uncertainty`) is loaded with the
same schema as whichever default file matches the active definition.

For full curve definitions and parameter tables see
[`INDICATORS.md` — Measurement uncertainty](INDICATORS.md#measurement-uncertainty-uo).

---

## Statistical conventions

### Strict `>` for exceedance

The contingency table uses **strict** `>` on both axes:

```
GA+ = #{ M > T  AND  O > T }
FA  = #{ M > T  AND  O ≤ T }
MA  = #{ M ≤ T  AND  O > T }
GA− = #{ M ≤ T  AND  O ≤ T }
```

A day where the observation exactly equals the threshold is counted
as a **non-exceedance**. Matches IDL `elaboration_routines.pro` lines
1214–1218 (forecast categorical) and 1438–1441 (persistence
categorical, with `obs2` in place of `M`).

### Pearson correlation

`R = np.corrcoef(M, O)[0,1]`. Range [−1, +1]. Used in the persistence
categorical scores and in the residual decomposition behind the
Forecast Target Plot.

### NaN handling — paired drop

For any indicator computed from `(O, M)` pairs, days where either `O`
or `M` (or `obs2`, when needed) is NaN are dropped from the paired
set before any statistic is computed. This is the "paired
complete-case" convention; `fmf_eval` applies it inside
`stats_core._aligned`:

```python
mask = ~(np.isnan(o) | np.isnan(m))
o_clean, m_clean = o[mask], m[mask]
```

NaNs do **not** propagate silently — they are dropped, and the
per-station `N_valid` count is logged.

### NaN vs zero for categorical denominators

Categorical scores have denominators that can hit zero (e.g.
`SR = GA+ / (GA+ + FA)` is undefined if the forecast never exceeds the
threshold). `fmf_eval` returns **NaN** in those cases, not 0 — a
forecast that has never said "exceedance" is not a forecast with `SR =
0`; it has no SR at all. `ACC` is always defined whenever `N > 0`.

### 75 %-completeness rule (daily aggregation)

A station's daily series requires ≥ 75 % of the hourly values to be
present (e.g. ≥ 18 of 24 for daily-mean pollutants). Days that fail
this stay as NaN and are never paired into the indicator calculation.
Stations whose total valid-day fraction falls below `min_data_capture`
are excluded from the network aggregation. Both checks are logged.

---

## Sign conventions

| Quantity | Sign | Aggregation |
|---|---|---|
| `BIAS` | signed | — (per station; feeds Target y-axis) |
| `RMSE`, `RMSE_p`, `CRMSE`, `MFU` | always ≥ 0 | — |
| `MQIf`, `MPI1`, `MPI2` | always ≥ 0 | `p90` across stations |
| `MFE_f`, `MFE_p` | always ≥ 0 | — (already absolute-value mean) |
| `obstempDiff` (persistence residual) | signed before `Crit`, signed after | — (cancels out in `RMSE_p`, `MFE_p`) |
| Categorical counts `GA+`, `GA−`, `FA`, `MA` | always ≥ 0 (integer) | — |
| `ACC, SR, POD, FBIAS, TS, GSS` | always ≥ 0 (NaN possible) | distribution across stations (Figs. 7, 8) |
| `POD/PODp`, `SR/SRp` | always ≥ 0 (NaN possible) | `p10` across stations |

### Target Plot x-axis sign rule

The Forecast Target Plot's x-axis is `CRMSE / RMSE_p`, with sign:

```
x = +CRMSE / RMSE_p   if  FA  > MA       (false-alarm-dominated)
x = −CRMSE / RMSE_p   if  FA ≤ MA        (miss-dominated)
```

The condition is **strict `>`** for the right side, matching the IDL.
A station with `FA == MA` lands on the negative (miss) side. Both
backends (matplotlib `forecast_target.py` and its plotly counterpart)
implement this rule identically.

---

## 90th-percentile rule

`fmf_eval` reproduces the **IDL DeltaTool** percentile interpolation
by default (`indicators.percentile_worst(..., method="idl")`):

```python
# IDL DeltaTool convention (elaboration_routines.pro):
#   n   = 0.90 * N
#   nf  = floor(n)
#   pos = nf - 1 + (n - nf)        # real-valued 0-indexed rank
#   MQIf_p90_worst = interp between sorted[floor(pos)] and sorted[floor(pos)+1]
MQIf_p90_worst = indicators.percentile_worst(mqif_per_station, 90)   # method="idl"
```

This is **bit-identical to the IDL** rank position. For `N = 18` it
interpolates at rank `pos = 16 − 1 + 0.2 = 15.2` (0-indexed) on the
ascending-sorted values.

A `method="numpy"` opt-in is retained for backwards compatibility; it
calls `numpy.percentile(..., method="linear")`, whose position is
`(N − 1)·0.9 = 15.3` for `N = 18`. The two differ by a fraction of a
rank and can flip green→red at the MQOf boundary — see
[Reference-implementation alignment](#reference-implementation-alignment).

The 10th-percentile-worst persistence-normalised ratios use the same
helper (and therefore the same IDL convention by default):

```python
POD_over_PODp_p10 = indicators.percentile_worst(pod_over_podp, 10)   # method="idl"
```

---

## MPC and MQOf criteria

| Indicator group       | Criterion implemented                                | Status                                                |
|-----------------------|------------------------------------------------------|-------------------------------------------------------|
| `MQIf` (per station)  | `MQIf ≤ 1`                                           | JRC146007 Eq. 1                                       |
| **MQOf** (network)    | `MQIf_p90_worst ≤ 1`                                 | JRC146007 §1.1 — Modelling Quality Objective         |
| `MPI1` (per station)  | `MPI1 ≤ 1`                                           | JRC146007 Eq. 6                                       |
| `MPI2` (per station)  | `MPI2 ≤ 1`                                           | JRC146007 Eq. 7                                       |
| **MPC** (per station) | `MPI1 ≤ 1 AND MPI2 ≤ 1`                              | JRC146007 §1.2 — Modelling Performance Criterion     |
| `POD/PODp`, `SR/SRp`  | persistence-normalised; green when ratio > 1         | JRC146007 §2.3 — no formal cut-off                    |
| Six categorical scores (raw) | **No criterion**                              | JRC146007 §3 — **open issue**                         |
| AQI distributional match | **No criterion**                                  | JRC146007 §3 — **open issue**                         |
| `FBIAS / FBIASp`      | two-sided band `(0.75, 1.25)` on the ratio (Fig. 8)  | FAIRMODE convention for ±25 % closeness-to-1          |

The open-issue criteria are reported as bare numbers in the
per-station and summary CSVs; the tool does **not** invent a
pass/fail threshold for them.

### Network-level fulfilment fractions

Three summary scalars are written to `indicators_summary_FH<n>.csv`:

```
MQOf_fulfilled_fraction       = mean(MQIf ≤ 1)
MPC_both_fulfilled_fraction   = mean(MPI1 ≤ 1 AND MPI2 ≤ 1)
POD_over_PODp_p10             = np.percentile(POD/PODp, 10)
SR_over_SRp_p10               = np.percentile(SR/SRp, 10)
```

The MQOf is conventionally "fulfilled" when
`MQIf_p90_worst ≤ 1`, equivalently when at least 90 % of stations
have `MQIf ≤ 1`.

---

## Time conventions

### Local time and the GMT lag

`local_time = UTC + GMTLag` per station. `GMTLag` is read from
`startup.ini` (7th column, e.g. `GMT`, `GMT+1`; `GMT` is treated as
`GMT+0`). All daily aggregations (NO₂ max, O₃ max-8h, PM mean) are
performed in **local time** so that "daily" corresponds to the
calendar day at the station.

### Daily aggregation rule by pollutant

| Pollutant | Daily statistic                              |
|-----------|----------------------------------------------|
| NO₂       | daily **maximum** of hourly values           |
| O₃        | daily **maximum of 8-hour moving averages**  |
| PM10      | daily **mean** of hourly values              |
| PM2.5     | daily **mean** of hourly values              |

These match the EU Ambient Air Quality Directive aggregation rules.
The 75 %-completeness rule is applied at every aggregation step.

### Persistence lag

`obs2_i = obs_{i − 1 − FH}` for `i ≥ 1 + FH`. The first `1 + FH` days
of any horizon-FH series are therefore unusable for persistence-based
indicators. The pipeline drops them silently from the paired set.

---

## Reference-implementation alignment

`fmf_eval` is a Python port of the IDL DeltaTool's forecast
evaluation routines.

### Inspectable ground truth

The reference source is `elaboration_routines.pro` (4309 lines), the
human-readable IDL file shipped with the DeltaTool installer. The
compiled `app/delta.sav` is **not** inspected: its EULA forbids
reverse engineering. Routines defined **outside**
`elaboration_routines.pro` are flagged below as unaudited.

### Verified bit-identical formulas

| Quantity                                  | IDL line(s)   |
|-------------------------------------------|---------------|
| Persistence shift (`obs2 = SHIFT(obs, 1+FH)`) | 652       |
| Sign-preserving `Crit` inflation          | 680–686       |
| `RMSE_p` denominator (`sqrt(mean(obstempDiff²))`) | 1252  |
| `MFE_p` numerator and denominator (plain sum `O + obs2`) | 1473 |
| `MFU` (U evaluated on `obs`, not `obs2`)  | 1471–1474     |
| Strict `>` for forecast categorical       | 1214–1218     |
| Persistence categorical (`obs2` substituted for `M`) | 1438–1441 |

### Documented IDL ↔ Python divergences

These are **not bugs** — they are deliberate, documented choices
where the Python port differs from the IDL.

#### 1. Threshold sensitivity (Fig. 6)

- **IDL**: evaluates POD/SR at 5 points (`LV ± 1`, `LV ± 0.5`, `LV`)
  and reports the *slope*.
- **Python**: evaluates at 2 points (`T ± threshold_sensitivity`,
  default `±1 µg/m³`) and stores absolute scores `POD_minus`,
  `POD_plus`, `SR_minus`, `SR_plus`.

Same underlying data, different reported metric. Fig. 6 shows the
2-point version.

#### 2. Percentile interpolation — now aligned

The percentile interpolation **matches the IDL** by default
(`percentile_worst(..., method="idl")`):

- **IDL / fmf_eval default**: `pos = nf − 1 + (n − nf)` with
  `n = 0.9·N`, `nf = floor(n)`, on the ascending-sorted values.
- **`method="numpy"` (opt-in)**: `numpy.percentile(..., method="linear")`,
  position `(N − 1)·0.9`.

For `N = 18` the IDL/default position is `15.2`; numpy is `15.3`. The
default now reproduces the IDL value bit-for-bit; the numpy form is
retained only for backwards compatibility and can flip green/red on
`MQIf_p90_worst` at the MQOf boundary.

#### 3. AQI categorisation

- **IDL**: bin convention from the unaudited `aqindex` routine (lives
  in a separate IDL file).
- **Python**: `[lower, upper)` for all categories except the last,
  which is `[lower, upper]`.

Unverified equivalence — only matters at the exact bin edges.

#### 4. `obs_dir` field accepts a `.cdf` file

For backwards compatibility the YAML field `inputs.obs_dir` accepts
either a directory of CSVs (the DeltaTool layout) **or** a NetCDF
file (DeltaTool's `OBS_TIME.cdf` convention). The field name is kept
as-is to avoid breaking existing YAMLs.

### Routines defined outside `elaboration_routines.pro`

Unaudited; behaviour matched best-effort against the DeltaTool User
Guide:

- `aqindex` — AQI bin classification.
- `CheckCriteria`, `CheckCriteriaForecast` — convenience wrappers.
- `bias()`, `crmse()` — helper functions.

### Relation to `fmm_assess`

The companion `fmm_assess` package targets the same DeltaTool family
but for the **assessment** workflow (not forecasts). It uses the
`mqor` R package as primary reference and IDL only where `mqor` is
silent (the §3.7.3 coherence MPIs and §3.7.2 spatial-gradient MPIs).
`fmf_eval` has no equivalent of `mqor` in the forecast space, so the
IDL is the single normative reference for every indicator here.

---

*For full formulas and reading guides see
[`INDICATORS.md`](INDICATORS.md) and [`PLOTS.md`](PLOTS.md). For the
full specification (run config, IDL mapping, every decision and why)
see [`TECH_SPEC.md`](TECH_SPEC.md).*

## Statistics reporting basis (classic stats vs MQI/MPI)

The per-station classic statistics (`BIAS`, `RMSE`, `CRMSE`, and DT's
`MO`/`MM`/`SO`/`SM`/`1-R`) are reported on the **regulatory averaging time** of
the pollutant, matching the DeltaTool *Summary Report*:

| Pollutant | Statistics basis | MQI / MPI basis |
|---|---|---|
| NO2  | hourly                  | daily maximum |
| O3   | daily maximum of 8h     | daily maximum of 8h |
| PM10 | daily mean              | daily mean |
| PM25 | daily mean              | daily mean |

Only **NO2** has a sub-daily statistics basis: its reported `RMSE`/`BIAS`/`CRMSE`
are computed on the hourly paired series, whereas its MQI/MPI are computed on the
daily-maximum series. For O3/PM the two bases coincide. This is implemented by
`aggregations.to_stats_basis()` / `aggregations.stats_uses_subdaily()`; the daily
series feeding `MQIf` (via `rmse_f`) is never altered. Reporting the daily-max
RMSE for NO2 would inflate it ~2× relative to DT, while leaving the MQI unchanged.

## MFu / MPI2 — known DeltaTool discrepancy

`fmf_eval` computes the observation uncertainty per JRC146007 Eq. 4:

    U(O) = Urv · sqrt( (1-α²)·O² + α²·LV² )            [spec, used here]

The compiled DeltaTool implements a different parenthesisation in
`CheckCriteriaForecast`:

    U(O)_DT = Urv · sqrt( (1-α²)·(O² + α²·LV²) )        [DT, confirmed bug]

The DT form expands the LV² coefficient to `(1-α²)·α²` instead of `α²`, and does
**not** reduce to `Urv·O` as α→0 (it should). It has been confirmed as a bug in
DT. Because `MFu = mean(2·U(O)/O)` feeds `MPI2 = MFEf / MFu`, DT's smaller `MFu`
yields a systematically **larger** `MPI2`. The gap grows with α and is therefore
pollutant-dependent (empirical, sample dataset):

| Pollutant | α | MPI2 gap (DT − fmf_eval) |
|---|---|---|
| NO2  | 0.20 | ≈ +2.0 % |
| PM10 | 0.25 | ≈ +2.7 % |
| PM25 | 0.50 | ≈ +13 %  |
| O3   | 0.79 | ≈ +34 %  |

`fmf_eval` intentionally keeps the JRC146007 formula; `MQIf` and `MPI1` are
unaffected and match DT bit-for-bit. `MPI2` will not reconcile with DT until the
DT binary is corrected.
