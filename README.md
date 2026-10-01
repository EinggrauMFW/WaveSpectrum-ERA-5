# WaveSpectrum-ERA-5

Retrieve ERA5 2D ocean wave spectra from the Copernicus Climate Data Store (CDS) and reduce them to wave statistics (Hs, Tp, Tm01, Tm02, Dpm, Dspr, ...) with [`wavespectra`](https://wavespectra.readthedocs.io/).

The repository covers the whole path from raw spectra to usable numbers:

```
CDS API (ECMWF ERA5, param 251.140)  ->  output.nc  ->  wavespectra  ->  spectrum plots, statistics, CSV time series
        cdsapidata_retrieval.py                          wavespectra.ipynb / onesitewavestatistic.ipynb
```

## Repository contents

| File | Purpose |
|---|---|
| `cdsapidata_retrieval.py` | Annotated guide to pulling ERA5 data with `cdsapi`: single-level fields, raw MARS access, and 2D wave spectra (Section 3). Includes helpers to rebuild the frequency/direction axes and convert log10 density back to spectral density. |
| `wavespectra.ipynb` | Loads the spectra with `wavespectra`, computes gridded statistics, plots a single 2D spectrum, and compares a peak-Hs and a calm sea state. |
| `onesitewavestatistic.ipynb` | Maps the available grid points, selects the grid point nearest to a target site, computes the full set of wave parameters there, and exports them to CSV. |
| `output.nc` | Example dataset (about 7.7 MB): ERA5 wave spectra for April 2020 over 0-6°N, 95-100°E. |

## Example dataset

`output.nc` was retrieved with the Section 3 request in `cdsapidata_retrieval.py`:

- **Source:** `reanalysis-era5-complete`, stream `wave`, parameter `251.140` (2D wave spectra), `expver=1`
- **Period:** 2020-04-01 to 2020-04-30, 6-hourly (00/06/12/18 UTC), so 120 time steps
- **Area:** `[N, W, S, E] = [6, 95, 0, 100]`, `0.5° x 0.5°` grid, so 13 x 11 points
- **Spectral bins:** 24 directions x 30 frequencies (0.0345 to 0.548 Hz, each bin 1.1x the previous one)

## Requirements

Python 3 with:

```
cdsapi  xarray  netCDF4  numpy  pandas  matplotlib  dask  wavespectra  cartopy
```

`cfgrib` and `eccodes` are only needed if you request GRIB instead of NetCDF. Cartopy is only used by the mapping code in `onesitewavestatistic.ipynb`.

```bash
pip install cdsapi xarray netCDF4 numpy pandas matplotlib dask wavespectra cartopy
```

## 1. Set up CDS access

1. Create a free account at <https://cds.climate.copernicus.eu>.
2. Copy your API key from your profile page.
3. Create `~/.cdsapirc` (on Windows, `%USERPROFILE%\.cdsapirc`):

   ```
   url: https://cds.climate.copernicus.eu/api
   key: <your-uid>:<your-api-key>
   ```

## 2. Retrieve wave spectra

The request that produced `output.nc`:

```python
import cdsapi

c = cdsapi.Client()
c.retrieve(
    "reanalysis-era5-complete",
    {
        "class": "ea",
        "date": "2020-04-01/to/2020-04-30",
        "direction": "/".join(str(i) for i in range(1, 25)),   # 24 direction bins
        "domain": "g",
        "expver": "1",
        "frequency": "/".join(str(i) for i in range(1, 31)),   # 30 frequency bins
        "param": "251.140",                                    # 2D wave spectra
        "stream": "wave",
        "time": "00/06/12/18",
        "type": "an",
        "area": [6, 95, 0, 100],                               # [North, West, South, East]
        "grid": "0.5/0.5",
        "format": "netcdf",
    },
    "output.nc",
)
```

To change the study area or period, edit `date`, `area` and `grid`.

Notes on `cdsapidata_retrieval.py`:

- **It runs three requests in sequence.** The file is a guide, and its three sections are independent examples. Running it as a whole script triggers all three downloads, so delete or comment out the sections you do not need. Section 3 is the one that produces wave spectra.
- **NetCDF output loses the real axes.** The `direction` and `frequency` dimensions come back as bin indices (1-24 and 1-30). `reconstruct_wave_spectra_axes()` rebuilds them: directions are `7.5 + 15 * n` degrees, and frequencies are `0.03453 * 1.1**n` Hz.
- **Values are log10 of density.** ERA5 stores the spectra as log10 of spectral density. `to_spectral_density()` converts them back to m²/(Hz·rad) and fills missing bins with 0. The notebooks do not need these two helpers, because `wavespectra.read_era5` returns spectra with real frequency and direction coordinates.
- **Large requests are slow.** `reanalysis-era5-complete` reads from the ECMWF MARS archive, so split long periods or big areas into smaller requests.
- **Check `expver` near the present.** `expver=1` is final ERA5 and `5` is preliminary ERA5T. Mixed-period requests can blend both.

## 3. Compute wave statistics

### Gridded statistics and spectrum plots: `wavespectra.ipynb`

```python
from wavespectra import read_era5

dset = read_era5("output.nc")                 # dims: time, lat, lon, freq, dir
hs = dset.spec.hs()                           # significant wave height
stats = dset.spec.stats(["hs", "tp", "tm01", "tm02", "dpm", "dspr"])

# One 2D spectrum (frequency x direction) at a single time and point
point = dset.sel(time="2020-04-01T00:00", lat=6, lon=96, method="nearest")
point.spec.plot(kind="contourf")
```

The notebook also exports a point time series (`point_timeseries.csv`), plots the Hs time series at one grid point, and compares the 2D spectrum at the time of maximum Hs with a calm period.

### Single-site analysis: `onesitewavestatistic.ipynb`

1. Lists every (lat, lon) in the dataset, handles both gridded and site-based layouts, and saves the list to `era5_coordinates.csv`.
2. Maps all grid points and the selected site on one shared extent, using Cartopy.
3. Selects the grid point nearest to `SITE_LAT` / `SITE_LON` (default `6, 95`).
4. Reports the record length, time step and number of steps.
5. Computes `hs, tp, tm01, tm02, dpm, dspr, dp, dm, gamma`, prints a summary and saves the time series to `selected_site_wave_stats.csv` with one plot per parameter.

To analyse another location, change `SITE_LAT` and `SITE_LON`. Cell 1 of this notebook uses a `dset` variable, so run `dset = read_era5("output.nc")` first if you run the cells out of order or in a fresh kernel.

### Result for the example dataset

At the grid point nearest 6°N, 95°E, April 2020:

| Parameter | Value |
|---|---|
| Hs | mean 1.30 m, min 0.92 m, max 1.96 m |
| Tp | mean about 14.0 s, min 11.4 s (long-period swell) |
| Peak Hs | 2020-04-12 12:00 UTC |
| Calm period (comparison case) | 2020-04-20 00:00 UTC |

## Statistics reference

| Name | Meaning |
|---|---|
| `hs` | Significant wave height |
| `tp` | Peak period |
| `tm01` | Mean period (first moment) |
| `tm02` | Zero-crossing period (second moment) |
| `dpm` | Mean direction at the spectral peak |
| `dspr` | Directional spreading |
| `dp` | Peak direction |
| `dm` | Mean direction |
| `gamma` | Peak enhancement (JONSWAP) parameter |

See the [`wavespectra` documentation](https://wavespectra.readthedocs.io/) for exact definitions.

## Data source and attribution

Contains modified Copernicus Climate Change Service information. ERA5 data: Hersbach, H. et al. (2023), *ERA5 hourly data on single levels from 1940 to present*, Copernicus Climate Change Service (C3S) Climate Data Store (CDS), <https://doi.org/10.24381/cds.adbb2d47>. Use of the data is subject to the [Copernicus licence](https://cds.climate.copernicus.eu/licences). Wave statistics are computed with [`wavespectra`](https://github.com/wavespectra/wavespectra).
