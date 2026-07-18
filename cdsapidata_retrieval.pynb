"""
=============================================================================
 COMPLETE GUIDE: ACQUIRING ERA5 DATA WITH cdsapi
=============================================================================
This file walks through every common way of pulling ERA5 data from the
Copernicus Climate Data Store (CDS), with inline comments explaining each
argument. Run only the section(s) you need — they are independent examples.
 
SETUP (do this once, outside of any script):
1. Create a free account at https://cds.climate.copernicus.eu
2. Go to your profile page and copy your API key (UID:APIKEY string)
3. Create a file at ~/.cdsapirc (Linux/Mac) or %USERPROFILE%\\.cdsapirc (Win)
   containing:
       url: https://cds.climate.copernicus.eu/api
       key: <your-uid>:<your-api-key>
4. pip install cdsapi xarray netCDF4 cfgrib eccodes
=============================================================================
"""
 
import cdsapi
import xarray as xr
import numpy as np
 
c = cdsapi.Client()  # reads credentials automatically from ~/.cdsapirc
 
 
# =============================================================================
# SECTION 1 — "reanalysis-era5-single-levels"
# The easiest, most common dataset: surface / single-level variables
# (2m temperature, precipitation, wind, etc.) via a friendly, non-MARS API.
# =============================================================================
 
c.retrieve(
    'reanalysis-era5-single-levels',  # dataset name — surface/single-level fields
    {
        'product_type': 'reanalysis',       # 'reanalysis' (best estimate) vs 'ensemble_mean/spread/members'
        'variable': [                        # human-readable variable names (not GRIB codes)
            '2m_temperature',
            '10m_u_component_of_wind',
            '10m_v_component_of_wind',
            'total_precipitation',
        ],
        'year': '2020',                      # single year as string
        'month': '04',                       # zero-padded month
        'day': [f'{d:02d}' for d in range(1, 31)],  # list of zero-padded days
        'time': [f'{h:02d}:00' for h in range(0, 24)],  # hourly steps, 'HH:MM'
        'area': [6, 95, 0, 100],             # [North, West, South, East] bounding box in degrees
        'grid': [0.25, 0.25],                # output resolution in degrees (native res is ~0.25°)
        'format': 'netcdf',                  # 'netcdf' or 'grib' — netcdf is safe here (regular fields)
    },
    'era5_single_levels.nc'                  # output filename
)
 
 

# =============================================================================
# SECTION 2 — "reanalysis-era5-complete" (raw MARS access)
# Use this ONLY when the variable you need isn't exposed in the friendly
# datasets above (e.g. model-level fields, spectral wave data, obscure
# GRIB parameters). Requests follow strict MARS syntax, not human names.
# Find parameter codes at https://apps.ecmwf.int/codes/grib/param-db
# =============================================================================
 
c.retrieve(
    'reanalysis-era5-complete',
    {
        'class': 'ea',                       # 'ea' = ERA5 (obsolete key, but harmless to include)
        'date': '2020-04-01/to/2020-04-30',  # MARS date range syntax
        'expver': '1',                       # 1 = ERA5 final, 5 = ERA5T (preliminary, near-real-time)
        'levtype': 'sfc',                    # 'sfc' surface, 'ml' model level, 'pl' pressure level
        'param': '167.128',                  # GRIB param code — 167.128 = 2m temperature
        'stream': 'oper',                    # 'oper' = atmospheric operational stream
        'time': '00/06/12/18',               # MARS shorthand: 'HH/HH/HH/HH', no colons needed
        'type': 'an',                        # 'an' = analysis, 'fc' = forecast
        'area': [6, 95, 0, 100],             # [N, W, S, E]
        'grid': '0.25/0.25',                 # MARS grid syntax uses '/' not a list
        'format': 'netcdf',                  # safe for regular scalar fields like this
    },
    'era5_complete_t2m.nc'
)
 
 
# =============================================================================
# SECTION 5 — ERA5 2D WAVE SPECTRA (special case, param 251.140)
# The wave model stores data per spectral bin (direction x frequency), which
# is a genuinely different shape of request. Two important caveats apply
# (see notes at the bottom of this section).
# =============================================================================
 
c.retrieve(
    'reanalysis-era5-complete',
    {
        'class': 'ea',
        'date': '2020-04-01/to/2020-04-30',
        'direction': '/'.join(str(i) for i in range(1, 25)),   # 24 directional bins, MARS wants indices 1-24
        'domain': 'g',                        # 'g' = global wave domain
        'expver': '1',
        'frequency': '/'.join(str(i) for i in range(1, 31)),   # 30 frequency bins, indices 1-30
        'param': '251.140',                   # GRIB code for 2D wave spectra
        'stream': 'wave',                     # wave model stream (not 'oper')
        'time': '00/06/12/18',
        'type': 'an',
        'area': [6, 95, 0, 100],
        'grid': '0.5/0.5',                    # wave model native res is coarser than atmosphere (~0.5°)
        'format': 'netcdf',                   # NOTE: see caveat 1 below
    },
    'era5_wave_spectra.nc'
)
 
# --- CAVEAT 1: NetCDF conversion loses real frequency/direction values ---
# The output file's 'direction' and 'frequency' dimensions will just be
# index numbers (1-24, 1-30), NOT the actual physical values, because the
# GRIB->NetCDF converter doesn't carry ECMWF's local spectral metadata.
# Reconstruct the real axes yourself after loading:
 
def reconstruct_wave_spectra_axes(ds: xr.Dataset) -> xr.Dataset:
    """Attach true direction (deg) and frequency (Hz) coordinates to a
    wave-spectra dataset that was retrieved in netcdf format, where the
    'direction' and 'frequency' dims are just index numbers 1-24 / 1-30."""
    n_dir = ds.sizes['direction']              # should be 24
    n_freq = ds.sizes['frequency']              # should be 30
 
    # Oceanographic convention: bin 1 starts at 7.5 deg, +15 deg per bin
    direction_deg = 7.5 + 15.0 * np.arange(n_dir)
 
    # Frequency bins grow geometrically: f(1)=0.03453 Hz, f(n)=f(n-1)*1.1
    frequency_hz = 0.03453 * (1.1 ** np.arange(n_freq))
 
    ds = ds.assign_coords(direction=direction_deg, frequency=frequency_hz)
    ds['direction'].attrs['units'] = 'degrees'
    ds['frequency'].attrs['units'] = 'Hz'
    return ds
 
# --- CAVEAT 2: values are log10 of spectral density, not the density itself ---
def to_spectral_density(ds: xr.Dataset, varname: str = 'd2fd') -> xr.DataArray:
    """Convert the raw log10(density) values ERA5 stores into actual
    spectral density (m^2 / (Hz * rad)). Missing/near-zero bins were
    discarded during compression and are effectively 0."""
    log_density = ds[varname]
    density = 10 ** log_density
    density = density.fillna(0.0)
    return density
 
# Example usage after retrieval:
#   ds = xr.open_dataset('era5_wave_spectra.nc')
#   ds = reconstruct_wave_spectra_axes(ds)
#   density = to_spectral_density(ds)
 
 
# =============================================================================
# SECTION 3 — GENERAL TIPS & GOTCHAS
# =============================================================================
#
# 1. FORMAT CHOICE: 'netcdf' is fine for regular lat/lon scalar fields
#    (temperature, wind, precip). For wave spectra or other GRIB-local-table
#    parameters, consider 'format': 'grib' + decoding with cfgrib/eccodes if
#    you need the real metadata preserved.
#
# 2. AREA ORDER: always [North, West, South, East] — a common source of
#    empty or malformed downloads if reversed.
#
# 3. TIME FORMAT: the friendly datasets (single-levels, pressure-levels,
#    land) want 'HH:MM' strings. The raw MARS interface
#    ('reanalysis-era5-complete') wants MARS shorthand like '00/06/12/18'
#    or '00/to/23/by/6' — colons are optional there.
#
# 4. GRID SYNTAX DIFFERS: friendly datasets take a list [0.25, 0.25];
#    the raw MARS interface wants a slash-separated string '0.25/0.25'.
#
# 5. expver: '1' = ERA5 final data; '5' = ERA5T (preliminary, used for the
#    most recent ~3 months before final ERA5 QC is complete). Mixed-period
#    netCDF requests can silently blend both — check the 'expver' variable
#    in the output if working near the present.
#
# 6. RATE LIMITS / QUEUE: reanalysis-era5-complete pulls from ECMWF's tape
#    archive (MARS) and can be much slower than the CDS-native datasets.
#    Large multi-month/global requests are best split into smaller chunks.
#
# 7. PARAMETER CODES: for anything not exposed as a friendly 'variable'
#    name, look up the GRIB code at https://apps.ecmwf.int/codes/grib/param-db
#    and pass it as 'param': '<code>.128' (or '.140' for wave params, etc.)
#
# 8. CHECK YOUR REQUEST FIRST: on the CDS dataset webpage, use the
#    "Show API request" / "View MARS request" button after building a
#    request in the web form — it generates a correct, ready-to-paste
#    dict for whichever dataset you're using.
# =============================================================================
 
