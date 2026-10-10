# Design

How the experiments are designed: the data and its known biases, the split protocol, the models, and the tests that check whether a melanoma classifier actually looks at the lesion.

## 1. Problem

Melanoma is the deadliest common skin cancer, and early detection decides survival. CNNs trained on dermoscopic images report dermatologist-level accuracy — but several studies have shown that such models can rely on cues that are not part of the lesion: surgical ink markings (Winkler et al., 2019), rulers and image borders, and, in the extreme, still classify above chance when most of the lesion is hidden (Bissoto et al., 2020). A score earned through such shortcuts will not survive a change of clinic, camera or protocol.

This project asks three concrete questions on HAM10000:

1. **How much does evaluation design inflate the score?** Image-level vs lesion-level splits.
2. **Does the model need the lesion?** Lesion-only and background-only images, and Grad-CAM measured against lesion masks.
3. **Does the dataset itself carry a shortcut?** The diagnosis method (`dx_type`) is not independent of the diagnosis.

## 2. Data

**HAM10000** (Tschandl et al., 2018): 10,015 dermoscopic images from two sites, seven diagnoses (melanoma, melanocytic nevus, benign keratosis, basal cell carcinoma, actinic keratosis / Bowen's disease, dermatofibroma, vascular lesion). Each image has a `lesion_id` (several images can show the same lesion), the diagnosis method `dx_type` (histopathology, follow-up, expert consensus, confocal microscopy), age, sex and body site. **Lesion masks** are those released for HAM10000 by Tschandl et al. (2020). Licence: CC BY-NC 4.0.

**Tasks.** Melanoma vs all other diagnoses (default); malignant (melanoma, basal cell carcinoma, actinic keratosis) vs benign; and an **acquisition probe** — among nevi only, *follow-up* vs *histopathology* diagnosis — which asks whether the way a lesion was documented is visible in the image at all.

## 3. Splits

| Split | Unit | Purpose |
|---|---|---|
| **lesion** (default) | `lesion_id` | all images of a lesion in one split — no lesion seen in training appears in test |
| image | image | the common, leaky practice; kept only to measure its optimism |

Both are stratified by label (70 / 10 / 20 %) and use a fixed split seed, so every model and every training seed is evaluated on the same test images. Each run records the share of test images whose lesion also occurs in training (zero for the lesion split).

## 4. Models and training

ImageNet-pretrained ResNet-50, DenseNet-121 and EfficientNet-B0 with a single-logit head; 224 × 224 input; random resized crops, flips, rotations and mild colour jitter; class-weighted binary cross-entropy (melanoma is about 11 % of images); AdamW, one-cycle schedule, mixed precision. The checkpoint is selected by validation AUROC, and the operating threshold is fixed on the validation set (Youden index) before it is applied to the test set. Three seeds per configuration.

## 5. Shortcut tests

**Masked images.** Removed regions are filled with smooth noise matching the image's own skin colour, so that the model cannot recognise masking from a black or flat patch.

| Test | Train on | Test on | Interpretation |
|---|---|---|---|
| A | full images | full images | standard performance |
| B | full images | lesion only | is the surrounding skin needed? |
| C | full images | background only | how much does the model use the background? |
| **D** | **background only** | **background only** | **can melanoma be predicted without seeing the lesion?** AUROC well above 0.5 means the context carries diagnostic shortcuts |

For background-only images a **box of fixed area** (half of the image, the same for every image) centred on the lesion is removed, together with any lesion pixels outside it. Removing exactly the lesion-shaped mask would leave the lesion's size and outline visible — and melanomas tend to be larger and more irregular than nevi, so the outline alone would be a shortcut. The share of lesions that reach beyond the box is reported with every analysis.

**Grad-CAM, quantified.** For every test image: the share of Grad-CAM energy that falls on the lesion relative to the lesion's share of the image, and whether the Grad-CAM maximum lies on the lesion (pointing game).

**Acquisition bias.** The diagnosis method is cross-tabulated with the diagnosis. In HAM10000, *follow-up* documents benign lesions that were monitored over time, while melanomas are, as a rule, confirmed histopathologically — so the diagnosis method and the label are linked (the dataset audit reports the exact cross-tabulation). Two analyses follow:

* performance on the **histopathology-confirmed subset** of the test set, where both classes share the same diagnosis method;
* the **acquisition probe** (section 2): if follow-up and histopathology nevi can be told apart from the image, the acquisition setting is visible to the model and can stand in for the diagnosis. Follow-up images largely come from one image source (the `dataset` field), so the probe is read together with the cross-tabulation of diagnosis method × image source, and results are also broken down by image source.

**Subgroups.** AUROC and sensitivity by diagnosis method, image source, sex, age band and body site, wherever both classes are sufficiently represented.

## 6. Metrics

AUROC; sensitivity and specificity at the validation-chosen threshold; balanced accuracy; **sensitivity at 90 % specificity** — the clinically relevant regime, where missing a melanoma costs far more than an extra biopsy.

## 7. Limitations

* Two sites and dermoscopic images only; clinical (smartphone) photographs behave differently.
* HAM10000 contains few images of dark skin; subgroup analysis by skin type is not possible with this dataset.
* The masking tests remove information as well as shortcuts; a drop under masking is interpreted together with the background-only training control (test D), not on its own.

## References

1. P. Tschandl, C. Rosendahl, H. Kittler, "The HAM10000 dataset, a large collection of multi-source dermatoscopic images of common pigmented skin lesions," *Scientific Data*, 5, 2018.
2. P. Tschandl et al., "Human–computer collaboration for skin cancer recognition," *Nature Medicine*, 26, 2020 (lesion segmentations).
3. A. Bissoto, E. Valle, S. Avila, "Debiasing Skin Lesion Datasets and Models? Not So Fast," *CVPR Workshops*, 2020.
4. J. K. Winkler et al., "Association Between Surgical Skin Markings in Dermoscopic Images and Diagnostic Performance of a Deep Learning Convolutional Neural Network for Melanoma Recognition," *JAMA Dermatology*, 155(10), 2019.
5. R. Geirhos et al., "Shortcut learning in deep neural networks," *Nature Machine Intelligence*, 2, 2020.
6. R. R. Selvaraju et al., "Grad-CAM: Visual Explanations from Deep Networks via Gradient-based Localization," *ICCV*, 2017.
