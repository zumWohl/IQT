# Crop redesign — decision record

Notes from the session that redesigned `data_prep.ipynb`'s crop from a shared native
96×96 canvas to a single constant square `CROP_SIZE`, per-dataset landmark positioning.
Written to survive outside the conversation that produced it.

## CROP_SIZE = 80

Derivation used the **99th percentile** of a 3-consecutive-dynamic sustained pool of
each dataset's own up/down/left/right extent (not the raw max — see below), over each
dataset's `MEASUREMENT_EXCLUDE`-clean slices.

Robust global maxima: height = 58.0px (TV1), width = 64.0px (TV1).
`round_up_to_16(64.0) = 64` — but a direct truncation comparison at 64 vs. 80 showed 64
still truncated real, non-excluded lung pixels (TV1 5.7%, TV2 1.3%/14px worst overflow),
while 80 reduced that to near-zero. **CROP_SIZE is committed at 80 for that reason, not
as a bare round-up of the robust maxima.**

## Why the result is square (not because the constraint was reimposed)

The supervisor decision that started this session explicitly *dropped* the square
constraint — height and width were meant to be derived independently, each only needing
to be divisible by 16. A rectangle was live for part of this session: an early diagnostic
(raw max, before TV6's artifacts were identified and before the robust statistic
replaced the max) showed height=86.0px vs. width=98.0px, i.e. genuinely different axes.

Once TV6's contaminated slices were excluded from `MEASUREMENT_EXCLUDE` and the
percentile-based statistic replaced the raw max, the two axes' independently-derived
robust maxima (58.0px height, 64.0px width, both from TV1) happened to round up to the
**same** multiple of 16 (64), and the subsequent 64→80 bump (driven by the truncation
comparison, not axis coupling) preserved that equality. The square outcome is an
emergent property of the artifact-free data, not a rule put back into the derivation —
if a future dataset's extents don't round to the same value, the mechanism will produce
a genuine rectangle again.

## Robust statistic vs. raw max

Raw max (no percentile, TV2's shoulder-tissue artifact still in the pool) would have
given height=86.0px, width=98.0px → `round_up_to_16` → **96×112** — and 112 exceeds the
native 96×96 acquisition matrix outright, i.e. the naive max-based approach doesn't even
produce a usable crop. The 99th-percentile + 3-consecutive-dynamic-sustain statistic is
what makes CROP_SIZE=80 (well inside the native frame) possible at all.

## MEASUREMENT_EXCLUDE vs. TRAINING_EXCLUDE

Two separate lists, deliberately not merged:

- **MEASUREMENT_EXCLUDE** — union of the automated area-based full-range contamination
  screen (≥10% of a slice's dynamics), the automated extent-outlier detector (median ±
  3×MAD, ≥10% of dynamics), `MANUAL_OVERRIDE_EXCLUDE` (visually confirmed), and
  `SUBJECT_SLICE_EXCLUSIONS` (pre-existing reliability exclusions). Used only to keep bad
  data out of the CROP_SIZE/anchor *derivation*.
- **TRAINING_EXCLUDE** — frozen as exactly `MANUAL_OVERRIDE_EXCLUDE`'s contents: slices
  that were *visually/manually* confirmed bad, nothing promoted from an automated flag
  alone. This is the "don't train on this" list.

| Dataset | MEASUREMENT_EXCLUDE | Reason | In TRAINING_EXCLUDE? |
|---|---|---|---|
| TV1 | none | — | — |
| TV2 | [5, 6] | Manual — visually confirmed shoulder-tissue artifact (slice 5 dyn 30); neither automated screen caught it | Yes |
| TV3 | [6] | `SUBJECT_SLICE_EXCLUSIONS` — 188/190 segmentation failures, a reliability issue, not contamination | Yes |
| TV4 | none | — | — |
| TV5 | [5, 6] | Automated area screen + manual override agree | Yes |
| TV6 | [2, 4, 5, 6] | Slice 2: automated area screen only (**false positive**, see below). Slices 4–6: manual override after visually confirming the slice-6 dyn-49 mis-located mask | Only [4, 5, 6] |
| TV7 | [4, 5, 6] | Automated area screen + manual override agree | Yes |
| TV8 | [6] | Automated extent-outlier screen only (**false positive**, see below) | No |
| IQT_HR | [4, 5] | Automated extent-outlier + area screens; not manually reviewed | No |

## Two confirmed false positives

Flagged by an automated screen, checked visually, no real defect found:

- **TV8 slice 6** — extent-outlier detector fired at 38.4% of dynamics, but the detector
  is two-sided and fired on a *smaller*-than-typical mask (dyn 1, down=9.0px vs. slice
  median 19.0px). A small mask cannot inflate a crop size, and the overlay at that frame
  showed a normal, clean bilateral mask.
- **TV6 slice 2** — automated area-based full-range screen fired at 33/190 dynamics, but
  was never manually confirmed. Overlay at the subject's own max (dyn 249) and min (dyn
  39) expansion frames showed clean, anatomically normal bilateral lungs at both
  extremes.

Both are now back in `TRAINING_EXCLUDE`-clean training data (380 frames total — 190
each) after `included` in the export was switched from a `MEASUREMENT_EXCLUDE`-based
proxy to `TRAINING_EXCLUDE`.

## Two artifacts neither automated screen caught

- **TV2 slice 5, dyn 30** — a visually confirmed shoulder-tissue mask (not lungs), the
  original driving frame behind TV2's raw-max height and width. The area screen missed
  it because the mask's pixel *count* wasn't anomalous relative to TV2's other slices;
  the extent-outlier detector missed it because a single-frame spike gets washed out by
  the sustained-pool percentile by design. Only added to `MEASUREMENT_EXCLUDE`/
  `TRAINING_EXCLUDE` after the truncation gate traced real measured-slice truncation
  back to it.
- **TV6 slices 4–6** — the case that originally motivated this whole investigation
  (slice 6 dyn 49's mis-located mask). Missed by the area screen for the same
  count-isn't-anomalous reason, and missed *entirely* by the extent-outlier detector
  (zero outliers fired on TV6, on any slice) because the bias is systematic, not a
  single-frame spike — nothing about that dynamic looks unusual relative to that slice's
  own (already-biased) history. Added by direct visual confirmation of the overlay only.

## Margin-collapse fix and IQT_HR's zero achieved margin

The per-dataset anchor is clipped from a "balanced" starting value into a feasible
interval per axis (contain the dataset's own robust extent, stay inside the native
96×96 frame). This was originally a hard `assert lower <= upper` — and it was
**infeasible for IQT_HR**: its up-extent (99th pct, 41.2px) is numerically equal to its
own `diaphragm_y` (41.2px), meaning the lung mask's top row sits at row 0 of the native
frame for the worst-normal dynamic — there is no margin available above it at all, let
alone the requested 3px.

Fixed by collapsing the interval (`lower = min(lower, upper)`) instead of raising: the
margin shrinks to whatever is actually available, and the shortfall is reported per
dataset rather than silently absorbed or fatally raised. IQT_HR ends up with
anchor=(41, 41) and **0.0px margin achieved on the row axis vs. 3.0px requested** — the
only dataset with a reported shortfall; all others kept their full margin (or needed
only a position shift, not a margin cut).

The `_crop_origin` containment assertion (checks that the dataset's own robust extent
fits inside the crop window — a different, still-enforced invariant) passes for IQT_HR
at this anchor; margin and containment are independent checks; a margin shortfall does
not imply a containment failure.

## Open items

- **Flip-angle question for Mina** — raised this session, not investigated. Needs
  follow-up (relevant given the IQT LR/HR naming convention, `WIP_LowVFA5DEG` /
  `WIP_HighVFA5DEG` — variable flip angle protocols — in the source folder names).
- **Degradation-model edge residuals** — the HR-vs-synthetic-LR difference images
  (`slide3_hr_vs_synthetic_lr_examples.png`) show the largest residuals concentrated at
  the top edge of the crop (body/frame boundary) across all four example subjects
  checked. Not investigated further; could be a resize/blur edge effect at the crop
  boundary rather than a meaningful signal.
- **Dice needs an oracle upper bound** — Step 7's phase-bin pairing produces Dice scores
  in the 0.83–0.94 range (30 pairs), but there is no reference ceiling (e.g. a
  same-subject, same-phase repeat comparison, or HR-vs-HR at adjacent dynamics) to say
  whether that range is "good" or just "not catastrophic." Needs a follow-up calculation
  before the Dice distribution is used to argue the pairing is trustworthy.

## Note on `lr_synth_frames`

The `lr_synth_frames` array in `tv_hr_crop_synthetic_lr_export.npz` is produced by
`degrade_hr_to_lr` with a **fixed** blur sigma (1.0) and downsample factor (1.5) — a
simple, uncalibrated placeholder degradation, not fitted or validated against the real
IQT LR/HR relationship. Treat it as a **fixed-sigma reference artefact for QC/visual
comparison in this session, not the final training input** — an actual training pipeline
will likely need a degradation model tuned or learned against real paired data before
`lr_synth_frames` (or its replacement) is trusted as model input.
