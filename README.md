# 3D CT Post-Processing and Denoising

> **Research/demo project.** This repository contains code for
> generating generated 3D CT-like phantom data, training a 3D U-Net
> denoiser, and experimenting with post-processing on reconstructed
> volumes. Generated phantom outputs are not clinical CT images and must
> not be presented as validated medical results.

## Overview

The workflow in `post_processing.py` includes:

1.  **Phantom dataset generation** --- creates simplified 3D body,
    lung-like, and bone-like regions.
2.  **Noise simulation** --- adds image-domain Gaussian noise to create
    paired clean/noisy synthetic volumes.
3.  **Dataset splitting** --- separates volumes into training,
    validation, and test subsets.
4.  **3D U-Net training** --- trains a convolutional 3D U-Net using 3D
    patches.
5.  **Synthetic evaluation** --- compares predictions with synthetic
    targets and calculates PSNR and SSIM.
6.  **Reconstruction experiments** --- includes code paths for running
    inference on a supplied 3D reconstructed volume, visualizing slices,
    and saving denoised volumes.
7.  **Self-supervised baseline** --- includes a separate masked-voxel
    training experiment that does not require a clean target volume.


## Requirements

The script uses Python libraries including:

-   NumPy
-   PyTorch
-   Matplotlib
-   scikit-image
-   SciPy
-   IPython / Google Colab utilities

Install common dependencies in a notebook or environment:

``` bash
pip install numpy torch matplotlib scikit-image scipy
```

For Google Colab, select a GPU runtime if available. Some code sections
use Colab's `google.colab.files` upload utility and `/content` paths, so
they are designed for Colab.

## Phantom dataset configuration

The dataset generator uses these defaults:

  Setting                                                   Default
  ---------------------- ------------------------------------------
  Number of volumes                                             500
  Volume shape               `(142, 232, 194)` in `(Z, Y, X)` order
  Gaussian noise level     `0.04` times the maximum clean intensity
  Random seed                                                  `42`
  Output directory                  `/content/ct_denoising_dataset`
  Train split                                                   70%
  Validation split                                              15%
  Test split                                                    15%
  Saved volume type                                       `float16`

The generator creates the following structure:

``` text
/content/ct_denoising_dataset/
├── metadata.json
├── train/
│   ├── clean/
│   └── noisy/
├── val/
│   ├── clean/
│   └── noisy/
└── test/
    ├── clean/
    └── noisy/
```

Each clean volume is paired with a noisy volume of the same filename.
`metadata.json` records the volume shape, noise model, split counts,
data type, and random seed.

**Storage warning:** 500 volumes of this size require substantial disk
space. The script estimates storage and stops if the available Colab
disk space is insufficient. Reduce `NUM_VOLUMES` or the volume
dimensions for a quick test.

## Model and training

The supervised model is a 3D U-Net with 3D convolution blocks,
max-pooling, transposed-convolution upsampling, and skip connections.
Training uses:

-   Random 3D patches of size `64 × 64 × 64`
-   Batch size `2`
-   Adam optimizer
-   Mean squared error (MSE) loss
-   Validation loss to select and save the best checkpoint

The supervised checkpoint is saved at:

``` text
/content/best_ct_unet3d.pth
```

The script also includes a separate self-supervised 3D U-Net experiment.
It masks a fraction of voxels in randomly sampled patches and computes a
loss at the masked locations. Its checkpoint and output paths are:

``` text
/content/self_supervised_ct_denoiser.pth
/content/self_supervised_denoised_volume.npy
```

These are separate experiments; do not assume their weights or outputs
are interchangeable.

## Running in Google Colab

1.  Upload `post_processing.py` to Colab or copy its code into notebook
    cells.
2.  Review the configuration values and available disk space before
    generating the full generated phantom dataset.
3.  Run the generated phantom dataset-generation section.
4.  Inspect a clean/noisy sample to confirm that the generated pairs and
    shapes are correct.
5.  Run the supervised 3D U-Net training section and confirm that the
    checkpoint is saved.
6.  Review the phantom test metrics and visual comparison.
7.  If testing a real reconstruction, upload the correct `.npy` volume
    and verify its shape, axis order, intensity units, and value range
    first.
8.  Run only the intended inference section, then inspect the saved
    output and compare slices using the same display scale.

### Input and output paths used in the script

  ------------------------------------------------------------------------------
  Purpose                       Path
  ----------------------------- ------------------------------------------------
  Phantom dataset \`/           content/ct_denoising_dataset\`

  Supervised model checkpoint   `/content/best_ct_unet3d.pth`

  Example real input volume     `/content/reconstructed_volume.npy`

  Supervised inference output   `/content/denoised_chest_volume_3d.npy`

  Self-supervised checkpoint    `/content/self_supervised_ct_denoiser.pth`

  Self-supervised output        `/content/self_supervised_denoised_volume.npy`
  ------------------------------------------------------------------------------

Paths may need to be changed to match the actual files in your runtime.
The script uses more than one reconstruction filename in different
sections; verify the intended input path before inference.

## Evaluation

For held-out phantom volumes, the code calculates:

-   **PSNR (Peak Signal-to-Noise Ratio):** compares prediction error
    with the clean phantom target.
-   **SSIM (Structural Similarity Index):** compares local structural
    similarity with the clean phantom target.

Metrics should be computed on data not used for training. They measure
performance on the generated phantom data distribution only. They do not
establish artifact removal or diagnostic quality on real CT scans.

For real reconstructed volumes without ground truth, use carefully
controlled visual comparisons, inspect multiple slices and orientations,
check for changes in intensity statistics, and avoid claiming a
quantitative improvement based on synthetic metrics.

## Troubleshooting

-   **Dataset generation stops due to disk space:** reduce `NUM_VOLUMES`
    or test with smaller dimensions.
-   **`FileNotFoundError`:** confirm that the dataset, input volume, or
    checkpoint exists at the exact path used by that code section.
-   **Shape mismatch:** confirm the volume is 3D and that model patch
    dimensions are compatible with the U-Net pooling/upsampling path.
-   **Image looks almost black:** inspect percentiles and use a
    documented, robust display window. Do not change the stored voxel
    values just to improve visualization.
-   **Rings or structured patterns remain:** generic denoising may not
    remove reconstruction artifacts. Investigate the reconstruction
    geometry, calibration, projections/sinogram, intensity conversion,
    and axis ordering with the reconstruction team.
-   **Output looks too smooth:** reduce smoothing strength or reassess
    model training and data realism; do not assume lower variation means
    better image quality.
