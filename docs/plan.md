# MedQA Thesis - Detailed Implementation Plan

## Title
**Systematic Comparison of Multiple Retrieval-Augmented Generative AI Architectures for Evidence-Based Medical Question Answering with Explainability and Hallucination Control**

---

## Project Overview

This project compares four RAG architectures (Naive RAG, Sparse RAG, Hybrid RAG, Multi-Hop Explainable RAG) on the MedQA USMLE dataset (12,723 clinical questions) with 18 English medical textbooks as the retrieval knowledge base. All architectures share the same LLM, embedding model, prompt template, and retrieval infrastructure — only the retrieval strategy varies.

---

## Dataset Summary

| Component | Details |
|-----------|---------|
| Questions (Train) | 10,178 questions (`medqa-data/questions/US/4_options/phrases_no_exclude_train.jsonl`) |
| Questions (Dev) | 1,272 questions (`medqa-data/questions/US/4_options/phrases_no_exclude_dev.jsonl`) |
| Questions (Test) | 1,273 questions (`medqa-data/questions/US/4_options/phrases_no_exclude_test.jsonl`) |
| Total Questions | 12,723 |
| Question Format | JSONL: `question`, `answer`, `options` (A/B/C/D), `answer_idx`, `meta_info`, `metamap_phrases` |
| Textbooks | 18 English medical textbooks (~85MB total) in `medqa-data/textbooks/en/` |
| Largest Textbook | Harrison's Internal Medicine (22MB) |
| Smallest Textbook | Pathoma (400KB) |

### Textbooks Available
1. InternalMed_Harrison.txt (22.4MB)
2. Surgery_Schwartz.txt (11.5MB)
3. Neurology_Adams.txt (8.4MB)
4. Obstentrics_Williams.txt (6.6MB)
5. Gynecology_Novak.txt (5.6MB)
6. Pharmacology_Katzung.txt (5.1MB)
7. Cell_Biology_Alberts.txt (4.9MB)
8. Pathology_Robbins.txt (3.8MB)
9. Immunology_Janeway.txt (3.3MB)
10. Physiology_Levy.txt (3.1MB)
11. Histology_Ross.txt (3.1MB)
12. Pediatrics_Nelson.txt (3.0MB)
13. Psichiatry_DSM-5.txt (2.9MB)
14. Anatomy_Gray.txt (2.3MB)
15. Biochemistry_Lippincott.txt (1.4MB)
16. First_Aid_Step2.txt (1.0MB)
17. First_Aid_Step1.txt (0.7MB)
18. Pathoma_Husain.txt (0.4MB)

---

## Directory Structure

```
MedQA-Thesis/
├── plan.md                              # This file
├── medqa-data/                          # Raw dataset (already exists)
│   ├── questions/US/4_options/          # JSONL question files
│   └── textbooks/en/                   # 18 medical textbooks
├── experiments/                         # Experiment tracking
│   └── ThesisExperimentData.xlsx       # 29 experiments (config pre-filled, metrics empty)
├── data/                                # Processed data (generated)
│   ├── processed/                      # Cleaned DataFrames
│   └── indices/                        # FAISS + BM25 indices
├── notebooks/                           # All implementation notebooks
│   ├── local/                          # Local (MacBook) versions
│   └── colab/                          # Google Colab versions
├── results/                             # Experiment outputs
│   ├── experiments/                    # Component selection experiment results & plots
│   ├── naive_rag/                      # Naive RAG results
│   ├── sparse_rag/                     # Sparse RAG results
│   ├── hybrid_rag/                     # Hybrid RAG results
│   ├── multihop_rag/                   # Multi-Hop RAG results
│   ├── evaluation/                     # RAGAS evaluation results
│   ├── explainability/                 # LIME & SHAP results
│   └── comparative/                    # Final comparative analysis
├── config/                              # Configuration files
│   └── config.py                       # API keys, model names, hyperparams
└── utils/                               # Shared utility functions
    ├── data_utils.py                   # Data loading helpers
    ├── retrieval_utils.py              # Retrieval pipeline helpers
    ├── evaluation_utils.py             # Evaluation metric helpers
    └── visualization_utils.py          # Plotting helpers
```

---

## Notebook Plan (10 Notebooks)

Each notebook has two versions:
- **Local version**: `notebooks/local/XX_name.ipynb` (runs on MacBook M1 Pro)
- **Colab version**: `notebooks/colab/XX_name_colab.ipynb` (runs on Google Colab with Drive mount + pip installs)

---

### Notebook 01: Data Preprocessing & EDA
**File**: `01_data_preprocessing.ipynb`

| Cell | Description |
|------|-------------|
| 1 | Install/import dependencies (pandas, numpy, matplotlib, seaborn, json) |
| 2 | Load MedQA JSONL files (train/dev/test) into pandas DataFrames |
| 3 | Data validation: check for missing fields, duplicates, malformed entries |
| 4 | Standardize schema: ensure `question`, `answer`, `options`, `answer_idx`, `meta_info` columns |
| 5 | **EDA - Questions**: question length distribution, answer distribution (A/B/C/D balance), meta_info breakdown (step1 vs step2&3) |
| 6 | **EDA - Options**: option text length distribution, check for empty/null options |
| 7 | Load all 18 English textbooks, compute per-book statistics (char count, word count, line count) |
| 8 | **Textbook cleaning**: normalize whitespace, fix encoding issues, strip control characters |
| 9 | **Medical terminology normalization**: standardize abbreviations, fix OCR artifacts |
| 10 | Standardize text encoding to UTF-8 |
| 11 | Save processed data: `data/processed/medqa_train.parquet`, `medqa_dev.parquet`, `medqa_test.parquet` |
| 12 | Save cleaned textbook corpus: `data/processed/textbook_corpus.json` (list of {book_name, text}) |
| 13 | Summary statistics and visualizations |

**Output Files**:
- `data/processed/medqa_train.parquet`
- `data/processed/medqa_dev.parquet`
- `data/processed/medqa_test.parquet`
- `data/processed/textbook_corpus.json`

---

### Notebook 02: Component Selection Experiments
**File**: `02_component_experiments.ipynb`

This notebook systematically evaluates different chunking techniques, embedding models, vector databases, and LLM models to select the optimal configuration for the final RAG pipelines. Results are recorded in the Excel spreadsheet: `experiments/ThesisExperimentData.xlsx`.

**Experiment Strategy**: Sequential elimination (one variable at a time, fix the rest):
1. Phase 1 → find best **chunking** (fix embedding, VectorDB, LLM)
2. Phase 2 → find best **embedding** (use best chunking from P1)
3. Phase 3 → find best **VectorDB** (use best chunking + embedding from P1-P2)
4. Phase 4 → find best **LLM** (use best chunking + embedding + VectorDB from P1-P3)
5. Phase 5 → final comparison of all **4 RAG architectures** with best config
6. Phase 6 → cross-validation: test other RAGs with alternative chunking strategies

All experiments run on **50-100 dev set questions** first. Full test set evaluation only in Phase 5.

#### Excel Spreadsheet: `experiments/ThesisExperimentData.xlsx`

**Sheet 1 (Sheet1)**: 29 experiments, headers + config pre-filled, metric columns empty.

**Headers (21 columns)**:
`SN | RAG Name | Chunking Technique | Embedding | VectorDB | Generator Model | Accuracy | Precision | Recall | F1 | Exact_Match | Token_F1 | Recall@3 | Recall@5 | Recall@10 | MRR | RAGAS_Faithfulness | RAGAS_Answer_Correctness | RAGAS_Context_Precision | RAGAS_Context_Recall | RAGAS_Answer_Relevancy`

**Sheet 2 (Experiment Phases)**: Legend explaining each phase.

#### Phase 1: Chunking Strategy Comparison (Experiments 1-7)
Fix: Embedding=all-MiniLM-L6-v2, VectorDB=FAISS, LLM=LLaMA 3.3 70B, RAG=Naive

| SN | Chunking Technique |
|----|--------------------|
| 1 | Fixed-Size (200 tokens) |
| 2 | Fixed-Size (512 tokens) |
| 3 | Overlapping (512 tokens, 50 overlap) |
| 4 | Overlapping (512 tokens, 100 overlap) |
| 5 | Recursive (512 tokens) |
| 6 | Semantic Passage (200 tokens) |
| 7 | Sentence Window (4 sentences, overlap=1) |

**Decision**: Select chunking strategy with best Accuracy + Recall@5 + RAGAS_Faithfulness.

#### Phase 2: Embedding Model Comparison (Experiments 8-10)
Fix: Chunking=best from P1, VectorDB=FAISS, LLM=LLaMA 3.3 70B, RAG=Naive

| SN | Embedding Model | Dimensions | Params |
|----|----------------|-----------|--------|
| 8 | BAAI/bge-large-en-v1.5 | 1024 | 335M |
| 9 | ncbi/MedCPT-Query-Encoder | 768 | 110M |
| 10 | all-MiniLM-L6-v2 | 384 | 22M |

**Decision**: Select embedding with best Recall@5 + MRR + RAGAS_Context_Precision.

#### Phase 3: VectorDB Comparison (Experiments 11-13)
Fix: Chunking=best from P1, Embedding=best from P2, LLM=LLaMA 3.3 70B, RAG=Naive

| SN | VectorDB | Type |
|----|----------|------|
| 11 | FAISS (IndexFlatIP) | In-memory library |
| 12 | ChromaDB | Embedded database |
| 13 | Qdrant (in-memory) | Vector search engine |

**Decision**: Select VectorDB with best query latency + Recall@5 (accuracy should be near-identical for exact search).

#### Phase 4: LLM Model Comparison (Experiments 14-16)
Fix: Chunking=best from P1, Embedding=best from P2, VectorDB=best from P3, RAG=Naive

| SN | Generator Model | Params | API |
|----|----------------|--------|-----|
| 14 | LLaMA 3.3 70B | 70B | Groq |
| 15 | LLaMA 3.1 8B | 8B | Groq |
| 16 | Mixtral 8x7B | 46.7B | Groq |

**Decision**: Select LLM with best Accuracy + RAGAS_Faithfulness + RAGAS_Answer_Correctness.

#### Phase 5: Final RAG Architecture Comparison (Experiments 17-20)
All components = best from Phases 1-4

| SN | RAG Architecture | Retrieval Method |
|----|-----------------|------------------|
| 17 | Naive RAG | FAISS dense semantic search only |
| 18 | Sparse RAG | BM25 keyword search only |
| 19 | Hybrid RAG | FAISS + BM25 with RRF fusion (k=60) |
| 20 | Multi-Hop RAG | Iterative FAISS retrieval (max 3 hops) |

**This is the core comparison of the thesis.**

#### Phase 6: Cross-Validation Experiments (Experiments 21-29)
Test whether chunking preference varies by RAG architecture.

| SN | RAG | Chunking |
|----|-----|----------|
| 21 | Sparse RAG | Semantic Passage (200 tokens) |
| 22 | Sparse RAG | Sentence Window (4 sentences, overlap=1) |
| 23 | Sparse RAG | Overlapping (512 tokens, 50 overlap) |
| 24 | Hybrid RAG | Semantic Passage (200 tokens) |
| 25 | Hybrid RAG | Sentence Window (4 sentences, overlap=1) |
| 26 | Hybrid RAG | Overlapping (512 tokens, 50 overlap) |
| 27 | Multi-Hop RAG | Semantic Passage (200 tokens) |
| 28 | Multi-Hop RAG | Sentence Window (4 sentences, overlap=1) |
| 29 | Multi-Hop RAG | Overlapping (512 tokens, 50 overlap) |

#### Notebook 02 Cell Structure

| Cell | Description |
|------|-------------|
| 1 | Install/import dependencies (langchain, sentence-transformers, faiss-cpu, chromadb, qdrant-client, rank-bm25, groq, ragas, openpyxl) |
| 2 | Load cleaned textbook corpus and sample dev set questions (50-100) |
| 3 | **Define evaluation metrics functions**: Accuracy, Precision, Recall, F1, Exact_Match, Token_F1, Recall@3, Recall@5, Recall@10, MRR, RAGAS metrics |
| 4 | **Define experiment runner**: function(chunking, embedding, vectordb, llm, rag_type) → dict of all metrics |
| 5 | **Implement all 7 chunking strategies** using LangChain text splitters |
| 6 | Chunk statistics per strategy (total chunks, avg/min/max length) |
| 7 | **Load all 3 embedding models** |
| 8 | Embedding speed benchmarks |
| 9 | **Build indices** for FAISS, ChromaDB, Qdrant |
| 10 | **LLM API setup** with placeholder for API key |
| 11 | **Run Phase 1 experiments** (SN 1-7): chunking comparison |
| 12 | Phase 1 results visualization + best chunking selection |
| 13 | **Run Phase 2 experiments** (SN 8-10): embedding comparison |
| 14 | Phase 2 results visualization + best embedding selection |
| 15 | **Run Phase 3 experiments** (SN 11-13): VectorDB comparison |
| 16 | Phase 3 results visualization + best VectorDB selection |
| 17 | **Run Phase 4 experiments** (SN 14-16): LLM comparison |
| 18 | Phase 4 results visualization + best LLM selection |
| 19 | **Run Phase 5 experiments** (SN 17-20): final RAG architecture comparison |
| 20 | Phase 5 results visualization |
| 21 | **Run Phase 6 experiments** (SN 21-29): cross-validation |
| 22 | Phase 6 results visualization |
| 23 | **Write all results to Excel**: update `experiments/ThesisExperimentData.xlsx` with metric values |
| 24 | **Export summary**: `results/experiments/component_selection_results.csv` |
| 25 | **Final recommendation**: print best config for each component |

**Output Files**:
- `experiments/ThesisExperimentData.xlsx` (updated with metric values)
- `results/experiments/component_selection_results.csv`
- `results/experiments/plots/` (visualization PNGs)

**IMPORTANT**: After this notebook, the user reviews the Excel spreadsheet and selects the final configuration for:
- Chunking strategy
- Embedding model
- Vector database
- LLM model

These selections are then hardcoded in `config/config.py` and used consistently across ALL four RAG architectures in Notebooks 03-07.

---

### Notebook 03: Knowledge Base Setup (Final Configuration)
**File**: `03_knowledge_base_setup.ipynb`

Uses the **selected configuration** from Notebook 02 experiments.

| Cell | Description |
|------|-------------|
| 1 | Install/import dependencies |
| 2 | Load configuration (selected chunking, embedding, vectordb from experiments) |
| 3 | Load cleaned textbook corpus |
| 4 | **Apply selected chunking strategy** to all 18 textbooks |
| 5 | Store chunk metadata: `source_book`, `chunk_id`, `char_offset`, `chunk_text` |
| 6 | Report chunk statistics (total chunks, avg length, per-book distribution) |
| 7 | **Generate embeddings** using selected embedding model (batch processing) |
| 8 | **Build FAISS index** (required for Naive RAG, Hybrid RAG, Multi-Hop RAG) |
| 9 | **Build BM25 index** (required for Sparse RAG, Hybrid RAG) |
| 10 | **Build alternative VectorDB index** if Qdrant/ChromaDB was selected |
| 11 | Save all indices to `data/indices/` |
| 12 | Save chunk metadata to `data/indices/chunk_metadata.json` |
| 13 | **Validation**: run 5 sample queries, inspect top-5 retrieved chunks from each index |
| 14 | Verify retrieval quality visually before proceeding |

**Output Files**:
- `data/indices/faiss_index.bin`
- `data/indices/bm25_index.pkl`
- `data/indices/chunk_metadata.json`
- `data/indices/embeddings.npy`

---

### Notebook 04: Naive RAG
**File**: `04_naive_rag.ipynb`

Naive RAG = Query Embedding → FAISS Vector Search → Retrieved Context → Prompt → LLM → Answer

| Cell | Description |
|------|-------------|
| 1 | Install/import dependencies |
| 2 | Load FAISS index, chunk metadata, embedding model, LLM config |
| 3 | Load MedQA dev set (1,272 questions) and test set (1,273 questions) |
| 4 | **LLM setup**: Groq API with selected model, `temperature=0` for reproducibility |
| 5 | **Prompt template** (evidence-grounded, shared across all architectures): |

```
You are a medical expert answering USMLE-style clinical questions.
Use ONLY the provided evidence passages to answer. If the evidence
is insufficient, state that rather than guessing.

Evidence Passages:
{context}

Question: {question}

Options:
A) {option_a}
B) {option_b}
C) {option_c}
D) {option_d}

Instructions:
1. Analyze the evidence passages relevant to the question.
2. Select the single best answer (A, B, C, or D).
3. Provide a brief explanation citing specific evidence passages.

Answer format:
Answer: [letter]
Explanation: [your reasoning with evidence citations]
```

| Cell | Description |
|------|-------------|
| 6 | **Naive RAG pipeline function**: embed_query → faiss_search(top_k=5) → build_prompt → llm_generate → parse_answer |
| 7 | **Run on small sample** (50 questions from dev) for pipeline validation |
| 8 | Inspect sample outputs: verify retrieval quality, answer parsing |
| 9 | **Run on full dev set** (1,272 questions) with progress bar and rate limiting |
| 10 | **Run on full test set** (1,273 questions) |
| 11 | **Exact Match Accuracy** computation |
| 12 | **Token-level F1** computation |
| 13 | **Retrieval metrics**: Recall@3, Recall@5, Recall@10, MRR |
| 14 | **RAGAS evaluation**: Faithfulness, Answer Correctness, Context Precision, Context Recall, Answer Relevancy |
| 15 | **Hallucination flagging**: identify answers with Faithfulness < 0.5, log for error analysis |
| 16 | Save raw results: `results/naive_rag/naive_rag_predictions.json` |
| 17 | Save metrics: `results/naive_rag/naive_rag_metrics.json` |
| 18 | **Visualizations**: accuracy by meta_info (step1 vs step2&3), faithfulness distribution, retrieval recall distribution |
| 19 | **Error analysis**: sample of incorrect answers — was it retrieval failure or generation failure? |

**Output Files**:
- `results/naive_rag/naive_rag_predictions.json` (per-question: question, retrieved_contexts, generated_answer, predicted_option, ground_truth)
- `results/naive_rag/naive_rag_metrics.json` (aggregate metrics)
- `results/naive_rag/naive_rag_flagged_hallucinations.json`

---

### Notebook 05: Sparse RAG
**File**: `05_sparse_rag.ipynb`

Sparse RAG = Keyword Extraction → BM25 Keyword Search → Retrieved Context → Prompt → LLM → Answer

| Cell | Description |
|------|-------------|
| 1 | Install/import dependencies |
| 2 | Load BM25 index, chunk metadata, LLM config |
| 3 | Load MedQA dev set and test set |
| 4 | **Sparse RAG pipeline function**: tokenize_query → bm25_search(top_k=5) → build_prompt (same template) → llm_generate → parse_answer |
| 5 | **Run on small sample** (50 questions) for validation |
| 6 | **Run on full dev set** (1,272 questions) |
| 7 | **Run on full test set** (1,273 questions) |
| 8 | Compute all metrics (same as Notebook 04) |
| 9 | RAGAS evaluation |
| 10 | Hallucination flagging (Faithfulness < 0.5) |
| 11 | Save results to `results/sparse_rag/` |
| 12 | Visualizations and error analysis |

**Output Files**: Same structure as Naive RAG under `results/sparse_rag/`

---

### Notebook 06: Hybrid RAG
**File**: `06_hybrid_rag.ipynb`

Hybrid RAG = (FAISS search ∥ BM25 search) → Reciprocal Rank Fusion (k=60) → Retrieved Context → Prompt → LLM → Answer

| Cell | Description |
|------|-------------|
| 1 | Install/import dependencies |
| 2 | Load FAISS index, BM25 index, chunk metadata, embedding model, LLM config |
| 3 | Load MedQA dev set and test set |
| 4 | **RRF fusion function**: `RRF(d) = Σ 1/(k + rank_r(d))` with k=60 |
| 5 | **Hybrid RAG pipeline function**: (embed_query → faiss_search) + (tokenize → bm25_search) → rrf_merge → deduplicate → top_k=5 → build_prompt → llm_generate → parse_answer |
| 6 | **Run on small sample** (50 questions) for validation |
| 7 | **Run on full dev set** (1,272 questions) |
| 8 | **Run on full test set** (1,273 questions) |
| 9 | Compute all metrics |
| 10 | RAGAS evaluation |
| 11 | Hallucination flagging |
| 12 | Save results to `results/hybrid_rag/` |
| 13 | Visualizations and error analysis |
| 14 | **Ablation**: compare RRF weights, vary k parameter |

**Output Files**: Same structure under `results/hybrid_rag/`

---

### Notebook 07: Multi-Hop Explainable RAG
**File**: `07_multihop_rag.ipynb`

Multi-Hop RAG = Question Decomposition → Hop 1 (Initial FAISS Retrieval) → Intermediate Reasoning → Hop 2 (Refined Retrieval) → Hop 3 (Gap-filling Retrieval) → Accumulated Evidence → Prompt → LLM → Answer

| Cell | Description |
|------|-------------|
| 1 | Install/import dependencies |
| 2 | Load FAISS index, chunk metadata, embedding model, LLM config |
| 3 | Load MedQA dev set and test set |
| 4 | **Question decomposition prompt**: LLM breaks complex medical question into 2-3 focused sub-questions |
| 5 | **Hop 1**: For each sub-question → embed → FAISS top-k retrieval → collect initial evidence |
| 6 | **Intermediate reasoning prompt**: LLM analyzes Hop 1 evidence, identifies knowledge gaps, generates refined queries |
| 7 | **Hop 2**: Refined queries → FAISS retrieval → collect additional evidence |
| 8 | **Hop 3** (if needed): Final gap-filling retrieval |
| 9 | **Termination logic**: stop when all sub-questions have supporting evidence OR max 3 hops reached |
| 10 | **Accumulate evidence** from all hops, deduplicate, rank by relevance |
| 11 | **Final answer generation**: accumulated evidence → prompt → LLM → answer |
| 12 | **Store reasoning chain**: log each hop's queries, retrieved evidence, intermediate reasoning |
| 13 | **Run on small sample** (50 questions) for validation |
| 14 | **Run on full dev set** (1,272 questions) |
| 15 | **Run on full test set** (1,273 questions) |
| 16 | Compute all metrics |
| 17 | RAGAS evaluation |
| 18 | Hallucination flagging |
| 19 | Save results including reasoning chains to `results/multihop_rag/` |
| 20 | Visualizations: accuracy vs number of hops used, evidence accumulation analysis |

**Output Files**: Same structure under `results/multihop_rag/` plus `reasoning_chains.json`

---

### Notebook 08: RAGAS Evaluation (Comprehensive)
**File**: `08_ragas_evaluation.ipynb`

| Cell | Description |
|------|-------------|
| 1 | Install/import dependencies (ragas) |
| 2 | Load all results from Notebooks 04-07 |
| 3 | **RAGAS evaluation** across all 4 architectures with 5 metrics: |

| Metric | What It Measures |
|--------|-----------------|
| Faithfulness | Whether generated claims are supported by retrieved context |
| Answer Correctness | Overall correctness (0.75 × F1 + 0.25 × cosine similarity) |
| Context Precision | Whether relevant chunks are ranked above irrelevant ones |
| Context Recall | Whether retrieved context contains all necessary information |
| Answer Relevancy | Whether the answer addresses the question asked |

| Cell | Description |
|------|-------------|
| 4 | **Non-LLM metrics** (objective cross-checks): Exact Match Accuracy, Retrieval Recall |
| 5 | **Comparison table**: all metrics × all 4 architectures |
| 6 | **Statistical significance testing**: paired t-tests / McNemar's test between architectures |
| 7 | **Hallucination rate comparison**: % of answers with Faithfulness < 0.5 per architecture |
| 8 | Save to `results/evaluation/ragas_comparison.csv` |
| 9 | Visualizations: radar chart, grouped bar charts, box plots per metric |

**Output Files**:
- `results/evaluation/ragas_comparison.csv`
- `results/evaluation/statistical_tests.json`
- `results/evaluation/plots/`

---

### Notebook 09: Explainability Analysis (LIME & SHAP)
**File**: `09_explainability_analysis.ipynb`

| Cell | Description |
|------|-------------|
| 1 | Install/import dependencies (lime, shap) |
| 2 | Load results from all 4 architectures (a representative sample, e.g., 100-200 questions) |
| 3 | **LIME setup**: adapt to passage-level — each retrieved chunk is a feature, perturb by including/excluding chunks |
| 4 | **LIME analysis per architecture**: for each question, generate local explanation showing which passages most influenced the answer |
| 5 | **SHAP setup**: Kernel SHAP at passage level — compute Shapley values for each chunk's contribution |
| 6 | **SHAP analysis per architecture**: compute passage-level SHAP values |
| 7 | **Compare LIME vs SHAP**: do they agree on which passages are most important? |
| 8 | **Cross-architecture comparison**: which architecture best traces answers to specific evidence? |
| 9 | **Visualization**: LIME explanation plots, SHAP summary plots, SHAP force plots |
| 10 | **Evidence attribution quality**: % of answers where top-contributing passage actually contains the answer |
| 11 | Save to `results/explainability/` |

**Output Files**:
- `results/explainability/lime_results.json`
- `results/explainability/shap_results.json`
- `results/explainability/plots/`

---

### Notebook 10: Comparative Analysis & Final Framework
**File**: `10_comparative_analysis.ipynb`

| Cell | Description |
|------|-------------|
| 1 | Load all results from Notebooks 08 and 09 |
| 2 | **Master comparison table**: all 4 architectures × all metrics (accuracy, RAGAS, explainability) |
| 3 | **Hallucination analysis**: hallucination rate per architecture, error categorization |
| 4 | **Retrieval failure vs generation failure**: breakdown of where each architecture fails |
| 5 | **Performance by question type**: step1 vs step2&3, by medical subject |
| 6 | **Statistical significance**: comprehensive paired tests across all metric comparisons |
| 7 | **Deployment framework**: map clinical requirements → recommended architecture |

| Clinical Priority | Recommended Architecture (expected) |
|-------------------|-------------------------------------|
| Maximum accuracy | TBD based on results |
| Minimum hallucination | TBD based on results |
| Best explainability | TBD based on results |
| Speed/efficiency | TBD based on results |
| Balance of all | TBD based on results |

| Cell | Description |
|------|-------------|
| 8 | **Research question answers**: explicitly address all 4 research questions with data |
| 9 | **Visualizations**: comprehensive comparison charts, heatmaps, spider/radar charts |
| 10 | **Export final results** for thesis writing |

**Output Files**:
- `results/comparative/master_comparison.csv`
- `results/comparative/deployment_framework.json`
- `results/comparative/plots/`
- `results/comparative/research_question_answers.json`

---

## Technology Stack

| Category | Tool/Library | Purpose |
|----------|-------------|---------|
| Language | Python 3.13 | Core programming |
| Data | pandas, numpy | Data manipulation |
| Visualization | matplotlib, seaborn | Plots and charts |
| LLM Framework | LangChain, LangChain-Community | Prompt chains, retrieval pipelines |
| Embeddings | sentence-transformers | Dense embedding generation |
| Dense Retrieval | faiss-cpu | Vector similarity search |
| Sparse Retrieval | rank-bm25 | BM25 keyword retrieval |
| Alternative VectorDB | chromadb, qdrant-client | For component experiments |
| LLM Inference | groq (API) | LLaMA 3.3 70B via Groq Cloud |
| RAG Evaluation | ragas | 5 RAGAS metrics |
| Explainability | lime, shap | Passage-level XAI |
| Version Control | git, GitHub | Code versioning |

---

## Evaluation Metrics Reference

### Experiment Spreadsheet Columns
(Used in Notebook 02 for component selection and all subsequent evaluations)

| Column | Description | Formula/Method |
|--------|-------------|----------------|
| SN | Serial number | Auto-increment |
| RAG Name | Architecture name | Naive/Sparse/Hybrid/Multi-Hop |
| Chunking Technique | Chunking strategy used | From experiment config |
| Embedding | Embedding model used | From experiment config |
| VectorDB | Vector database used | From experiment config |
| Generator Model | LLM used for generation | From experiment config |
| Accuracy | % of correct answers (exact match) | correct / total |
| Precision | Precision of answer selection | TP / (TP + FP) |
| Recall | Recall of answer selection | TP / (TP + FN) |
| F1 | Harmonic mean of precision and recall | 2 × (P × R) / (P + R) |
| Exact_Match | Binary exact match rate | Predicted == Ground Truth |
| Token_F1 | Token-level overlap F1 | Token overlap between predicted and reference |
| Recall@3 | Retrieval recall at k=3 | % queries with relevant doc in top-3 |
| Recall@5 | Retrieval recall at k=5 | % queries with relevant doc in top-5 |
| Recall@10 | Retrieval recall at k=10 | % queries with relevant doc in top-10 |
| MRR | Mean Reciprocal Rank | Mean of 1/rank of first relevant result |
| RAGAS_Faithfulness | RAGAS faithfulness score | (supported claims) / (total claims) |

### RAGAS Metrics (Used in Notebooks 04-08)

| Metric | Formula |
|--------|---------|
| Faithfulness | (1/M) Σ 1[claim_i supported by context] |
| Answer Correctness | 0.75 × F1(TP,FP,FN) + 0.25 × cos(a, g) |
| Context Precision | (1/K) Σ Precision@k × Rel_k |
| Context Recall | (1/G) Σ 1[gt_claim_j ∈ context] |
| Answer Relevancy | (1/N) Σ cos(q_i, q) |

---

## Implementation Order & Dependencies

```
[01_data_preprocessing]
        │
        ▼
[02_component_experiments]  ◄── USER DECISION POINT: select final config
        │
        ▼
[03_knowledge_base_setup]   ◄── uses selected config from 02
        │
        ├──────────┬──────────┬──────────┐
        ▼          ▼          ▼          ▼
[04_naive_rag] [05_sparse] [06_hybrid] [07_multihop]
        │          │          │          │
        └──────────┴──────────┴──────────┘
                       │
                       ▼
              [08_ragas_evaluation]
                       │
                       ▼
           [09_explainability_analysis]
                       │
                       ▼
           [10_comparative_analysis]
```

---

## API Keys & Configuration

All API keys are stored in `config/config.py` (gitignored) with placeholders:

```python
# config/config.py
GROQ_API_KEY = "your-groq-api-key-here"

# Selected after Notebook 02 experiments (update after running experiments)
SELECTED_CHUNKING = "recursive"           # placeholder
SELECTED_CHUNK_SIZE = 512                 # placeholder
SELECTED_CHUNK_OVERLAP = 50              # placeholder
SELECTED_EMBEDDING_MODEL = "BAAI/bge-large-en-v1.5"  # placeholder
SELECTED_VECTORDB = "faiss"              # placeholder
SELECTED_LLM_MODEL = "llama-3.3-70b-versatile"  # placeholder

# Shared hyperparameters
TOP_K = 5
LLM_TEMPERATURE = 0
FAITHFULNESS_THRESHOLD = 0.5
MAX_HOPS = 3  # for Multi-Hop RAG
RRF_K = 60    # for Hybrid RAG
```

---

## Hallucination Control Strategy (Applied Across All Architectures)

| Layer | Strategy | Implementation |
|-------|----------|---------------|
| Layer 1 | Evidence-Grounded Prompt Design | Prompt instructs model to answer ONLY from retrieved evidence |
| Layer 2a | Retrieval Optimization | Compare 4 retrieval strategies to find best evidence coverage |
| Layer 2b | Multi-Hop Retrieval Validation | Each hop validated before proceeding (Multi-Hop RAG only) |
| Layer 3 | Answer Rejection Mechanism | Flag answers with RAGAS Faithfulness < 0.5 as unreliable |

---

## Rate Limiting Strategy (Groq API)

Groq free tier has rate limits. Strategy:
1. **Small sample first**: 50 questions for pipeline validation
2. **Dev set**: 1,272 questions with rate limiting (sleep between requests if needed)
3. **Test set**: 1,273 questions for final evaluation
4. **Batch processing**: group requests, handle retries with exponential backoff
5. **Caching**: save all LLM responses to avoid re-running on failure
6. **Fallback**: if Groq is down, switch to Together AI or HuggingFace Inference API

---

## Colab Notebook Differences

Each Colab version includes these additional cells at the top:

```python
# Cell 0 (Colab only): Mount Google Drive
from google.colab import drive
drive.mount('/content/drive')
PROJECT_ROOT = '/content/drive/MyDrive/MedQA-Thesis'

# Cell 1 (Colab only): Install dependencies
!pip install langchain langchain-community sentence-transformers faiss-cpu rank-bm25 groq ragas lime shap pandas numpy matplotlib seaborn
```

Local notebooks use relative paths from the project root.

---

## Current Status

- [x] Research proposal completed
- [x] MedQA dataset loaded into workspace
- [x] Plan created (this document)
- [ ] Notebook 01: Data Preprocessing & EDA
- [ ] Notebook 02: Component Selection Experiments
- [ ] **USER DECISION**: Select chunking, embedding, vectordb, LLM
- [ ] Notebook 03: Knowledge Base Setup
- [ ] Notebook 04: Naive RAG
- [ ] Notebook 05: Sparse RAG
- [ ] Notebook 06: Hybrid RAG
- [ ] Notebook 07: Multi-Hop Explainable RAG
- [ ] Notebook 08: RAGAS Evaluation
- [ ] Notebook 09: Explainability Analysis (LIME & SHAP)
- [ ] Notebook 10: Comparative Analysis & Final Framework
