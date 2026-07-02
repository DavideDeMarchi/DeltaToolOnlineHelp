# Indicators reference

Every quantity `fmf_eval` computes, with formulas, sign conventions,
and a short reading guide. The mathematical reference is the FAIRMODE
Guidance Document on Quality control indicators for AQ forecasts
(JRC146007, Vitali et al., 2026). Cross-references like *GD §1.2*
point to that document.

## Contents

- [At a glance](#at-a-glance)
- [Foundations](#foundations)
- [Measurement uncertainty U(O)](#measurement-uncertainty-uo)
- [Continuous indicators (MQIf, MPI1, MPI2)](#continuous-indicators-mqif-mpi1-mpi2)
- [Categorical indicators (the six exceedance scores)](#categorical-indicators-the-six-exceedance-scores)
- [Air Quality Index (AQI)](#air-quality-index-aqi)
- [Cross-walk to reference implementations](#cross-walk-to-reference-implementations)

---

## At a glance

Every indicator in one table. The continuous indicators (MQIf, MPI1,
MPI2) are dimensionless and pass when `≤ 1`. The categorical scores
(ACC, SR, POD, FBIAS, TS, GSS) have no formal pass criterion in
JRC146007 (§3 — open issue) and are reported as bare numbers.

> **Aggregation rule.** Continuous indicators are aggregated to the
> network level using the **90th-percentile-worst-station** rule
> across stations. The persistence-normalised categorical ratios
> (`POD/PODp`, `SR/SRp`) use the **10th-percentile-worst-station**
> instead — both rules following JRC146007 §1 and the Forecast Target
> Plot side-panel convention.

The 90 subscript in the table denotes the 90th percentile across
stations. `obs2` is the obs series shifted by `1 + FH` days (the
persistence prediction at horizon FH). `obstempDiff = obs − obs2`
after the sign-preserving `Crit` inflation (see
[Foundations](#foundations)).

| Indicator | Per-station formula | Aggregation | Pass | GD section |
|---|---|---|---|---|
| `BIAS` | `mean(M − O)` | — (per station) | — (diagnostic) | §1.1 |
| `RMSE` | `√mean((M−O)²)` | — (per station) | — (diagnostic) | §1.1 |
| `CRMSE` | `√(RMSE² − BIAS²)` | — (per station) | — (diagnostic) | §1.1 |
| `RMSE_p` | `√mean(obstempDiff²)` | — (per station) | reference baseline | §1.1, Eq. 2 |
| **`MQIf`** | `RMSE / RMSE_p` | `p90` across stations | **MQOf**: `MQIf_p90 ≤ 1` | §1.1, Eq. 1 |
| `MFE_f` | `mean(2·\|M−O\| / (M+O))` | — (per station) | — (ingredient of MPI1) | §1.2, Eq. 3 |
| `MFE_p` | `mean(2·\|obstempDiff\| / (O+obs2))` | — (per station) | reference baseline | §1.2 |
| `MFU` | `mean(2·U(O) / O)` | — (per station) | reference noise floor | §1.2, Eq. 4 |
| **`MPI1`** | `MFE_f / MFE_p` | `p90` across stations | `MPI1 ≤ 1` | §1.2, Eq. 6 |
| **`MPI2`** | `MFE_f / MFU` | `p90` across stations | `MPI2 ≤ 1` | §1.2, Eq. 7 |
| **MPC** | both `MPI1 ≤ 1` *and* `MPI2 ≤ 1` | — (per station) | per-station joint criterion | §1.2 |
| `GA+`, `GA−`, `FA`, `MA` | contingency-table counts (days) | — (per station) | — (diagnostic) | §1.3 |
| `ACC` | `(GA+ + GA−) / N` | distribution (Figs. 7, 8) | — (no GD criterion) | §1.3, Table 2 |
| `SR` | `GA+ / (GA+ + FA)` | distribution | — | §1.3 |
| `POD` | `GA+ / (GA+ + MA)` | distribution | — | §1.3 |
| `FBIAS` | `(GA+ + FA) / (GA+ + MA)` | distribution | optimum at **1** (two-sided) | §1.3 |
| `TS` | `GA+ / (GA+ + FA + MA)` | distribution | — | §1.3 |
| `GSS` | `(GA+ − H) / (GA+ + FA + MA − H)`, `H = (GA++MA)(GA++FA)/N` | distribution | — | §1.3 |
| `ACCp, SRp, PODp, FBIASp, TSp, GSSp` | same six formulas with `obs2` in place of `M` | distribution | reference baseline | §1.3 |
| `POD/PODp`, `SR/SRp` | per-station ratio | `p10` across stations | network: `p10 ≥ 1` (green) | §2.3 |
| `POD_minus`, `POD_plus`, `SR_minus`, `SR_plus` | POD/SR recomputed at `T ± Δ` | — (per station) | low spread is good | §2.4 |
| AQI counts | per-station observed-vs-forecast day counts per category | — | — (open issue, §3) | §1.4 |

---

## Foundations

### Notation

For station *s* and day *t* (after daily aggregation):

- `O_t` — observed daily concentration at day *t* (µg/m³)
- `M_t` — forecast daily concentration at day *t*, issued for day *t*
  with forecast horizon `FH` days ahead (µg/m³)
- `obs2_t = O_{t − 1 − FH}` — the persistence prediction at horizon FH
- `obstempDiff_t` — signed persistence residual after `Crit` inflation
  (see [The master ratio](#the-master-ratio))
- `Crit_t` — persistence measurement-uncertainty envelope (Eq. 2 of GD)
- `U(O)` — measurement-uncertainty function (§measurement-uncertainty)
- `T` — exceedance threshold (µg/m³, set via `threshold_ugm3`)
- `FH` — forecast horizon (0, 1, 2, … days)
- `N` — number of valid paired days at a station
- `N_s` — number of valid stations contributing to the network

The bar (e.g. `Ō`) is reserved for annual means; the forecast
indicators here are daily, so the bar notation does not appear.

### Daily aggregation

All indicators are computed on **daily values** derived from the
hourly forecast and observation series, with a 75 % minimum data
capture applied at each aggregation step. The daily statistic depends
on the pollutant (matching the EU Ambient Air Quality Directive
aggregation rules):

| Pollutant | Daily statistic                              |
|-----------|----------------------------------------------|
| NO₂       | daily **maximum** of hourly values           |
| O₃        | daily **maximum of 8-hour moving averages**  |
| PM10      | daily **mean** of hourly values              |
| PM2.5     | daily **mean** of hourly values              |

After this step the rest of the indicator pipeline sees only paired
daily series `M_t` (forecast) and `O_t` (observation) at each station.
Reference implementation: `fmf_eval/aggregations.py`.

### The persistence model

The reference baseline used throughout JRC146007 is

$$
P_t = O_{t-1-FH} \;\pm\; U\bigl(O_{t-1-FH}\bigr) \tag{Eq. 2}
$$

— yesterday's observation (or the appropriate lag for forecast
horizon FH), with the measurement uncertainty as an envelope. Every
forecast must beat this trivial baseline to be considered useful.

### The master ratio

Every continuous indicator in `fmf_eval` is a **ratio**: the forecast's
own error metric in the numerator, the corresponding persistence (or
measurement-uncertainty) baseline in the denominator. The shared
building block is `obstempDiff`, the signed persistence residual
**after** the sign-preserving `Crit` inflation:

$$
\mathrm{obstempDiff}_t = O_t - \mathrm{obs2}_t \;\pm\; \mathrm{Crit}_t
$$

with

$$
\mathrm{Crit}_t = U_{rv95r}\;\sqrt{(1-\alpha^2)\,\mathrm{obs2}_t^2 + \alpha^2\,LV^2}
$$

and the `±` applied in the **same direction** as the raw residual:
positive residuals get `+Crit`, non-positive ones get `−Crit`.

This is **not** the simpler form `sqrt(mean((P−O)² + U²))` (quadrature
inflation). The IDL formula contains the extra cross-term `2·|P−O|·U`
inside the mean and is what DeltaTool actually uses. See IDL
`elaboration_routines.pro` lines 680–686.

The pass/fail threshold is therefore **`indicator ≤ 1`** for every
continuous indicator: a value of 1 means "forecast error exactly at
the persistence baseline".

### Core paired statistics

These are the building blocks. Every continuous indicator below is a
ratio of two of these quantities.

| Quantity | Formula | Notes |
|---|---|---|
| `BIAS` | `mean(M − O)` | Signed; positive ⇒ forecast over-predicts on average. |
| `RMSE` | `√mean((M − O)²)` | Total forecast-error magnitude. |
| `CRMSE` | `√(RMSE² − BIAS²)` | Centred RMSE — error after removing the bias. |
| `RMSE_p` | `√mean(obstempDiff²)` | Persistence RMSE (with `±Crit` envelope). |
| `MFE_f` | `mean(2·\|M − O\| / (M + O))` | Forecast Mean Fractional Error. |
| `MFE_p` | `mean(2·\|obstempDiff\| / (O + obs2))` | Persistence MFE (plain sum in denom). |
| `MFU` | `mean(2·U(O) / O)` | Mean Fractional Uncertainty (noise floor). |

The four "f" / "p" / "U" quantities share `obstempDiff` and `U(O)` as
ingredients. NaN handling is paired-drop at the very start (see
[`CONVENTIONS.md` — Statistical conventions](CONVENTIONS.md#statistical-conventions)).

### 90th-percentile-worst rule

Most continuous indicators come back as one value per station. To
produce a single network-level summary, `fmf_eval` takes the **90th
percentile across stations** using the **IDL DeltaTool interpolation**
(`indicators.percentile_worst(..., method="idl")`, the default):

```
# n = 0.9·N ; nf = floor(n) ; pos = nf − 1 + (n − nf)   (0-indexed rank)
MQIf_p90_worst = indicators.percentile_worst(mqif_per_station, 90)
```

This reproduces the IDL value bit-for-bit. A `method="numpy"` opt-in
(`numpy.percentile(..., method="linear")`) is retained for backwards
compatibility — see
[`CONVENTIONS.md` — Reference-implementation alignment](CONVENTIONS.md#reference-implementation-alignment).
The same helper (with `percentile=10`) computes `POD_over_PODp_p10`
and `SR_over_SRp_p10`.

---

## Measurement uncertainty U(O)

The persistence envelope `Crit` and the noise floor `MFU` both depend
on a curve `U(O)` giving the expected measurement uncertainty as a
function of concentration. `fmf_eval` implements both definitions in
common use; the choice is run-time-selectable via `uncertainty_def`.

### `fairmode` — smooth quadratic (default; Janssen & Thunis 2022)

$$
U_O(O) = U_{rv95r}\;\sqrt{(1 - \alpha^2)\,O^2 + \alpha^2\,RV^2}
$$

This is the legacy DeltaTool `goals_criteria_oc.dat` form. Reduces to
`α·U_{rv95r}·RV` at `O = 0` and to `U_{rv95r}·O` for very large `O`.
Parameters live in
[`fmf_eval/data/uncertainty_default.csv`](../data/uncertainty_default.csv):

| pollutant | Urv95r | RV (µg/m³) | α    | Np  | Nnp | limit_value | k |
|-----------|--------|------------|------|-----|-----|-------------|---|
| NO₂       | 0.24   | 200        | 0.20 | 5.5 | 5.2 | 200         | 2 |
| O₃        | 0.18   | 120        | 0.79 | 3   | 11  | 120         | 2 |
| PM10      | 0.28   | 50         | 0.25 | 1.5 | 20  | 50          | 2 |
| PM2.5     | 0.36   | 25         | 0.50 | 1.5 | 20  | 25          | 2 |

This is the original behaviour of DeltaTool. `run.yaml` files without
the `uncertainty_def` key continue to use this formulation.

### `aaqd` — piecewise (JRC146736 Table 1, AAQD 2024/2881)

$$
U_O(O) = \begin{cases}
U_\varphi + O \cdot \dfrac{U_{O,\mathrm{fix}}(LV) - U_\varphi}{LV} & O \le LV \\[6pt]
O \cdot \dfrac{U_{O,\mathrm{fix}}(LV)}{LV} & O > LV
\end{cases}
$$

with `U_{O,fix}(LV) = U_Ofix_LV_rel · LV`. Parameters live in
[`fmf_eval/data/uncertainty_aaqd.csv`](../data/uncertainty_aaqd.csv):

| pollutant | time res.    | LV (µg/m³) | U_Ofix(LV) | U_φ (µg/m³) | β_short |
|-----------|--------------|------------|------------|-------------|---------|
| NO₂       | hour         | 200        | 15 %       | 3           | 3.2     |
| O₃        | daily 8h max | 120        | 15 %       | 10          | 2.2     |
| PM10      | day          | 45         | 25 %       | 4           | 2.2     |
| PM2.5     | day          | 25         | 25 %       | 3           | 2.5     |

Both formulations return the **absolute** observation uncertainty in
µg/m³. The stringency factor `β` is not baked into `U_of_O` itself —
`beta_short` is carried on the params object for downstream use only.

A user-supplied override CSV (`inputs.uncertainty`) is loaded with the
same schema as whichever default file matches the active definition.
Reference implementation: `fmf_eval/uncertainty.py` (`U_of_O`,
`UncertaintyParams`).

### Which argument is `U(O)` evaluated on?

The two use sites take **different arguments** — and this matches the
IDL exactly:

| Use site                 | `U(O)` evaluated on   | IDL line |
|--------------------------|-----------------------|----------|
| Persistence `Crit` (Eq. 2) | `obs2`              | 680      |
| `MFU` (Eq. 4)            | `obs` (today's obs)   | 1471     |

The split is documented in
[`CONVENTIONS.md` — Master denominator](CONVENTIONS.md#master-denominator).

---

## Continuous indicators (MQIf, MPI1, MPI2)

These three are the "is the forecast better than persistence?"
indicators. All dimensionless, all pass when `≤ 1`.

### Short-term MQIf (GD §1.1, Eq. 1)

For each station, from the paired daily series:

$$
\mathrm{MQIf}_s = \frac{\mathrm{RMSE}_s}{\mathrm{RMSE}_{p,s}}
\;=\; \frac{\sqrt{\mathrm{mean}((M_t - O_t)^2)}}
           {\sqrt{\mathrm{mean}(\mathrm{obstempDiff}_t^2)}}
$$

`RMSE_p` uses `obstempDiff` (signed, with `±Crit` inflation), not the
naïve `√mean((O − obs2)² + U²)`. See
[The master ratio](#the-master-ratio) for the `Crit` definition. IDL
reference: `elaboration_routines.pro` lines 1252 (`RMSE_p`), 661
(`obstempDiff`), 680–686 (`Crit`).

### Modelling Performance Indicators MPI1, MPI2 (GD §1.2, Eqs. 6, 7)

$$
\mathrm{MPI1}_s = \frac{\mathrm{MFE}_{f,s}}{\mathrm{MFE}_{p,s}}
\qquad
\mathrm{MPI2}_s = \frac{\mathrm{MFE}_{f,s}}{\mathrm{MFU}_s}
$$

with the three ingredients:

$$
\mathrm{MFE}_f = \mathrm{mean}\!\left(\frac{2\,|M_t - O_t|}{M_t + O_t}\right)
\qquad
\mathrm{MFE}_p = \mathrm{mean}\!\left(\frac{2\,|\mathrm{obstempDiff}_t|}{O_t + \mathrm{obs2}_t}\right)
\qquad
\mathrm{MFU} = \mathrm{mean}\!\left(\frac{2\,U(O_t)}{O_t}\right)
$$

Three implementation details that matter:

1. The `MFE_p` numerator uses `|obstempDiff|`, **with** the
   sign-preserving `+Crit` inflation already applied.
2. The `MFE_p` denominator is the **plain sum** `O + obs2` — no
   absolute value, no extra `U` term — matching the IDL exactly
   (line 1473). Earlier drafts of this port had `|M+O| + U` in the
   denominator; the bug is now removed.
3. `MFU` evaluates `U(O)` on the **observation itself** (not on
   `obs2`) — line 1471. This is the same `U(O)` that goes into `Crit`,
   but at a different argument.

### Network-level summary and MQOf (GD §1.1)

The Modelling Quality Objective for forecasts (MQOf) is

$$
\mathrm{MQIf}_{p90\text{-worst}} \le 1
$$

equivalently: at least 90 % of stations have `MQIf ≤ 1`. The pipeline
reports two related fractions in `indicators_summary_FH<n>.csv`:

```
MQOf_fulfilled_fraction     = mean(MQIf ≤ 1)            # fraction of stations
MPC_both_fulfilled_fraction = mean(MPI1 ≤ 1 AND MPI2 ≤ 1)
```

### How to interpret

- **MQIf < 1** → forecast more accurate than persistence (= good).
- **MQIf = 1** → no better than persistence.
- **MQIf > 1** → worse than persistence (= bad).

- **MPI1 < 1** → forecast error smaller than persistence error.
- **MPI2 < 1** → forecast error smaller than measurement uncertainty
  ("within the noise of the measurement").
- Both must be `< 1` simultaneously to satisfy the per-station MPC.

The colour code on the Forecast Target Plot follows MQIf: green when
`MQIf < 1`, red when `MQIf ≥ 1`. The Forecast MPI Plot uses three
zones — green when both MPI1 and MPI2 are `< 1`, orange when only one
is, white when neither.

Reference implementation: `fmf_eval/indicators.py` (`mqif`, `mpi1`,
`mpi2`, `percentile_worst`, `fraction_meeting`).

---

## Categorical indicators (the six exceedance scores)

These six indicators measure how well the forecast predicts
**exceedances of a threshold** (e.g. a regulatory limit value). The
threshold is set via `threshold_ugm3` in `run.yaml`.

### Contingency table (GD §1.3)

The exceedance comparison uses **strict** `>` on both axes (IDL lines
1214–1218):

| forecast \\ observation | exceedance (`O > T`) | non-exceedance (`O ≤ T`) |
|-------------------------|----------------------|---------------------------|
| **`M > T`**             | `GA+` (hit)          | `FA` (false alarm)        |
| **`M ≤ T`**             | `MA` (miss)          | `GA−` (correct rejection) |

### The six derived scores (GD §1.3, Table 2)

| Indicator | Formula | Sign | What it captures |
|---|---|---|---|
| `ACC` | `(GA+ + GA−) / N` | ≥ 0 | overall fraction of days got right |
| `SR` | `GA+ / (GA+ + FA)` | ≥ 0 | when the model cried wolf, was it right? |
| `POD` | `GA+ / (GA+ + MA)` | ≥ 0 | when an exceedance happened, was it warned? |
| `FBIAS` | `(GA+ + FA) / (GA+ + MA)` | ≥ 0 | exceedance count over- vs under-prediction (optimum at **1**) |
| `TS` | `GA+ / (GA+ + FA + MA)` | ≥ 0 | hits / (hits + misses + false alarms) — CSI |
| `GSS` | `(GA+ − H) / (GA+ + FA + MA − H)`, `H = (GA++MA)(GA++FA)/N` | signed | TS corrected for chance |

NaN handling: when a denominator hits zero (no observed exceedances,
or no forecast exceedances) the score is **NaN, not 0** — a forecast
that never said "exceedance" is not a forecast with `SR = 0`; it has
no SR at all. `ACC` is always defined whenever `N > 0`.

### Persistence categorical scores

The same six scores are also computed with `obs2` (the persistence
prediction) substituted for `M`. They appear as `ACCp`, `SRp`, `PODp`,
`FBIASp`, `TSp`, `GSSp` in `indicators_per_station_FH<n>.csv` and
drive the persistence-normalised Summary Report (Fig. 8). IDL
reference: `elaboration_routines.pro` lines 1438–1441.

### Threshold sensitivity (GD §2.4)

For each station, POD and SR are also recomputed at thresholds
shifted by `±threshold_sensitivity` µg/m³ (default `±1 µg/m³`). The
four resulting scores are stored as `POD_minus`, `POD_plus`,
`SR_minus`, `SR_plus` and visualised in Fig. 6 as overlay bars on
top of each station's nominal-T bar.

> **Divergence from IDL.** IDL evaluates POD/SR at 5 points
> (`LV ± 1`, `LV ± 0.5`, `LV`) and reports the *slope*. The Python
> port evaluates at 2 points and stores absolute scores. Same data,
> different metric — see
> [`CONVENTIONS.md`](CONVENTIONS.md#documented-idl--python-divergences).

### Network-level summary

JRC146007 does **not** prescribe a fitness-for-purpose criterion for
the six categorical scores (§3 — open issue). `fmf_eval` reports the
distribution across stations (Figs. 7, 8) and two percentile summaries
for the persistence-normalised ratios:

```
POD_over_PODp_p10 = np.percentile(POD/PODp, 10)
SR_over_SRp_p10   = np.percentile(SR/SRp,   10)
```

These are the 10th-percentile-worst-station ratios — the level
reached by the worst-decile of stations on each ratio.

### How to interpret

- **POD low and FBIAS low** ⇒ many missed exceedances. The model is
  *conservative*: it under-predicts exceedance counts.
- **SR low and FBIAS high** ⇒ many false alarms. The model is *eager*:
  it over-predicts exceedance counts.
- **TS** combines POD and SR (insensitive to correct rejections);
  **GSS** corrects TS for chance hits.
- **ACC** is the most coarse summary — high ACC is easy to achieve
  when exceedances are rare (`ACC ≈ GA−/N` then), so it should always
  be read alongside POD / FBIAS.

Reference implementation: `fmf_eval/categorical.py`
(`ContingencyTable`, the six derived metrics).

---

## Air Quality Index (AQI)

The AQI assessment is a **distributional** comparison: for each
station the count of days falling into each AQI category in the
forecast vs the observation. No per-day pairing — the timing of
category changes is **not** evaluated (JRC146007 §3 flags this as an
open issue: matched-day AQI accuracy is the more demanding criterion
and is better captured by the categorical scores above).

Four schemes are bundled and selectable via `aqi_scheme` in
`run.yaml`:

| Scheme       | Categories                                | Reference                     |
|--------------|-------------------------------------------|-------------------------------|
| `EEA_AQI`    | Very Good → Extremely Poor (6 bands)      | European Environment Agency   |
| `UK4_AQI`    | Low / Moderate / High / Very High (4)     | UK DAQI banding               |
| `UK10_AQI`   | 1..10 numeric bands (4 colour groups)     | UK DAQI banding               |
| `USEPA_AQI`  | Good → Hazardous (6)                      | US EPA bands                  |

A user-supplied scheme can be provided via `inputs.aqi_scales` (same
schema as `data/aqi_scales_default.csv`).

### How to interpret

A matching colour stack between the model and observation bars at a
station means the AQI category distribution is well reproduced.
Systematic shifts (e.g. model showing more *Medium* when observation
shows more *Good*) signal a calibration issue specific to that
station.

> **Caveat.** A model whose AQI category distribution matches the
> observation perfectly but is *uncorrelated day-by-day* will look
> good on Fig. 9. Always pair it with the categorical scores
> (Figs. 4–8) for the matched-day view.

Reference implementation: `fmf_eval/aqi.py`. Per-station counts are
written to `aqi_counts_FH<n>.csv`; the visual is Fig. 9 (see
[`PLOTS.md` — Fig. 9](PLOTS.md#forecast_aqi_fhnpng--forecast-aqi-plot)).

---

## Cross-walk to reference implementations

`fmf_eval` is a Python port of the **IDL DeltaTool** forecast
evaluation routines. The reference source is `elaboration_routines.pro`
(4309 lines), the human-readable IDL file shipped with the DeltaTool
installer. The compiled `app/delta.sav` is not inspected (EULA
forbids it).

### vs IDL DeltaTool

Every per-station indicator is tied to an explicit line of
`elaboration_routines.pro`.

| Indicator | fmf_eval (Python) | IDL DTL line(s) |
|---|---|---|
| `obs2` (persistence shift) | `persistence.build_persistence` | 652 |
| Sign-preserving `Crit` | `persistence.persistence_uncertainty` | 680–686 |
| `RMSE_p` | `stats_core.rmse_ou` | 1252 |
| `MFE_p` numerator + denominator | `stats_core.mfe_ou` | 1473 |
| `MFU` (U on `obs`, not `obs2`) | `stats_core.mfu` | 1471–1474 |
| Strict `>` forecast categorical | `categorical.ContingencyTable` | 1214–1218 |
| Persistence categorical (`obs2` in place of `M`) | same module | 1438–1441 |

### Documented divergences (not bugs)

| # | What | Effect |
|---|---|---|
| 1 | **Threshold sensitivity** (Fig. 6) — Python: 2-point scores at `T ± Δ`; IDL: 5-point slope through `LV ± 1, LV ± 0.5, LV`. | Same data, different reported metric. |
| 2 | **Percentile interpolation** — now **aligned**: default `method="idl"` reproduces IDL's `pos = nf − 1 + (n − nf)`. A `method="numpy"` opt-in (`numpy.percentile` `linear`) is retained for backwards compatibility. | Default matches IDL bit-for-bit; numpy opt-in differs by a fraction of a rank (pos 15.2 vs 15.3 at `N = 18`). |
| 3 | **AQI bin edges** — Python: `[lower, upper)` for all categories except the last (which is `[lower, upper]`); IDL: convention from the unaudited `aqindex` routine. | Only affects rare on-edge cases. |
| 4 | **`obs_dir` field accepts a `.cdf` file** — kept for backwards compatibility with existing YAMLs that treated obs_dir as either a directory or a NetCDF file. | Purely cosmetic — name doesn't reflect what it is. |

### Routines defined outside `elaboration_routines.pro` (unaudited)

- `aqindex` — AQI bin classification; lives in a separate IDL file.
- `CheckCriteria`, `CheckCriteriaForecast` — convenience wrappers.
- `bias()`, `crmse()` — helper functions.

Behaviour for these is matched best-effort against the public
DeltaTool User Guide v7.3.1.

### Why no `mqor` cross-walk?

Unlike `fmm_assess`, the `mqor` R package does not target the
forecast workflow. JRC146007 indicators (MQIf, MPI1, MPI2, persistence
categorical scores, the 8 forecast figures) are **not** defined in
`mqor`. The IDL DeltaTool is therefore the single normative reference
for `fmf_eval`.

### Decision rule for future indicators

For the forecast workflow there is currently only one reference
implementation (the IDL DTL). New indicators should be added by:

1. Matching the IDL formula bit-for-bit when one exists, with a
   line-number citation in the code comment.
2. Documenting any deliberate divergence in this section and in
   [`CONVENTIONS.md` — Documented IDL ↔ Python divergences](CONVENTIONS.md#documented-idl--python-divergences).

---

*Cross-references throughout: `GD §x.y` = FAIRMODE Guidance Document
on Quality control indicators for AQ forecasts (JRC146007, Vitali et
al., 2026). For plot-reading guidance see [`PLOTS.md`](PLOTS.md). For
the conventions cheat-sheet (signing rules, NaN handling, percentile
rule) see [`CONVENTIONS.md`](CONVENTIONS.md).*
