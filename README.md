# CoronaryViz — CAD & vessel-stenosis risk on an interactive 3D heart

Web prototype: scikit-learn models (overall CAD + LAD / LCX / RCA stenosis) behind a FastAPI backend,
a Three.js viewer whose coronary arteries are recoloured live from the predicted probabilities, and a
dashboard with SHAP explanations. **Decision support / educational use only — not a substitute for
diagnostic imaging.** The disclaimer is shown in a banner, on the 3D canvas, and in `/api/meta`.

> **Data note.** This build ships with a **synthetic stand-in dataset** (`data/synthetic_cad.csv`, 303 rows, same
> columns as Z-Alizadeh Sani). Reported metrics describe the synthetic data only. To use the real dataset see
> "Using the real dataset".

## Quick start
```bash
pip install -r requirements.txt
python -m cadviz.train              # trains 4 models -> models/*.joblib + models/meta.json
uvicorn cadviz.app:app --port 8000  # open http://localhost:8000
```
Three.js is vendored in `frontend/vendor/`, so no Node/npm and no internet are needed at run time.

Tests: `python -m pytest -q tests` (pipeline, leakage guard, API, SHAP additivity) and
`npm install && npm test` (headless 3D geometry/colour tests).

## Using the real dataset
Download the Z-Alizadeh Sani extension file (UCI / Kaggle) and run
```bash
python -m cadviz.train --data path/to/Z-Alizadeh-sani-dataset.xlsx   # .csv also works
```
`cadviz/data.py` normalises the original column names (`Current Smoker`, `EF-TTE`, `Region RWMA`, …) and string codes
(`Y/N`, `Male/Fmale`, `Stenotic/Normal`, `Cad/Normal`). Real-data features that are not in `cadviz/schema.py`
(e.g. `Weak Peripheral Pulse`, `Lung rales` variants) are ignored until you add them to `FEATURES`.

## Layout
```
cadviz/schema.py   feature list (single source of truth: form, API, training)
cadviz/synth.py    synthetic dataset generator
cadviz/data.py     loading, cleaning, leakage guard
cadviz/train.py    CV model selection, hold-out evaluation, saves weights + metrics
cadviz/explain.py  SHAP (tree + linear) wrapper
cadviz/app.py      FastAPI: /api/meta /api/samples /api/predict + static frontend
frontend/          index.html, style.css, app.js (UI), anatomy.js (procedural 3D anatomy)
models/            trained weights (*.joblib) and meta.json (metrics, samples)
docs/              project documentation (≤ 6 pages)
```

## Extending
* **New clinical feature:** add one entry to `FEATURES` in `schema.py` (and a column in your data), retrain. The form,
  API and SHAP chart pick it up automatically.
* **New model:** add it to `candidates()` in `train.py`; the best by CV ROC-AUC is chosen per target.
* **New anatomical structure:** add a mesh in `anatomy.js` with `userData.pick = 'Name'` — hover, click and the
  inspector work automatically; give it a vessel key to attach a risk colour.
* **Real mesh (BodyParts3D / Sketchfab .glb):** vendor `GLTFLoader.js` next to `OrbitControls.js`, load the heart
  mesh, use it in place of the procedural `heart`, and re-run the `onSurface()` projection of the artery control
  points against the new mesh. Respect each asset's licence (BodyParts3D is CC BY-SA).

## Design notes
* LAD, LCX, RCA and Cath are never inputs (asserted in `data.split_xy`).
* Every vessel segment/node shows the *vessel-level* probability; the dataset has no lesion-location labels, so the
  app does not claim to localise a lesion. Violet shading on the myocardium is the echo "Region with RWMA" input.
* Colour ramp (teal → amber → red) is always paired with text (percentage and Low/Moderate/High).
