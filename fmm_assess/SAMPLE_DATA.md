# Sample data — demo dataset walkthrough

The `sample_data/` directory at the root of the repository contains a
complete, ready-to-run example: hourly observations and model output for
50 European monitoring stations, four pollutants, covering the full year
2021. No additional data download is required.

---

## Directory layout

```
sample_data/
├── startup50_FINAL.ini          station metadata (coordinates, types, variables)
├── run_ModCsv_assessment.yaml   run configuration — point fmm_assess here
└── CSV/
    ├── obs50/                   observations — one CSV per station (50 files)
    └── mod50_FH0/               model output — one CSV per station (50 files)
```

Each CSV file is named after the EEA station code (e.g. `AT30104.csv`)
and contains semicolon-separated hourly data:

```
year;month;day;hour;NO2;O3;PM25;PM10
2021;1;1;0;4.5;44.0;12.32;14.27
2021;1;1;1;5.0;41.5;13.77;15.77
2021;1;1;2;6.0;38.5;14.74;16.77
...
```

Missing observations are encoded either as `-999` or as empty fields
(both are handled by the reader). The header row is fixed in this
exact order; only the four pollutant columns visible above are read by
the pipeline.

**Dataset characteristics:**

| Property | Value |
|---|---|
| Period | 1 January – 31 December 2021 |
| Stations | 50 (pan-European, diverse station types and areas) |
| Pollutants | NO₂, O₃, PM₂.₅, PM₁₀ |
| Time resolution | Hourly |
| File format | CSV (semicolon-delimited) |

---

## The startup file — `startup50_FINAL.ini`

The startup file supplies station metadata to the pipeline. It has
three sections.

**`[MODEL]`** — dataset-level attributes:

```ini
[MODEL]
;Year
;frequency
;Scale
2021
hour
urban
```

These three lines declare the reference year, the temporal resolution
(`hour`), and the default spatial scale.

**`[PARAMETERS]`** — pollutants present in the CSV files:

```ini
[PARAMETERS]
;Specie*type*measureunit
NO2;GAS;ugm-3
O3;GAS;ugm-3
PM25;PM;ugm-3
PM10;PM;ugm-3
```

Each line declares a pollutant, its physical type (`GAS` / `PM`), and
its measurement unit. The pollutant names here must match the column
headers in the CSV files.

**`[MONITORING]`** — one row per station, with a fixed 12-column header:

```ini
[MONITORING]
StationCode;StationName;Stationabbreviation;Altitude;Lon;Lat;GMTLag;Region;StationType;AreaType;Siting;listOfvariables
AD0945A;AD0945A;3;2515.00;1.716990;42.534900;GMT;AD;background;rural;Encamp;O3*;
AT2M611;AT2M611;22;470.00;14.379400;46.760600;GMT;AT;background;suburban;Carinthia;O3*;
AT30104;AT30104;32;265.00;14.519500;48.192200;GMT;AT;traffic;rural;LowerAustria;NO2*O3*PM25*PM10*;
...
```

| Column | Meaning |
|---|---|
| `StationCode` | EEA short code; **must match** the CSV filename (`<StationCode>.csv`) |
| `StationName` | Human-readable label used on plots |
| `Stationabbreviation` | Short numeric/alphanumeric ID |
| `Altitude` | Metres above sea level |
| `Lon`, `Lat` | WGS-84 decimal degrees |
| `GMTLag` | UTC offset (string; `GMT` = no offset) |
| `Region` | Country / sub-region label (free-text) |
| `StationType` | `background` ∣ `traffic` ∣ `industrial` |
| `AreaType` | `urban` ∣ `suburban` ∣ `rural` |
| `Siting` | Free-text local-context label |
| `listOfvariables` | `*`-separated pollutant codes this station measures (e.g. `NO2*O3*PM10*`) |

A station is included for a given pollutant only if that pollutant
appears in its `listOfvariables`. The `StationType` × `AreaType`
combination drives the SI / MPI classification (UB, RB, UT,
industrial/traffic/background coherence cells, etc.).

An optional 13th column `MeasurementType` (`fixed` | `indicative`) can
be appended to a station's row to override the run-level default for
that station. When absent or blank, the YAML `measurement_type` field
is used.

---

## The run configuration — `run_ModCsv_assessment.yaml`

```yaml
run_id: NO2
pollutant: NO2
year: 2021

aggregations:
  - short
  - long

uncertainty_def: aaqd
measurement_type: fixed

resolution_short: hourly
resolution_long:  annual

min_data_capture: 0.75
min_stations: 10
include_station_types: [background, traffic, industrial]
include_area_types:    [urban, suburban, rural]

inputs:
  startup_ini: ./startup50_FINAL.ini
  obs_dir:    ./CSV/obs50
  model_file: ./CSV/mod50_FH0
  uncertainty: null

outputs:
  dir: ./out_no2_aaqd_final
  formats: [png, pdf]
  dpi: 200

timeseries:
  stations: [""]
  nuts:     []
  show_obs: true
```

**Key fields:**

| Field | Meaning |
|---|---|
| `run_id` | Free-text run identifier; appended to every output filename |
| `pollutant` | One of `NO2`, `O3`, `PM10`, `PM25`, `SO2`, `CO`, `benzene`, `lead`, `arsenic`, `cadmium`, `nickel`, `bap` |
| `year` | Reference year (used for time-axis labelling and seasonal slicing) |
| `aggregations` | Subset of `[short, long]` — selects which MQIs to compute |
| `uncertainty_def` | `aaqd` (AAQD piecewise uncertainty curve) |
| `measurement_type` | `fixed` ∣ `indicative` — run-level default; per-station overrides take precedence (see startup file) |
| `resolution_short`, `resolution_long` | Optional time-resolution disambiguators (e.g. `hourly` ∣ `daily` for short-term NO2). When omitted, the FAIRMODE normative choice for the pollutant is used |
| `min_data_capture` | 75 % rule on the active aggregation |
| `min_stations` | Below this, the network 90th-percentile is unreliable (warning logged) |
| `include_station_types`, `include_area_types` | Filters applied before any indicator is computed |
| `inputs.startup_ini` | Path to the startup `.ini` (relative to YAML location) |
| `inputs.obs_dir` | Either a directory of per-station CSVs **or** a single NetCDF file |
| `inputs.model_file` | Same — directory or NetCDF; auto-detected |
| `inputs.uncertainty` | `null` for bundled defaults; otherwise a path to a custom CSV |
| `outputs.dir` | Where CSVs, logs and figures are written |
| `outputs.formats` | Subset of `[png, pdf]` for matplotlib figures (plotly figures are always written as HTML) |
| `timeseries.stations` | Optional explicit list of station codes for time-series panels |
| `timeseries.nuts` | NUTS-region prefix filter (alternative to explicit list); panels show the mean across matching stations |

All paths in the YAML are resolved **relative to the YAML's parent
directory**, so the example above can be run from anywhere as long as
the file structure under `sample_data/` is preserved.

---

## Running the example

From a Python shell:

```python
from fmm_assess.pipeline import run_assessment

res = run_assessment("sample_data/run_ModCsv_assessment.yaml")
print(f"MQI_short p90 = {res.aggregate.MQI_short_p90:.3f}  "
      f"({'PASS' if res.aggregate.MQO_short_pass else 'FAIL'})")
print(f"MQI_long  p90 = {res.aggregate.MQI_long_p90:.3f}  "
      f"({'PASS' if res.aggregate.MQO_long_pass else 'FAIL'})")
```

From the command line (after editing `cli.py` to point at the YAML):

```bash
python -m fmm_assess.cli
```

The pipeline typically finishes in well under a minute on the demo
dataset.

---

## Expected outputs

After a successful run, `sample_data/out_no2_aaqd_final/` will contain:

```
out_no2_aaqd_final/
├── per_station_NO2.csv                     one row per station, all indicators
├── summary_NO2.csv                         pooled network-level scalars
├── summary_by_type_NO2.csv                 fixed / indicative split (AAQD Annex VI)
├── summary_temporal_coherence_NO2.csv      3×3 temporal-coherence grid
├── run.log                                 timestamped run log
├── mplotl/                                 matplotlib PNG / PDF figures
│   ├── radar.{png,pdf}
│   ├── target.{png,pdf}
│   ├── ts_report.{png,pdf}
│   ├── scatter.{png,pdf}
│   ├── taylor.{png,pdf}
│   ├── scatter_dyneval.{png,pdf}
│   ├── bars.{png,pdf}
│   └── timeseries_*.{png,pdf}
└── pltly/                                  plotly interactive HTMLs
    ├── radar.html
    ├── target.html
    └── ...                                  (same set of eight diagrams)
```

The two backends produce **bit-identical numerical content** — same
indicators, same dot positions, same pass/fail colouring. The difference
is interactivity, not data.

**Per-station CSV columns** (`per_station_<run_id>.csv`):

`station, station_type, area_type, measurement_type, n_valid_short,
n_valid_long, mean_O, mean_M, sigma_O, sigma_M, R, BIAS, RMSE, CRMSE,
O_bar, M_bar, D_short, D_long, MQI_short, MQI_long, TI_B, TI_R, TI_sigma,
dWS_obs, dWS_mod, dWkWe_obs, dWkWe_mod, dDN_obs, dDN_mod, coh_seasonal,
coh_week, coh_day, MPI_UT_UB, MPI_UB_RB`.

The last five columns are the per-station §3.7.3 contributions (the
network-level 90th-percentile of these reproduces the corresponding
cell in `summary_temporal_coherence_<run_id>.csv` and the `MPI_UT_UB`
/ `MPI_UB_RB` scalars in `summary_<run_id>.csv`).

---

## Trying other pollutants

The bundled `run_ModCsv_assessment.yaml` is configured for NO₂. To
assess O₃, PM₁₀ or PM₂.₅ instead, change the four lines at the top:

```yaml
run_id: O3              # any free-text label — appears in output filenames
pollutant: O3
resolution_short: max daily 8hr mean   # required for O3 short
resolution_long:  seasonal             # required for O3 long
```

For PM10 / PM25, set `resolution_short: daily` and `resolution_long: annual`.
See [`TECH_SPEC_fmm_assess.md`](TECH_SPEC_fmm_assess.md) §4 for the full
valid-value matrix per pollutant.

---

## Customising the dataset

The demo dataset is intentionally small (50 stations, one model
realisation, one year). The same pipeline scales to:

- larger station networks (hundreds of stations) — the only constraint
  is that every station appearing in `[MONITORING]` has matching CSV
  filenames under `obs_dir` and `model_file`;
- NetCDF inputs — replace `obs_dir` / `model_file` with paths to
  `.cdf` / `.nc` / `.nc4` files (DeltaTool format, one variable per
  station, with a global `Parameters` attribute listing the species
  on the V axis);
- mixed fixed + indicative networks — append a 13th `MeasurementType`
  column to the relevant `[MONITORING]` rows.

For everything else, see the [Full specification](TECH_SPEC_fmm_assess.md).
