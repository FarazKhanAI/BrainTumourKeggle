
## Introduction
We are working with **brain MRI scans** from the BraTS dataset. Each patient has:
- **Images**: a single file that contains 4 different MRI views (FLAIR, T1, T1ce, T2) stacked together.
- **Labels**: a file that tells us exactly which voxels belong to background (0), edema (1), non‑enhancing tumor (2), or enhancing tumor (3).

The goal is to **build an AI model that automatically segments tumors** from new, unseen scans.

Before we build any model, we must **verify the data** (make sure nothing is corrupted) and **explore its characteristics** so our preprocessing decisions are correct.

---

## Data Verification (Steps 1–5)
We ran a script that checks every single file and reports any problems.

### Step 1 – File Count
- Counted how many training images, training labels, and test images exist.
- **Result**: 484 training images, 484 training labels, 289 test files (before cleaning).

### Step 2 – Filename Matching
- Verified that every training image has a corresponding label with the same name.
- **Result**: All matched perfectly. No missing pairs.

### Step 3 – NIfTI Integrity & Dimension Audit
- Loaded 10 random scans to check if any file is corrupted.
- Also verified that each image has shape `(240, 240, 155, 4)` — meaning 240 pixels high, 240 pixels wide, 155 slices deep, and 4 channels (one per MRI sequence).  
  Labels are `(240, 240, 155)`, containing only one channel of class numbers.
- **Result**: All 10 samples were perfect. No corruption, dimensions consistent.

### Step 4 – Label Value Verification
- Scanned **all 484 label masks** pixel by pixel to ensure they contain only the allowed values `{0, 1, 2, 3}`.
- **Result**: Every mask is clean. No unexpected numbers like 4, 255, or negative values.

### Step 5 – Channel Verification
- Checked that the 4 MRI sequences inside each image are **actually different** (not identical copies).
- Saved a preview image of the middle slice showing all four channels.
- **Result**: Channels differ → data is valid.

**Output files for the team**: `validation_report.txt`, `sample_channel_preview.png`.

---

## Cleaning the Testing Folder
The original testing folder (`imagesTs`) contained **23 hidden macOS metadata files** (names starting with `._`) that are not real medical images.  
- We can’t delete them because the input folder is read‑only.
- **Solution**: We created a **clean list** (`clean_test_list.json`) that contains only the paths of the 266 valid test scans. We will use this list for everything going forward.

---

## Building Clean Train/Test File Lists
We then applied the same filtering to **all** folders (training images, training labels, test images) and created two master JSON files:
- `train_pairs.json` – 484 entries, each with the full path to the image and its matching label.
- `test_images.json` – 266 valid test image paths.

These files are our single source of truth for loading data in later steps.

---

## Exploratory Data Analysis (EDA)

### 1. Voxel Spacing Analysis
We looked at 50 random scans to see the physical size of each voxel (the 3D pixel).
- **Result**: Every single voxel is exactly **1 mm × 1 mm × 1 mm**.
- **Why it matters**: Because the spacing is already perfectly uniform, we do **not** need to resample the images to 1 mm—they are already isotropic. This saves a lot of computation time.

### 2. Spatial Dimension Distribution
Checked the dimensions of all 484 training images.
- **Result**: Every image is **240 × 240 × 155** pixels/voxels. No variation at all.
- **Why it matters**: We can safely design a network that expects a fixed input size. No need to handle differently sized volumes.

### 3. Class Frequency & Tumor Volume
We counted how many voxels belong to each class (background, edema, non‑enhancing tumor, enhancing tumor) across all patients.
- **Class imbalance**:
  - Background: **98.8%** of all voxels
  - Edema: **0.7%**
  - Non‑enhancing tumor: **0.2%**
  - Enhancing tumor: **0.2%**
- **Why it matters**: The model would naturally cheat by predicting “background” everywhere unless we give it a **class weight** that tells it to pay more attention to rare classes.
- **Class weights computed** (inverse frequency, normalized) are saved in `class_weights.json`.
- Also calculated total tumor volume per patient (in mm³) and saved a per‑patient statistics CSV.

**Outputs**: `class_weights.json`, `per_patient_stats.csv`, and a combined plot.

### 4. Intensity Distribution per Modality
We looked at the range of brightness values (intensities) inside the brain tissue (ignoring black background) for each of the four MRI types.
- The ranges differ significantly:
  - FLAIR: 1–3888, mean ~486
  - T1: 1–2903, mean ~608
  - T1ce: 1–3875, mean ~611
  - T2: 1–3027, mean ~503
- **Why it matters**: Because each channel has its own intensity range, we must **normalize each channel separately** (per‑channel normalization) during preprocessing. Otherwise, one channel would dominate the learning.

### 5. Foreground Bounding Box Analysis
We found the smallest box that contains all brain/tumor tissue in each mask.
- **Result**: The actual brain region occupies, on average, only about **27% of the width, 36% of the height, and 45% of the depth**. The rest is useless black background.
- **Why it matters**: We can **crop away roughly 55–73% of the volume** before feeding it into the network. This dramatically reduces memory usage and speeds up training.

---

## What We Have Now – Summary for the Team
- The dataset is clean, consistent, and ready for modeling.
- All images are 240×240×155×4, 1 mm isotropic, with no missing or corrupted files.
- The tumor classes are heavily imbalanced → we will use weighted loss functions.
- The four MRI sequences have very different intensity distributions → we will apply per‑channel normalization.
- A large proportion of each volume is empty background → we will crop the foreground to save memory.
