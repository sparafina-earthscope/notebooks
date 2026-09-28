# Forward Modeling in Cascadia with Green's Functions

This notebook builds synthetic seismograms for the Cascadia subduction zone from known sources and precomputed Green's functions: **source + Green's functions → predicted seismogram**. It follows Section B of Lecture 17 (equations 7–9) in MIT 12.510 *Introduction to Seismology*.

The notebook has two parts:

| Part | Source | What it does |
|---|---|---|
| 1 | 2001 M6.8 Nisqually intraslab earthquake | Builds a synthetic from six elementary Green's functions, checks that the superposition is exact, and compares the result with seismograms recorded in 2001. |
| 2 | Scenario M9.0 full-margin megathrust rupture | Divides the Slab2 plate interface into about 100 subfaults, computes one Green's function per subfault, and sums them with rupture delays. It then reuses the same Green's functions for a second hypocenter to show directivity, without new requests. |

The Green's functions come from the EarthScope [Syngine](https://ds.iris.edu/ds/products/syngine/) service. It extracts them from precomputed AxiSEM databases for 1D Earth models (default `ak135f_5s`). The results are long-period (longer than 20–30 s) and use a 1D Earth model, so they illustrate the method and are **not** shaking-hazard estimates. The notebook explains its limitations in detail.

## Contents

| File | Description |
|---|---|
| `cascadia_greens_forward_model.ipynb` | The notebook. |
| `environment.yml` | Conda environment for GeoLab (or any conda install). |

## Setting up in GeoLab

### 1. Start a server and clone the repository

1. Log in to GeoLab and launch a server with the default **GeoLab** image. The smallest server size is enough. The largest array, the Part 2 Green's-function bank, is a few tens of MB.
2. Open a terminal (**File → New → Terminal**) and clone the repository into your home directory:

   ```bash
   cd ~
   git clone https://github.com/sparafina-earthscope/notebooks.git
   ```

   The files are in `~/notebooks/greens-functions`. To get later updates, run `git pull` inside `~/notebooks`.

### 2. Check or create the environment

The notebook needs:

- **Required:** ObsPy 1.4.1 or later, NumPy, SciPy, Matplotlib, pandas, xarray, and requests. Earlier ObsPy versions don't recognize the `EARTHSCOPE` FDSN provider.
- **netCDF backend:** `netCDF4` or `h5netcdf`. The Slab2 grids are netCDF-4 (HDF5) files. The notebook lists these packages as optional, but Part 2 stops at the Slab2 step if neither is installed.
- **Optional:** Cartopy, for coastlines and borders on the maps. Without it, the maps use plain longitude/latitude axes.

Open the notebook and run the first code cell. It prints the version of every package and stops if a required one is missing. If everything is present, including `netCDF4` or `h5netcdf`, skip to step 3. Otherwise, choose one of these options.

**Option A: persistent environment (recommended).** It's stored in your home directory, so you build it once and it stays available across sessions. In the terminal, run:

```bash
cd ~/notebooks/greens-functions
conda env create -f environment.yml
conda activate cascadia-gf
python -m ipykernel install --user --name cascadia-gf --display-name "Python (cascadia-gf)"
```

The build takes a few minutes. Then open the notebook, go to **Kernel → Change Kernel…**, and select **Python (cascadia-gf)**.

**Option B: quick install into the default kernel.** Add this as the first cell of the notebook and run it:

```python
%pip install "obspy>=1.4.1" netCDF4
```

Then restart the kernel (**Kernel → Restart Kernel**). This is faster, but the packages are gone the next time you start a server, so you need to run the cell again each session. Cartopy is easier to install with conda (Option A) than with pip.

### 3. Run the notebook

Open `cascadia_greens_forward_model.ipynb` and choose **Kernel → Restart Kernel and Run All Cells**.

- **Network access.** The notebook contacts `service.earthscope.org` (Syngine, FDSN stations, and waveforms), `earthquake.usgs.gov` (the ComCat moment tensor), and `www.sciencebase.gov` (Slab2).
- **First run.** Part 1 sends about 8 Syngine requests. Part 2 sends one request per subfault, about 100 in total, four at a time, which takes a few minutes.
- **Later runs.** Every response is cached, so a re-run with the same settings sends no Syngine requests. The notebook creates two folders next to itself:
  - `cache_cascadia_gf/`: Syngine responses, as miniSEED files named by a hash of the request.
  - `data_cascadia/`: the Slab2 grids.

  To force fresh downloads, delete these folders.

When you're done, stop your server (**File → Hub Control Panel → Stop My Server**) so it doesn't use up your compute quota.

## Setting parameters

All parameters are in the **Configuration** cell. Throughout the notebook, labels like `[A4]` mark modeling assumptions, and each one names the parameter that controls it. The ones you're most likely to change:

| Parameter | Default | Description |
|---|---|---|
| `MODEL` | `"ak135f_5s"` | Syngine Earth model. The notebook checks that it exists on the service. |
| `MT_SOURCE` | `"comcat"` | Nisqually source: the ComCat moment tensor, or `"ichinose2004"` for the published fault plane. |
| `P1_N_STATIONS`, `P1_MAXRADIUS_DEG` | `6`, `10.0` | Number of stations (one per azimuth sector) and search radius for Part 1. |
| `P1_PERIOD_BAND_S` | `(20.0, 100.0)` | Period band for the comparison with recorded data. |
| `DEPTH_EDGES_KM` | `[0, 7, 13, 19, 25]` | Down-dip rows of subfaults. The last value is the down-dip limit of slip. |
| `SLIP_DOWNDIP_WEIGHTS` | `[1.0, 1.0, 0.7, 0.35]` | Relative slip per row. It needs one value per row. |
| `HYPOCENTERS` | south and north | Rupture start points, as (latitude, depth in km). |
| `RUPTURE_VELOCITY_KMS` | `2.5` | Rupture speed. |
| `RECEIVERS` | six cities | Sites for the Part 2 synthetics. |
| `MAX_WORKERS` | `4` | Parallel Syngine requests. Keep this small, because Syngine is a shared public service. |

**What triggers new requests.** Changing slip, rupture velocity, rise time, or hypocenters reuses the Part 2 Green's-function bank, so the notebook sends no new Syngine requests. Changing the geometry does send new requests: `LAT_MIN`, `LAT_MAX`, `DLAT`, `DEPTH_EDGES_KM`, `RAKE_DEG`, `RECEIVERS`, `MODEL`, or `P2_REQUEST_S`.

## Troubleshooting

- **`ImportError: Required packages are missing`:** set up the environment as in step 2.
- **`Could not open ... with any netCDF engine`:** install `netCDF4` or `h5netcdf`, then restart the kernel.
- **`Syngine request failed after 3 attempts`:** the service is busy or unreachable. Wait a few minutes and re-run the cell. Requests that already succeeded are cached and aren't sent again.
- **`ComCat has no moment-tensor product`:** set `MT_SOURCE = "ichinose2004"`.
- **A station is `DROPPED` in Section 2.8:** its 2001 recording has a gap or no instrument response. The notebook leaves it out of the comparison and continues. It stops only if no station has usable data.
- **Slab2 download fails:** download `cas_slab2_dep_02.24.18.grd`, `cas_slab2_str_02.24.18.grd`, and `cas_slab2_dip_02.24.18.grd` from the [ScienceBase item](https://www.sciencebase.gov/catalog/item/5aa312cde4b0b1c392ea3ef5) and put them in `data_cascadia/`. The notebook uses local files first.
- **A `[C#]` check raises `AssertionError`:** the notebook tested an assumption and the test failed. The error message says which assumption failed and what to check. Don't continue past a failed check.
