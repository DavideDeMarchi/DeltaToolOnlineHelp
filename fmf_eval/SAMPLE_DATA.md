# Sample data — demo dataset walkthrough

The `sample_data/` directory at the root of the repository contains a
complete, ready-to-run example: hourly observations and day-ahead
forecasts for 50 European monitoring stations, four pollutants, three
forecast horizons, covering the full year 2021.  No additional data
download is required.

---

## Directory layout

```
sample_data/
├── startup50.ini           station metadata (coordinates, types, variables)
├── run_ModCsv.yaml         run configuration — point fmf_eval here
└── CSV/
    ├── obs50/              observations  — one CSV per station (50 files)
    ├── mod50_FH0/          forecast horizon 0 — one CSV per station (50 files)
    ├── mod50_FH1/          forecast horizon 1 — one CSV per station (50 files)
    └── mod50_FH2/          forecast horizon 2 — one CSV per station (50 files)
```

Each CSV file is named after the EEA station code (e.g. `AT30104.csv`)
and contains semicolon-separated hourly data:

```
year;month;day;hour;NO2;O3;PM25;PM10
2021;1;1;0;23.04;2.30;62.0;77.0
2021;1;1;1;22.37;3.49;62.5;76.5
...
```

Missing values are encoded as `-999`.

**Dataset characteristics:**

| Property | Value |
|---|---|
| Period | 1 January – 31 December 2021 |
| Stations | 50 (pan-European, diverse station types and areas) |
| Pollutants | NO₂, O₃, PM₂.₅, PM₁₀ |
| Forecast horizons | FH0 (same-day), FH1 (+1 day), FH2 (+2 days) |
| Time resolution | Hourly |
| File format | CSV (semicolon-delimited) |

---

## The startup file — `startup50.ini`

The startup file supplies station metadata to the pipeline.  It has
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

**`[PARAMETERS]`** — pollutants available in the CSV files and their
physical type:

```ini
[PARAMETERS]
;Specie*type*measureunit
NO2;GAS;ugm-3
O3;GAS;ugm-3
PM25;PM;ugm-3
PM10;PM;ugm-3
```

**`[MONITORING]`** — one row per station, semicolon-separated:

```ini
[MONITORING]
StationCode;StationName;Stationabbreviation;Altitude;Lon;Lat;GMTLag;Region;StationType;AreaType;Siting;listOfvariables
AT30104;AT30104;32;265.00;14.519500;48.192200;GMT;AT;traffic;rural;LowerAustria;NO2*O3*PM25*PM10*;
BETN121;BETN121;260;430.00;5.202390;49.877100;GMT;BE;background;rural;Wallonia;NO2*O3*PM25*PM10*;
...
```

The `listOfvariables` column (last field, asterisk-delimited) declares
which pollutants are valid at each station.  Stations without a
given pollutant are automatically excluded from that pollutant's
evaluation.

---

## The run configuration — `run_ModCsv.yaml`

```yaml
run_id: data_sample
uncertainty_def: fairmode   # "aaqd" or "fairmode" (default)
pollutant: NO2
forecast_horizons: [0, 1, 2]
threshold_ugm3: 120         # exceedance threshold for categorical metrics
threshold_sensitivity: 1
aqi_scheme: EEA_AQI
min_data_capture: 0.75
min_stations: 4

inputs:
  startup_ini: ./startup50.ini
  obs_dir:     ./CSV/obs50
  model_files:
    0: ./CSV/mod50_FH0
    1: ./CSV/mod50_FH1
    2: ./CSV/mod50_FH2

outputs:
  dir:     ./out_NO2
  formats: [png]
  dpi:     150
```

Key fields:

| Field | Meaning |
|---|---|
| `run_id` | Label used in output filenames and figure titles |
| `uncertainty_def` | Uncertainty parametrisation: `fairmode` (quadratic, JRC146007 default) or `aaqd` (piecewise, AAQD-aligned) |
| `pollutant` | Which pollutant column to evaluate (`NO2`, `O3`, `PM25`, `PM10`) |
| `forecast_horizons` | List of FH indices; must match the keys in `model_files` |
| `threshold_ugm3` | Concentration threshold for exceedance-based categorical scores (FA, MA, GA+, etc.) |
| `threshold_sensitivity` | Number of additional threshold values to test above and below (0 = single threshold only) |
| `aqi_scheme` | AQI colour scale used for the AQI diagram (`EEA_AQI`, `CAQI`, `US_AQI`, or `IT_IQA`) |
| `min_data_capture` | Fraction of valid time steps required for a station to contribute (0.75 = 75 %) |
| `min_stations` | Minimum number of stations required to compute network-level summary indicators |
| `inputs.startup_ini` | Path to the `.ini` metadata file (relative to the YAML) |
| `inputs.obs_dir` | Directory of per-station observation CSVs, **or** a DeltaTool NetCDF file |
| `inputs.model_files` | Mapping of FH index → directory of per-station forecast CSVs, **or** a NetCDF file per FH |
| `outputs.dir` | Directory where CSVs and figures are written |
| `outputs.formats` | Figure formats: `png`, `pdf`, `svg` (matplotlib); plotly always writes HTML |
| `outputs.dpi` | Resolution for raster outputs |

All paths in the YAML are resolved relative to the YAML file itself,
so the configuration works from any working directory as long as you
pass the correct path to `run_evaluation()`.

---

## Running the demo

**From the command line** (run from the `sample_data/` directory):

```bash
cd sample_data
python -c "
from fmf_eval.pipeline import run_evaluation
run_evaluation('run_ModCsv.yaml')
"
```

**From Spyder or any IDE:**

1. Open `fmf_eval/cli.py`.
2. Set `CONFIG_PATH` to the absolute path of `sample_data/run_ModCsv.yaml`.
3. Press **F5** (or run the script).

**From Python directly:**

```python
from fmf_eval.pipeline import run_evaluation

results = run_evaluation("sample_data/run_ModCsv.yaml")

# Network-level summary for FH0
summary = results.summary[0]           # dict keyed by indicator name
print(f"MQIf p90 worst : {summary['MQIf_p90_worst']:.3f}")
print(f"MQOf fulfilled : {summary['MQOf_fulfilled_fraction']:.0%}")

# Per-station indicators for FH0
df = results.per_station[0]            # pandas DataFrame, one row per station
print(df[["station", "MQIf", "MFEf", "MFEp", "MPI1", "MPI2"]].head())
```

---

## Expected outputs

After a successful run, the `sample_data/out_NO2/` directory will
contain:

```
out_NO2/
├── indicators_per_station_FH0.csv
├── indicators_per_station_FH1.csv
├── indicators_per_station_FH2.csv
├── indicators_summary_FH0.csv
├── indicators_summary_FH1.csv
├── indicators_summary_FH2.csv
├── aqi_counts_FH0.csv
├── aqi_counts_FH1.csv
├── aqi_counts_FH2.csv
├── run.log
├── mplotl/
│   ├── forecast_summary_FH0.png
│   ├── forecast_summary_pnorm_FH0.png
│   ├── forecast_target_FH0.png
│   ├── forecast_mpi_FH0.png
│   ├── forecast_threshold_perf_FH0.png
│   ├── forecast_threshold_perf_norm_FH0.png
│   ├── forecast_pod_sr_sensitivity_FH0.png
│   ├── forecast_aqi_FH0.png
│   └── ... (same set for FH1, FH2)
└── pltly/
    ├── forecast_summary_FH0.html
    └── ... (interactive versions of the same eight diagrams per FH)
```

The `indicators_per_station_FH<n>.csv` files contain one row per
station with all computed indicators (BIAS, RMSE, MQIf, MFEf, MFEp,
MFU, MPI1, MPI2, contingency counts, ACC, SR, POD, FBIAS, TS, GSS,
and AQI category counts).  The `indicators_summary_FH<n>.csv` files
aggregate these to network-level scalars (p90 worst-case MQIf, MQOf
fulfilled fraction, mean MPI1/MPI2, etc.).

---

## Trying other pollutants

To evaluate O₃ instead of NO₂, copy `run_ModCsv.yaml` to
`run_ModCsv_O3.yaml` and change two lines:

```yaml
pollutant:      O3
threshold_ugm3: 120   # O3 target value (µg m⁻³)
outputs:
  dir: ./out_O3
```

Then run:

```python
run_evaluation("sample_data/run_ModCsv_O3.yaml")
```

The same CSV files are used; `fmf_eval` reads the `O3` column
automatically and applies the correct FAIRMODE uncertainty parameters
for ozone.

For PM₂.₅ (`pollutant: PM25`, `threshold_ugm3: 25`) and PM₁₀
(`pollutant: PM10`, `threshold_ugm3: 50`) the procedure is identical.
Note that some stations in `startup50.ini` do not carry all four
pollutants; those stations are silently excluded from runs for
pollutants not listed in their `listOfvariables` field.

---

## Further reading

- [`fmf_eval/docs/INDICATORS.md`](INDICATORS.md) — full formula
  reference for every indicator produced by the pipeline.
- [`fmf_eval/docs/PLOTS.md`](PLOTS.md) — how to read each of the
  eight diagnostic diagrams.
- [`fmf_eval/docs/TECH_SPEC.md`](TECH_SPEC.md) — complete YAML field
  reference and package architecture.
- [`fmf_eval/docs/CONVENTIONS.md`](CONVENTIONS.md) — NaN handling,
  persistence convention, percentile rule, and other implementation
  decisions.
