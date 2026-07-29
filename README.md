# DC-Seg Reproduction: Missing-Modality Robustness and Representation Analysis

**English** | [中文](README_CN.md)

An independent reproduction and mechanism-oriented study of
[DC-Seg: Disentangled Contrastive Learning for Brain Tumor Segmentation with Missing Modalities](https://papers.miccai.org/miccai-2025/0213-Paper0653.html)
on BraTS2020.

This public repository presents the main results and aggregate analysis from
the project. It is intentionally separated from the private working archive:
datasets, checkpoints, server paths, intermediate feature arrays, logs, and
case-level prediction files are not included.

> **Main finding.** The reproduced model closely matches the released
> checkpoint in segmentation performance and missing-modality degradation.
> The modality branch is strongly disentangled, while convincing subject-level
> anatomical alignment emerges mainly **after feature fusion**, rather than in
> the raw pre-fusion anatomical encoders.

## Project at a glance

| Item | Setting |
|---|---|
| Dataset | BraTS2020 |
| MRI modalities | FLAIR, T1ce, T1, T2 |
| Missing-modality settings | All 15 non-empty modality combinations |
| Test set | 100 cases |
| Segmentation targets | Whole Tumor (WT), Tumor Core (TC), Enhancing Tumor (ET) |
| Reproduction model | 300 epochs, 45,000 iterations |
| Reference | Authors' released `model_last` checkpoint |
| Extended analyses | Robustness, representation probing, FP/FN decomposition, label transitions, modality attribution, failure cases |

## Detailed reports

1. DC-Seg Reproduction and Representation Analysis  
   [English](reports/01_DC-Seg_Reproduction_and_Representation_Analysis_en.pdf) · [中文](reports/01_DC-Seg_Reproduction_and_Representation_Analysis_cn.pdf)  
   Training and evaluation reproduction, all 15 missing-modality settings, and
   quantitative analysis of modality, pre-fusion anatomical, and fused
   anatomical representations.
2. DC-Seg Error Decomposition and Modality Contribution Analysis  
   [English](reports/02_DC-Seg_Error_Decomposition_and_Modality_Contribution_Analysis_en.pdf) · [中文](reports/02_DC-Seg_Error_Decomposition_and_Modality_Contribution_Analysis_cn.pdf)  
   Region- and label-level error decomposition, context-balanced modality
   contribution, and representative failure-case analysis.

Each report is available in both English and Chinese. The English versions
mirror the analytical scope of their Chinese counterparts.

## What I did

1. Reconstructed the BraTS2020 training and evaluation pipeline, including
   modality-mask sampling and sliding-window inference.
2. Evaluated every non-empty combination of the four MRI modalities under the
   same 100-case test split.
3. Probed modality, pre-fusion anatomical, and fused anatomical features using
   clustering and matched-versus-shuffled similarity analysis.
4. Decomposed segmentation errors beyond Dice using false-positive and
   false-negative statistics and label-transition matrices.
5. Estimated how each modality suppresses specific error transitions across
   matched input contexts.
6. Inspected representative spatial failure modes introduced by missing
   modalities.

## Reproduction results

Average Dice across all 15 modality combinations:

| Model | WT | TC | ET (post-processed) |
|---|---:|---:|---:|
| Reproduction, epoch 300 | **0.87** | **0.79** | **0.64** |
| Released checkpoint | 0.88 | 0.79 | 0.65 |

Under full-modality input, the reproduced model obtained
**0.91 / 0.86 / 0.80** for WT / TC / post-processed ET, compared with
**0.91 / 0.86 / 0.82** from the released checkpoint.

The reproduction used fewer iterations than the authors' run
(45,000 versus approximately 55,000). Because training was resumed in stages,
the polynomial learning-rate schedule restarted at stage boundaries. I
therefore interpret the result as a reproduction of the main inference
behaviour, not as evidence that the optimization schedules were identical.

The per-mask aggregate table is available at
[`results/dice_by_modality_mask.csv`](results/dice_by_modality_mask.csv). It
reports both raw and post-processed ET scores.

## Robustness to missing modalities

Dice drop relative to full-modality inference:

| Available modalities | WT drop | TC drop | ET drop |
|---|---:|---:|---:|
| One modality | 0.09 | 0.15 | 0.25 |
| Two modalities | 0.03 | 0.07 | 0.14 |
| Three modalities | 0.01 | 0.03 | 0.06 |

- **FLAIR and T2** are most important for edema and overall WT extent.
- **T1ce** is difficult to replace for TC and especially ET.
- More modalities reduce degradation, but cannot fully recover information
  supplied directly by the missing pathology-sensitive sequence.

## Representation analysis

### The modality branch is strongly disentangled

K-means on modality features recovered all four MRI modalities with
**ARI = 1.00** and **NMI = 1.00** for both the reproduced and released models.
The silhouette scores were 0.99 and 0.98, respectively.

![t-SNE visualization of modality features](figures/modality_feature_tsne.png)

### Pre-fusion anatomical features remain modality dependent

For the reproduced model, same-subject cross-modality similarity was 0.41,
whereas different subjects from the same modality reached 0.68. The released
checkpoint showed the same ordering, 0.40 versus 0.57. The raw anatomical
encoder outputs therefore contain subject information, but should not be
interpreted as fully modality-invariant shared anatomy.

![Pre-fusion anatomical features remain clustered by modality](figures/prefusion_anatomical_feature_tsne.png)

### Subject-level alignment emerges after fusion

| Model | Same subject, different input | Different subjects, all inputs | Different subjects, same input |
|---|---:|---:|---:|
| Reproduction | **0.64** | 0.43 | 0.49 |
| Released checkpoint | **0.65** | 0.43 | 0.47 |

![Matched-subject alignment improves after feature fusion](figures/fusion_subject_alignment_gap.png)

This supports a more precise interpretation of DC-Seg: the modality branch
learns a clear modality-specific space, while the fusion module is where a
more convincing shared anatomical representation is formed.

## Error decomposition and modality contribution

- **WT:** errors are generally false-positive dominated. FLAIR and T2 jointly
  preserve edema coverage.
- **TC:** errors are mostly false-negative dominated. Without T1ce, true core
  voxels increasingly flow to background or edema.
- **ET:** has the strongest false-positive problem and the largest sensitivity
  to T1ce absence.

Label-level analysis further showed that background-edema exchange (`0 ↔ 2`)
drives much of the WT expansion and contraction, while NCR/NET-to-enhancing
tumor confusion (`1 → 3`) is an important source of ET over-segmentation.

Across matched modality additions, T1ce most strongly suppressed
enhancing-tumor `3 → 2` errors (16.44 percentage points) and NCR/NET `1 → 3`
errors (15.74 points). FLAIR and T2 most clearly suppressed edema `2 → 0`
errors.

## Qualitative inspection

The example below compares a single BraTS2020 case under all 15 modality masks
using a common slice and color scheme.

![Segmentation predictions across modality combinations](figures/qualitative_missing_modality_case.png)

Recurring failure modes included:

1. joint FLAIR/T2 absence turning connected edema regions into background;
2. T1ce absence converting enhancing tumor into non-enhancing labels;
3. false enhancing-tumor expansion in some cases without true ET.

## Repository contents

```text
figures/
  modality_feature_tsne.png
  prefusion_anatomical_feature_tsne.png
  fusion_subject_alignment_gap.png
  qualitative_missing_modality_case.png
results/
  dice_by_modality_mask.csv
  modality_clustering_summary.csv
  fusion_alignment_summary.csv
reports/
  01_DC-Seg_Reproduction_and_Representation_Analysis_cn.pdf
  01_DC-Seg_Reproduction_and_Representation_Analysis_en.pdf
  02_DC-Seg_Error_Decomposition_and_Modality_Contribution_Analysis_cn.pdf
  02_DC-Seg_Error_Decomposition_and_Modality_Contribution_Analysis_en.pdf
DATA_SCOPE.md
README.md
README_CN.md
```

Only aggregate statistics and selected presentation figures are included.
See [DATA_SCOPE.md](DATA_SCOPE.md) for the public-release boundary.

## Status and attribution

This is ongoing independent reproduction work, not an official implementation
or a verification of every claim in the paper. All credit for the original
method and implementation belongs to the DC-Seg authors. See the
[MICCAI 2025 paper](https://papers.miccai.org/miccai-2025/0213-Paper0653.html)
and the [official repository](https://github.com/CuCl-2/DC-Seg).
# DC-Seg Reproduction: Missing-Modality Robustness and Representation Analysis

**English** | [中文](README_CN.md)

An independent reproduction and mechanism-oriented study of
[DC-Seg: Disentangled Contrastive Learning for Brain Tumor Segmentation with Missing Modalities](https://papers.miccai.org/miccai-2025/0213-Paper0653.html)
on BraTS2020.

This public repository presents the main results and aggregate analysis from
the project. It is intentionally separated from the private working archive:
datasets, checkpoints, server paths, intermediate feature arrays, logs, and
case-level prediction files are not included.

> **Main finding.** The reproduced model closely matches the released
> checkpoint in segmentation performance and missing-modality degradation.
> The modality branch is strongly disentangled, while convincing subject-level
> anatomical alignment emerges mainly **after feature fusion**, rather than in
> the raw pre-fusion anatomical encoders.

## Project at a glance

| Item | Setting |
|---|---|
| Dataset | BraTS2020 |
| MRI modalities | FLAIR, T1ce, T1, T2 |
| Missing-modality settings | All 15 non-empty modality combinations |
| Test set | 100 cases |
| Segmentation targets | Whole Tumor (WT), Tumor Core (TC), Enhancing Tumor (ET) |
| Reproduction model | 300 epochs, 45,000 iterations |
| Reference | Authors' released `model_last` checkpoint |
| Extended analyses | Robustness, representation probing, FP/FN decomposition, label transitions, modality attribution, failure cases |

## Detailed reports

1. [DC-Seg Reproduction and Representation Analysis](reports/01_DC-Seg_Reproduction_and_Representation_Analysis.pdf)  
   Training and evaluation reproduction, all 15 missing-modality settings, and
   quantitative analysis of modality, pre-fusion anatomical, and fused
   anatomical representations.
2. [DC-Seg Error Decomposition and Modality Contribution Analysis](reports/02_DC-Seg_Error_Decomposition_and_Modality_Contribution_Analysis.pdf)  
   Region- and label-level error decomposition, context-balanced modality
   contribution, and representative failure-case analysis.

Both reports are written in Chinese, with English terminology retained for the
main methods, metrics, and anatomical labels.

## What I did

1. Reconstructed the BraTS2020 training and evaluation pipeline, including
   modality-mask sampling and sliding-window inference.
2. Evaluated every non-empty combination of the four MRI modalities under the
   same 100-case test split.
3. Probed modality, pre-fusion anatomical, and fused anatomical features using
   clustering and matched-versus-shuffled similarity analysis.
4. Decomposed segmentation errors beyond Dice using false-positive and
   false-negative statistics and label-transition matrices.
5. Estimated how each modality suppresses specific error transitions across
   matched input contexts.
6. Inspected representative spatial failure modes introduced by missing
   modalities.

## Reproduction results

Average Dice across all 15 modality combinations:

| Model | WT | TC | ET (post-processed) |
|---|---:|---:|---:|
| Reproduction, epoch 300 | **0.87** | **0.79** | **0.64** |
| Released checkpoint | 0.88 | 0.79 | 0.65 |

Under full-modality input, the reproduced model obtained
**0.91 / 0.86 / 0.80** for WT / TC / post-processed ET, compared with
**0.91 / 0.86 / 0.82** from the released checkpoint.

The reproduction used fewer iterations than the authors' run
(45,000 versus approximately 55,000). Because training was resumed in stages,
the polynomial learning-rate schedule restarted at stage boundaries. I
therefore interpret the result as a reproduction of the main inference
behaviour, not as evidence that the optimization schedules were identical.

The per-mask aggregate table is available at
[`results/dice_by_modality_mask.csv`](results/dice_by_modality_mask.csv). It
reports both raw and post-processed ET scores.

## Robustness to missing modalities

Dice drop relative to full-modality inference:

| Available modalities | WT drop | TC drop | ET drop |
|---|---:|---:|---:|
| One modality | 0.09 | 0.15 | 0.25 |
| Two modalities | 0.03 | 0.07 | 0.14 |
| Three modalities | 0.01 | 0.03 | 0.06 |

- **FLAIR and T2** are most important for edema and overall WT extent.
- **T1ce** is difficult to replace for TC and especially ET.
- More modalities reduce degradation, but cannot fully recover information
  supplied directly by the missing pathology-sensitive sequence.

## Representation analysis

### The modality branch is strongly disentangled

K-means on modality features recovered all four MRI modalities with
**ARI = 1.00** and **NMI = 1.00** for both the reproduced and released models.
The silhouette scores were 0.99 and 0.98, respectively.

![t-SNE visualization of modality features](figures/modality_feature_tsne.png)

### Pre-fusion anatomical features remain modality dependent

For the reproduced model, same-subject cross-modality similarity was 0.41,
whereas different subjects from the same modality reached 0.68. The released
checkpoint showed the same ordering, 0.40 versus 0.57. The raw anatomical
encoder outputs therefore contain subject information, but should not be
interpreted as fully modality-invariant shared anatomy.

![Pre-fusion anatomical features remain clustered by modality](figures/prefusion_anatomical_feature_tsne.png)

### Subject-level alignment emerges after fusion

| Model | Same subject, different input | Different subjects, all inputs | Different subjects, same input |
|---|---:|---:|---:|
| Reproduction | **0.64** | 0.43 | 0.49 |
| Released checkpoint | **0.65** | 0.43 | 0.47 |

![Matched-subject alignment improves after feature fusion](figures/fusion_subject_alignment_gap.png)

This supports a more precise interpretation of DC-Seg: the modality branch
learns a clear modality-specific space, while the fusion module is where a
more convincing shared anatomical representation is formed.

## Error decomposition and modality contribution

- **WT:** errors are generally false-positive dominated. FLAIR and T2 jointly
  preserve edema coverage.
- **TC:** errors are mostly false-negative dominated. Without T1ce, true core
  voxels increasingly flow to background or edema.
- **ET:** has the strongest false-positive problem and the largest sensitivity
  to T1ce absence.

Label-level analysis further showed that background-edema exchange (`0 ↔ 2`)
drives much of the WT expansion and contraction, while NCR/NET-to-enhancing
tumor confusion (`1 → 3`) is an important source of ET over-segmentation.

Across matched modality additions, T1ce most strongly suppressed
enhancing-tumor `3 → 2` errors (16.44 percentage points) and NCR/NET `1 → 3`
errors (15.74 points). FLAIR and T2 most clearly suppressed edema `2 → 0`
errors.

## Qualitative inspection

The example below compares a single BraTS2020 case under all 15 modality masks
using a common slice and color scheme.

![Segmentation predictions across modality combinations](figures/qualitative_missing_modality_case.png)

Recurring failure modes included:

1. joint FLAIR/T2 absence turning connected edema regions into background;
2. T1ce absence converting enhancing tumor into non-enhancing labels;
3. false enhancing-tumor expansion in some cases without true ET.

## Repository contents

```text
figures/
  modality_feature_tsne.png
  prefusion_anatomical_feature_tsne.png
  fusion_subject_alignment_gap.png
  qualitative_missing_modality_case.png
results/
  dice_by_modality_mask.csv
  modality_clustering_summary.csv
  fusion_alignment_summary.csv
reports/
  01_DC-Seg_Reproduction_and_Representation_Analysis.pdf
  02_DC-Seg_Error_Decomposition_and_Modality_Contribution_Analysis.pdf
DATA_SCOPE.md
README.md
README_CN.md
```

Only aggregate statistics and selected presentation figures are included.
See [DATA_SCOPE.md](DATA_SCOPE.md) for the public-release boundary.

## Status and attribution

This is ongoing independent reproduction work, not an official implementation
or a verification of every claim in the paper. All credit for the original
method and implementation belongs to the DC-Seg authors. See the
[MICCAI 2025 paper](https://papers.miccai.org/miccai-2025/0213-Paper0653.html)
and the [official repository](https://github.com/CuCl-2/DC-Seg).
