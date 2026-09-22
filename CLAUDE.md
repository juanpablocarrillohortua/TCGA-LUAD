# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Breast cancer (oncología de seno) exploratory data analysis. The dataset in
`data/` is `GSE20194_MDACC_Sample_Info.xls` — clinical annotation for the
MDACC arm of the MAQC-II breast cancer gene-expression study. Analysis is
driven from notebooks in `notebooks/` (currently `analisis_genes.ipynb`,
empty scaffold) using the helpers in `utils/`.

There is no dependency manifest (no `requirements.txt`/`pyproject.toml`) and
no committed virtualenv. Per `utils/README.md`, the expected environment has
`pandas`, `numpy`, `scipy`, `matplotlib`, `seaborn`, `statsmodels`, `IPython`
installed; `scikit-learn` and `plotly` are needed for two modules but are not
installed (see below).

## Running notebooks / imports

`utils` is a plain package with no packaging metadata, so notebooks resolve
it by inserting the repo root onto `sys.path` at the top of the notebook:

```python
import sys
from pathlib import Path

sys.path.insert(0, str(Path.cwd().parent))  # notebooks/ -> repo root

from utils.fancy_barplot import fancy_bars
from utils.target_against_features import test_features_against_target
```

There is no test suite and no lint/build command configured in this repo.

## Architecture: `utils/`

One self-contained module per analysis idea, each independently importable.
**Modules never import each other**, and `utils/__init__.py` re-exports
nothing — always `from utils.<module> import <fn>`, never `from utils import
<fn>`. If two modules need the same helper, it is duplicated rather than
shared. Full parameter tables and a module overview live in
[utils/README.md](utils/README.md); read it before modifying or adding a
module — it documents the public API, test-selection logic, and shared
conventions in detail. Key points:

- **Two modules cannot import today**: `fast_OLS` needs `plotly`,
  `mutual_inf_plot` needs `scikit-learn`; neither is installed.
- **Contract split** — most modules take an `ax`/`axes` argument and return
  an `Axes`/`DataFrame`/`dict` without calling `plt.show()`, so they compose
  into subplot grids. Two older modules (`bivaraite_boxplot`,
  `category_vs_numeric_plot`) break this: no `ax` param, return nothing, call
  `plt.show()` internally, and cannot be composed.
- **`title=None` convention**: auto-generates a title from the column name
  (`"foo_bar"` → `"Foo Bar"`) across the newer modules.
- `target_against_features.py` / `cat_num_dep.py` auto-select a statistical
  test based on inferred variable type (categorical/discrete/continuous) —
  the type-inference rule (integer-valued + low cardinality ⇒ categorical,
  not continuous) is the part most likely to surprise a caller; see the
  README's type-inference and test-selection tables before adding a variable
  with an unusual encoding.
- Figure text and `ValueError` messages are in Spanish; identifiers are in
  English. Line length is 79 columns, ruff `py313` style.

## `domain_utils/`

This package is **not wired into the current project** and its imports will
fail here: `medical_criteria.py` imports `data_loader.torch_dataloader` and a
local `config` module (with `config.LUNA25_LABELS` /
`config.NODULE_BLOCKS_ROOT`) that do not exist in this repo, plus `torch`,
which is not installed. Its domain is CT-scan lung-nodule morphometry
(LUNA25/NLST datasets) — a different clinical domain from the breast-cancer
gene-expression work in `data/` and `notebooks/`. Treat it as reference code
carried over from another project rather than part of this repo's working
pipeline unless/until it is actually connected up (a `config.py` and a
`data_loader` package would need to be added).

- `medical_criteria.py` — GPU (torch) nodule segmentation and morphometry
  (size, margin irregularity, density) from CT patches; has a `selftest()`
  entry point that validates against synthetic phantoms with known ground
  truth.
- `nodule_match.py` — one-to-one geometric matching between LUNA25 nodules
  and NLST `ctab` anomalies via `scipy.optimize.linear_sum_assignment`
  (laterality + anatomical-height cost); pure pandas/numpy/scipy, importable
  standalone.
- `visualizer_3d.py` — ipywidgets-based interactive 3-plane + 3D point-cloud
  viewer for a segmented nodule mask (`explore_mask(volume, mask, ...)`);
  needs `ipywidgets`.
- `__init__.py` is empty (no re-exports), same convention as `utils/`.
