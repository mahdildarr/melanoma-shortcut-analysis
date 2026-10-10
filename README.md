# Does the Melanoma Classifier Look at the Lesion?

Melanoma classification on HAM10000 with an evaluation designed to detect shortcut learning: is the model using the lesion, or the skin around it and the way the image was taken?

<p align="center"><img src="docs/assets/pipeline.svg" width="100%" alt="Pipeline"></p>

## Motivation

CNNs for melanoma detection report dermatologist-level accuracy, but earlier work has shown that such models can react to surgical ink markings and can still classify above chance when most of the lesion is hidden. HAM10000 also has two properties that make results easy to overestimate: many lesions appear in more than one image, and benign lesions documented by follow-up were imaged differently from melanomas confirmed by histopathology.

## Approach

- Splits: all images of a lesion stay in the same split. The common image-level split is also run, only to measure how much it inflates the results.
- Shortcut tests: models are evaluated on lesion-only and background-only images, and one model is trained and tested without seeing the lesion. The lesion is removed with a fixed-area box and skin-coloured fill, so its size and outline do not leak.
- Grad-CAM, measured against the lesion masks (share of the heat map on the lesion, pointing game).
- Acquisition bias: diagnosis vs diagnosis method, evaluation on the histopathology-confirmed subset, and a probe that tests whether follow-up and histopathology nevi can be told apart from the image.
- Metrics: AUROC and sensitivity at 90% specificity, with the decision threshold fixed on the validation set; subgroups by sex, age, body site and image source.
- Models: ResNet-50, DenseNet-121 and EfficientNet-B0.

The full experiment design is in [docs/DESIGN.md](docs/DESIGN.md).

## Roadmap

- [x] Data loading, lesion-level splits, masking and Grad-CAM analysis
- [x] Training and evaluation pipeline, tested on synthetic data
- [ ] Dataset audit (images per lesion, diagnosis method)
- [ ] Lesion-level vs image-level split (three seeds)
- [ ] Shortcut tests and Grad-CAM analysis
- [ ] Acquisition probe and subgroup analysis

## Code

The implementation (data pipeline, masking, training and analysis scripts, tests and a Colab notebook) is kept in a private repository for now. I am happy to share it on request.

## Data

HAM10000 and its lesion segmentations (CC BY-NC 4.0). The data is not redistributed here.

## References

- P. Tschandl, C. Rosendahl, H. Kittler. The HAM10000 dataset, a large collection of multi-source dermatoscopic images of common pigmented skin lesions. Scientific Data, 2018.
- A. Bissoto, E. Valle, S. Avila. Debiasing Skin Lesion Datasets and Models? Not So Fast. CVPR Workshops, 2020.
- J. K. Winkler et al. Association Between Surgical Skin Markings in Dermoscopic Images and Diagnostic Performance of a Deep Learning Convolutional Neural Network for Melanoma Recognition. JAMA Dermatology, 2019.
