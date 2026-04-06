# Multi-Label Prediction of Peptide Functions Using Protein Language Models

---

## Overview

Antimicrobial peptides (AMPs) are short amino acid sequences with natural antibacterial, antiviral, antifungal, and anticancer activity - often simultaneously. Predicting these functions computationally is a multi-label classification problem, substantially harder than the binary AMP identification task that has dominated the literature.

This project frames peptide function prediction as a multi-label sequence classification task and systematically compares four approaches of increasing complexity: a classical k-mer baseline, and three pre-trained protein language models (ProtBERT, ESM-2, ProtT5). All models are evaluated on two independently curated repositories (DRAMP and dbAMP) under identical experimental conditions, with sequence-identity-based data splits to prevent leakage. Each pLM-based model is trained twice - once with standard Binary Cross-Entropy (BCE) and once with Asymmetric Loss (ASL) - to quantify the contribution of loss function choice to minority label recovery.

Beyond classification performance, the project includes an interpretability and error analysis pipeline covering embedding geometry and cross-dataset false negative profiling.

---

## Models

| Model | Type | Parameters | Embedding |
|---|---|---|---|
| K-mer + Logistic Regression | Classical ML | 5 000 | Bag-of-k-mers (k=3), OvR |
| ProtBERT | Transformer encoder | 420M | [CLS] token, 1024-dim |
| ESM-2 (t6-8M) | Transformer encoder | 8M | [CLS] token, 320-dim |
| ProtT5-XL | Transformer encoder | 3B | Mean pooling, 1024-dim |

All pLM models share the same classification head: a dropout layer (rate 0.3) followed by a linear projection to four output logits with independent sigmoid activations per label.

---

## Datasets

**DRAMP** - 5,527 unique sequences after preprocessing (filtered from 17,481 raw rows). Near-universal antimicrobial prevalence (99.3%), with antiviral (5.2%) and anticancer (2.7%) as the rarest classes. Sourced from the DRAMP 4.0 repository.

**dbAMP** - 25,902 unique sequences after preprocessing (filtered from 51,786 raw rows). Antimicrobial dominates at 90.2%, with anticancer the rarest class at only 67 sequences (0.3%). Sourced from dbAMP 3.0.

Both datasets are mapped onto four unified activity labels: **antimicrobial**, **antiviral**, **antifungal**, and **anticancer**. Coarser sub-categories are merged into a single label following the label-consolidation philosophy of ESCAPE. Both datasets are split using sequence-identity-based clustering via MMseqs2 at a 40% identity threshold to prevent data leakage, with a fixed `random_state=42` for reproducibility.

### Label Distribution

| Label | DRAMP Count | DRAMP % | dbAMP Count | dbAMP % |
|---|---|---|---|---|
| antimicrobial | 5,488 | 99.3 | 23,366 | 90.2 |
| antifungal | 1,786 | 32.3 | 5,460 | 21.1 |
| antiviral | 285 | 5.2 | 2,541 | 9.8 |
| anticancer | 147 | 2.7 | 67 | 0.3 |
| multi-label | 2,062 | 37.3 | 5,252 | 20.3 |
| **Total** | **5,527** | | **25,902** | |

---

## Results

### DRAMP dataset (ASL)

| Model | Accuracy | Macro F1 | Macro AUC | Macro Recall |
|---|---|---|---|---|
| **ProtT5** | **0.711** | **0.638** | 0.779 | **0.703** |
| ESM-2 | 0.671 | 0.632 | 0.788 | 0.681 |
| Baseline | 0.608 | 0.542 | 0.725 | 0.612 |
| ProtBERT | 0.547 | 0.492 | **0.795** | 0.538 |

### dbAMP dataset (ASL)

| Model | Accuracy | Macro F1 | Macro AUC | Macro Recall |
|---|---|---|---|---|
| **ProtBERT** | **0.680** | **0.524** | 0.781 | 0.557 |
| ESM-2 | 0.635 | 0.520 | **0.856** | **0.604** |
| Baseline | 0.629 | 0.455 | 0.740 | 0.481 |
| ProtT5 | 0.476 | 0.446 | 0.688 | 0.523 |

### ASL vs. BCE comparison (Macro F1)

| Model | DRAMP BCE | DRAMP ASL | dbAMP BCE | dbAMP ASL |
|---|---|---|---|---|
| Baseline | 0.542 | 0.542 | 0.455 | 0.455 |
| ProtBERT | **0.603** | 0.492 | 0.440 | **0.524** |
| ESM-2 | 0.461 | **0.632** | **0.532** | 0.520 |
| ProtT5 | 0.639 | **0.638** | 0.238 | **0.446** |

**Key findings:**
- No single model dominates across both datasets. ProtT5 leads on DRAMP, ProtBERT leads on dbAMP, and ESM-2 is the most consistent model on macro AUC across both settings.
- The k-mer baseline remains competitive on DRAMP macro F1 (0.542), outperforming ProtBERT - consistent with prior findings that feature-based methods can match transformers on imbalanced multi-label tasks.
- ASL improves minority label recovery in five of six model–dataset combinations, with the largest gain on ProtT5 on dbAMP (+0.208), where BCE near-collapses to 0.238.
- Class imbalance is the dominant factor: all models achieve near-perfect antimicrobial recovery while antiviral and anticancer remain systematically underrecovered. Macro F1 is recommended over aggregate accuracy as the primary evaluation metric.

---

## Analysis & Interpretability

Beyond classification metrics, the project includes two interpretability analyses run on both datasets.

| Script | Description |
|---|---|
| `08_embedding_visualization.py` | t-SNE and UMAP projections of sequence embeddings for all three pLM models, with per-label silhouette scores. |
| `09_error_analysis.py` | Per-label false negative rate profiling and F1 breakdown across models and both datasets. |

---

## Project Structure

```
Multi-AMP/
├── data/
│   ├── dRAMP/
│   │   ├── Anticancer_amps.fasta
│   │   ├── Antifungal_amps.fasta
│   │   ├── antimicrobial/
│   │   │   ├── Anti-Gram-_amps.fasta
│   │   │   ├── Anti-Gram-positive_amps.fasta
│   │   │   ├── Antibacterial_amps.fasta
│   │   │   ├── Antimicrobial_amps.fasta
│   │   │   ├── Antiparasitic_amps.fasta
│   │   │   └── Insecticidal_amps.fasta
│   │   └── antiviral/
│   │       ├── Anti-SARS-CoV-2_amps.fasta
│   │       └── Antiviral_amps.fasta
│   ├── dbAMP/
│   │   ├── antifungal/
│   │   │   ├── dbAMP_Antifungal_2024.fasta
│   │   │   └── dbAMP_Antiyeast_2024.fasta
│   │   ├── antimicrobial/
│   │   │   ├── dbAMP_AntiGram_n_2024.fasta
│   │   │   ├── dbAMP_AntiGram_p_2024.fasta
│   │   │   ├── dbAMP_AntiMRSA_2024.fasta
│   │   │   ├── dbAMP_Antibacterial_2024.fasta
│   │   │   ├── dbAMP_Antibiofilm_2024.fasta
│   │   │   ├── dbAMP_Antimicrobial_2024.fasta
│   │   │   ├── dbAMP_Antiparasitic_2024.fasta
│   │   │   ├── dbAMP_Antiprotozoal_2024.fasta
│   │   │   └── dbAMP_Insecticidal_2024.fasta
│   │   ├── antiviral/
│   │   │   ├── dbAMP_AntiHIV_2024.fasta
│   │   │   └── dbAMP_Antiviral_2024.fasta
│   │   └── dbAMP_Antitumour_2024.fasta
│   ├── dramp.csv
│   ├── dramp_train/val/test.csv
│   ├── dbamp.csv
│   └── dbamp_train/val/test.csv
├── notebooks/
│   ├── 01_data_preparation.ipynb
│   ├── 02_exploratory_data_analysis.ipynb
│   ├── 03_baseline_model.ipynb
│   └── 07_results_comparison.ipynb
├── scripts/
│   ├── 04_run_protbert.py
│   ├── 05_run_esm2.py
│   ├── 06_run_prott5.py
│   ├── 08_embedding_visualization.py
│   └── 09_error_analysis.py
├── results/
│   ├── all_results.csv
│   ├── baseline_results.png
│   ├── results_protbert.csv
│   ├── results_esm2.csv
│   ├── results_prott5.csv
│   ├── embedding_viz/
│   │   ├── embedding_summary.csv
│   │   ├── embedding_comparison_3models_dramp.png
│   │   └── embedding_comparison_3models_dbamp.png
│   ├── error_analysis/
│   │   ├── metrics_summary.csv
│   │   ├── fn_rate_heatmap_dramp.png
│   │   ├── fn_rate_heatmap_dbamp.png
│   │   ├── per_label_f1_dramp.png
│   │   └── per_label_f1_dbamp.png
│   └── training_viz/
│       ├── protbert_dramp_training.png
│       ├── protbert_dbamp_training.png
│       ├── protbert_multilabel_comparison.png
│       ├── esm2_dramp_training.png
│       ├── esm2_dbamp_training.png
│       ├── esm2_multilabel_comparison.png
│       ├── prott5_dramp_training.png
│       ├── prott5_dbamp_training.png
│       └── prott5_comparison.png
└── README.md
```

---

## Reproducing the Results

### 1. Install dependencies

```bash
pip install torch transformers scikit-learn pandas numpy biopython matplotlib seaborn scipy umap-learn tqdm mmseqs2
```

### 2. Data preparation

Run `01_data_preparation.ipynb` with the FASTA files from the `data/dRAMP/` and `data/dbAMP/` directories. The notebook applies sequence filtering (length 5–100, standard IUPAC characters), deduplication, multi-label merging, and MMseqs2 cluster splitting at 40% sequence identity. Outputs the processed CSV files and the six train/val/test splits into `data/`.

### 3. Exploratory data analysis

Run `02_exploratory_data_analysis.ipynb` to inspect label distributions, sequence length statistics, and multi-label co-occurrence patterns across both datasets.

### 4. Baseline model

Run `03_baseline_model.ipynb` to train the k-mer logistic regression baseline on both datasets. Results are saved to `results/baseline_results.png`.

### 5. Transformer models

Each script trains the corresponding pLM in two independent runs (BCE and ASL) on both datasets. Results are appended to `results/results_all_models.csv`.

```bash
python scripts/04_run_protbert.py
python scripts/05_run_esm2.py
```

> Note: ProtT5 requires ~11 GB GPU memory:

```bash
python scripts/06_run_prott5.py
```

Each script auto-detects device availability and applies early stopping based on macro F1 on the validation set.

### 6. Results comparison

Run `07_results_comparison.ipynb` after all models have been trained to generate the aggregate performance tables and macro F1 / macro AUC bar charts across models and datasets.

### 7. Interpretability and error analysis

Both analysis scripts require the trained `.pth` weight files in `results/`. Each script loads models sequentially and releases GPU memory between models.

```bash
python scripts/08_embedding_visualization.py   # requires: pip install umap-learn
python scripts/09_error_analysis.py
```

---

## Dependencies

- Python 3.10+
- PyTorch 2.0+
- Hugging Face Transformers
- scikit-learn
- BioPython
- MMseqs2
- umap-learn
- pandas, numpy, matplotlib, seaborn, scipy, tqdm

---

## Authors

Oliver Buteski (226023) · Jana Angeloska (226040)
