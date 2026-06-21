# European Electricity Demand and Wind Resource Analysis

A reproducible, end-to-end pipeline for two complementary studies on the European energy transition:

1. **Behind-the-Meter (BTM) Solar Estimation** — inferring rooftop photovoltaic self-consumption from Germany's grid demand signal (2020–2025).
2. **Wind Power Density (WPD) Forecasting** — computing and mapping Europe-wide WPD from NOAA GFS ensemble forecasts and ranking countries by short-term wind resource potential.

---

## Repository Structure

```
.
├── gfs_data_downloader.ipynb          # Step 1 – download GFS atmospheric forecasts
├── btm_solar_estimation.ipynb         # Step 2 – BTM solar estimation (Germany)
├── wind_power_density_analysis.ipynb  # Step 3 – WPD mapping & country ranking
│
├── countries/
│   └── country_bbox.json              # Bounding-box definitions (see §3 below)
│
├── gfs_data/                          # GFS NetCDF outputs (created by Step 1)
│   └── gfs_Europe_YYYYMMDD_run??Z_3day.nc
│
├── figures/                           # Saved WPD maps (created by Step 3)
│
│── germany_electricity_demand_observation_q2.csv
├── germany_solar_observation_q2.csv
├── germany_atm_features_q2.csv
│
├── btm_solar_estimation_hourly.csv    # Hourly BTM estimates (output of Step 2)
├── monthly_true_demand_analysis.csv   # Monthly aggregation (output of Step 2)
└── country_wind_ranking_per_run.csv   # Per-run country ranking (output of Step 3)
```

---

## 1. Background and Motivation

### BTM Solar Estimation

Grid-level electricity demand statistics systematically **under-report** true consumption
wherever behind-the-meter (rooftop) PV is widespread.  Self-consumed solar generation
never crosses the metering boundary, so observed demand shrinks even as underlying load
grows — due to electrification of heating, transport, and industry.

The effect is material: Germany's installed BTM capacity exceeded 80 GW by 2025.
Separating the solar signal from genuine load changes matters for:

- Capacity planning and grid investment
- Demand forecasting models trained on historical metered data
- Policy assessment of electrification programmes

### Wind Power Density Forecasting

Short-range (0–72 h) WPD forecasts drive both day-ahead electricity market bidding and
wind-farm operational decisions.  GFS provides four daily initialisation runs (00Z, 06Z,
12Z, 18Z), each yielding a slightly different wind-field estimate.  Comparing runs
quantifies intra-ensemble spread and gives a simple, parameter-free measure of forecast
uncertainty — without requiring full probabilistic ensembles.

---

## 2. Notebooks

### `gfs_data_downloader.ipynb`

Downloads sub-regional GFS atmospheric forecasts from the UCAR THREDDS Data Server via
the NCSS (Network Common Subsetted Service) interface.

**What it does:**
- Queries the GFS 0.25° global grid for four variables at 100 m and 80 m height:
  U-wind, V-wind, temperature, pressure.
- Iterates over all four daily model runs (00Z, 06Z, 12Z, 18Z).
- Crops to a user-specified country bounding box.
- Saves one NetCDF file per run: `gfs_<Country>_<YYYYMMDD>_run<HH>Z_3day.nc`.

**Runtime:** ≈ 7–10 minutes per execution (network-bound).

---

### `btm_solar_estimation.ipynb`

Estimates hourly behind-the-meter solar generation in Germany from 2020 to 2025 using a
night-only machine-learning demand baseline.

**Inputs:**

| File                                            | Content                               |
|-------------------------------------------------|---------------------------------------|
| `germany_electricity_demand_observation_q2.csv` | Hourly metered grid demand (MW)       |
| `germany_solar_observation_q2.csv`              | Hourly grid-injected solar power (MW) |
| `germany_atm_features_q2.csv`                   | ERA5 atmospheric reanalysis features  |

**Method:**

1. **Feature engineering** — cyclical time encodings (hour, day-of-year), HDD/CDD
   temperature features, interaction terms, and an astronomically precise daylight flag
   (via `pvlib`).  Target and solar columns are excluded to prevent leakage.

2. **Night-only training** — an `ExtraTreesRegressor` is trained exclusively on hours
   where grid solar power < 50 MW *and* solar elevation < 6°.  During these hours,
   observed demand equals true demand by construction.  Hyperparameters are selected via
   randomised search with 5-fold time-series cross-validation.

3. **Full-period prediction** — the trained model predicts counterfactual *true demand*
   for all 47,000+ hours in the dataset.

4. **BTM displacement** — the non-negative gap between predicted true demand and observed
   demand estimates rooftop self-consumption:
   `BTM = max(0, predicted_true_demand − observed_demand)`.

5. **Temporal aggregation** — monthly averages reveal long-run trends in both BTM solar
   and underlying demand relative to the January 2020 baseline.

**Key model performance (validation set):**

| Metric         | Value     |
|----------------|-----------|
| Validation R²  | 0.921     |
| Validation MAE | ~1,800 MW |
| Validation RMSE| ~2,580 MW |

**Outputs:**
- `btm_solar_estimation_hourly.csv` — hourly observed demand, predicted true demand, and BTM estimate.
- `monthly_true_demand_analysis.csv` — monthly aggregation with growth percentages.
- `btm_solar_displacement.png` — two-panel visualisation.

---

### `wind_power_density_analysis.ipynb`

Processes the four GFS NetCDF files produced by `gfs_data_downloader.ipynb` to generate
daily WPD maps and a country-level wind resource ranking.

**Inputs:** `gfs_data/gfs_Europe_*.nc` (four files, one per run hour).

**Method:**

1. **WPD computation** — for each run file, extracts U and V wind at 100 m, temperature
   at 100 m, and pressure at 80 m; computes air density via the ideal gas law; computes
   `WPD = 0.5 × ρ × v³`.

2. **Ensemble concatenation** — the four run-hour DataArrays are concatenated along a
   `run_hour` dimension into a single `(run_hour, time, latitude, longitude)` DataArray.

3. **Daily maps** — the ensemble mean is computed first, then resampled to daily means.
   Each day's map is plotted with `cartopy` in PlateCarree projection, saved to `figures/`.

4. **Country ranking** — for each run separately, the full 3-day WPD mean is computed
   and zonal statistics (mean WPD per country polygon) are extracted via `rasterstats`.
   Countries are ranked by mean WPD within each run and the results are saved to CSV.

**Outputs:**
- `figures/wpd_europe_YYYY-MM-DD.png` — one map per forecast day.
- `country_wind_ranking_per_run.csv` — per-run country rankings.

---

## 3. Country Bounding-Box Configuration

`gfs_data_downloader.ipynb` requires a JSON file at `countries/country_bbox.json`.
The schema is:

```json
{
  "EU": ["Europe",  [270.0, 24.5,  45.0, 84.75]],
  "DE": ["Germany", [5.5,   47.0,  15.5, 55.5]],
  "FR": ["France",  [354.5, 41.0,  10.0, 51.5]]
}
```

Each key maps to `[country_name, [west_lon, south_lat, east_lon, north_lat]]`.
Longitudes must be in the **0–360° convention** used by the GFS NCSS endpoint.

---

## 4. Installation

```bash
pip install siphon xarray netCDF4 tqdm pandas numpy matplotlib \
            scikit-learn pvlib cartopy geopandas rioxarray \
            rasterstats affine shapely
```

> **Note:** `cartopy` requires system-level PROJ and GEOS libraries.
> On Ubuntu/Debian: `sudo apt-get install libproj-dev libgeos-dev`.
> Conda users: `conda install -c conda-forge cartopy`.

---

## 5. Workflow

Run the notebooks in order:

```
1. gfs_data_downloader.ipynb        →  produces gfs_data/*.nc
2. btm_solar_estimation.ipynb       →  produces btm_solar_estimation_hourly.csv
                                        monthly_true_demand_analysis.csv
3. wind_power_density_analysis.ipynb →  produces figures/*.png
                                        country_wind_ranking_per_run.csv
```

Notebooks 2 and 3 are independent of each other and can be run in any order once
Step 1 is complete.

---

## 6. Key Results (August 2025 sample run)

### BTM Solar Estimation (Germany, 2020–2025)

- Observed grid demand has **declined** relative to the 2020 baseline, but the night-only
  model reveals that **true underlying demand has grown** — the apparent decline is caused
  entirely by the masking effect of behind-the-meter solar self-consumption.
- BTM solar displaced approximately **9,200 GWh** of metered demand in 2024, equivalent
  to roughly 1.5% of Germany's annual electricity consumption.
- The divergence between observed and true demand growth widens year-on-year, consistent
  with continued rooftop PV deployment.

### Wind Power Density Ranking (Europe, 6 August 2025)

| Rank | Country        | Mean 3-day WPD (W m⁻²) |
|------|----------------|------------------------|
| 1    | France         |       ~2,500           |
| 2    | Ireland        |       ~1,850           |
| 3    | Italy          |       ~960             |
| 4    | Belgium        |       ~780             |
| 5    | United Kingdom |       ~710             |

Rankings are highly consistent across all four GFS runs, indicating low forecast spread
and high confidence for this particular synoptic situation.

---

## 7. Limitations and Future Work

- **BTM baseline year:** January 2020 is used as the reference month for growth
  percentages.  Results should be interpreted relative to this specific starting point,
  not as absolute annual statistics.
- **Night-mask sensitivity:** The BTM estimation assumes that hours with grid solar
  < 50 MW and solar elevation < 6° are fully free of BTM generation.  Systematic errors
  in this mask propagate into the displacement estimates.
- **GFS resolution:** The 0.25° grid (≈ 25 km) may smear fine-scale topographic and
  coastal effects.  Sub-km downscaling (e.g., WRF) would improve site-level WPD estimates.
- **Short forecast window:** Three days is insufficient for climatological resource
  assessment.  An archive of GFS forecasts over multiple months would yield more robust
  country rankings.

---

## 8. Data Sources

| Dataset                    | Provider                         | Access                                                          |
|----------------------------|----------------------------------|-----------------------------------------------------------------|
| GFS 0.25° Forecasts        | NOAA / NCEP                      | [UCAR THREDDS](https://thredds.ucar.edu/thredds/catalog/grib/NCEP/GFS/Global_0p25deg/catalog.html) |
| Germany Electricity Demand | ENTSO-E / Open Power System Data | Included in repo (2020–2025)                                    |
| Germany Solar Generation   | ENTSO-E / Open Power System Data | Included in repo                                                |
| ERA5 Atmospheric Features  | ECMWF via CDS                    | Included in repo                                                |
| Country Polygons           | Natural Earth                    | Downloaded at runtime from [naciscdn.org](https://naciscdn.org) |
