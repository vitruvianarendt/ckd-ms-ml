# Machine Learning for the Detection of Chronic Kidney Disease from Plasma Mass Spectrometry

Code accompanying the master's thesis *Application of Machine Learning Methods
for the Detection of Chronic Kidney Disease Using Mass Spectrometry Data*
(Hana Hasanicaj, FINKI, Ss. Cyril and Methodius University, Skopje).

The analysis works with untargeted blood-plasma lipidomics acquired by
LC-ESI-MS. It asks whether chronic kidney disease can be detected from these
data under an evaluation protocol that controls for acquisition-order
confounding, and measures what is lost when spectra are converted to images.

## Repository layout

```
notebooks/ the analysis pipeline, in run order (00 -> 17)
archive/ the preliminary pipeline the thesis critiques, kept for provenance
data/ placeholder; real data is not committed (see below)
requirements.txt
```

Each notebook opens with a header cell stating its purpose, the thesis section
it maps to, and its inputs and outputs.

## Pipeline order

| Notebook | Does |
|----------|------|
| `00_inspect_data` | Inspect raw ion lists: scale, mass range, centroiding, intensity floor |
| `01_build_labels` | Reconcile the six spreadsheet tabs into one label file per task |
| `02_feature_extraction` | Single full pass -> mass profile, coarse 2-D map, images |
| `03_generate_images_occ` | Occupancy-masked / TIC-normalised image variant |
| `04_extract_embeddings` | Frozen ImageNet encoders -> cached embeddings |
| `05_batch_diagnostics` | Patient-identifier baseline, block baseline, acquisition-order matching |
| `06_mz_alignment` | Mass-axis dispersion and bin-width check |
| `07_region_discovery` | Is the signal localised on the mass axis or distributed? |
| `08_representation_redundancy` | Representational similarity across variants and encoders |
| `09_signal_search` | Main evaluation: every task x representation, permutation controls |
| `10_bootstrap_ci` | Bootstrap confidence intervals on out-of-fold predictions |
| `11_improved_features` | Feature-engineering variants on profile / map |
| `12_peak_picked_images` | Peak-picked images (threshold + rolling-window maxima) |
| `13_threshold_profiles` | Threshold-profile generation and evaluation |
| `13b_threshold_pilot` | Small raw-vs-picked occupancy pilot |
| `14_frozen_encoder_eval` | Cached embeddings with ridge / XGBoost heads |
| `15_finetune_encoders` | End-to-end fine-tuning (seeded, stratified, class-weighted) |
| `16_second_cohort_second cohort` | Second cohort (different instrument); acquisition-date batch confound |
| `17_figures` | Result figures (ROC, confusion, margins, cohort, acquisition order) |

## Data layout (not committed)

The raw acquisitions and all derived artefacts are **not** part of this
repository. They are large and patient-derived.
The notebooks expect the folders below, relative to the repository root.
Create and populate them as described, or adjust the path constants near the
top of each notebook.

| Folder | Contains | Produced by | Consumed by |
|--------|----------|-------------|-------------|
| `../ahen_data/filtered_tsv_2/` | One TSV per patient: `time`, `mz`, `intensity` (~28M rows each, ~118 GB total). The raw input. | supplied upstream | 00, 02, 12, 13 |
| `labels/` | One JSON per task (`labels_<task>.json`), patient ID -> 0/1. | 01 | most |
| `profiles/` | `mz_profiles.npy` (80k-bin profile/patient), `coarse_maps.npy` (512x64), `mz_centres.npy`, `profile_pids.json`. | 02 | 05-11, 14 |
| `images224/` | `grayscale_heatmap/` and `rgb_mz_split/`, one 224x224 PNG/patient. | 02 | 04, 08, 14 |
| `images_v2_occ_tic/` | Occupancy-masked, TIC-normalised images (same two subfolders). | 03 | 04, 14 |
| `images_picked/` | Peak-picked images (same two subfolders). | 12 | 04, 14 |
| `threshold_profiles/` | Threshold-profile arrays. | 13 | 13, 14 |
| `embeddings/` | Severity-task encoder embeddings: `<rep>__<model>.npy` + matching `.json` of IDs. | 04 | 09, 14 |
| `embeddings_detect/` | As above, detection patient set. | 04 | 14 |
| `embeddings_severity/` | As above, `severity_all`. | 04 | 14 |
| `second cohort/` | Second-cohort profiles: `mz_profiles.npy`, `profile_pids.json`, `metadata.json`. | 16 | 16 |
| `figures/` | Generated PDF/PNG figures used in the thesis. | 17 | thesis |
| `logs/` | Optional run logs. | several | --- |

### Image representations

The feature-extraction stage can render each acquisition as several image
types. The preliminary pipeline generated seven; the final analysis uses two.

| Image type | Axes / encoding | Used in final analysis? |
|------------|-----------------|-------------------------|
| `grayscale_heatmap` | m/z x retention time, intensity -> single channel | yes |
| `rgb_mz_split` | m/z split into three bands, one per colour channel | yes |
| `pseudocolor_heatmap` | colour map applied to the greyscale heatmap | no (no extra information) |
| `scatterplot` | occupied m/z x RT points | no (1-D-like, sparse) |
| `tic_plot` | total ion current over retention time | no (discards m/z) |
| `lineplot_intensity` | max intensity over retention time | no (discards m/z) |
| `recurrence_plot` | recurrence of the intensity series | no (discards m/z) |

The five unused types are retained in the generation code for completeness;
the thesis (Section 4.5.1) explains why only the two mass-preserving image
types are carried forward. Each type, when generated, is written to its own
subfolder under the image root (for example `images224/grayscale_heatmap/`).

### Second-cohort raw data

`16_second_cohort_second cohort` reads denoised mzML files (one scan per patient,
positive mode) from a path set at the top of the notebook. Adjust that
constant to point at their location.

## Reproducing

1. Create a virtual environment and `pip install -r requirements.txt`.
2. Place the raw `filtered_tsv_2/` files as above.
3. Run `01_build_labels`, then `02_feature_extraction` (the slow step: a full
 pass over ~118 GB).
4. Run the remaining notebooks in numeric order. Each caches its outputs, so
 later notebooks do not repeat earlier computation.

Every result uses repeated stratified cross-validation (5 folds, 5-10 repeats,
seed 42) with all feature selection performed inside the training fold. See
Chapter 4 of the thesis for the protocol.

## Where the data and generated images live

The raw acquisitions and all generated image representations are held on the
collaboration's compute server, not in this repository. They are large and
patient-derived, and access is available on
request.

- Raw ion-list files (`filtered_tsv_2/`): on the analysis server.
- Generated images (all types above): produced and stored on the same server
 by `02_feature_extraction` and `03_generate_images_occ`.
- Second-cohort mzML: on the server under the shared omics volume.

> **Data location:** Held on the collaboration analysis server. Access is
> available on request.

## The `archive/` notebooks

These are the preliminary pipeline, including the MLflow-tracked end-to-end
fine-tuning of several transformer architectures. The thesis reports and
analyses this work (Section 5.1) as the starting point whose headline result
it sets out to explain. They are kept for provenance and are not part of the
final analysis.
