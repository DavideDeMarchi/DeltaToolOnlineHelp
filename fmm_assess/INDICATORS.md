# Indicators reference

Every quantity `fmm_assess` computes, with formulas, sign conventions,
and a short reading guide. The mathematical reference is the FAIRMODE
Guidance Document v4.0 (JRC146736). Cross-references like *GD §3.7.1*
point to that document.

## Contents

- [At a glance](#at-a-glance)
  - [Mixed measurement types (fixed + indicative)](#mixed-measurement-types-fixed--indicative)
- [Foundations](#foundations)
- [Measurement uncertainty U(O)](#measurement-uncertainty-uo)
- [Modelling Quality Indicator (MQI)](#modelling-quality-indicator-mqi)
- [Complementary temporal MPIs (TI)](#complementary-temporal-mpis-ti)
- [Complementary spatial MPIs (SI)](#complementary-spatial-mpis-si)
- [Spatial gradient MPIs (UT-UB, UB-RB)](#spatial-gradient-mpis-ut-ub-ub-rb)
- [Temporal-coherence MPIs (3x3 grid)](#temporal-coherence-mpis-3x3-grid)
- [Cross-walk to reference implementations](#cross-walk-to-reference-implementations)

---

## At a glance

Every indicator in one table. All are dimensionless and pass when
`≤ 1` (after absolute value where applicable).

> **Aggregation rule.** Every indicator is aggregated to the network
> level using the **90th-percentile rule** (Eq. 5) across stations
> — **except the SI family** (`SI_RMSE`, `SI_B`, `SI_R`, `SI_σ`),
> which are already single scalars computed across the whole station
> population (no per-station value exists to aggregate).

> **Mixed networks.** When the network contains both fixed and
> indicative stations, each station's indicator is normalised by **its
> own** `U(O)` curve, and the network-level value is reported **twice**:
> once pooled (`summary_<run>.csv`) and once split by measurement type
> (`summary_by_type_<run>.csv` — the AAQD-aligned form). SI is the one
> exception: its pooled value is `NaN` in mixed networks and only the
> per-type values are meaningful. See
> [Mixed measurement types](#mixed-measurement-types-fixed--indicative)
> below for the full rule, and each indicator section restates how it
> behaves in that case.

The 90 subscript in the table denotes the 90th percentile across
stations using Eq. 5.

| Indicator | Per-station formula | Aggregation | Pass | GD section |
|---|---|---|---|---|
| `MQI_short` | `RMSE / D_short` | `p90` across stations | `MQI_short,90 ≤ 1` | §3.3 |
| `MQI_long` | `\|M̄ − Ō\| / D_long` | `p90` across stations | `MQI_long,90 ≤ 1` | §3.3 |
| `TI_B` | `(M̄ − Ō) / D_short` | `p90(\|.\|)` | `\|TI_B\|_90 ≤ 1` | §3.7.1 |
| `TI_R` | `√(2·σ_O·σ_M·(1−R)) / D_short` | `p90` | `TI_R,90 ≤ 1` | §3.7.1 |
| `TI_σ` | `(σ_M − σ_O) / D_short` | `p90(\|.\|)` | `\|TI_σ\|_90 ≤ 1` | §3.7.1 |
| `SI_RMSE` | `RMSE_sp / D_sp` | none — already scalar | `SI_RMSE ≤ 1` | §3.7.2 |
| `SI_B` | `BIAS_sp / D_sp` | none — already scalar | `\|SI_B\| ≤ 1` | §3.7.2 |
| `SI_R` | `√(2·σ_Ō^sp·σ_M̄^sp·(1−R_sp)) / D_sp` | none — already scalar | `SI_R ≤ 1` | §3.7.2 |
| `SI_σ` | `(σ_M̄^sp − σ_Ō^sp) / D_sp` | none — already scalar | `\|SI_σ\| ≤ 1` | §3.7.2 |
| `MPI_UT-UB` | signed gradient (see spatial-gradients) | `p90(\|.\|)` across UT | `MPI_UT-UB ≤ 1` | §3.7.2 |
| `MPI_UB-RB` | signed gradient (see spatial-gradients) | `p90(\|.\|)` across UB | `MPI_UB-RB ≤ 1` | §3.7.2 |
| `MPI_coh[contrast,class]` | signed Δ (see coherence) | `p90(\|.\|)` per cell | `cell ≤ 1` | §3.7.3 |

Denominator key:

- `D_short = √(1+β²_short) · RMSU_O` — for MQI_short, TI rows; computed
  over paired hourly/daily obs.
- `D_long = √(1+β²_long) · U(Ō_s)` — for MQI_long; per-station, using
  each station's annual mean.
- `D_sp = √(1+β²_long) · RMSU_Ō` — for SI rows; spatial counterpart of
  D_long, computed across stations' annual means.
- `RMSU_X = √(mean(U(X_i)²))` — RMS of the uncertainty curve evaluated
  at each value `X_i`.

### Mixed measurement types (fixed + indicative)

Per AAQD Annex VI a network may contain **both fixed and indicative**
monitoring stations; their uncertainty curves differ (different `α`,
`U_{O,r}(RV)`, `β` for indicative vs fixed) and so the indicators are
not directly comparable across types. `fmm_assess` handles this as
follows:

- **Per-station computation** — each station's MQI and TI are
  normalised by **its own** `U(O)` (selected by
  `Station.measurement_type` from the fixed or indicative row of the
  uncertainty-parameter CSV). No mixing happens at the station level:
  a fixed station is always assessed against the fixed curve, an
  indicative station against the indicative curve.
- **Network aggregation** is reported **twice** in every run:
  - A **pooled** summary (`summary_<run>.csv`) where all stations
    contribute to the same p90 regardless of type — useful for an
    overall MQO statement when both sub-networks are equally weighted.
  - A **per-type** summary (`summary_by_type_<run>.csv`) with one row
    per measurement type — this is the AAQD-aligned view, since fixed
    and indicative stations have separate `β` and MPC thresholds.
- **SI is the one exception**: a pooled SI is not defined for a
  mixed network (the spatial RMSE/correlation would mix two different
  uncertainty regimes in its denominator). In a mixed run the pooled
  SI fields are written as `NaN` and only the per-type SI values in
  `summary_by_type_<run>.csv` are meaningful. This matches MQOR's
  `calc_longterm_type_stats` policy.
- **Temporal-coherence MPIs are not partitioned by measurement type**:
  the 9-cell grid lives in its own `summary_temporal_coherence_<run>.csv` (long format:
  `contrast`, `class`, `MPI`, `pass`) and contains the pooled values
  only. A measurement-type split is not emitted by design — the cells
  are already partitioned along the `class ∈ {industry, traffic,
  background}` axis (Station Type), and the typical FAIRMODE network
  rarely has enough indicative stations per class to support a further
  sub-split that would yield a robust p90. The per-station Δ_s used
  inside each cell still respects the per-station denominator rule
  (each station normalised by its own `D_long`), so in mixed runs the
  cell pools the two uncertainty regimes at the *aggregation* step, as
  IDL DTL does.

Each indicator section below restates how it behaves in a mixed
network.

---

## Foundations

### Notation

For station *s* and time-step *t* (hourly or daily, depending on the
short-term aggregation):

- `O_t` — observed concentration at time *t* (µg/m³)
- `M_t` — modelled concentration at time *t* (µg/m³)
- `Ō_s`, `M̄_s` — annual mean of obs / model at station *s*
- `σ_O`, `σ_M` — *temporal* standard deviation of obs / model at station *s*
- `σ_Ō^sp`, `σ_M̄^sp` — *spatial* standard deviation across stations
- `R` — Pearson correlation between obs and model at station *s* (temporal)
- `R_sp` — Pearson correlation between station means (spatial)
- `BIAS = mean(M − O)`, `RMSE`, `CRMSE` — paired statistics (formulas below)
- `β_short`, `β_long` — stringency factors (AAQD parameters)
- `U_O(O)` — measurement-uncertainty function (§measurement-uncertainty)
- `RMSU_O` — root-mean-square of `U_O(O_t)` over the time series

The bar (e.g. `Ō`) means *temporal* mean at a station; the superscript
*sp* (e.g. `σ_Ō^sp`) means *spatial* statistic across the network's
station means.

### Two time scales

Every indicator is computed for one of two terms:

- **Short term** — at each station, from the time series of hourly or
  daily values. Captures whether the model gets the timing of
  variations right. Uses `β_short`.
- **Long term** — at each station, from a single annual-mean value.
  Captures whether the model gets the absolute level right. Uses
  `β_long`.

For some indicators only one term is meaningful (e.g. the spatial
indicators SI_* are only defined for the long term, since they look
across stations).

### The master formula

Every indicator in `fmm_assess` uses the same denominator from GD §3.3:

$$
D = \sqrt{1 + \beta^2} \cdot \mathrm{RMSU}_O
$$

with `β = β_short` for short-term indicators and `β = β_long` for
long-term indicators. The factor `√(1+β²)` is the *stringency*
adjustment: it scales the uncertainty so that a model just within the
acceptable observation-plus-model error budget has indicator value
exactly 1.

The pass/fail threshold is therefore **`indicator ≤ 1`** for every
indicator (after absolute value where the indicator is signed).

The §3.7.3 coherence MPIs and the §3.7.2 gradient MPIs use each
station's own `U_O(Ō_s)` per-station — same formula, evaluated at that
station's annual mean.

### Core paired statistics

These are the building blocks. Every indicator below is some
combination of these quantities and `D`.

| Quantity | Formula | Notes |
|---|---|---|
| `BIAS` | `mean(M − O)` | Signed; positive ⇒ model over-predicts on average. |
| `RMSE` | `√(mean((M − O)²))` | Total error magnitude. |
| `CRMSE` | `√(mean(((M − M̄) − (O − Ō))²))` | "Centred" RMSE — error after removing both means. |
| `σ_O`, `σ_M` (temporal) | `√(mean((x − x̄)²))`, ddof = 0 | Population estimator (`mqor::vec_sd(term="short")`). |
| `σ_Ō^sp`, `σ_M̄^sp` (spatial) | `√(Σ(x − x̄)² / (N − 1))`, ddof = 1 | Sample estimator (`mqor::vec_sd(term="long")`). |
| `R` | Pearson correlation, `np.corrcoef(M, O)[0,1]` | Range −1 to +1. |

The three are linked by the Pythagoras identity used throughout the
Guidance Document:

$$
\mathrm{RMSE}^2 = \mathrm{BIAS}^2 + \mathrm{CRMSE}^2
              = \mathrm{BIAS}^2 + \sigma_O^2 + \sigma_M^2 - 2\sigma_O\sigma_M R
$$

This identity is what makes the MQI decomposable into three independent
temporal MPIs (TI_B, TI_R, TI_σ).

### 90th-percentile rule (GD Eq. 5)

Most indicators come back as one value per station. To produce a single
network-level summary, `fmm_assess` takes the **90th percentile across
stations**, using the linear-interpolation formula from §3.4 Eq. 5:

```
N90 = floor(0.9 · N)
D90 = 0.9 · N − N90
p90 = sorted_vals[N90 − 1]
      + D90 · (sorted_vals[N90] − sorted_vals[N90 − 1])
```

The R-style 1-indexing in the Guidance Document is translated to
0-indexed Python by shifting by 1. Verified to match
`mqor::mqo_percentile` exactly.

---

## Measurement uncertainty U(O)

The denominator `D` depends on a curve `U_O(O)` giving the expected
measurement uncertainty as a function of concentration. `fmm_assess`
implements the AAQD definition from the Guidance Document.

### AAQD piecewise (GD §3.3)

Two-segment linear curve from `U_∅` at `O = 0` to a fixed relative
uncertainty above the limit value `LV`:

$$
U_O(O) = \begin{cases}
U_\emptyset + O \cdot \dfrac{U_{O,\text{fix}}(LV) - U_\emptyset}{LV} & O \le LV \\[6pt]
O \cdot \dfrac{U_{O,\text{fix}}(LV)}{LV} & O > LV
\end{cases}
$$

For short-term PM10, PM2.5, O3, NO2, SO2 and CO the AAQD prescribes
specific `U_∅` values (4, 3, 10, 3, 5 µg/m³ and 0.5 mg/m³ respectively);
for everything else `U_∅ = U_{O,fix}(LV)`. Parameters live in
[`fmm_assess/data/uncertainty_aaqd.csv`](../data/uncertainty_aaqd.csv).

### RMSU

For a time series of length `N`:

$$
\mathrm{RMSU}_O = \sqrt{\tfrac{1}{N} \sum_{t=1}^{N} U_O(O_t)^2}
$$

For a single annual-mean value `Ō`, `RMSU_O` collapses to `U_O(Ō)`.
For spatial indicators, the RMS is taken across stations:

$$
\mathrm{RMSU}_{\bar O} = \sqrt{\tfrac{1}{N_s} \sum_{s=1}^{N_s} U_O(\bar O_s)^2}
$$

where `N_s` is the number of stations and `Ō_s` is the annual mean
observation at station `s`.

---

## Modelling Quality Indicator (MQI)

The MQI is the headline indicator: it quantifies the total discrepancy
between modelled and observed concentrations, normalised by the
acceptable uncertainty budget. It comes in two flavours.

### Short-term MQI (GD §3.3 Eq. 1)

For each station, from the hourly/daily time series:

$$
\mathrm{MQI}^{\text{short}}_s = \frac{\mathrm{RMSE}_s}{\sqrt{1 + \beta_{\text{short}}^2}\;\mathrm{RMSU}_{O,s}}
$$

### Long-term MQI (GD §3.3 Eq. 3)

For each station, from the single annual-mean pair `(M̄_s, Ō_s)`:

$$
\mathrm{MQI}^{\text{long}}_s = \frac{|\bar{M}_s - \bar{O}_s|}{\sqrt{1 + \beta_{\text{long}}^2}\;U_O(\bar{O}_s)}
$$

### Network-level summary and MQO (GD §3.4)

The Modelling Quality Objective is:

$$
\mathrm{MQI}^{\text{short}}_{90} \le 1 \quad\text{and}\quad \mathrm{MQI}^{\text{long}}_{90} \le 1
$$

where the *90* subscript means the 90th percentile across stations
using Eq. 5. For each term, the network passes if at least 90% of
stations have `MQI ≤ 1`.

### How to interpret

- **MQI = 0** — perfect agreement.
- **MQI = 1** — model error exactly at the acceptable uncertainty limit.
- **MQI > 1** — model exceeds the uncertainty budget at that station.

The 90th-percentile rule allows for a small fraction of poorly-modelled
stations (network design constraints, atypical sites) without
invalidating the whole assessment.

### Mixed networks

Per-station MQI uses each station's own `U(O)` curve (the `fixed` or
`indicative` row of the parameter CSV, selected by
`Station.measurement_type`). At the network level the run produces
**both** a pooled p90 (all stations, in `summary_<run>.csv`) and a
per-type p90 (one value per measurement type, in
`summary_by_type_<run>.csv`). The MQO assessment should be read off the
per-type file when fixed and indicative stations have separate `β`
thresholds in the AAQD.

---

## Complementary temporal MPIs (TI)

The MQI condenses three independent skills (bias, correlation,
amplitude) into one number. The TI_* triplet *decomposes* the
short-term MQI into those three skills:

$$
(\mathrm{MQI}^{\text{short}}_s)^2 = \mathrm{TI}_B^2 + \mathrm{TI}_R^2 + \mathrm{TI}_\sigma^2
$$

with denominator `D_short = √(1+β²_short) · RMSU_O`:

| Indicator | Formula | Sign | What it captures |
|---|---|---|---|
| `TI_B` | `BIAS / D_short` | signed | systematic offset (over/under-prediction) |
| `TI_R` | `√(2·σ_O·σ_M·(1−R)) / D_short` | ≥ 0 | timing skill (low R inflates this) |
| `TI_σ` | `(σ_M − σ_O) / D_short` | signed | amplitude skill (model too smooth → negative; too noisy → positive) |

### MPC

The Modelling Performance Criteria are:

```
|TI_B|_90 ≤ 1     |TI_R|_90 ≤ 1     |TI_σ|_90 ≤ 1
```

with `|.|_90` denoting the 90th percentile of absolute values (GD
Eq. 5 with abs).

### Reading guide

When `MQI > 1`, the TI_* triplet tells you **why**:

- **High `TI_B`, low `TI_R` and `TI_σ`** — the model has a systematic
  offset but otherwise tracks the variations correctly. Typically an
  emission-inventory issue.
- **High `TI_σ`** — the model is too smooth or too noisy in amplitude.
  Typically a mixing-height or chemistry-mechanism issue.
- **High `TI_R`** — the model gets the *timing* wrong. Typically a
  meteorology issue, a clock-offset bug, or a wrong diurnal-cycle
  parameterisation.

### Mixed networks

Same as MQI: each station's TI uses its own `U(O)` in `D_short`, and
the network-level p90s are reported both pooled
(`summary_<run>.csv`) and split by measurement type
(`summary_by_type_<run>.csv`). The MPC thresholds (`|TI|_90 ≤ 1`)
should be evaluated separately for the fixed and indicative
sub-networks when the AAQD prescribes different `β_short` for them.

---

## Complementary spatial MPIs (SI)

To assess how well the model reproduces *spatial* variation across the
network, `fmm_assess` first reduces each station to its annual mean and
then computes the same statistics — but *across* stations rather than
*along* time. The denominator uses `β_long`:

$$
D^{\text{sp}} = \sqrt{1 + \beta_{\text{long}}^2}\;\mathrm{RMSU}_{\bar{O}^{\text{sp}}}
$$

| Indicator | Formula | Sign |
|---|---|---|
| `SI_RMSE` | `RMSE_sp / D_sp` | ≥ 0 |
| `SI_B` | `BIAS_sp / D_sp` | signed |
| `SI_R` | `√(2·σ_Ō^sp·σ_M̄^sp·(1−R_sp)) / D_sp` | ≥ 0 |
| `SI_σ` | `(σ_M̄^sp − σ_Ō^sp) / D_sp` | signed |

These are **single scalars per run**, not per-station — they describe
the network as a whole. There is no 90th-percentile step because
there's only one value to begin with.

### Reading guide

- **`SI_B`** — does the model systematically over- or under-predict
  across the network as a whole? Positive ⇒ all stations modelled too
  high.
- **`SI_R`** — does the spatial *pattern* match? A low correlation
  means the model puts the hotspots in the wrong places.
- **`SI_σ`** — does the network have the right *spatial spread*? A
  negative value means the model gives almost the same concentration
  at every station while the observations show clear gradients.
- **`SI_RMSE`** — the total spatial error budget; combines all three
  above.

### Mixed networks

SI is the only family where the **pooled value is not defined** for a
mixed network. The reason is the denominator `D_sp =
√(1+β²_long)·RMSU_Ō`: it would have to RMS-average two different
uncertainty curves (fixed and indicative) over the station-mean
population, which has no physical meaning. Accordingly:

- `summary_<run>.csv` reports `NaN` for `SI_*` whenever the network
  contains more than one measurement type.
- `summary_by_type_<run>.csv` is the authoritative source: it
  computes SI separately on the fixed and indicative subsets, each
  with its own `β_long` and its own `U(O)`. Read SI from this file in
  mixed runs.

This matches MQOR's `calc_longterm_type_stats` policy: MQOR does not
expose a pooled SI either.

---

## Spatial gradient MPIs (UT-UB, UB-RB)

These check whether the model reproduces *expected* gradients between
station types — e.g. urban-traffic vs urban-background (UT-UB) and
urban-background vs rural-background (UB-RB).

For each station *s* in the "near-source" class (UT for UT-UB, UB for
UB-RB), `fmm_assess` finds the **geographically nearest** station *s'*
of the reference class using the equirectangular distance

$$
d_{s,s'}^2 = (\phi_s - \phi_{s'})^2 + \bigl[(\lambda_s - \lambda_{s'})\cos\bar\phi_{s,s'}\bigr]^2
$$

(with $\phi$ = latitude, $\lambda$ = longitude, $\bar\phi_{s,s'}$ the
midpoint latitude — matches IDL DTL `DTLprog.pro` line 502). The
signed per-station gradient is then

$$
g_s = \frac{\bar{M}_s - \bar{O}_s}{D_s} - \frac{\bar{M}_{s'} - \bar{O}_{s'}}{D_{s'}}
$$

with **split denominators** — each term divided by its own station's
`D = √(1+β²_long)·U(Ō)`. The cell scalar is

$$
\mathrm{MPI}^{\text{grad}} = p_{90}\bigl( |g_s| \bigr)_{s \in \text{near-source}}
$$

i.e. the 90th percentile of the absolute values across near-source
stations. The signed per-station distribution is preserved (so
`ts_report.png` can show direction); only the aggregation step takes
the absolute value.

### Why split denominators?

This is the IDL DeltaToolLight convention. MQOR does not implement
spatial-gradient MPIs at all. See
[CONVENTIONS](CONVENTIONS.md#reference-implementation-alignment) for
the rationale.

### Reading guide

- **High UT-UB** — modelled traffic-vs-background contrast doesn't
  match observations. Usually a sign of unresolved street-canyon
  effects or a misspecified emission inventory for road transport.
- **High UB-RB** — modelled urban-vs-rural contrast is off. Typically
  a regional-background or urban-emissions issue.

### Mixed networks

The split-denominator form means each station term `(M_s−O_s)/D_s`
already uses **that station's own** `D = √(1+β²_long)·U(Ō_s)`. So a
gradient computed between, say, a fixed UT station and an indicative
UB neighbour normalises each side by its own measurement-type
uncertainty, which is the correct thing to do. The p90 is then taken
across all near-source stations regardless of type. If the user
needs gradients reported only within one measurement type, run the
pipeline once per subset.

---

## Temporal-coherence MPIs (3x3 grid)

These check whether the model reproduces three temporal *contrasts*
that matter for policy:

- **Seasonal** — winter (DJF) minus summer (JJA) means.
- **Week vs weekend** — weekday minus weekend means.
- **Day vs night** — 10:00–18:00 minus 22:00–06:00 means (local time;
  hourly aggregation only).

Each contrast is split by station type (industry, traffic, background),
giving a **3×3 grid**.

### Per-station formula

For each station *s* in a given (contrast, class) cell:

$$
\Delta_s = \frac{(\bar{M}^{\text{high}}_s - \bar{O}^{\text{high}}_s) - (\bar{M}^{\text{low}}_s - \bar{O}^{\text{low}}_s)}{\sqrt{1 + \beta_{\text{long}}^2}\;U_O(\bar{O}_s)}
$$

where "high/low" are e.g. winter/summer or weekday/weekend. **The
per-station Δ_s is signed** — no absolute value is taken when forming
Δ_s.

### Cell aggregation

The cell scalar is

$$
\mathrm{MPI}^{\text{coh}}[\text{contrast},\text{class}] = p_{90}\bigl( |\Delta_s| \bigr)_{s \in \text{class}}
$$

i.e. the 90th percentile of the **absolute values** of the per-station
Δ_s. This matches the IDL `calc_perc` routine (`np.abs(finite_arr)`
applied before the percentile in the reference Python port).

### Reading guide

- **High seasonal-background MPI** — model gets the seasonal cycle of
  regional-background concentrations wrong. Wintertime stagnation
  under-resolved, or summertime O3 chemistry off.
- **High week-traffic MPI** — model's weekday emission profile for
  road transport is misspecified.
- **High day-night background MPI** — model has issues with the
  mixing-height or nocturnal-boundary-layer scheme.

The choice of station class matters: a high seasonal-traffic MPI may
just reflect that traffic emissions vary less seasonally than
background concentrations, so the absolute scale of the contrast is
small and noise-sensitive.

### Mixed networks

Like the spatial gradients, each station's Δ_s is normalised by
**its own** `D = √(1+β²_long)·U(Ō_s)` in the denominator, so cells
that include both fixed and indicative stations of the same class
already have each contribution correctly scaled. The 3×3 grid is
populated using all stations of the relevant class regardless of
measurement type. To restrict the grid to a single type, filter the
network upstream and re-run.

---

## Cross-walk to reference implementations

`fmm_assess` is a Python port. This section documents how each
indicator aligns with the two reference implementations.

### vs MQOR (R package)

MQI, TI and SI are **identical** to MQOR — same formulas, same
coefficients, verified numerically. The AAQD parameter table
[`uncertainty_aaqd.csv`](../data/uncertainty_aaqd.csv) is aligned
with the normative rows of MQOR's `data-raw/params.csv`.

| Indicator | fmm_assess | MQOR R function |
|---|---|---|
| `MQI_short` | `RMSE(O,M) / D_short` | `vec_mqi(term='short')` |
| `MQI_long` | `\|M̄ − Ō\| / D_long` | `vec_mqi(term='long')` |
| `TI_B` | `(M̄ − Ō) / D_short` | `vec_pi_bias2` |
| `TI_R` | `√(2·σ_M·σ_O·(1−R)) / D_short` | `vec_pi_cor2` |
| `TI_σ` | `(σ_M − σ_O) / D_short` | `vec_pi_sd2` |
| `SI_RMSE` | `√(mean((M̄_s − Ō_s)²)) / D_sp` | `vec_si_rmse(term='long', station means)` |
| `SI_B` | `(mean(M̄_s) − mean(Ō_s)) / D_sp` | `vec_pi_bias(term='long', station means)` |
| `SI_R` | `√(2·σ_M̄s·σ_Ōs·(1−R_sp)) / D_sp` | `vec_pi_cor(term='long', station means)` |
| `SI_σ` | `(σ_M̄s − σ_Ōs) / D_sp` | `vec_pi_sd(term='long', station means)` |

**Numerical verification** (5-station × 200-hour synthetic NO₂ case):

- Per-station TI max |diff|: **2.78 × 10⁻¹⁷**
- Network SI max |diff|: **3.47 × 10⁻¹⁸**
- MQI max |diff|: **0.0 (exact)**

All below the float64 round-off floor (~10⁻¹⁵).

### vs IDL DeltaToolLight (AAQD only)

`MPI_coherence` (W-S / Wk-We / D-N × T/B/I) and `MPI_UT-UB`, `MPI_UB-RB`
follow the **IDL DTL convention**, not MQOR's. These indicators do not
exist in MQOR.

The IDL alignment is verified against a reference Python port of the
IDL routine (transcribed literally from `DTLprog.pro`):

| Indicator | fmm_assess | IDL DTL |
|---|---|---|
| MPI coherence per-station | signed Δ_s, single per-station D | identical |
| MPI coherence cell | p90 over `\|Δ_s\|` | identical |
| MPI gradient per-station | signed `g_s`, **split denominators** | identical |
| MPI gradient cell | p90 over `\|g_s\|` | identical |
| Nearest-neighbour distance | equirectangular | identical to IDL (some other ports use haversine but give the same neighbour for typical AQ spacing) |

**Numerical verification** against the reference IDL Python port
(synthetic case):

- Coherence W-S per-station max |diff|: **5.5 × 10⁻¹⁷**
- Coherence W-S p90 diff: **0.0**
- Gradient UT-UB per-station max |diff|: **0.0**
- Gradient UT-UB p90 diff: **0.0**

### Indicators not in MQOR

| Indicator | In MQOR? | In DTL? | In fmm_assess? |
|---|---|---|---|
| MQI_short, MQI_long | ✓ | ✓ | ✓ (matches MQOR) |
| TI_B, TI_R, TI_σ | ✓ | ✓ (algebraically different) | ✓ (matches MQOR) |
| SI_RMSE, SI_B, SI_R, SI_σ | ✓ | partial (only SI_R, SI_σ, with different form) | ✓ (matches MQOR) |
| MPI temporal coherence (3x3) | — | ✓ | ✓ (matches DTL) |
| MPI spatial gradients | — | ✓ | ✓ (matches DTL) |

### Decision rule for future indicators

Where IDL DTL and MQOR disagree on a formula:
- For indicators **present in both** (MQI, TI, SI), MQOR wins.
- For indicators **only in DTL** (MPI coherence, MPI gradients), DTL
  wins.

See [`CONVENTIONS.md` — Reference-implementation alignment](CONVENTIONS.md#reference-implementation-alignment)
for the full decision matrix and rationale.

---

*Cross-references throughout: `GD §x.y` = FAIRMODE Guidance Document
v4.0 (JRC146736). For plot-reading guidance see
[`PLOTS.md`](PLOTS.md). For the conventions cheat-sheet (signing
rules, σ ddof, percentile rule) see [`CONVENTIONS.md`](CONVENTIONS.md).*
