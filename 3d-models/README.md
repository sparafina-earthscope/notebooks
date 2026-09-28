# 3D-Printable Terrain Models from USGS 3DEP

This notebook turns USGS 3DEP elevation data into 3D-printable STL files. It produces a solid, watertight terrain model with a flat base, plus a hillshade preview image. The default settings model Mount St. Helens, but the notebook works for any area covered by 3DEP (the United States and its territories).

## Contents

| File | Description |
|---|---|
| `make_stl.ipynb` | Downloads a DEM for any bounding box and builds an STL with configurable vertical exaggeration. |
| `environment.yml` | Conda environment for GeoLab (or any conda install). |

## Setting up in GeoLab

### 1. Start a server and clone the repository

1. Log in to GeoLab and launch a server with the default **GeoLab** image. The smallest server size is enough: with the default settings a model uses about 0.5 GB of memory and builds in seconds, plus the time to download the DEM.
2. Open a terminal (**File → New → Terminal**) and clone the repository into your home directory (`/home/jovyan`):

   ```bash
   cd ~
   git clone https://github.com/sparafina-earthscope/notebooks.git
   ```

   The files are in `~/notebooks/3d-models`. To get later updates, run `git pull` inside `~/notebooks`.

With the default settings each STL is about 50 MB, well within the 50 GB home directory limit.

### 2. Create the environment

The default GeoLab image already includes `numpy` and `pyproj`, but not `tifffile`. You have two options.

**Option A: persistent environment (recommended).** Build it once and it stays available across sessions, because it is stored in your home directory. In the terminal, run:

```bash
cd ~/notebooks/3d-models
conda env create -f environment.yml
conda activate msh-3d-model
python -m ipykernel install --user --name msh-3d-model --display-name "Python (msh-3d-model)"
```

The build takes a few minutes. Then open the notebook, go to **Kernel → Change Kernel…**, and select **Python (msh-3d-model)**. You need to choose the kernel once for each notebook.

**Option B: quick install into the default kernel.** Add this as the first cell of the notebook and run it:

```python
%pip install tifffile pillow pyproj
```

This is faster, but the packages are gone the next time you start a server, so you'll need to run the cell again each session.

### 3. Run the notebook

Open `make_stl.ipynb`, edit the **Configuration** cell (see below), then choose **Kernel → Restart Kernel and Run All Cells**. The last cell shows the hillshade preview.

To download the STL, right-click it in the File Browser and choose **Download**.

When you're done, stop your server (**File → Hub Control Panel → Stop My Server**) so it doesn't use up your compute quota.

## Setting parameters

### Bounding box and vertical exaggeration

These are in the **Configuration** cell of `make_stl.ipynb`:

```python
BBOX = (-122.30, 46.13, -122.10, 46.26)   # lon_min, lat_min, lon_max, lat_max (WGS84)
EXAG = 2.5                                 # vertical exaggeration
```

- **`BBOX`** is `(west, south, east, north)` in decimal degrees. West longitudes are negative. To find coordinates, draw a box at [bboxfinder.com](http://bboxfinder.com) and copy the values it shows.
- **`EXAG`** multiplies the heights. `1.0` is true scale. Most terrain looks flat when printed at true scale, so values of `1.5`–`3` are typical. Use lower values for mountainous areas and higher values for gentle terrain.

The output files are named after the exaggeration. For example, `EXAG = 3` writes `mount_st_helens_3x.stl` and `preview_hillshade_mount_st_helens_3x.png`, so earlier models aren't overwritten.

The model keeps the real shape of the area. A bbox that is wider than it is tall produces a rectangular model, with the longer side set to `SIZE_MM`.

### Model parameters

These are in the **Model parameters** cell. The defaults work well for a typical FDM printer.

| Parameter | Default | Description |
|---|---|---|
| `SIZE_MM` | `152.4` | Length of the longer side of the model, in mm (152.4 mm = 6 in). |
| `GRID` | `500` | Number of samples along the longer side. Higher values give more detail but larger files: 500 gives about 1 million triangles and a 50 MB STL. Detail finer than about 0.3 mm is lost on a 0.4 mm nozzle. |
| `BASE_MM` | `3.0` | Thickness of the solid base below the lowest point, in mm. |
| `RES_M` | `10.0` | Resolution of the downloaded DEM, in metres. Increase it (e.g. `30`) for large areas. The server may reject requests much larger than a few thousand pixels per side. |

### How the DEM download works

The DEM is downloaded from the [USGS 3DEP ImageServer](https://elevation.nationalmap.gov/arcgis/rest/services/3DEPElevation/ImageServer) in the UTM zone of the bbox centre. It is saved under a name built from the bbox and resolution, such as `dem_-122.3000_46.1300_-122.1000_46.2600_10m.tif`:

- **Cached files:** re-running with the same settings reuses the saved file. Changing `BBOX` or `RES_M` downloads a new one.
- **Download failures:** the server sometimes returns 504 errors. The notebook retries up to 5 times. If it still fails, wait a few minutes and run the cell again.
- **Extra coverage:** a lon/lat box isn't a perfect rectangle in UTM, so the downloaded area can extend up to a few hundred metres past the bbox on some edges.

## Printing tips

- **Setup:** print flat, base down, with no supports. The terrain has no overhangs.
- **Layers:** use 0.12 mm layers, or variable layer height if your slicer has it, to reduce visible steps on gentle slopes.
- **Shell and infill:** 3 walls, 5–6 top layers, and 10–15% infill.
- **Adhesion:** a brim helps stop the corners of a large, flat base from lifting.
- **Filament:** matte grey, white, or tan PLA shows relief best.
