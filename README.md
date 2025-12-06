# CMPE-295-Project
## Aneurysm Segmentation with nnUNet (RSNA CTA Subset)

This repo contains a working pipeline for preparing the **RSNA Intracranial Aneurysm Detection** dataset for **nnUNet v2** training using CTA series only.

---

## 📌 Current Status

### 1. Reproducible Dataset Builder (Configurable)

Parameterized pipeline in `295-nnunet-12-5-preprocessing-done.ipynb` builds an nnUNet-ready dataset with configurable size and class balance:

```python
TARGET_TOTAL = 40      # total number of cases
POS_FRACTION = 0.75    # fraction of positives (based on localizers)
```

Pipeline steps:
- Load CTA series info from `train.csv`.
- Use `train_localizers.csv` aneurysm annotations.
- Sample positives and negatives according to `TARGET_TOTAL` and `POS_FRACTION`.
- Convert DICOM → NIfTI for selected series.
- Build `imagesTr/`, `labelsTr/`.
- Generate a valid `dataset.json` for nnUNet v2.

### 2. Label Construction (No Use of /segmentations)

We do **not** use the RSNA `/segmentations` folder. Labels come from `train_localizers.csv`:
- For each annotated aneurysm: use `SeriesInstanceUID` + `SOPInstanceUID` to find the slice index.
- Use coordinates `(x, y)` as pixel position.
- Draw a small 3D sphere around the center → label 1; all other voxels → label 0.
- Positives → masks with 0/1 (aneurysm spheres); negatives → all-zero masks.

Labels are saved as:
```
labelsTr/<SeriesInstanceUID>.nii.gz
```

### 3. Disk-Friendly Subset for Kaggle

Because of the 19 GB Kaggle limit:
- Current working subset: **40 CTA cases**.
- Preprocessed size (`3d_fullres`): ~9.2 GB.
- Raw + preprocessed fit under the quota.

Dataset layout:
```
nnUNet_raw/Dataset201_AneurysmCTA/
  imagesTr/
  labelsTr/
  dataset.json
```

### 4. Successful nnUNet Preprocessing

With the 40-case subset, preprocessing completes:
```bash
export nnUNet_raw=/kaggle/working/nnUNet_raw
export nnUNet_preprocessed=/kaggle/working/nnUNet_preprocessed
export nnUNet_results=/kaggle/working/nnUNet_results

nnUNetv2_plan_and_preprocess -d 201 --verify_dataset_integrity -c 3d_fullres -np 1
```
Note: `-np 1` avoids known blosc2 slice errors in multiprocess mode.

### 5. Ready for Training

Train (fold 0) with:
```bash
nnUNetv2_train 201 3d_fullres 0 --npz
```
Outputs:
```
nnUNet_results/Dataset201_AneurysmCTA.../nnUNetTrainer__3d_fullres/fold_0/
```

---

## ▶ How to Rebuild a Dataset

1) Open `295-nnunet-12-5-preprocessing-done.ipynb` and set:
```python
TARGET_TOTAL = <desired number, e.g. 30, 40, 60>
POS_FRACTION = <0–1 ratio, e.g. 0.7>
```
2) Run:
   - Config cell
   - Helper functions cell
   - Main pipeline cell (dataset build)
3) Then run:
```bash
nnUNetv2_plan_and_preprocess -d 201 -c 3d_fullres -np 1
nnUNetv2_train 201 3d_fullres 0 --npz
```
