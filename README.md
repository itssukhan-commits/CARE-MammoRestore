# ARED-Net and CARE-MammoRestore

This repository provides a PyTorch implementation of ARED-Net, CARE-MammoRestore, and the lesion-guided tumor analyzer described in the manuscript. It includes workflows for training, inference, paired evaluation, and component-removal studies. The modular implementation provides a foundation for further experiments, incremental improvements, and future research on tattoo artifact suppression and breast tumor analysis in mammography.

## Running

Use Python 3.11 and a compatible PyTorch / torchvision pair for your platform.

    python -m venv .venv
    source .venv/bin/activate
    python -m pip install -r requirements.txt
    python smoke.py
    python run.py train /path/to/views.csv runs/full
    python run.py evaluate /path/to/views.csv runs/full

Evaluation computes new predictions and metrics using the selected checkpoints. LPIPS uses pretrained AlexNet weights; provision them in the torchvision cache or allow download on first use. Use the same evaluation seed across configurations. For component removal, use a fresh output directory for each training run and pass --omit followed by transformer, frequency, gate, anatomy, cmi or rtss. Evaluation reads the saved configuration. Keep evaluation CSV files and model checkpoints in private storage.

## Input manifest

One row per retained original view. CSV header:

    patient,exam,view,split,image,tumor,artifact,label,paired,fidelity,breast

Use anonymized patient and unique examination identifiers, distinct view IDs per examination, two views per exam, patient-disjoint train/validation/test partitions and one binary examination label (1 malignant). Image and masks are aligned grayscale PNG/TIFF files, paths relative to the CSV. Supply a breast mask for each image. A blank artifact field explicitly denotes a verified artifact-free view. Exclude views with unknown artifact status or obtain the missing annotations before using this loader. Masks use positive foreground pixels. Set paired=1 for the selected affected test view per patient and fidelity=1 for the selected artifact-free recipient test view per patient; otherwise use 0. Set these selections in advance. Only training views with annotated artifacts can donate RTSS residuals. Clinical images, private manifests and labels are supplied by the investigator.

## Models and training

models.py implements masked-context anatomy estimation, residual feature gating, latent cosine separation, two-class Dirichlet evidence, positive and negative prototype refinement, bounded residual editing, CNN / patch-attention / Haar branches, a learned protection map and a lesion-weighted dual-task analyzer. run.py trains ARED-Net, then CARE with the selected ARED checkpoint, then the analyzer on both original and restored views. Each stage selects a validation checkpoint. The analyzer uses class-weighted cross entropy, BCE plus soft Dice, AdamW, warm-up/cosine decay, a joint Dice/AUROC validation criterion and patience 15.

Choices include width 24, global attention over nonoverlapping patch tokens, soft prototype attention, confidence exponent 2, protection factor 0.8, FP32 training, horizontal flip augmentation, unit restoration loss coefficients and three-scale SSIM. CMI uses a soft union (maximum gate) and smoothed confidence map. For the gate ablation the auxiliary protection supervision stays active. 

## Outputs and populations

views.csv stores per-view scores. exams.csv stores the mean score of two views for each exam and condition. paired.csv includes one affected test view per selected patient, original and restored scores, per-image tumor and artifact segmentation metrics and outside-reference-mask absolute change. fidelity.csv contains SSIM, PSNR and LPIPS on the breast bounding crop after one transferred donor residual with scale 0.7 per selected clean recipient. Evaluation prints summary metrics; malignant F1 uses the 0.50 threshold. Undefined segmentation ratios are missing and excluded from averages. Average precision uses sklearn's stepwise definition. The number of selected cases comes from the input manifest. Original-input F1 and AUROC can be computed when scores and both labels are available. These new outputs describe your run.

## Archiving

Upload the extracted source files to the intended GitHub repository, select an author-approved license, tag the reviewed commit and archive the same revision on Zenodo. Cite its actual version DOI and tagged GitHub URL. An older archive refers to the files in its own deposit. Record the Python and dependency versions for the run with pip freeze.

For inference on one de-identified grayscale image:

    python predict.py runs/full mammogram.png prediction

The output directory receives 16-bit PNG renderings on the standardized 256-by-256 grid.
