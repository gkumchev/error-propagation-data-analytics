# Wind turbine power forecast: how far off can it be?

Group project for *Intro to Data Analytics* (Frankfurt School, 2026).

We use one year (2025) of hourly SCADA data from a 1 MW wind turbine on the Bulgarian Black Sea coast to fit a power curve,
show how an error in the wind forecast grows in the power forecast, and build 90 % prediction intervals with conformal prediction
that hold, also for real day-ahead wind forecasts.

## Notebooks

Run them in order. Each one reads the CSV files written by the one before (in `data/`) and saves its slide figures to `Presentation/figures/`.

| Notebook | Content |
|---|---|
| `01_eda.ipynb` | Exploratory data analysis: wind distribution, seasonality, outages |
| `02_power_curve_bins_pchip.ipynb` | Data cleaning, 80/20 split, power curve (IEC method of bins + PCHIP), validation |
| `03_forecast_error.ipynb` | Error propagation: simulated wind noise (1 / 5 / 10 %) through the power curve |
| `04_conformal_prediction.ipynb` | Asymmetric split conformal prediction, classic vs. Mondrian (per wind bin) |
| `05_real_forecast_test.ipynb` | Test with real day-ahead wind forecasts from Open-Meteo |

## Main results

- Cleaning keeps 7 969 of 8 761 hours (91 %). The power curve has an error of about 25 kW (RMSE, leave-one-month-out).
- On the steep part of the curve a wind error comes out about **3× larger** in the power. Near rated power the error becomes one-sided,
  so a Gaussian ±σ band is no reliable interval.
- One conformal interval covers 90 % on average but only 55–67 % at 7–11 m/s. **Mondrian conformal prediction** (one interval per wind bin)
  covers 90–93 % in every bin and is 20–23 % narrower.
- Real day-ahead forecasts: 37 % wind error becomes 85 % power error. Calibrated on January–August and tested on September–December,
  89 % of the hours are inside the 90 % interval (Mondrian: 88–91 % in every bin).

## Data

The SCADA data of the turbine were made available to us for this course and are proprietary, so they are **not included** in this
repository: neither the raw export (`Wind_Data_WTG_01_20250101.xlsx`) nor the CSV files the notebooks derive from it in `data/`.
The notebooks are saved with their outputs, so you can follow the results without running them.

Included is `data/openmeteo_forecast_2025.csv`, the historical wind forecasts for our site from the public
[Open-Meteo Previous Runs API](https://open-meteo.com/en/docs/previous-runs-api) (ECMWF IFS, 80 m, made 0–7 days before each hour).
Notebook 05 uses them.

## Setup

Python 3 with `numpy`, `pandas`, `scipy`, `matplotlib` and `openpyxl` (all part of a standard Anaconda install).

## Rerun the project with your own SCADA data

The method works for any turbine with about one year of hourly data.

1. **Prepare the data.** One row per hour with the time, mean wind speed and mean power. Put the file in the project folder and set
   `FILE` in the first code cell of notebooks 01 and 02 (for a CSV file, use `pd.read_csv` instead of `pd.read_excel`).
   In the loading cell of both notebooks, rename your columns to ours:

   | Column | Our export | Content |
   |---|---|---|
   | `time` | `Time` | hour as date-time, local time |
   | `v` | `WindSpeedAve.(m/s)` | hourly mean wind speed in m/s |
   | `p` | `GeneratorOutput(kW)` | hourly mean power in kW |
   | `maint` | `MaintenanceStop Time(min)` | maintenance stop within the hour as `hh:mm:ss` (set it to 0 if you don't have it) |
   | `v_max` | `WindSpeedMax.(m/s)` | hourly maximum wind speed (only used in 01) |

2. **Adapt the settings to your turbine.** `CUT_IN` and `RATED_SPEED` (01), `STEEP_PART` (03) and the Mondrian bins `WIND_BINS`
   (03–05, the last bin should start at about rated wind speed). The plot axes are set for a 1 MW turbine (power up to 1150 kW).
3. **Run 01 → 05.** Create the folder `Presentation/figures/` first (02 creates `data/` itself). The markdown text and takeaways
   describe our turbine, so their numbers will differ from yours.
4. **Real forecasts (notebook 05).** The included forecasts are for our site. For your turbine, download them for its coordinates
   and the year of your data (05 needs the columns `time_utc`, `v_d1` and `v_d3`):

   ```python
   import pandas as pd
   import requests

   # hourly wind forecasts at 80 m, made 1 and 3 days before each hour
   lat, lon = 0.0, 0.0   # coordinates of your turbine
   names = {"wind_speed_80m_previous_day1": "v_d1", "wind_speed_80m_previous_day3": "v_d3"}
   params = {"latitude": lat, "longitude": lon, "models": "ecmwf_ifs025", "hourly": ",".join(names),
             "start_date": "2025-01-01", "end_date": "2025-12-31",
             "wind_speed_unit": "ms", "timezone": "GMT", "cell_selection": "land"}
   response = requests.get("https://previous-runs-api.open-meteo.com/v1/forecast", params=params, timeout=120)
   response.raise_for_status()
   forecasts = pd.DataFrame(response.json()["hourly"]).rename(columns={"time": "time_utc", **names})
   forecasts.to_csv("data/openmeteo_forecast_2025.csv", index=False)
   ```

   In 05, set the time zone of your turbine clock (`Europe/Sofia` for us), `SHIFT_HOURS` (+1 h matched our turbine best) and,
   if your data is not one calendar year, `CALIBRATION_MONTHS`.
