# The Data

The FITS images used by this project are **not included in this repository**.

Two reasons:

1. **Size** — the dataset is about **1.1 GB** (1,741 files), well beyond what belongs in a git repository.
2. **Ownership** — the images were provided to participants by the organisers of **HACK4DEV IRAQ 2026**. They are not ours to redistribute publicly, so we link to them rather than copy them.

Everything else needed to reproduce our results *is* in this repository: all six notebooks, the derived results in `outputs/`, and the exact package versions in `requirements.txt`.

## What the dataset is

| | |
|---|---|
| Source | MicroObservatory — "Cecilia" telescope, 6-inch, Whipple Observatory, Arizona |
| Provided by | HACK4DEV IRAQ 2026, Exoplanet Data Challenge |
| Contents | 1,741 FITS images — 1,681 science + 60 dark calibration frames |
| Targets | 8 stars with known planets: CoRoT-2, HAT-P-10, Qatar-1, TrES-1, TrES-3, TrES-5, WASP-2, WASP-10 |
| Coverage | 22 nights, each 2.5–5 hours, one image roughly every 3 minutes |
| Total size | ~1.1 GB |

## Where to put it

Place the `database/` folder next to the notebooks, exactly as the organisers distributed it:

```
hac4dev-exoplanet-track-f/
├── database/                  <-- add this folder yourself
│   ├── observations/YYYY-MM-DD/TARGET/session_01/*.fits
│   ├── calibration/YYYY-MM-DD/*.fits
│   ├── metadata/observations.csv
│   └── dataset_index.csv
├── notebooks/
├── outputs/
├── README.md
└── requirements.txt
```

The notebooks reference the data as `../database`, so no path editing is needed once the folder is in place.

## If you only want to read the results

You do **not** need the images to inspect our work. The notebooks in `notebooks/` are saved with their outputs, and the derived data is in `outputs/`:

| File | What it holds |
|---|---|
| `frames_index.csv` | one row per science image, with observing conditions and quality flags |
| `light_curves.csv` | 46,231 brightness measurements — 40 stars per night, 19 nights |
| `light_curves_detrended.csv` | the same curves after systematics removal (notebook 05) |
| `photometry_quality.csv` | per-night photometry quality |
| `nights_summary.csv`, `night_star_counts.csv` | per-night summaries from notebook 01 |
| `planet_parameters.csv` | published planet parameters from the NASA Exoplanet Archive |

## One external source

`planet_parameters.csv` comes from the **NASA Exoplanet Archive** (orbital period, transit duration, transit depth, reference transit time). Notebook 03 downloads it automatically if the file is missing, so an internet connection is needed the first time.

It is used only to know *when* real transits were expected — never to train the model. Notebook 06, which produces our final result, uses no external data at all.
