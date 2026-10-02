# A Hands-on Introduction to BioImage Simulation with DeepTrack2

Materials for **workshop CW20** at **SPAOM 2026**.

Training a deep-learning model to detect, classify or count objects in microscopy images usually
starts with the slowest and most expensive step: manually annotating real data. This workshop takes a
different route, using [DeepTrack2](https://github.com/DeepTrackAI/DeepTrack2) to simulate microscopy
images from physical principles, with exact ground truth and no manual annotation at all.

Everything lives in a single notebook, [`CW20_Guigo_SPAOM2026.ipynb`](CW20_Guigo_SPAOM2026.ipynb):

1. **Simulating images** — point particles, noise, Brownian dynamics, scatterers and the five imaging modalities.
2. **Simulating cells** — realistic fluorescent nuclei with their ground-truth masks.
3. **Training a U-Net to count cells** — trained only on simulated data, evaluated on the real [BBBC039](https://bbbc.broadinstitute.org/BBBC039) dataset.
4. **Going further** — two-color confocal synapses, a custom scatterer for bacteria, and custom textures for electron microscopy.
5. **Discussion** — which simulation choices matter, and when simulated data is a reasonable substitute for annotated data.

This community workshop is part of the activities of AIM-Net (RED2024-153844-T, funded by
MICIU/AEI/10.13039/501100011033).

## Setup

Requires **Python 3.10 or newer** and **git** (the notebook clones the cell-counting dataset).

```bash
git clone https://github.com/<your-user>/SPAOM-CW20-DeepTrack.git
cd SPAOM-CW20-DeepTrack

# Create and activate an environment (conda shown here; venv works just as well).
conda create -n deeptrack2 python=3.11
conda activate deeptrack2

# Install the workshop dependencies.
pip install -e .
```

`pip install -e .` installs DeepTrack2, deeplay and everything the notebook needs, including the
Jupyter kernel. To open the notebook in a browser rather than in an IDE, install the extra:

```bash
pip install -e ".[jupyter]"
jupyter lab
```

In VS Code, open the notebook and select the `deeptrack2` environment as the kernel.

### Google Colab / Kaggle

No setup is needed. Open the notebook and uncomment the first code cell:

```python
!pip install deeptrack deeplay
```

## Notes

- Section 3 trains a small U-Net on simulated data. It runs on a laptop CPU within the session, and
  is considerably faster on a GPU.
- Section 2 downloads the [cell counting dataset](https://github.com/DeepTrackAI/cell_counting_dataset)
  into the working directory the first time it runs.
