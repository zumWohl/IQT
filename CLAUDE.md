# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

MSc/research project: **"Image quality enhancement for dynamic lung MRI"** (UCL Hawkes
Institute, supervisors Danny Alexander & Mina Kim, IQT group). The goal is to adapt
image quality transfer (IQT) and super-resolution reconstruction techniques — previously
used for brain MRI — to dynamic (oxygen-enhanced / free-breathing) lung MRI, which has
much lower SNR, respiratory motion, and low proton density. See
`MScCGVI2025_IQT_Alexander-Kim.docx` for the full proposal and
`Deep_learning_for_improving_ZTE_MRI_images_in_free_breathing.pptx` for background slides.

This is a data-analysis/model-training workspace built around Jupyter notebooks in
`Development/`, operating directly on the MRI datasets in the sibling folders below. It
is a git repository (`Development/` is the repo root) with no package/build system —
there's no `requirements.txt`/`environment.yml`; dependencies live in pre-existing conda
environments (see below).

## Environment

Two conda environments matter, for different notebooks:

- **base Anaconda env** (`C:\Users\User\anaconda3`, Python 3.13): `pydicom`, `nibabel`,
  `numpy`, `scipy`, `scikit-image`, `pandas`, `matplotlib`. No `torch`. Used for
  `preprocessing.ipynb` (no GPU needed there).
- **`IQT` conda env** (registered as Jupyter kernel name `iqt`): everything the base env
  has, plus `torch`+CUDA. Used for `train.ipynb`. **Gotcha:** both `preprocessing.ipynb`
  and `train.ipynb` have `display_name: "IQT"` in their stored kernelspec metadata, but
  the kernelspec `name` field is actually `"python3"`, which resolves to the **base env**
  regardless of display name. To actually run `train.ipynb` on GPU you must override the
  kernel explicitly: `--ExecutePreprocessor.kernel_name=iqt`. Running it with the default
  `python3` kernel will fail (no `torch`) or silently run on CPU depending on what else is
  installed there.
- The `IQT` env's `matplotlib` has a known broken backend — even an empty figure crashes
  on `savefig`/`show` (a native crash, not a Python exception) with a broken freetype/font
  DLL from its conda-forge build. Do any plotting/figure-saving through the **base env**
  instead, even from a `torch`-heavy analysis; only run the GPU-dependent compute in `IQT`.
- Execute a notebook headlessly (adjust the env/kernel per notebook, see above):
  ```
  jupyter nbconvert --to notebook --execute --inplace preprocessing.ipynb --ExecutePreprocessor.timeout=1800
  ```
  For `train.ipynb` on GPU: use the `IQT` env's `jupyter` and add
  `--ExecutePreprocessor.kernel_name=iqt`. `preprocessing.ipynb` full-pipeline runs
  (segmenting every frame of all 8 TV subjects + IQT HR + IQT LR) take 30-60+ minutes —
  run in the background. Real `train.ipynb` training runs are gated behind explicit
  `RUN_*` flags (see below) and, once enabled, take on the order of 40+ minutes per
  40-epoch run even for the small model here — per-epoch time in the live notebook/kernel
  context has consistently run noticeably slower than an equivalent isolated script,
  budget accordingly rather than assuming a quick script-level timing estimate.
- **Check the actual exit code, not just that the command returned** — piping through
  `tee` (or similar) masks nbconvert's real exit status, since the shell reports the last
  command in the pipe; redirect straight to a file and capture `$?` immediately after
  instead. A `CellExecutionError` or `CellTimeoutError` still leaves the on-disk notebook
  in its pre-execution state (`--inplace` only writes back after a fully successful run),
  so a failed run doesn't corrupt anything, but it also means "the file changed size"
  doesn't tell you the run succeeded — read the log.
  If a run stalls or gets killed unexpectedly, check for a live VS Code Jupyter kernel with
  the same notebook open (`ipykernel_launcher` processes) — it competes for the same
  file/kernel resources and has caused exactly this. Long training runs launched as
  background shell tasks have also been observed to get killed by an external session
  limit (roughly an hour) independent of any timeout you pass to nbconvert — for a run
  expected to take that long, launch it as a fully OS-detached process (e.g. PowerShell
  `Start-Process -WindowStyle Hidden`, not a tracked background shell job) so it survives.
- After editing a notebook by direct JSON manipulation (see below), validate before
  executing: `python -c "import json,nbformat; nbformat.validate(json.load(open('train.ipynb', encoding='utf-8')))"`.

### Editing notebooks

`preprocessing.ipynb` and `train.ipynb` both grow large once executed (embedded PNG
outputs push `preprocessing.ipynb` well past 40MB), which makes the `Read`/`NotebookEdit`
tools fail with a token-limit error on the whole-file read `NotebookEdit` requires first.
When that happens, edit the notebook's JSON directly with a Python script (`json.load` →
mutate `nb["cells"]` → `json.dump`) instead — this is the approach used throughout this
project's history. Keep cell `id`s stable when patching cells in place; when doing a large
structural rewrite, it's cleaner to filter `nb["cells"]` down to the cells you want to keep
(by id) and rebuild the rest fresh. `ast.parse` every code cell's source (skip markdown
cells — they aren't Python) before writing, and `nbformat.validate` the whole file before
executing.

## Data layout

```
TVs_UCL_MCMR/                    HR training dataset — 8 subjects (TV1..TV8)
  TV<N>_<date>_sorted_UCL/
    IM_....                      extensionless multiframe DICOM (the image data — use this)
    XX_....                      extensionless metadata-only object (NOT for image loading)

IQT/
  HR/                            IQT HR target scan — Analyze .hdr/.img pairs,
                                  named IM-[slice]-[echo]-[time].hdr, e.g. IM-01-01-001.hdr
  LR/                            IQT LR scan — same naming convention as HR
```

`IQT_HR_ROOT`/`IQT_LR_ROOT` are resolved live (`_find_iqt_hr_lr_roots` in
`preprocessing.ipynb`) by scanning `IQT/` for the two folders containing `IM-*-*-*.hdr`
files and classifying by native resolution (higher = HR) — don't hardcode a path, this
folder layout has already changed once (from a nested `2023_08_10/<protocol>_raw/`
structure to the flat `HR`/`LR` above).

### HR dataset (`TVs_UCL_MCMR`) specifics

- Each subject: 96x96 images, 6 slices, 2 echoes (TE1~0.71ms, TE2~1.2ms), 340 dynamics,
  4080 frames total per subject, loaded via `pydicom.dcmread` on the `IM_....` file.
- Frame order is **not** implicit row-major — it was verified against
  `PerFrameFunctionalGroupsSequence`/`DimensionIndexValues` (`= [1, slice, dynamic, echo]`):
  slice-major (680-frame blocks per slice), dynamic-major within a slice, echo alternating
  (TE1, TE2, TE1, TE2, ...). This makes `frames.reshape(6, 340, 2, 96, 96)` valid — see
  `load_tv_volume`/`get_tv_frame` in `preprocessing.ipynb` for the canonical loader.
- Gas phases (1-based dynamic numbers): **1-60 = air, 61-210 = oxygen (always excluded
  from analysis), 211-340 = air.** Only the two air phases (`AIR_PHASE_DYNAMICS`, 190
  dynamics total) are used; the oxygen phase is a hard exclusion in every task so far.

### IQT HR/LR specifics

- Different participants than the TV cohort, and TV/IQT are never treated as
  anatomically registered — comparisons are visual/standardised-view only.
- IQT HR and IQT LR are **different pulse sequences** (`HighVFA5DEG` vs `LowVFA5DEG` —
  different flip angles), not simply the same acquisition at two resolutions. Confirmed to
  matter: after independently percentile-normalising each, real IQT LR's low-frequency
  (near-DC) spectral power is still only ~45% of real IQT HR's — a gap resolution loss
  alone cannot produce, since low frequencies are essentially untouched by any plausible
  blur/truncation. Treat real HR-vs-LR intensity/contrast differences as partly a protocol
  difference, not purely a resolution effect, when interpreting real-data evaluation
  results.
- 96x96 (HR) vs **64x64 (LR, not 96x96)** — confirmed live via a ported `iqt_index` scan,
  6 slices, 2 echoes, 30 dynamics each, 360 `.hdr`/`.img` pairs per resolution. Every
  active call site uses **TE1 only** — TE2 is loaded nowhere in the live pipeline.
  `load_iqt_lr_frame` always returns native 64x64; the only place LR gets upscaled (cubic,
  to 96x96) is a segmentation-only helper, so segmentation can reuse the constants already
  calibrated at 96x96 rather than deriving/validating a second set — this never reaches
  training data.
- Real IQT HR/LR headers report an identical, flat, uncalibrated 1.0mm/voxel spacing for
  both resolutions (confirmed by checking the affine matrices too) — this is almost
  certainly an Analyze-format placeholder, not real acquisition geometry. Don't trust these
  headers for physical FOV/spacing; a downstream analysis that needs "cycles/mm" has to
  make (and state) an explicit shared-FOV assumption instead.

## `Development/preprocessing.ipynb` — segmentation, respiratory phase, and pairing

Renamed from `data_prep.ipynb`; superseded earlier copies (`data_prep.ipynb`,
`data_prep_old.ipynb`, `data_exploration.ipynb` — a colleague's notebook, never modify it)
live in `Development/old_notebooks/`, kept only as history/reference.

Two parts:

**Part 1** (`find_project_root`, `load_tv_volume`/`get_tv_frame`, `load_iqt_hr_frame`/
`load_iqt_lr_frame`, `normalise_for_display`): the shared loaders used everywhere else in
the notebook, plus a side-by-side visual comparison of both datasets.

**Part 2** — lung segmentation and phase-based anatomical alignment, used to derive
everything downstream needs (training crops, and the real LR↔HR pairing):
1. `segment_lungs_te1` — deterministic baseline lung segmentation on a single TE1 frame:
   body-silhouette isolation, a percentile ladder of body-relative darkness for lung
   candidates, midline-split handling for merged bilateral blobs, shape/position
   filtering (explicitly rejects shoulder/rib tissue via a central-thoracic-region band),
   and bilateral pair validation (separation, vertical overlap, size ratio).
2. `compute_subject_aggregate_curve` — per-dynamic sum of lung area across all 6 slices
   (NaN-safe per-slice failures), with a running-median temporal-consistency filter and
   run-edge exclusion, then selects one subject-level max/min-expansion dynamic from the
   smoothed curve.
3. `estimate_landmarks`/`extract_standardised_frame` — per-subject landmark (median
   `mid_x`/`diaphragm_y` across slices), cropped via integer-offset slicing only (no
   interpolation/padding). `CROP_SIZE = 80` (committed after confirming 64 would truncate
   real lung pixels), with a per-dataset anchor derived from that dataset's own robust
   extent (`per_subject_crop`) — covers TV and IQT HR; **IQT LR is not yet in
   `per_subject_crop`**, so pairing/Dice work upscales LR for segmentation instead of
   cropping it.
4. `SUBJECT_SLICE_EXCLUSIONS` (currently `{"TV3...": [6]}`) / `TRAINING_EXCLUDE` — per-
   subject per-slice reliability exclusions threaded through curve/landmark computation
   and (separately) the training export; kept distinct from `MEASUREMENT_EXCLUDE`
   (crop-sizing diagnostics), which carries additional automated-screen flags never
   promoted to a training exclusion without visual confirmation.
5. **Step 7 — real IQT HR/LR respiratory-phase pairing** (`compute_subject_aggregate_curve`
   on IQT LR, `find_local_extrema`, `assign_phase_bins`, `pair_lr_to_hr_by_phase`,
   `dice_coefficient`): each dynamic's phase is its fractional position within its own
   local half-cycle (bounded by consecutive local extrema of its own smoothed curve, not a
   single global `[0,1]` rescaling — that older approach is kept only as a superseded
   comparison), discretized into `PHASE_BINS=10` bins with an `"inhale"`/`"exhale"`
   direction. `pair_lr_to_hr_by_phase` matches each LR dynamic to an HR dynamic sharing the
   same `(phase_bin, direction)` key (nearest-smoothed-area tiebreak among candidates),
   falling back to nearest-area-across-all-valid-HR (flagged `fallback=True`) when no HR
   dynamic shares that key. `dice_coefficient` on the resulting pairs' lung masks
   (`pairing_quality_df`) is **report-only, not gated** — every pair is kept, low Dice is a
   diagnostic signal (e.g. a phase-mismatched pair), not an exclusion.

All constants are defined near the top of Part 2 and documented inline with why each value
was chosen — check there before tuning behaviour rather than hardcoding new numbers.

Design principles that carry over to any extension:
- 1-based slice/echo/dynamic numbers in all function signatures and plot titles; convert
  to 0-based only for array indexing.
- Never modify the original loaded arrays — normalisation/segmentation always work on
  copies.
- Segmentation success/failure is an explicit `dict` field (`"success"`, `"reason"`), not
  an exception or a silent skip.
- TE2 is never independently segmented or used; only TE1 drives the live pipeline.

## `Development/train.ipynb` — degradation model, U-Net, and CV

Loads `Outputs/tv_hr_crop_synthetic_lr_export.npz` (the pre-cropped 80x80 TV frames,
produced by `preprocessing.ipynb`'s export step) once and keeps it resident in memory;
that export's own baked-in `lr_synth_frames` field is never used for training/eval — LR is
always regenerated on the fly via `degrade_hr_to_lr` (Gaussian blur, edge-attenuation-
corrected, then resize down by `SCALE_FACTOR=1.5` and back up, both cubic) so sigma can
vary per sample. Training sigma is randomised in `[SIGMA_MIN, SIGMA_MAX]`
(`SIGMA_REF * [0.7, 1.3]`, `SIGMA_REF` derived from FWHM=`SCALE_FACTOR` via the standard
FWHM→sigma conversion); val/test always use fixed `SIGMA_REF` so their metrics are
comparable across epochs. `UNet` (`CHANNELS = [8, 16, 32, 64]`) predicts the **residual**
`hr - lr`, not `hr` directly. Optional geometric augmentation (`USE_GEOMETRIC_AUG`,
rotation ≤5°/translation ≤3px, applied to HR *before* re-degrading so the pair stays
consistent) is on in the frozen config.

Split is always by **subject**, never by frame (`VAL_SUBJECT="TV4..."`,
`TEST_SUBJECT="TV8..."` held out as the internal synthetic test — real IQT evaluation is a
separate, later concern). The notebook runs, in order: a single-split baseline, a sampler
comparison, an augmentation comparison, then a **leave-one-subject-out CV across
TV1–TV7** (`cv_fold_<subject>_history.csv` + `cv_summary.csv`, `best_epoch` = highest
val_psnr epoch per fold), then a **final combined run** training once on all of TV1–7 for
the CV-median epoch count with no further validation, evaluated exactly once on TV8. TV8
never enters model selection or hyperparameter choices anywhere upstream of that one
evaluation.

Every actual training loop is gated behind its own boolean flag, default `False`
(`RUN_TRAINING`, `RUN_SAMPLER_COMPARISON`, `RUN_AUG_COMPARISON`, `RUN_CV`, `RUN_FINAL`) —
there's no single master switch, so re-running the notebook top-to-bottom trains nothing
and never overwrites existing checkpoints/CSVs by accident. Flip the specific flag you
mean to run, deliberately, per section.

## Downstream evaluation (planned)

The intended pipeline is `preprocessing.ipynb` → `train.ipynb` → `evaluate.ipynb` (frozen
model's performance in the controlled synthetic domain: CV summary, synthetic TV8 test) →
`test.ipynb` (real acquired IQT LR → frozen model → paired real IQT HR, using
`preprocessing.ipynb`'s respiratory-phase pairing, kept conceptually separate from the
synthetic evaluation). As of writing, `evaluate.ipynb` exists but is empty and
`test.ipynb` doesn't exist yet — don't assume either has any implementation without
checking the file first.

## Output conventions

Generated figures/data go to `Development/Outputs/` (created via `OUTPUT_DIR.mkdir(...)`
in the notebooks) with descriptive filenames. Never write derived files back into
`TVs_UCL_MCMR/` or `IQT/`. Two artifact families currently coexist there — don't confuse
them:
- The frozen `train.ipynb` pipeline's own artifacts: `spatial_baseline_best.pt` (best
  val-loss checkpoint from the single-split baseline run), `spatial_baseline_final.pt`
  (unconditional last-epoch weights from the final CV-epoch-count combined run — this is
  the one real evaluation work should load), `spatial_baseline_history.csv`,
  `cv_fold_<subject>_history.csv` / `cv_summary.csv`.
- `Outputs/checkpoints/` and `Outputs/results/` belong to an earlier, separate experiment
  track (a k-space-truncation degradation alternative that was explored and is not part of
  the current frozen `train.ipynb` pipeline) — don't mix its checkpoints into the frozen
  pipeline's evaluation.

# Instructions for Claude

## Plotting

- When plotting, don't save the plots to `Outputs/`. `Outputs/` is reserved for the
  notebooks' own pipeline artifacts (see Output conventions above); if a plot is only
  for your own diagnostic purposes, display it inline or write it to the scratchpad
  directory instead.

## Writing style

- Avoid em dashes. Use commas, parentheses, or separate sentences instead.
- Never use these words: delve, dive into, navigate (figurative), underscore, bolster, foster, harness, leverage, unpack, shed light on, pave the way, pivotal, groundbreaking, cutting-edge, transformative, game-changing, innovative, robust, comprehensive, seamless, intricate, vibrant, multifaceted, holistic, testament, landscape (figurative), realm.
- Never use these phrases: "In today's...", "It's important to note...", "When it comes to...", "This is where X comes in", "Let's break it down".
- Avoid these structures: "It's not just X — it's Y", "Not only X but Y", "This isn't about X. It's about Y."
- Use contractions. Write plain prose, not uniform bullet-point lists.
- Vary paragraph and sentence length. Don't signpost ("Let's explore"). Start on substance, not context-setting.
