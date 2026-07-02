# Conventions cheat-sheet

A short reference card for the decisions baked into `fmm_assess`. Use
this document when verifying that `fmm_assess` matches a paper formula
or another tool. For full derivations see [`INDICATORS.md`](INDICATORS.md).

## Contents

- [Master denominator](#master-denominator)
- [Which β?](#which-β)
- [Statistical conventions](#statistical-conventions)
- [Sign conventions](#sign-conventions)
- [90th-percentile rule](#90th-percentile-rule-eq-5)
- [MPC and MQO criteria](#mpc-and-mqo-criteria)
- [Time conventions](#time-conventions)
- [Reference-implementation alignment](#reference-implementation-alignment)

---

## Master denominator

Every indicator in `fmm_assess` uses the same denominator from GD §3.3:

$$
D = \sqrt{1 + \beta^2} \cdot \mathrm{RMSU}_O
$$

| Denominator | Used for | Formula |
|---|---|---|
| `D_short` | MQI_short, TI rows | `√(1+β²_short) · RMSU_O` over paired hourly/daily obs |
| `D_long` | MQI_long (per-station) | `√(1+β²_long) · U(Ō_s)` for one station's annual mean |
| `D_sp` | SI rows (single scalar) | `√(1+β²_long) · RMSU_Ō` over station annual means |

The `√(1+β²)` factor is the *stringency* adjustment from the Guidance
Document.

The §3.7.3 coherence MPIs and §3.7.2 gradient MPIs use **each station's
own** `U_O(Ō_s)` per-station — same `D = √(1+β²_long)·U_O(Ō_s)`,
evaluated at that station's annual mean.

### Mixed measurement types (fixed vs indicative)

When a network contains **both fixed and indicative** stations, the
AAQD prescribes different uncertainty parameters (`α`, `U_{O,r}(RV)`,
and in general `β`) for the two types. `fmm_assess` resolves this at
two levels:

- **Per-station denominator (always per-station).** Each station's
  `D_short` and `D_long` are computed from **its own** `U_O` curve,
  selected by `Station.measurement_type` (`fixed` ⇒ fixed row of the
  parameter CSV, `indicative` ⇒ indicative row). The `RMSU_O` inside
  `D_short` is the RMS of the uncertainty curve evaluated at **that
  station's own time series**, not across stations. A fixed station is
  therefore always normalised by the fixed-station uncertainty, an
  indicative station by the indicative one. No averaging across types
  happens at the station level.
- **Network-level p90 aggregation.** For per-station indicators
  (MQI_short, MQI_long, TI_*, and the per-station MPI gradient /
  coherence distributions) the network value is the 90th percentile
  across stations. This aggregation step is reported **twice**:
  - In `summary_<run>.csv` (pooled), the p90 is taken over all
    stations regardless of type — useful for an overall MQO statement
    when both sub-networks are equally weighted.
  - In `summary_by_type_<run>.csv` (per-type), the p90 is taken
    within each measurement-type subset — this is the AAQD-aligned
    view, since fixed and indicative stations have separate `β` and
    MPC thresholds.
- **`D_sp` (the SI denominator) is the only genuinely network-level
  denominator** in the package. It uses `RMSU_Ō` aggregated across
  station annual means, and `β_long` from one parameter row — both of
  which would have to combine the two uncertainty regimes for a mixed
  network. There is no clean way to do this, so pooled `SI_*` are
  written as `NaN` in mixed runs and only the per-type SI values in
  `summary_by_type_<run>.csv` are meaningful. Matches MQOR's
  `calc_longterm_type_stats` policy.

See [`INDICATORS.md` — Mixed measurement types](INDICATORS.md#mixed-measurement-types-fixed--indicative)
for the per-indicator behaviour.

---

## Which β?

`β` is read from the AAQD uncertainty parameters CSV
([`uncertainty_aaqd.csv`](../data/uncertainty_aaqd.csv)). Which row gets
selected depends on the indicator's term:

| Indicator | β used | Reason |
|---|---|---|
| `MQI_short`, `TI_B`, `TI_R`, `TI_σ` | `β_short` | Short-term indicators — hourly or daily aggregation |
| `MQI_long`, `SI_*` | `β_long` | Long-term indicators — annual means |
| `MPI_UT-UB`, `MPI_UB-RB` | `β_long` | Spatial gradients use annual means |
| `MPI_coherence` (all 9 cells) | `β_long` | Coherence MPIs use seasonal/weekly/diurnal averages — the long-term parameters apply |

The mapping is identical to MQOR (`mqor::default_params` rows are
identified by `term ∈ {short, long}`).

---

## Statistical conventions

### Standard deviation

| Context | ddof | Why |
|---|---|---|
| Temporal σ (`σ_O`, `σ_M`) | **0** (`/N`) | Population estimator — matches `mqor::vec_sd(term="short")` and GD Eq. 2 |
| Spatial σ (`σ_Ō^sp`, `σ_M̄^sp`) | **1** (`/(N−1)`) | Sample estimator — matches `mqor::vec_sd(term="long")` for cross-station spread |

This is a deliberate asymmetry — the temporal σ treats the year as
the entire population (no inference beyond the observed period), while
the spatial σ treats the station network as a sample of the underlying
distribution of locations.

### Centred RMSE

Computed **directly** from the centred residuals:

$$
\mathrm{CRMSE} = \sqrt{\tfrac{1}{N} \sum_t \bigl((M_t - \bar M) - (O_t - \bar O)\bigr)^2}
$$

**Not** derived from the Pythagoras identity `√(RMSE² − BIAS²)` (which
would be more vulnerable to round-off when bias ≈ RMSE). Matches
`mqor::vec_crmse` exactly.

### Pearson correlation

`R = np.corrcoef(M, O)[0,1]`. Range [−1, +1]. Matches `mqor::vec_cor`
(which uses `stats::cor(method="pearson")`).

### NaN handling — paired drop

For any indicator computed from `(O, M)` pairs, time-steps where either
`O` or `M` is NaN are dropped from both before any statistic is
computed. This is the "paired complete-case" convention; `fmm_assess`
applies it at the start of every per-station statistic:

```python
mask = ~(np.isnan(o) | np.isnan(m))
o_clean, m_clean = o[mask], m[mask]
```

NaNs do **not** propagate silently — they are dropped, and the count
of dropped time-steps is logged.

The 75 %-completeness rule (GD §3.4.1) is applied **after** paired
drop — if fewer than 75 % of expected time-steps remain at a station,
the station is excluded from the network-level aggregations. See
[`fmm_assess/filters.py`](../filters.py).

---

## Sign conventions

| Indicator | Sign | Aggregation |
|---|---|---|
| `BIAS`, `TI_B` | signed | `p90(\|.\|)` |
| `TI_σ` | signed | `p90(\|.\|)` |
| `TI_R` | always ≥ 0 | `p90` (no abs needed — already non-negative) |
| `SI_B`, `SI_σ` | signed | no aggregation — already single scalars |
| `SI_R`, `SI_RMSE` | always ≥ 0 | no aggregation |
| MPI coherence Δ_s | signed per-station | `p90(\|Δ_s\|)` per cell |
| MPI gradient `g_s` | signed per-station | `p90(\|g_s\|)` |

**Rule of thumb**: per-station / per-time-step quantities are **signed**;
the absolute value is taken **only at the percentile aggregation step**.
This matches MQOR (`mqor::mqo_percentile` takes `abs()` internally,
line 22) and the IDL DTL reference Python port (`calc_perc` line 218).

### Why store signed values?

The signed per-station values are needed for:

- **`ts_report.png`** distribution rows — to show whether the worst
  stations cluster on one side of zero (systematic bias) or are
  symmetric.
- **`per_station_<run>.csv`** — to let users see direction.
- **MPI coherence interpretation** — "winter excess vs summer excess"
  has a meaningful sign that the absolute value would erase.

The aggregation `p90(|.|)` is what feeds the MPC pass criterion;
the signed per-station value is what feeds the visualisation.

---

## 90th-percentile rule (Eq. 5)

Linear-interpolation across stations, R-style 1-indexed; translated to
0-indexed Python by shifting:

```python
def percentile_90(values):
    s = sorted(v for v in values if np.isfinite(v))
    N = len(s)
    if N < 1:
        return float("nan")
    if N == 1:
        return s[0]
    N90 = int(0.9 * N)             # floor
    D90 = 0.9 * N - N90
    if N90 == 0:                    # very small N
        return s[0]
    if N90 >= N:
        return s[-1]
    return s[N90 - 1] + D90 * (s[N90] - s[N90 - 1])
```

Verified to match `mqor::mqo_percentile` exactly. **Not** equivalent to
`numpy.percentile(..., 90)` — numpy's default uses a different
interpolation kind.

---

## MPC and MQO criteria

### MQO (Modelling Quality Objective) — GD §3.4

The network passes the **MQO** if **both** of:

```
MQI_short_p90 ≤ 1   AND   MQI_long_p90 ≤ 1
```

Equivalently: `MQI ≤ 1` at at least 90% of stations, separately for
short and long terms.

### MPC (Modelling Performance Criteria) — GD §3.7

Per-indicator MPC for the complementary MPIs:

```
|TI_B|_90  ≤ 1     |TI_R|_90  ≤ 1     |TI_σ|_90  ≤ 1
|SI_B|     ≤ 1     SI_R       ≤ 1     |SI_σ|     ≤ 1     SI_RMSE  ≤ 1
MPI_UT-UB ≤ 1      MPI_UB-RB  ≤ 1
MPI_coh[contrast,class] ≤ 1   (for all 9 cells)
```

The MPC are diagnostic — they don't determine MQO pass/fail directly,
but they explain *why* the MQO passes or fails.

---

## Time conventions

### Local time

`local_time = UTC + GMTLag` per station. `GMTLag` is read from
`startup.ini` (12th column, or 13th if the optional `MeasurementType`
column is present).

All temporal-coherence MPIs (W-S, Wk-We, D-N) operate in **local
time**.

### Calendar definitions

| Period | Definition |
|---|---|
| **Winter** | December, January, February (DJF) |
| **Summer** | June, July, August (JJA) |
| **Weekday** | Monday – Friday |
| **Weekend** | Saturday – Sunday |
| **Day** | 10:00 – 17:59 local (hours 10–17 inclusive) |
| **Night** | 22:00 – 05:59 local (hours 22–23 and 0–5 inclusive) |

The Day/Night windows deliberately exclude the transition hours
(06:00–09:59 and 18:00–21:59) — these are the most sensitive to
boundary-layer mixing-height errors and would dominate the contrast
diagnosis.

The Day/Night MPI is computed **only for hourly aggregation**. For
daily aggregation the row appears as `NaN` in `summary_temporal_coherence_<run>.csv`
and as `no data` in `ts_report.png`.

### 75 %-completeness rule (GD §3.4.1)

A station's time series is **dropped from the network-level
aggregations** if fewer than 75 % of expected time-steps have a valid
paired `(O, M)`. The threshold is applied *after* paired NaN drop:

```
n_valid_paired / n_expected ≥ 0.75
```

A station that fails this check is logged but contributes nothing to
any aggregate. The corresponding row in `per_station_<run>.csv` still
appears (with the per-station numbers we could compute) but is flagged
as excluded.

---

## Reference-implementation alignment

`fmm_assess` is a Python port. Two reference implementations exist for
different parts of the indicator set.

### Indicator-by-indicator alignment

| Indicator | MQOR (R) | IDL DTL | fmm_assess follows |
|---|---|---|---|
| `MQI_short`, `MQI_long` | ✓ | ✓ (different U(O) for AAQD) | **MQOR** |
| `TI_B`, `TI_R`, `TI_σ` | ✓ | ✓ (different formula for TI_R) | **MQOR** |
| `SI_RMSE`, `SI_B`, `SI_R`, `SI_σ` | ✓ | partial | **MQOR** |
| `MPI_UT-UB`, `MPI_UB-RB` | not implemented | ✓ | **IDL DTL** |
| `MPI_coherence` (3×3 grid) | not implemented | ✓ | **IDL DTL** |

### Decision rule

Where IDL DTL and MQOR disagree on a formula:

- **Indicator present in both** (MQI, TI, SI) → **MQOR wins**. The
  Python port targets numerical equivalence with MQOR to machine
  precision (verified at ~10⁻¹⁷). Rationale: MQOR is the curated
  reference for the Guidance Document and the FAIRMODE/JRC
  cross-evaluation tooling.

- **Indicator only in DTL** (MPI coherence, MPI gradients) → **DTL
  wins**. The Python port targets numerical equivalence with the IDL
  DeltaToolLight and its companion reference Python port
  (`indicators/DTL/core/dtl.py`). Rationale: these indicators are not
  defined in MQOR, so the only normative reference is the IDL.

### Verified numerical alignment

- **TI, SI, MQI vs MQOR** (5-station × 200-hour synthetic NO₂ case):
  max |diff| 2.78×10⁻¹⁷ (TI), 3.47×10⁻¹⁸ (SI), 0.0 (MQI).
- **MPI coherence W-S vs IDL DTL reference port**: per-station max |diff|
  5.5×10⁻¹⁷; p90 diff 0.0.
- **MPI gradient UT-UB vs IDL DTL reference port**: per-station and p90 max |diff|
  both 0.0.
- **AAQD parameter table**: aligned with MQOR's normative rows in
  `data-raw/params.csv` (rows of `def == "aaqd"`).

### Notes

- **AAQD U(O) curve**: `fmm_assess` uses the FAIRMODE-WG3 piecewise
  form (`U_φ + O·(U_LV − U_φ)/LV` for O ≤ LV; `O · U_LV/LV` for O > LV).
  IDL DTL uses the legacy Δ-Tool form `rel_T · max(O, LV)` (flat
  at `rel_T·LV` below LV). For NO₂ short-term, the difference produces
  MQI values ~3× larger in `fmm_assess` than in IDL DTL at typical
  hourly concentrations. The fmm_assess form is what MQOR uses too.

- **TI_R differs structurally between MQOR/fmm_assess and IDL DTL**.
  MQOR: `√(2·σ_M·σ_O·(1−R)) / D` (square root in numerator, single
  denominator). DTL: `2·σ_M·σ_O·(1−R) / D²` (no square root, squared
  denominator). The DTL form is essentially `MQOR_TI_R²`. `fmm_assess`
  follows MQOR.

- **SI_RMSE and SI_B don't exist in IDL DTL** — they're MQOR-only
  indicators. `fmm_assess` computes them per MQOR.

- **Equirectangular vs haversine distance**: `fmm_assess` uses
  equirectangular (matches IDL literally). Some other reference ports
  use haversine. For typical AQ-monitoring station spacing (km to tens
  of km in mid-latitudes), both pick the same nearest neighbour.

---

*For full formulas and reading guides see
[`INDICATORS.md`](INDICATORS.md) and [`PLOTS.md`](PLOTS.md). For the
full specification (run config, mqor mapping, every decision and why)
see [`TECH_SPEC_fmm_assess.md`](TECH_SPEC_fmm_assess.md).*
