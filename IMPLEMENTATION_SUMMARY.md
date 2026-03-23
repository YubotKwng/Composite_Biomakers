# Implementation Summary — Composite Biomarkers Reproduction

This document summarizes **what has been implemented so far**, **how it works**, and **why each piece exists**.

---

## 1) Notebook Structure

We split the pipeline into **two notebooks** as requested:

1. `notebooks/data_processing_merge.ipynb`
   - Purpose: **all data wrangling and merging**
   - Output: clean cohort CSVs + updated v1/v2 master

2. `notebooks/paper_method_reproduction.ipynb`
   - Purpose: **modeling and evaluation**
   - Includes baseline CV + demo CV variants + composite sensitivity blocks

---

## 2) Data Processing / Merging (`data_processing_merge.ipynb`)

### 2.1 Core Inputs
- `data/raw/raw_frda_v1_v2_updated.xlsx` (master table)
- `data/raw/mri_data.csv` (diffusion + structural)
- `data/raw/ROI_vbcb_p50_Vwm.csv` (peduncle volumes)
- `data/raw/frda_csa_table.xlsx` (CSA)
- `data/raw/frda_eccentricity_table.xlsx` (ECC)
- `data/raw/IMAGE_FRDA_demograhics.xlsx` (Melbourne demographics)
- `data/raw/Campinas_Demographics_FRDA_.xlsx` (Campinas demographics)

### 2.2 Campinas ses‑3 → visit‑2 filling
- For Campinas, **ses‑1 = visit‑1** and **ses‑3 = visit‑2**.
- We fill missing visit‑2 values using **ses‑3 rows**, as per the feature table instructions.
- Explicit mapping for visit‑2 columns includes:
  - MRI: FA/MD/AD/RD features, brainstem & cerebellar volumes
  - ROI: **SCP_v2 / MCP_v2 / ICP_v2 from (r + l)/2**
  - CSA/ECC: mapping from CSA/ECC tables

### 2.3 ROI peduncle mapping
From `ROI_vbcb_p50_Vwm.csv`, we compute:
- `SCP_v2 = (rSCP + lSCP) / 2`
- `MCP_v2 = (rMCP + lMCP) / 2`
- `ICP_v2 = (rICP + lICP) / 2`
- Converted to mm³ via **×1000**

These are written into:
- `updated_v1_v2.csv`
- `combined_frda_merged.csv`

### 2.4 Cohort outputs + schema alignment
We write **three cohort files**:
- `data/processed/melbourne_frda_merged.csv`
- `data/processed/campinas_frda_merged.csv`
- `data/processed/combined_frda_merged.csv`

Rules:
- **All‑empty columns dropped** (per cohort + combined)
- **Subject column removed** from outputs
- **Cohort ID column** inserted as first column:
  - Melbourne: `melb_id` (e.g., melfrd003)
  - Campinas: `campac_id` (e.g., campac01)

Column order is:
1) **raw v1/v2 columns** (same as `raw_frda_v1_v2_updated.xlsx`)
2) **demographics** (age/sex/duration/onset/GAA)
3) **scores** (SARA1/2, FARA1/2, FARS1/2)
4) **remaining columns**

### 2.5 Updated v1/v2 master output
- `data/processed/updated_v1_v2.csv`
- **Exact same columns and order as raw master** (no demographic columns added)

### 2.6 Missingness reporting
The notebook prints a table of:
- **missing_ses3 / filled_ses3 counts** per v2 column for Campinas

---

## 3) Modeling (`paper_method_reproduction.ipynb`)

### 3.1 Baseline CV
- Uses **ElasticNet**, feature combos matching the paper (QSM skipped)
- Outlier removal: **Tukey fences (3×IQR)**
- Prints outlier removal counts **per combo**
- Outputs `cv_df` with:
  - `r2`, `rmse`, `n_rows`, `lambda_mode`, `alpha_mode`, `lambda_mean`, `alpha_mean`

### 3.2 Demo CV (subject‑level LOO)
Added **new blocks** (no edits to old code) to:
- run subject‑level LOO (`GroupKFold(n_splits = n_subjects)`)
- standardize **X only**
- use a small hyperparameter grid
- output **cv_df_demo2** with the same format as above

Also added **SARA CV** block (`cv_df_demo2_sara`).

### 3.3 Composite & sensitivity
Multiple composite approaches were implemented:

1) **OOF composite (subject‑level LOO)**
   - uses v1/v2 long format
   - predicts on held‑out subject

2) **Tuned composite (inner GroupKFold per fold)**
   - hyperparameters chosen inside each fold
   - avoids shrinkage bias that reduces Cohen’s d
   - reports:
     - `cohens_d`, `p_value`, `mean_change`, `sd_change`

The tuned composite block was added **above** the “prep: ensure merged_long is wide” cell so it runs in the correct order.

---

## 4) Purpose of Key Blocks

- **Explicit mapping block**: ensures Campinas v2 is built from ses‑3 rows consistently with paper rules.
- **Updated v1/v2 output**: gives a clean master table that matches the original schema.
- **Cohort CSVs**: give parallel Melbourne/Campinas datasets for downstream modeling.
- **LOO CV blocks**: ensure no subject leakage and reflect paper design.
- **Tuned composite block**: removes bias from fixed alpha/l1_ratio and improves sensitivity metrics.

---

## 5) Files Created / Updated

### Notebooks
- `notebooks/data_processing_merge.ipynb`
- `notebooks/paper_method_reproduction.ipynb`

### Processed Outputs
- `data/processed/melbourne_frda_merged.csv`
- `data/processed/campinas_frda_merged.csv`
- `data/processed/combined_frda_merged.csv`
- `data/processed/updated_v1_v2.csv`

---

## 6) How to Run

1) Run `data_processing_merge.ipynb`
   - Produces cohort CSVs and updated v1/v2 master
   - Prints missing/fill counts

2) Run `paper_method_reproduction.ipynb`
   - Runs baseline CV and demo LOO CV
   - Generates composite sensitivity metrics

---

If you want, I can add a changelog section or link each new block to a specific supervisor instruction.
