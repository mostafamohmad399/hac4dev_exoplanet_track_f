# Exoplanet Data Challenge — Track F (AI Challenge)

**HACK4DEV IRAQ 2026**
**Team:** Mostafa & Sara

---

## 1. The question

> Given ground-based images of stars known to host planets, can we automatically tell a **real transit-like dip** in a star's brightness apart from a **false dip** caused by clouds, the star sinking towards the horizon, or measurement noise?

Track F asks for AI/ML that detects signals, ranks candidates, or **reduces false positives**. We chose false-positive reduction, because it is the real problem in this dataset.

**Headline result:** at the same detection rate as a simple depth threshold, our model cuts false positives from **306 to 43 — an 86% reduction** (notebook 06, leave-one-night-out validation).

## 2. The data

- **Source:** MicroObservatory ("Cecilia", 6-inch telescope, Whipple Observatory, Arizona), provided with the challenge.
- **Size:** 1,741 FITS images — 1,681 science images + 60 dark calibration frames.
- **Targets:** 8 stars with known planets (CoRoT-2, HAT-P-10, Qatar-1, TrES-1, TrES-3, TrES-5, WASP-2, WASP-10).
- **Coverage:** 22 nights, each 2.5–5 hours, one image about every 3 minutes.
- **External data:** planet parameters from the **NASA Exoplanet Archive**, used only to know when real transits were expected. Disclosed as required by the challenge rules.

## 3. What we found in the data (notebook 01)

| Finding | Evidence |
|---|---|
| `index.py` reads header key `EXPOSURE`, but the real key is `EXPTIME` | the `exposure` column of `dataset_index.csv` is empty for all 1,741 rows |
| Paths in `dataset_index.csv` use Windows backslashes | they break on macOS/Linux |
| `MJD-OBS` is rounded to 0.001 day (86 s) | we use `UT-OBS` (millisecond precision) instead |
| Only **dark** calibration frames exist — no bias, no flat | the dataset README claims all three |
| 6 of 22 nights were largely clouded out | the middle image shows 0–4 stars, versus 200–600 on clear nights |
| 33 images were taken after dawn | sky level reaches the camera limit (4095) |
| Star names are not catalogue names | `TRES-3` → `TrES-3`, `HATP-10` → `HAT-P-10` (= WASP-11) |
| **All 22 nights were scheduled on a predicted transit** | checked against NASA ephemerides — so "night with/without transit" cannot be a label |

## 4. Method

### 4.1 Images → light curves (notebook 02)

Dark subtraction → star detection → **frame alignment** (the field drifts up to ~55 px per night, so fixed apertures measure empty sky) → aperture photometry → **relative brightness** (each star divided by the sum of the others, which cancels effects that hit the whole sky).

**Result:** 46,231 measurements, 40 stars per night, **19 of 22 nights usable**, best night 1.13% scatter per star.

### 4.2 Removing systematics (notebook 05)

Notebook 04 exposed the real problem: for TrES-5 the formal error bar was **±0.22%**, but the true spread between planet-free stars was **±1.01%** — the systematic error was about **5× the random error**. No model can fix that, so we fixed the data.

We use **SysRem/PCA de-trending**: all 40 stars look through the same atmosphere, so patterns common to all of them are systematics. We remove the 3 strongest shared patterns from each star, fitting them **without that star** (leave-one-out) so a real transit cannot be subtracted as if it were a shared pattern.

| | Before | After (k=3) |
|---|---|---|
| Median scatter per star | 2.19% | **1.19%** (46% better) |
| Control-star spread (TrES-5) | 1.01% | **0.42%** |
| Injected 2% transit preserved | — | 83% |

### 4.3 Labels: injection and recovery (notebooks 03 → 06)

All 22 nights were scheduled on a transit, so there are **no natural negatives**, and 8 planets is far too few to train on. We therefore inject artificial transits (depth 1–5%, duration 40–120 min) into real light curves; injected windows are positives, dips found in untouched curves are false positives.

**Stated openly: our positive examples are injected, not discovered.**

### 4.4 Three corrections that made the numbers trustworthy

Notebook 03 was our first attempt. Notebook 06 fixes three flaws we found in it:

| Flaw in notebook 03 | Fix in notebook 06 |
|---|---|
| Transits injected **after** de-trending would never face the cleaning step | inject into the raw curve, **then** de-trend — measures the sensitivity we actually have |
| The model could partly guess from **when** a dip happened (`altitude_change` was the top feature) | candidate times restricted to the **middle 60%** of every night, for both classes |
| 5-fold split by night | **leave-one-night-out** — 16 nights, 16 independent tests, and we report the spread |

**Notebook 03's numbers were inflated by that time-of-night leakage. The numbers below are the honest ones.**

## 5. Results (notebook 06)

Dataset: **5,800 candidates** (640 injected transits, 5,160 false positives) from 16 nights, de-trended, leave-one-night-out.

| Model | Average precision | Precision at baseline recall | False positives |
|---|---|---|---|
| Baseline (depth > 2%) | 0.448 | 0.517 | 306 |
| **Random Forest** | **0.747** | **0.884** | **43** |
| HistGradientBoosting | 0.733 | 0.826 | 69 |
| Two-branch NN (1D-CNN + dense) | 0.733 | 0.822 | 71 |

### Confusion matrices (at identical recall)

| | TP | FP | FN | TN |
|---|---|---|---|---|
| Baseline (depth > 2%) | 328 | 306 | 312 | 4,854 |
| **Random Forest** | 328 | **43** | 312 | 5,117 |

Same transits caught, **263 fewer false alarms**.

### Which model, and why it matters

The **Random Forest won** — ahead of gradient boosting and ahead of the neural network. That is worth stating plainly: with 640 positives spread over 16 nights, the limiting factor is the amount of independent data, not model capacity. A 1D-CNN with two input channels (the star's own curve, and the average of the other stars) did not beat a simple ensemble of trees on engineered features.

**On LightGBM/XGBoost:** both fail to load on this machine because macOS lacks the OpenMP runtime they need. `HistGradientBoostingClassifier` is the same histogram-gradient-boosting algorithm, ships inside scikit-learn, and keeps the project runnable anywhere — so we used it.

### Stability between nights

| Model | Median AP | Worst night | Best night | Spread |
|---|---|---|---|---|
| Random Forest | 0.808 | 0.279 | 0.974 | 0.221 |
| HistGradientBoosting | 0.786 | 0.244 | 0.977 | 0.243 |
| Two-branch NN | 0.796 | 0.264 | 0.970 | 0.236 |

Performance varies a lot from night to night. A model that is excellent on a clear night and poor on a cloudy one is exactly what we would expect, and it is why we report the spread rather than a single number.

## 6. Limits — when does this fail?

Sensitivity of the best model on de-trended data, injection before cleaning:

| Injected depth | 1.0–1.5% | 1.5–2.0% | 2.0–2.5% | 2.5–3.0% | 3.5–4.0% | 4.5–5.0% |
|---|---|---|---|---|---|---|
| Detected | 14.7% | 41.2% | 52.2% | 59.0% | 55.2% | 58.2% |

These numbers are far lower than notebook 03 reported (70.7% at 1–1.5%). That difference **is** the leakage we removed. Below about 2% depth we are close to useless, and the transits in this dataset are 1.6–2.75% deep — which is precisely why we detect none of them.

## 7. Did we see a real transit? No — and we proved it twice

**Test 1 — folding + control stars (notebook 04).** We stacked all usable transits of each planet using the published period, then repeated the identical analysis on ordinary stars in the same images.

| Target | Transits stacked | Host depth | Published depth | Controls dipping deeper than the host |
|---|---|---|---|---|
| TrES-5 | 5 | +1.11% | 2.19% | **16 of 31 (52%)** |
| CoRoT-2 | 2 | +0.52% | 2.75% | 5 of 27 (19%) |

**Test 2 — the same test on cleaned data (notebook 05).**

| Target | Host depth before → after | Control spread before → after | Controls deeper, after |
|---|---|---|---|
| TrES-5 | 1.11% → 0.34% | 1.01% → 0.42% | 23% |
| CoRoT-2 | 0.52% → 0.15% | 0.39% → 0.23% | 30% |

De-trending made the data 2.4× cleaner — and the host's apparent dip shrank with it. That is the signature of a systematic, not a planet.

**Conclusion: no detection.** Our precision reaches ~1% on the best nights while these transits are 1.6–2.75% deep, and every night is centred on the transit so anything varying through a night imitates one.

This also retires an earlier hint: in notebook 03 the strongest candidate landed inside the real transit window on 13 of 16 nights (chance ≈ 8.1, p = 0.012). Since those windows sit mid-night where conditions are most stable, that test was most likely measuring the same night-shape systematic.

**Why the standard Kepler/TESS pipeline (BLS → folding → global/local views → CNN) does not apply:** we observed only **3–13% of each orbit**, all near phase 0, in 1–6 windows separated by many unobserved orbits — so BLS cannot recover a period and a "global view" does not exist. Classifiers of that family are trained on ~15,000 labelled signals; we have 8 stars and 22 nights of a single class. Full reasoning in notebook 04.

## 8. How to run

> **The images are not in this repository** — they are ~1.1 GB and belong to the organisers. See **[DATA.md](DATA.md)** for what the dataset is and where to place it. You do not need it to read the results: the notebooks are saved with their outputs and the derived data is in `outputs/`.

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/python -m ipykernel install --user --name hac4dev --display-name "Python (hac4dev)"
```

Open the notebooks in order, selecting the **Python (hac4dev)** kernel:

| Notebook | What it does | Runtime |
|---|---|---|
| `01_explore_data.ipynb` | check the data, find the quality problems | ~1 min |
| `02_photometry.ipynb` | images → light curves | ~1.5 min |
| `03_ai_model.ipynb` | first AI attempt (kept to show what we corrected) | ~1 min |
| `04_phase_folding.ipynb` | stack real transits, control-star test | ~10 s |
| `05_detrending.ipynb` | remove systematics, re-test | ~10 s |
| `06_best_model.ipynb` | **final model comparison** | ~7 min |

## 9. Use of AI tools

We used an AI assistant to help write and debug code and to explain astronomy concepts. Every notebook was run by us, and every decision is documented below with the evidence behind it. External data (NASA Exoplanet Archive) and libraries (`astropy`, `photutils`, `scikit-learn`, `torch`) are listed in `requirements.txt`.

## 10. Decision log

| # | Decision | Reason |
|---|---|---|
| 1 | Build our own index instead of `dataset_index.csv` | empty exposure column, Windows-only paths |
| 2 | Use `UT-OBS` for time, not `MJD-OBS` | `MJD-OBS` rounded to 86 s |
| 3 | Map folder names to official catalogue names | needed to query the NASA Exoplanet Archive |
| 4 | Flag bad frames instead of deleting them | lets us measure their effect |
| 5 | Dark frames only | no bias or flat frames exist |
| 6 | Subtract median dark of same/closest date | removes the camera's brightness gradient |
| 7 | Treat nights with very few stars as cloudy | star count agrees with the header weather score |
| 8 | Reference frame = image with most stars | the middle image can be cloudy |
| 9 | Align frames star-by-star | field drifts up to ~55 px |
| 10 | Keep only stars further from the edge than the night's drift | otherwise stars leave the image |
| 11 | 40 brightest stars, comparison = sum of the others | faint stars are noise |
| 12 | Drop unalignable frames and stars missing >10% of the time | missing data creates fake jumps |
| 13 | Injection-recovery for labels | no natural negatives; only 8 planets |
| 14 | Keep the *deepest* false positives when subsampling | makes the task harder, keeps the result honest |
| 15 | Split by night, never by image | images within a night are correlated → leakage |
| 16 | Compare baseline and model **at matched recall** | comparing false positives at different recalls is meaningless |
| 17 | Run an ablation instead of trusting feature importance | revealed that observing-condition features carried signal |
| 18 | Fold with the **published** period, never one "discovered" from our data | we cover only 3–13% of each orbit |
| 19 | Judge the folded depth against **control stars**, not its own error bar | the formal error was 5× smaller than the real star-to-star scatter |
| 20 | Match control stars across nights by position relative to the host | `star_id` is a per-night ranking, so the same id is a different star on another night |
| 21 | Remove shared patterns across stars (SysRem/PCA) before modelling | systematic error was ~5× the random error |
| 22 | Fit those patterns **without** the star being cleaned | otherwise a real transit could be removed as a "shared" pattern |
| 23 | Use k = 3 shared patterns | measured trade-off between noise removed and signal kept |
| 24 | **Inject before de-trending** | injecting afterwards would report a sensitivity we do not have |
| 25 | Restrict candidates to the middle 60% of each night | removes the model's ability to guess from time of night |
| 26 | **Leave-one-night-out** validation | 16 nights = 16 independent tests, and the spread is reported |
| 27 | Keep notebook 03 even though notebook 06 supersedes it | it documents the flaws we found and corrected |
| 28 | Use sklearn's HistGradientBoosting instead of LightGBM/XGBoost | both fail to load without an OpenMP runtime; same algorithm family, fully reproducible |
