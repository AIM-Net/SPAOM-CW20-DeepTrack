> [!NOTE]
> This repository is still being prepared for SPAOM 2026. Notebook links will be
> distributed before the workshop.

# A Hands-on Introduction to BioImage Simulation with DeepTrack2

**Community workshop CW20 — [SPAOM 2026](https://spaom2026.org)**

Guillem Guigó · Universitat de Vic – Universitat Central de Catalunya

Part of the activities of AIM-Net (RED2024-153844-T, funded by MICIU/AEI/10.13039/501100011033).

## Before the workshop

Please follow the [installation instructions](#installation) **before** attending, and check
that the first notebook runs end to end. No GPU is needed: everything is designed to run on a
laptop CPU within the session. If you would rather not install anything, the notebooks also run
on Google Colab.

## Workshop overview

Training a deep-learning model to detect, classify or count objects in microscopy images usually
starts with the slowest and most expensive step: manually annotating real data. This workshop
takes a different route, using [DeepTrack2](https://github.com/DeepTrackAI/DeepTrack2) to
**simulate** microscopy images from first principles. Because the images are generated rather than
collected, their ground truth is known exactly and no manual annotation is needed at all.

We build a complete pipeline from the ground up — **scatterer → optics → noise → image + ground
truth** — and then reuse it, almost unchanged, across increasingly complex samples: diffusing
single molecules, stained cell nuclei, bacteria, and two-color confocal images of synapses. We
close by asking the question that matters in practice: can a network trained only on simulated
cells count real ones?

## Contents of the workshop

- **Introduction** — what a scatterer is, how an optical system turns a physical object into an
  image, and why simulated noise matters.
- **Hands-on simulation** — point particles, noise, Brownian dynamics, imaging modalities.
- **Hands-on objects** — cells, bacteria and synapses, including writing your own scatterer.
- **Demonstration** — training a U-Net on simulated data only, and counting real cells with it.
- **Interactive wrap-up** — which simulation choices matter most for simulated-to-real transfer.

## Schedule

| Time | Activity |
|---|---|
| 15 min | Introduction: scatterers, optics, noise, and the DeepTrack2 pipeline |
| 50 min | Live-coded demo and hands-on session |
| 10 min | Interactive data collection and wrap-up discussion |

## Notebooks

| Notebook | Contents |
|---|---|
| [`CW20_Guigo_SPAOM2026.ipynb`](CW20_Guigo_SPAOM2026.ipynb) | The main workshop notebook: simulating images, cells, bacteria and synapses. |
| [`UNet-train.ipynb`](UNet-train.ipynb) | Optional companion: trains a U-Net on the simulated cells and counts real ones. Not covered live, as it takes longer than the session allows. |

## Software

| Package | Purpose |
|---|---|
| [DeepTrack2](https://github.com/DeepTrackAI/DeepTrack2) | Physics-informed simulation of microscopy images |
| [deeplay](https://github.com/DeepTrackAI/deeplay) | Deep-learning models, used for the U-Net |
| [PyTorch](https://pytorch.org) | Neural-network backend |
| [NumPy](https://numpy.org) · [SciPy](https://scipy.org) | Numerical and image-processing routines |
| [scikit-image](https://scikit-image.org) | Masks, morphology and connected components |
| [Matplotlib](https://matplotlib.org) | Figures |

## Installation

Requires **Python 3.10 or newer** and **git** (the cell notebook clones a public dataset).

```bash
git clone https://github.com/AIM-Net/SPAOM-CW20-DeepTrack.git
cd SPAOM-CW20-DeepTrack

# Create and activate an environment (conda shown here; venv works just as well).
conda create -n deeptrack2 python=3.11
conda activate deeptrack2

# Install the workshop dependencies.
pip install -e .
```

This installs DeepTrack2, deeplay and everything the notebooks need, including the Jupyter
kernel. In VS Code, open a notebook and select the `deeptrack2` environment as the kernel. To
work in a browser instead:

```bash
pip install -e ".[jupyter]"
jupyter lab
```

### Google Colab / Kaggle

No installation needed. Open the notebook with the badge below and uncomment the first code cell
(`!pip install deeptrack deeplay`).

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/AIM-Net/SPAOM-CW20-DeepTrack/blob/main/CW20_Guigo_SPAOM2026.ipynb)

## Practical requirements

- Laptop. No GPU required.
- **Basic familiarity with Python**: variables, functions, and running a Jupyter notebook.
- **No prior deep-learning or image-processing background** is needed.
- Laptops can be shared in pairs or trios for anyone without one.

## Acknowledgements

The confocal image of neuronal synapses in the main notebook is courtesy of Mercè
Izquierdo-Serra (Universitat de Barcelona).

## License

The course materials are released under the MIT License.
