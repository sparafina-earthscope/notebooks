# iMUSH Seismic Phase Picking with SeisBench

This notebook runs deep-learning phase picking on data from the iMUSH experiment at Mount St. Helens (network `XD`, 2014–2016). The name of the experiment is short for "Imaging Magma Under St. Helens". The notebook:

1. Downloads one hour of broadband waveforms from the EarthScope FDSN service.
2. Picks P and S arrivals with the [SeisBench](https://github.com/seisbench/seisbench) PhaseNet model, using the `volpick` weights, which were trained on volcanic seismicity.
3. Groups the picks into candidate events with a simple time-window rule.
4. Writes a JSON catalog, and plots each candidate event with its picks.

It also works as a GPU test. When the server has a GPU, PhaseNet runs on it; otherwise, the notebook uses the CPU.

> [!NOTE]
> The event grouping is a **heuristic, not a locator**. It groups picks that arrive close together in time at several stations. It doesn't compute a hypocenter, depth, or magnitude.

## Contents

| File | Description |
|---|---|
| `imush_phase_picking.ipynb` | Downloads waveforms, picks phases, groups them into events, and writes the catalog. |
| `environment.yml` | Conda environment for GeoLab (or any conda install). |

## Setting up in GeoLab

### 1. Start a server and clone the repository

1. Log in to GeoLab and launch a server.
   - **GPU:** choose a server option with a GPU if your account has one.
   - **CPU:** the default one-hour, 20-station run also finishes on a CPU server, just more slowly. Raising `MAX_STATIONS` or lengthening the window makes the GPU matter more.
2. Open a terminal (**File → New → Terminal**) and clone the repository into your home directory:

   ```bash
   cd ~
   git clone https://github.com/sparafina-earthscope/notebooks.git
   ```

   The files are in `~/notebooks/gpu-test-phase-picking`. To get later updates, run `git pull` inside `~/notebooks`.

### 2. Create the environment

The notebook needs `seisbench`, `torch` (PyTorch), `obspy` (1.4.1 or later), and `matplotlib`. If your server image already has PyTorch with GPU support, Option B is the simplest.

**Option A: persistent environment.** It's stored in your home directory, so you build it once and it stays available across sessions. In the terminal, run:

```bash
cd ~/notebooks/gpu-test-phase-picking
conda env create -f environment.yml
conda activate imush-picking
python -m ipykernel install --user --name imush-picking --display-name "Python (imush-picking)"
```

The build takes several minutes, because PyTorch is large. For a GPU build of PyTorch, run this command **on a GPU server**. conda detects the GPU and installs the CUDA build; on a CPU server, it installs the CPU build. Then open the notebook, go to **Kernel → Change Kernel…**, and select **Python (imush-picking)**.

**Option B: quick install into the default kernel.** Add this as the first cell of the notebook and run it:

```python
%pip install seisbench "obspy>=1.4.1"
```

Then restart the kernel (**Kernel → Restart Kernel**). If PyTorch isn't installed yet, pip installs it along with SeisBench. The packages are gone the next time you start a server, so you need to run the cell again each session.

### 3. Run the notebook

Open `imush_phase_picking.ipynb` and choose **Kernel → Restart Kernel and Run All Cells**.

- **GPU check.** The first code cell prints `seisbench GPU support: True` or `False`. The picking cell then names the GPU, or says that it's running on the CPU.
- **Model weights.** The first run downloads the PhaseNet weights and caches them in `~/.seisbench`.
- **Outputs.**
  - `imush_catalog.json`, next to the notebook: the candidate events, each with its picks (station, phase, time, and confidence).
  - `/tmp/imush_event_plots/`: one PNG for each event. The notebook also displays these PNGs inline.

When you're done, stop your server (**File → Hub Control Panel → Stop My Server**). This matters most on GPU servers.

## Setting parameters

All parameters are in the **Configuration** cell:

| Parameter | Default | Description |
|---|---|---|
| `WINDOW_START`, `WINDOW_END` | 2014-08-01, one hour | Analysis window. Use any date during the broadband deployment (2014–2016). |
| `CHANNEL` | `"HH?"` | 100 Hz broadband channels, the sample rate PhaseNet was trained on. |
| `MAX_STATIONS` | `20` | Number of stations to use, in alphabetical order. Stations with no data in the window are skipped. |
| `PHASENET_WEIGHTS` | `"volpick"` | SeisBench weight set. `"stead"` and `"original"` are general-purpose alternatives. |
| `PICK_CONFIDENCE_THRESHOLD` | `0.25` | Minimum probability for a P or S pick. |
| `ASSOCIATION_WINDOW_SECONDS` | `8.0` | Largest time gap between picks in one candidate event. |
| `MIN_STATIONS_PER_EVENT` | `3` | Minimum number of stations for a candidate event. |
| `CATALOG_PATH` | `imush_catalog.json` | Output file. |

Download time grows with `MAX_STATIONS` and the window length. iMUSH had thousands of nodal stations, so don't try to download the whole deployment in a notebook. Work in windows instead.

## Troubleshooting

- **`seisbench GPU support: False` on a GPU server:** the kernel's PyTorch is a CPU build. Rebuild the environment on the GPU server (Option A), or install a CUDA build of PyTorch.
- **`skipping <station>: No data available for request`:** that station has no `HH?` data in the window. This is normal for a few stations. The notebook continues with the others.
- **Few or no candidate events:** lower `PICK_CONFIDENCE_THRESHOLD` or `MIN_STATIONS_PER_EVENT`, or choose a different window.
- **Matplotlib cache warnings:** the notebook stores the Matplotlib cache in `/tmp/mplconfig`, so this warning should not appear. If it does, it's harmless.
