# GEMS Challenge: Fault Mapping and Geothermal Candidates in Western Nevada

These three notebooks predict fault locations in the GeoDAWN survey area of western Nevada from public geophysical, geodetic, seismic, and lidar data. They then list unmapped fault candidates, ranked by structural settings that favor geothermal systems. The analysis grid (EPSG:32611, 100 m) and the scoring metric (a distance-weighted Tversky index) follow the [DOE Geologic Enhanced Mapping System (GEMS) Prize Challenge](https://www.drivendata.org/competitions/306/competition-doe-gems/).

Run the notebooks in order. Each one reads the outputs of the notebooks before it from a shared store.

| File | Description |
|---|---|
| `01_data_assembly.ipynb`, `02_dem_derivatives_dask.ipynb`, `03_modeling_and_candidates.ipynb` | The three notebooks, described below. |
| `environment.yml` | Conda environment for GeoLab (or any conda install). |

| Notebook | What it does | Main outputs |
|---|---|---|
| `01_data_assembly.ipynb` | Builds the grid and fault labels (USGS Quaternary faults) and the feature stack. The features come from GeoDAWN magnetic and radiometric grids, ComCat earthquake density, and GNSS strain rate. | `grid.json`, `features.zarr`, `labels.zarr`, `qfaults_aoi.gpkg` |
| `02_dem_derivatives_dask.ipynb` | Computes terrain derivatives from USGS 3DEP 1 m lidar on a Dask Gateway cluster, including slope anomaly, Laplacian, TPI, and lineament orientation. It aggregates them to the 100 m grid. | `dem1m_features.zarr` |
| `03_modeling_and_candidates.ipynb` | Trains LightGBM with spatially blocked cross-validation, maps fault probability, and extracts and ranks unmapped fault candidates. An optional U-Net can be run for comparison. | `predictions/*.tif`, `predictions/fault_candidates.gpkg` |

> [!IMPORTANT]
> The mapped faults are incomplete. The scores measure agreement with the current fault map, so a real, unmapped fault counts as a false positive. The candidate list from notebook 03 is the scientific result, and experts must review it.

## Setting up in GeoLab

### 1. Start a server and clone the repository

1. Log in to GeoLab and launch a server with the default **GeoLab** (Pangeo-based) image. Choose a medium or large server. Notebook 01 holds the full-region grids in memory, and notebook 03 stops if its feature stack needs more than 60% of free RAM (`MEMORY_FRACTION_LIMIT`).
2. Open a terminal (**File → New → Terminal**) and clone the repository into your home directory:

   ```bash
   cd ~
   git clone https://github.com/sparafina-earthscope/notebooks.git
   ```

   The files are in `~/notebooks/GEMS-challenge`. To get later updates, run `git pull` inside `~/notebooks`.

### 2. Create the environment

The notebooks need these packages:

| Notebook | Packages |
|---|---|
| 01 | `rioxarray`, `rasterio`, `geopandas`, `pyogrio`, `shapely`, `pyproj`, `scipy`, `zarr`, `fsspec`, `requests` |
| 02 | Same as 01, plus `dask`, `distributed`, and `dask-gateway` |
| 03 | Same as 01, plus `lightgbm`, `scikit-image`, and `psutil` |
| 03, section 6 (optional U-Net) | `pytorch` and `segmentation-models-pytorch`, and a GPU server |

`environment.yml` installs all of them except the optional U-Net packages.

**Option A: persistent environment (recommended).** It's stored in your home directory, so you build it once and it stays available across sessions. In the terminal, run:

```bash
cd ~/notebooks/GEMS-challenge
conda env create -f environment.yml
conda activate gems
python -m ipykernel install --user --name gems --display-name "Python (gems)"
```

The build takes several minutes. Then open each notebook, go to **Kernel → Change Kernel…**, and select **Python (gems)**. You need to choose the kernel once for each notebook.

- **Optional U-Net:** uncomment the `pytorch` and `segmentation-models-pytorch` lines in `environment.yml` before you create the environment. Create it on a GPU server, so that conda installs the GPU build of PyTorch.
- **Updating:** if you change `environment.yml` later, update the environment with `conda env update -f environment.yml --prune`.

**Option B: quick install into the default kernel.** The default GeoLab image already has most of these packages. If an import fails, add this as the first cell of the notebook and run it, then restart the kernel:

```python
%pip install lightgbm scikit-image psutil
```

The packages are gone the next time you start a server, so you need to run the cell again each session.

> [!IMPORTANT]
> **Notebook 02 with Dask Gateway needs a custom image.** Gateway workers start from a container image. They can't see a conda environment in your home directory. The cluster cell calls `client.get_versions(check=True)`, which stops if the notebook's packages differ from the workers'. So a `gems` kernel from Option A fails that check. Use Option C for notebook 02.
>
> The `gems` kernel from Option A works for notebooks 01 and 03. It also works for notebook 02 with `USE_LOCAL_CLUSTER = True`, because the workers then run on the notebook server in the same environment.

**Option C: custom GeoLab image (needed for notebook 02 on Dask Gateway).** Build a GeoLab image that contains the packages in `environment.yml`. Then run the notebook server and the Gateway workers from that image:

1. **Build the image.** Add the packages in `environment.yml` to the image's **default** environment (the one the default kernel uses), not to a separate environment. The Gateway workers use the image's default environment.
2. **Start the notebook server** from your custom image, and run notebook 02 in the default kernel.
3. **Set the worker image.** `GATEWAY_IMAGE = "auto"` in the Configuration cell (the default) passes the server's own image to the workers. JupyterHub puts the image name in the `JUPYTER_IMAGE_SPEC` environment variable. If that variable isn't set, or you want a different tag, set `GATEWAY_IMAGE` to the full image name, for example `"registry.example.org/geolab/gems:2026.09"`.

The cluster cell prints the worker image it requests. If the Gateway only allows images from a fixed list, it prints that list, and the cell stops with a `ValueError` when your image isn't on it. Ask the GeoLab administrators to add it.

Use a fixed image tag rather than `latest`. Otherwise the workers can pull a newer build than the one your server runs, and `get_versions` then stops the cluster.

**Storage location.** By default, the notebooks write to `~/gems` and cache downloads in `/tmp/gems_cache`. To use other locations, set these environment variables before you start the kernel:

```bash
export GEMS_STORE=s3://my-bucket/gems   # an S3 URL also needs s3fs
export GEMS_CACHE=/tmp/gems_cache
```

A local path works for all three notebooks, because only the notebook server writes to the store; the Dask workers return their results to it. `/tmp` is cleared when the server stops, so the next session downloads the inputs again.

### 3. Run the notebooks

Open each notebook and run the cells in order. Read the annotations as you go:

- `VERIFY:` marks an assumption about an external service or file format. Check it on the first run.
- `NOTE:` marks a design decision that you can change.

The notebooks stop with an error when an input is missing or malformed. They don't fall back silently.

#### Notebook 01: data assembly

- **Downloads:** the two GeoDAWN GeoTIFF bundles (about 290 MB), the Quaternary fault database, the ComCat catalog, and the NGL MIDAS velocities.
- **GeoDAWN file table:** review the table the notebook prints before the grids load. `GEODAWN_INCLUDE` selects only the `*_tiffs.zip` bundles. The larger `*_gdb.zip` and `*_csv.zip` files are on ScienceBase's newer file manager, which sends scripts an HTML page instead of the file.

#### Notebook 02: lidar derivatives

1. **Test one tile.** The notebook runs one tile on the notebook server before it starts the cluster. This checks access to the 3DEP bucket and the memory use for your `decimate` setting.
2. **Size the workers.** Start the notebook server from your custom image (Option C in step 2). The cluster cell prints the Gateway options and their allowed values. Set `instance_type` and `worker_resource_allocation` in `GATEWAY_OPTIONS` so that each worker has at least 4 GB with `decimate = 2`, or about 16 GB with `decimate = 1`.
3. **Start small.** `MAX_TILES = 8` processes a test subset. Set it to `None` for the full region, which reads about 0.4 GB per tile.
4. **Shut down the cluster.** When the output is written, run `cluster.shutdown()`.

If Dask Gateway isn't available, set `USE_LOCAL_CLUSTER = True` to run on the notebook server. This is only practical for a few tiles.

#### Notebook 03: modeling and candidates

- **Without lidar features:** set `REQUIRE_DEM = False` to run without notebook 02's output.
- **U-Net comparison:** set `RUN_UNET = True` on a GPU server.

When you're done, stop your server (**File → Hub Control Panel → Stop My Server**) so it doesn't use up your compute quota.

## Setting parameters

Each notebook has a **Configuration** cell in section 0. The settings you're most likely to change:

| Notebook | Parameter | Default | Description |
|---|---|---|---|
| 01 | `AOI_LONLAT_OVERRIDE` | `None` | Smaller area as (lon_min, lat_min, lon_max, lat_max). `None` uses the GeoDAWN bounding box. |
| 01 | `GEODAWN_INCLUDE`, `GEODAWN_EXCLUDE` | TIFF bundles only | Regular expressions on file names. |
| 01 | `EQ_START`, `EQ_END`, `EQ_MIN_MAG` | 1980–2026, M1.0 | Earthquake catalog selection. |
| 01 | `MIDAS_URL` | `midas.IGS.txt` | GNSS velocity file. `midas.NA.txt` gives North-America-fixed velocities; the strain rate is the same. |
| 01 | `EXTRA_RASTERS`, `EXTRA_FAULT_VECTORS` | empty | Your own rasters (for example gravity or INGENIOUS layers) and fault traces. |
| 02 | `PARAMS["decimate"]` | `2` | `1` = full 1 m resolution (about 16 GB per worker); `2` = 2 m (about 4 GB). |
| 02 | `MAX_TILES` | `8` | Test subset. `None` = all tiles. |
| 02 | `GATEWAY_OPTIONS`, `MIN_WORKERS`, `MAX_WORKERS` | defaults, 2–40 | Worker size and number. |
| 02 | `GATEWAY_IMAGE` | `"auto"` | Worker image. `"auto"` uses the notebook server's image; `None` uses the Gateway default. |
| 03 | `N_FOLDS`, `BLOCK_PX`, `BUFFER_PX` | 5, 20 km, 1 km | Spatial cross-validation. |
| 03 | `NEG_MIN_DIST_M`, `UNLABELED_WEIGHT` | 1000 m, 0.5 | Positive-unlabeled sampling. These are heuristics, so test their effect. |
| 03 | `CAND_THRESHOLD`, `CAND_MIN_DIST_M`, `CAND_MIN_LENGTH_M` | 0.5, 500 m, 2 km | Candidate extraction. |
| 03 | `FAVOR_WEIGHTS` | see notebook | Weights of the geothermal ranking score. |

## Troubleshooting

- **`... is not a zip file ... First bytes: b'<!doctype html>...'` (notebook 01):** ScienceBase served a web page instead of the file. This happens for files on its newer file manager. Exclude that file, or download it by hand in a browser and put it in the cache folder. The notebook has already deleted the bad cached copy.
- **`HTTPError: 404` for the MIDAS file (notebook 01):** NGL renamed the file. Check the file list at <https://geodesy.unr.edu/gps_timeseries/IGS20/midas/> and update `MIDAS_URL`.
- **`Gateway option ... is not offered` (notebook 02):** use the option names that the cluster cell prints.
- **`scheduler-connection-lost` or `Sending large graph` (notebook 02):** the Dask scheduler ran out of memory or dropped the connection. The per-tile results are held on the notebook server, so re-run only the failed cell.
- **`get_versions` mismatch (notebook 02):** the notebook kernel and the Gateway workers use different package versions. Start the notebook server from your custom image, run notebook 02 in its default kernel, and check the `Worker image:` line that the cluster cell prints. It must name the same image and tag as your server.
- **`dem1m_features.zarr not found` (notebook 03):** run notebook 02 first, or set `REQUIRE_DEM = False`.
- **Memory error in notebook 03:** use a larger server, reduce `FEATURE_SCALES_PX`, or add layers to `EXCLUDE_FEATURES`.
