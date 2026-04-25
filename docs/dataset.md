# Dataset Overview

This document describes the two primary datasets used in the thesis: the **MedQA USMLE question dataset** and the **medical textbook corpus**. All data was loaded, validated, cleaned, and explored in the `notebooks/local/01_data_preprocessing.ipynb` notebook.

---

## 1. MedQA USMLE Question Dataset

### Source

The MedQA dataset originates from the [MedQA benchmark](https://github.com/jind11/MedQA) and contains real-world clinical vignette-style questions modelled after the United States Medical Licensing Examination (USMLE). This project uses the **4-option English (US)** variant.

### Raw Files

Located at `medqa-data/questions/US/4_options/`:

| File | Description |
|------|-------------|
| `phrases_no_exclude_train.jsonl` | Training split |
| `phrases_no_exclude_dev.jsonl` | Development/validation split |
| `phrases_no_exclude_test.jsonl` | Test split |

### Split Sizes

| Split | Questions |
|-------|-----------|
| Train | 10,178 |
| Dev | 1,272 |
| Test | 1,273 |
| **Total** | **12,723** |

### Record Schema

Each JSONL record contains the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `question` | string | Full clinical vignette and question stem |
| `answer` | string | Text of the correct answer |
| `options` | dict | Four answer choices keyed as `A`, `B`, `C`, `D` |
| `answer_idx` | string | Letter of the correct option (`A`/`B`/`C`/`D`) |
| `meta_info` | string | USMLE step classification (`step1` or `step2&3`) |
| `metamap_phrases` | list | MetaMap-extracted medical concepts from the question |

### Standardized DataFrame Schema

After preprocessing, the data is stored as Parquet files with 11 columns:

| Column | Type | Description |
|--------|------|-------------|
| `id` | string | Unique identifier (e.g., `train_0`, `dev_15`, `test_42`) |
| `split` | string | Dataset split (`train`, `dev`, `test`) |
| `question` | string | Full question text |
| `option_a` | string | Answer choice A |
| `option_b` | string | Answer choice B |
| `option_c` | string | Answer choice C |
| `option_d` | string | Answer choice D |
| `answer` | string | Correct answer text |
| `answer_idx` | string | Correct answer letter |
| `meta_info` | string | USMLE step type |
| `metamap_phrases` | list | Medical concept phrases |

### Answer Distribution

The correct answers are roughly balanced across the four options:

| Option | Count | Percentage |
|--------|-------|------------|
| A | 3,267 | 25.7% |
| B | 3,279 | 25.8% |
| C | 3,255 | 25.6% |
| D | 2,922 | 23.0% |

### USMLE Step Breakdown

| Step | Count | Percentage |
|------|-------|------------|
| Step 1 | 7,009 | 55.1% |
| Step 2 & 3 | 5,714 | 44.9% |

### Question Length Statistics (Word Count)

| Statistic | Value |
|-----------|-------|
| Mean | 116.6 words |
| Std Dev | 44.9 words |
| Min | 9 words |
| 25th Percentile | 84 words |
| Median (50th) | 112 words |
| 75th Percentile | 144 words |
| Max | 530 words |

### Option Length Statistics (Character Count)

| Statistic | Option A | Option B | Option C | Option D |
|-----------|----------|----------|----------|----------|
| Mean | 26.8 | 26.8 | 26.9 | 27.0 |
| Std Dev | 19.6 | 19.4 | 19.7 | 19.4 |
| Min | 1 | 1 | 1 | 1 |
| Median | 22 | 22 | 22 | 22 |
| Max | 230 | 206 | 251 | 235 |

### Data Quality

| Check | Result |
|-------|--------|
| Missing values | None |
| Empty strings | None |
| Duplicate questions | 2 |
| Invalid answer indices | 0 |
| Answer-option mismatches | 0 |

---

## 2. Medical Textbook Corpus

### Source

The corpus consists of **18 English medical textbooks** provided with the MedQA dataset, covering a broad range of medical disciplines. These textbooks serve as the knowledge base for retrieval-augmented generation (RAG) experiments.

### Raw Files

Located at `medqa-data/textbooks/en/` as plain `.txt` files.

### Textbook Details

| # | Book | Words | Characters | File Size (MB) |
|---|------|-------|------------|----------------|
| 1 | InternalMed_Harrison | 3,206,314 | 22,312,859 | 21.34 |
| 2 | Surgery_Schwartz | 1,601,640 | 11,441,715 | 10.95 |
| 3 | Neurology_Adams | 1,266,188 | 8,365,987 | 8.00 |
| 4 | Obstentrics_Williams | 958,801 | 6,583,361 | 6.28 |
| 5 | Gynecology_Novak | 816,205 | 5,619,750 | 5.39 |
| 6 | Pharmacology_Katzung | 730,734 | 5,122,267 | 4.90 |
| 7 | Cell_Biology_Alberts | 754,863 | 4,868,257 | 4.67 |
| 8 | Pathology_Robbins | 453,577 | 3,784,898 | 3.63 |
| 9 | Immunology_Janeway | 498,650 | 3,315,092 | 3.18 |
| 10 | Physiology_Levy | 408,117 | 3,049,236 | 2.93 |
| 11 | Histology_Ross | 462,808 | 3,047,020 | 2.91 |
| 12 | Pediatrics_Nelson | 425,380 | 3,002,297 | 2.87 |
| 13 | Psichiatry_DSM-5 | 418,604 | 2,893,358 | 2.77 |
| 14 | Anatomy_Gray | 352,200 | 2,281,157 | 2.18 |
| 15 | Biochemistry_Lippincott | 203,998 | 1,349,616 | 1.29 |
| 16 | First_Aid_Step2 | 146,337 | 1,030,582 | 0.99 |
| 17 | First_Aid_Step1 | 89,908 | 665,018 | 0.64 |
| 18 | Pathoma_Husain | 57,413 | 399,834 | 0.38 |

### Corpus Totals

| Metric | Value |
|--------|-------|
| Number of books | 18 |
| Total words | 12,851,737 |
| Total characters | 89,132,304 |
| Total size | 85.3 MB |

### Text Cleaning

The raw textbook text was cleaned through the following steps:

1. **Control character removal** -- Stripped non-printable characters (keeping newlines and tabs)
2. **Unicode normalization** -- Standardized smart quotes, dashes, and special characters to ASCII equivalents
3. **Whitespace normalization** -- Collapsed multiple consecutive newlines and spaces
4. **Line-level trimming** -- Removed leading/trailing whitespace per line

The cleaning was minimal by design, with an average character reduction of only **0.04%** across all books. The largest reduction was for Surgery_Schwartz at 0.11%, and First_Aid_Step1 at 0.41%.

---

## 3. Processed Output Files

All processed data is saved to `data/processed/`:

| File | Description | Size |
|------|-------------|------|
| `medqa_train.parquet` | Training questions (10,178 rows) | 7.43 MB |
| `medqa_dev.parquet` | Dev questions (1,272 rows) | 0.92 MB |
| `medqa_test.parquet` | Test questions (1,273 rows) | 0.95 MB |
| `textbook_corpus.json` | Cleaned textbook corpus (18 books) | 85.51 MB |

### EDA Visualizations

| File | Description |
|------|-------------|
| `eda_answer_distribution.png` | Bar charts of answer distribution per split |
| `eda_question_length.png` | Histogram of question length (characters and words) |
| `eda_meta_info.png` | USMLE step type distribution |
| `eda_option_lengths.png` | Box plot of option text lengths |
| `eda_textbook_sizes.png` | Horizontal bar chart of textbook word counts |

---

## 4. Sample Question

```json
{
  "question": "A 23-year-old pregnant woman at 22 weeks gestation presents with burning upon urination. She states it started 1 day ago and has been worsening despite drinking more water and taking cranberry extract. She otherwise feels well and is followed by a doctor for her pregnancy. Her temperature is 97.7°F (36.5°C), blood pressure is 122/77 mmHg, pulse is 80/min, respirations are 19/min, and oxygen saturation is 98% on room air. Physical exam is notable ...",
  "answer": "Nitrofurantoin",
  "options": {
    "A": "Ampicillin",
    "B": "Ceftriaxone",
    "C": "Doxycycline",
    "D": "Nitrofurantoin"
  },
  "answer_idx": "D",
  "meta_info": "step2&3"
}
```
