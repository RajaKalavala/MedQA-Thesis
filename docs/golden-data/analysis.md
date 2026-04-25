# Golden Dataset — Quality-Control Analysis

Analysis of the **100-question construction run completed on 2026-04-25**. All numbers are computed directly from the JSONL artifacts in `data/processed/`.

---

## 1. Pipeline integrity

Zero API failures across all three GPT-4 passes. Every stage produced exactly 100 rows.

| Stage | Rows | Status |
|---|---|---|
| Seed | 100 | ✓ |
| Candidates (10 chunks each) | 100 | ✓ |
| Pass 1 — evidence selection | 100 | 100 / 100 GPT-4 calls succeeded |
| Pass 2 — reference answer | 100 | 100 / 100 |
| Pass 3 — validation | 100 | 100 / 100 |
| Audited | 100 | ✓ |
| **Final accepted** | **64** | within healthy 65–85 range |
| Needs review | 32 | |
| Rejected | 4 | |

---

## 2. Stratification — drifted from the 40 / 40 / 20 spec

| Bucket | Target | Actual |
|---|---|---|
| Step 1 | 40 | **45** |
| Step 2&3 | 40 | **55** |
| Long-vignette (>200 words) | 20 | **23** *(overlaps both step buckets)* |

The drift is by design, not a bug. Long-vignette questions are sampled first; the step buckets top up from the remainder. Long vignettes are themselves predominantly Step 2&3 (clinical case stems), which is why Step 2&3 ends up over-represented.

**Implication for the thesis methodology section:** describe the sampling as *"20 long-vignette questions sampled first; remaining 80 stratified across Step 1 and Step 2&3"*, not as three independent buckets.

---

## 3. Pass 1 — evidence selection

Healthy distribution. GPT-4 found supporting evidence for almost every question, and predominantly classified it as strong.

| Indicator | Value | Read |
|---|---|---|
| `is_evidence_sufficient = True` | 99 / 100 | retrieval is finding relevant chunks for nearly every question |
| Mean selected chunks per question | 1.83 (range 1–5) | reasonable parsimony |
| Support-level distribution | 133 strong · 48 moderate · 2 weak | strong-heavy, as desired |
| Mean evidence keywords per question | 4.70 (range 3–8) | enough for the audit's keyword check to be meaningful |

---

## 4. Pass 2 — reference answer + explanation

| Field | Distribution | Comment |
|---|---|---|
| `question_type` | 48 diagnosis · 20 management · 17 mechanism · 14 treatment · 1 other | sensible USMLE mix |
| `requires_multihop` | **77 yes · 23 no** | **suspiciously high — see §6** |
| `hallucination_check_points` per Q | mean 2.95 (range 2–4) | enough atomic claims for downstream RAGAS hallucination scoring |

---

## 5. Pass 3 — validation scores

| Metric | Value | Read |
|---|---|---|
| `answer_match: True` | 100 / 100 | **structurally tautological — do not cite** |
| Mean `evidence_relevance_score` | 3.73 / 5 | mediocre — retrieval finds plausible chunks, not always the most supportive |
| Mean `faithfulness_score` | 3.86 / 5 | OK |
| Mean `explanation_quality_score` | 4.32 / 5 | high — GPT-4 writes well, expected |
| `hallucination_risk` | 61 low · 32 medium · 7 high | a third of rows have medium+ risk; inspect the 7 `high` |
| `final_status` (LLM verdict) | 65 accepted · 27 needs_review · 8 rejected | |

**Why `answer_match` is meaningless here:** Pass 2 is given the gold answer as input and instructed to write a reference around it. Pass 3 then "checks" whether the reference matches the gold answer — a check that is structurally guaranteed to pass. This field is not a quality signal and should be excluded from any thesis claim about the construction process.

---

## 6. Two findings worth flagging in the thesis

### Finding 1 — `requires_multihop = yes` rate is 77 %

USMLE questions are typically single-hop (one mechanism, one drug, one diagnosis). GPT-4 labelling 77 % as multi-hop is suspiciously high and risks **inflating the apparent value of the Multi-Hop RAG architecture** at evaluation time.

**Mitigation:** manually inspect 10 of the questions labelled `requires_multihop = yes`. If most do not actually require chained retrieval, either:
- relabel via a stricter prompt (define "multi-hop" as "the answer requires combining facts from two distinct clinical entities or two separate textbook passages"), or
- drop the `requires_multihop` claim from the thesis stratification.

### Finding 2 — keyword hallucination caught by the audit

The automated audit flagged **10 rows** with `no_evidence_keyword_in_gold_context` — meaning Pass 1 declared `evidence_keywords` that do not actually appear in the `best_gold_context` text. GPT-4's self-validation (Pass 3) did not catch this; the mechanical audit did.

**This is the strongest defensibility signal in the run** and worth highlighting in the thesis methodology section: the multi-stage construction with an independent non-LLM audit catches failure modes that a single-LLM pipeline would rubber-stamp.

---

## 7. Retrieval coverage

Top books across all 1,000 candidate chunks (10 per question × 100 questions):

| Rank | Book | Candidates | Share |
|---|---|---|---|
| 1 | InternalMed_Harrison | 339 | 33.9 % |
| 2 | Pharmacology_Katzung | 103 | 10.3 % |
| 3 | Gynecology_Novak | 78 | 7.8 % |
| 4 | Neurology_Adams | 77 | 7.7 % |
| 5 | Pediatrics_Nelson | 68 | 6.8 % |

Harrison's at 34 % over-represents its share of the corpus by word count (24.9 %, per [docs/dataset_exploration.md](../dataset_exploration.md)). This is corpus-frequency bias — Harrison's has more chunks in the index, so more compete for the top-k slots. It is shared by every architecture in the comparison and therefore cancels out for *relative* architecture analysis, but the absolute retrieval picture is internal-medicine-heavy.

---

## 8. Final accepted dataset — composition

`data/processed/medqa_ragas_golden.jsonl` (64 rows):

| Slice | Distribution |
|---|---|
| `meta_info` | 33 step2&3 · 31 step1 |
| `question_type` | 28 diagnosis · 13 management · 11 mechanism · 12 treatment |
| `requires_multihop` | 46 yes · 18 no *(see Finding 1)* |

This is the dataset Notebook 05 (pending) will consume.

---

## 9. What still needs human eyes

Three spot-check tasks before declaring the dataset thesis-ready:

| # | Task | Rows to inspect | What to look for |
|---|---|---|---|
| 1 | Read the 10 audit-flagged rows | `medqa_ragas_golden_needs_review.jsonl` filtered to `audit_issues` containing `"no_evidence_keyword_in_gold_context"` | Salvage (audit issue is harmless) vs drop (Pass 1 hallucinated keywords). |
| 2 | Sanity-check accepted rows | 5 random rows from `medqa_ragas_golden.jsonl` | Does `reference_explanation` actually ground in the cited `gold_context`? |
| 3 | Audit the multi-hop label | 10 rows from `medqa_ragas_golden.jsonl` with `requires_multihop = "yes"` | Does the question genuinely require chained retrieval? |

Outcomes documented in [roadmap.md](roadmap.md).

---

## 10. Summary verdict

**The run is healthy and the dataset is suitable for the next stage** (Notebook 05 RAGAS evaluation). Two caveats — the inflated multi-hop rate and the structural tautology of `answer_match` — affect *how the dataset is described in the thesis*, not whether it is usable for the architecture comparison.
