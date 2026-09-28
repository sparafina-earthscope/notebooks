# 3D View of Mount St. Helens with Seismic Stations

This notebook draws an interactive 3D terrain model of Mount St. Helens and places markers for the seismic stations that operated during the iMUSH experiment (2014–2016). You can rotate and zoom the figure, and hover over a station to see its code, position, and elevation.

## Contents

| File | Description |
|---|---|
| `IMUSH_sations_3d.ipynb` | Downloads a DEM and station metadata and builds the Plotly 3D figure. |
| `environment.yml` | Conda environment for GeoLab (or any conda install). |

## How it works

1. **Terrain.** [py3dep](https://docs.hyriver.io/readme/py3dep.html) downloads a 30 m DEM from USGS 3DEP for the bounding box and reprojects it to longitude/latitude.
2. **Stations.** ObsPy requests station metadata from the EarthScope FDSN service. It asks for stations inside the bounding box that started before 2017 and were still running after the start of 2014. This includes stations from **every network** that was active in that window, not only the iMUSH network (`XD`).
3. **Figure.** The notebook samples the DEM at each station and places the marker 50 m above the terrain, so it stays visible. Plotly draws the terrain as a surface and the stations as red diamonds.

## Setting up in GeoLab

### 1. Start a server and clone the repository

1. Log in to GeoLab and launch a server with the default **GeoLab** image. The smallest server size is enough: the DEM is only a few hundred thousand cells.
2. Open a terminal (**File → New → Terminal**) and clone the repository into your home directory:

   ```bash
   cd ~
   git clone https://github.com/sparafina-earthscope/notebooks.git
   ```

   The files are in `~/notebooks/3d-vizualizations`. To get later updates, run `git pull` inside `~/notebooks`.

### 2. Create the environment

The notebook needs `py3dep`, `plotly`, `obspy` (1.4.1 or later, for the `EARTHSCOPE` provider name), and `rioxarray`. You have two options.

**Option A: persistent environment (recommended).** It's stored in your home directory, so you build it once and it stays available across sessions. In the terminal, run:

```bash
cd ~/notebooks/3d-vizualizations
conda env create -f environment.yml
conda activate msh-3d-viz
python -m ipykernel install --user --name msh-3d-viz --display-name "Python (msh-3d-viz)"
```

The build takes a few minutes. Then open the notebook, go to **Kernel → Change Kernel…**, and select **Python (msh-3d-viz)**.

**Option B: quick install into the default kernel.** The first cell of the notebook holds a commented-out install command. Replace it with this line and run it:

```python
%pip install py3dep plotly "obspy>=1.4.1" rioxarray
```

Then restart the kernel (**Kernel → Restart Kernel**). The packages are gone the next time you start a server, so you need to run the cell again each session.

### 3. Run the notebook

Open `IMUSH_sations_3d.ipynb` and choose **Kernel → Restart Kernel and Run All Cells**. The figure appears below the second cell. The notebook needs network access to the USGS 3DEP services and to `service.earthscope.org`.

When you're done, stop your server (**File → Hub Control Panel → Stop My Server**) so it doesn't use up your compute quota.

## Setting parameters

All settings are in the second cell:

| Setting | Default | Description |
|---|---|---|
| `bbox` | `(-122.30, 46.13, -122.10, 46.26)` | Area as (west, south, east, north) in decimal degrees. |
| `resolution=30` | 30 m | DEM resolution in metres, in the `py3dep.get_dem` call. Use `10` for more detail; for large areas, use a coarser value. |
| `startbefore`, `endafter` | 2017-01-01, 2014-01-01 | Keep only the stations that were running during this period. |
| `network=` | not set | Add `network="XD"` to the `get_stations` call to show only iMUSH stations. |
| `aspectratio` | `z=0.35` | Vertical exaggeration of the figure. Increase `z` to make the relief stronger. |

## Troubleshooting

- **`ModuleNotFoundError`:** set up the environment as in step 2, and select the right kernel.
- **The figure is blank:** Plotly needs JupyterLab 3 or later, which GeoLab provides. Reload the browser tab and run the cell again.
- **`No data available for request` from `get_stations`:** no station matches the bounding box and dates. Widen `bbox` or the date range.
- **The 3DEP download fails or times out:** the USGS service is sometimes busy. Wait a few minutes and run the cell again.
