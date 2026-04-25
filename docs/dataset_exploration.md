# Dataset Exploration

A narrative walkthrough of the exploratory data analysis (EDA) performed on the MedQA USMLE question dataset and the medical textbook corpus, as carried out in `notebooks/local/01_data_preprocessing.ipynb`. This document focuses on **observations, distributions, and implications for the downstream RAG experiments**, complementing the structural reference in [`dataset.md`](dataset.md).

---

## 1. Scope of Exploration

| Asset | Purpose in Thesis |
|-------|-------------------|
| MedQA USMLE questions (12,723) | Evaluation benchmark for the four RAG architectures |
| Medical textbook corpus (18 books, ~12.8M words) | Retrieval knowledge base feeding the RAG pipelines |

The EDA was designed to answer four practical questions before any RAG pipeline is built:

1. Is the dataset **clean and trustworthy** enough to use as a benchmark?
2. Is the answer label distribution **balanced**, or does it bias evaluation?
3. How **long and complex** are the questions — and what does that imply for embedding/chunking?
4. How is the **knowledge base distributed** across medical specialties — and where will retrieval be biased?

---

## 2. Question Dataset — Key Findings

### 2.1 Split Sizing

| Split | Count | Share |
|-------|-------|-------|
| Train | 10,178 | 80.0% |
| Dev | 1,272 | 10.0% |
| Test | 1,273 | 10.0% |
| **Total** | **12,723** | **100%** |

The dataset follows a clean 80/10/10 split. For this thesis, the test set (1,273 questions) is the primary evaluation cohort across all four RAG architectures, while train/dev are reserved for prompt tuning and retrieval-quality validation.

### 2.2 Answer Label Distribution

| Option | Count | Share |
|--------|-------|-------|
| A | 3,267 | 25.7% |
| B | 3,279 | 25.8% |
| C | 3,255 | 25.6% |
| D | 2,922 | 23.0% |

**Observation.** Options A–C are near-uniform; D is mildly under-represented (~3 pp below the others). A naive "always pick B" baseline would score ~25.8%, which sets the absolute floor for any RAG system. This near-balance is important: it means accuracy on this benchmark is not inflated by a positional prior, so improvements attributable to retrieval are meaningful.

### 2.3 USMLE Step Composition

| Step | Count | Share |
|------|-------|-------|
| Step 1 | 7,009 | 55.1% |
| Step 2 & 3 | 5,714 | 44.9% |

**Observation.** The corpus is roughly evenly split between **basic-science** Step 1 questions (anatomy, biochemistry, pathology, pharmacology, immunology) and **clinical** Step 2&3 questions (diagnosis, management, ethics, patient care). This bimodal composition has a direct implication for retrieval: a single embedding model will need to handle both terse mechanism-recall stems and long clinical vignettes equally well — a stress test the four RAG architectures will be compared on.

### 2.4 Question Length

| Statistic | Words |
|-----------|-------|
| Mean | 116.6 |
| Std Dev | 44.9 |
| Min | 9 |
| 25th pct | 84 |
| **Median** | **112** |
| 75th pct | 144 |
| Max | 530 |

**Observation.** Half of all questions are 84–144 words, but the long tail reaches 530 words — these are typically the multi-paragraph "Patient Information" vignettes (full history, vitals, exam, social history). Two implications:

- **Embedding models with short context windows (e.g., 256 tokens) will truncate the longest ~5–10% of questions**, potentially dropping clinically critical context. This is a hyperparameter to watch when comparing architectures.
- **Retrieval queries built from full questions will be heterogeneous in length** — short stems behave like keyword queries, long vignettes behave like document-to-document similarity. Architectures that decompose or summarize the question (e.g., HyDE, query rewriting) may benefit asymmetrically.

### 2.5 Option Text Length

| Stat | A | B | C | D |
|------|---|---|---|---|
| Mean (chars) | 26.8 | 26.8 | 26.9 | 27.0 |
| Median | 22 | 22 | 22 | 22 |
| Max | 230 | 206 | 251 | 235 |

**Observation.** Option lengths are remarkably uniform across A/B/C/D — there is no length-based shortcut a model could exploit. Most options are short noun phrases (~22 chars), but a tail extends to ~250 chars, mostly clinical-management answer choices ("Place the infant in a supine position on a firm mattress…"). For LLM prompting, this means the answer-choice block is consistently small, leaving budget for retrieved context.

### 2.6 Data Quality

| Check | Result |
|-------|--------|
| Missing values | 0 |
| Empty strings | 0 |
| Invalid `answer_idx` (not in A–D) | 0 |
| Answer text vs. option mismatch | 0 |
| Duplicate questions | **2** |

**Observation.** The dataset is exceptionally clean. The only quality issue is a single pair of duplicated long-vignette questions in the training split (different `id`s but identical text). Because both duplicates fall in the train split, the test evaluation is not contaminated, and the duplicates were retained as-is.

### 2.7 MetaMap Phrases

Every question ships with a list of MetaMap-extracted clinical concepts (medications, anatomical structures, presenting symptoms, durations, vitals). Sample for the first training question:

```
["23 year old pregnant woman", "burning", "urination",
 "cranberry extract", "costovertebral angle tenderness",
 "gravid uterus", "best treatment", ...]
```

These pre-extracted phrases are a free signal for **concept-aware retrieval architectures** (e.g., entity-anchored retrieval, hybrid BM25+dense). They are preserved through preprocessing (kept as a list column in the parquet output) and available for downstream experiments without re-running an NER pipeline.

---

## 3. Textbook Corpus — Key Findings

### 3.1 Corpus Size

| Metric | Value |
|--------|-------|
| Books | 18 |
| Total words | 12,851,737 |
| Total characters | 89,132,304 |
| On-disk size | 85.3 MB |

### 3.2 Coverage by Specialty (sorted by size)

| Rank | Book | Words | Share of Corpus |
|------|------|-------|-----------------|
| 1 | InternalMed_Harrison | 3,206,314 | 24.9% |
| 2 | Surgery_Schwartz | 1,601,640 | 12.5% |
| 3 | Neurology_Adams | 1,266,188 | 9.9% |
| 4 | Obstetrics_Williams | 958,801 | 7.5% |
| 5 | Gynecology_Novak | 816,205 | 6.4% |
| 6 | Cell_Biology_Alberts | 754,863 | 5.9% |
| 7 | Pharmacology_Katzung | 730,734 | 5.7% |
| 8 | Immunology_Janeway | 498,650 | 3.9% |
| 9 | Histology_Ross | 462,808 | 3.6% |
| 10 | Pathology_Robbins | 453,577 | 3.5% |
| 11 | Pediatrics_Nelson | 425,380 | 3.3% |
| 12 | Psychiatry_DSM-5 | 418,604 | 3.3% |
| 13 | Physiology_Levy | 408,117 | 3.2% |
| 14 | Anatomy_Gray | 352,200 | 2.7% |
| 15 | Biochemistry_Lippincott | 203,998 | 1.6% |
| 16 | First_Aid_Step2 | 146,337 | 1.1% |
| 17 | First_Aid_Step1 | 89,908 | 0.7% |
| 18 | Pathoma_Husain | 57,413 | 0.4% |

**Observation.** The corpus is **highly skewed toward internal medicine and surgery**, which together account for ~37% of all words. The bottom three books — the high-yield review references (First Aid and Pathoma) — are tiny by comparison (~2.2% combined) despite being the most concept-dense per word. Two implications for retrieval:

- **Naive top-k retrieval will be biased** toward Harrison's and Schwartz simply because they have more chunks competing in the index. Per-book or per-specialty re-ranking may be worth testing.
- **First Aid and Pathoma are disproportionately concept-dense** relative to their size. They likely "punch above their weight" for Step 1 questions and may warrant up-weighting in hybrid retrieval.

### 3.3 Specialty Coverage Gaps

The corpus has solid coverage of the major clinical and basic-science domains, but a few are notably thin or absent:

- **Radiology, dermatology, ophthalmology, ENT** — no dedicated textbook; reliant on coverage within Harrison's.
- **Emergency medicine** — no dedicated textbook.
- **Medical ethics / professionalism** — no coverage despite Step 2 questions touching this.

These gaps are a structural ceiling on RAG performance for any question whose ground-truth knowledge sits outside the 18 books, and should be acknowledged in the thesis when interpreting per-category accuracy.

### 3.4 Cleaning Footprint

| Metric | Value |
|--------|-------|
| Avg character reduction | 0.04% |
| Max reduction | 0.41% (First_Aid_Step1) |
| Min reduction | ~0% (most books) |

**Observation.** The cleaning step (Unicode normalization, whitespace collapsing, control-character removal) was deliberately conservative — average shrinkage is 0.04%, confirming the source `.txt` files were already well-formed. A spot-check of `Pathoma_Husain` showed raw and cleaned outputs visually identical for the first 500 characters. This means downstream tokenization and chunking will operate on essentially the original text — clean enough for embedding without aggressive preprocessing that risks losing medical terminology.

---

## 4. Implications for the RAG Experiments

The EDA produces a small set of design points that should inform every architecture comparison:

1. **Random-baseline floor is ~25.8%** — any RAG architecture below this is performing worse than guessing.
2. **Question length is bimodal** (short stems vs. long vignettes) — track per-length accuracy, not just overall accuracy.
3. **Step 1 vs. Step 2&3 split is ~55/45** — report per-step accuracy to detect architectures that overfit to one mode.
4. **Corpus is internal-medicine-heavy** — per-specialty accuracy is needed to detect coverage-driven failures (vs. retrieval-quality failures).
5. **MetaMap phrases are pre-computed** — hybrid/concept-aware retrieval gets a free input signal worth comparing against pure dense retrieval.
6. **Two duplicate questions exist in the training split only** — test set is uncontaminated.

---

## 5. Reproducing this Exploration

The full EDA pipeline lives in:

```
notebooks/local/01_data_preprocessing.ipynb
```

Outputs (parquet splits, cleaned corpus, EDA charts) are written to `data/processed/`. The five EDA charts referenced in this document are:

| File | Shows |
|------|-------|
| `eda_answer_distribution.png` | Per-split A/B/C/D distribution |
| `eda_question_length.png` | Question length histograms (chars and words) |
| `eda_meta_info.png` | Step 1 vs. Step 2&3 split, overall and per-split |
| `eda_option_lengths.png` | Per-option length box plots |
| `eda_textbook_sizes.png` | Per-book word-count bar chart |

---

*Companion to: [`dataset.md`](dataset.md) (structural reference) · [`research_proposal.md`](research_proposal.md) (thesis context) · `notebooks/local/01_data_preprocessing.ipynb` (source of truth)*
