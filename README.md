# TakeMeter — Langfuse GitHub Issue Classifier

Fine-tuned DistilBERT classifier for triaging [Langfuse OSS GitHub issues](https://github.com/langfuse/langfuse/issues) into `bug_report`, `feature_request`, or `support_question`.

**Data:** `data/labeled_issues.csv` (200 issues)  
**Notebook:** `model_fine_tuning_project.ipynb` (Google Colab, T4 GPU)

---

## Demo video

**Link:** _[https://drive.google.com/file/d/1nwyCt3czO0Rj5E8HbXANqOyEg3VuING3/view?usp=sharing]_

Submitted via the CodePath assignment portal (Project 3 submission form) alongside this GitHub repo URL.

---

## Labels

| Label | Definition |
|---|---|
| `bug_report` | Broken behavior with repro steps, env, or expected vs actual |
| `feature_request` | New capability or improvement; no failure reported |
| `support_question` | How-to or config question; no clear bug or feature proposal |

---

## Baseline prompt (Groq `llama-3.3-70b-versatile`)

```
You classify Langfuse GitHub issues into exactly one label.

Labels:
- bug_report: broken behavior with repro steps, env details, or expected vs actual
- feature_request: asks for new capability or improvement; no failure reported
- support_question: how-to or config question; no clear bug or feature proposal

Rules:
- Self-hosting "X doesn't work" WITH repro/env → bug_report
- "How do I configure…" or "Is this expected?" → support_question
- Desired new behavior with use case → feature_request

Issue:
{text}

Reply with ONLY one label: bug_report, feature_request, or support_question
No explanation. No punctuation. No quotes
```

Temperature: 0. Evaluated on the same locked 30-example test set as the fine-tuned model.

---

## Evaluation summary

| Model | Test accuracy | Macro F1 |
|---|---|---|
| Zero-shot baseline (Groq) | **0.90** | 0.60 |
| Fine-tuned Run 1 (default Trainer) | 0.63 | 0.26 |
| Fine-tuned Run 3 (WeightedTrainer, 5ep, lr 3e-5) | **0.83** | ~0.56 |

**Takeaway:** Baseline wins on accuracy. Default fine-tuning collapsed to majority class; class-weighted training recovered feature detection but did not beat Groq. `support_question` (10/200 training examples) was not learned by either model on the single test case.

**Artifacts:** `evaluation_results_after.json`, `confusion_matrix_after.png`, `confusion_matrix_before.png`

---

## Confusion matrix — fine-tuned best run (test set, n=30)

|  | Pred: bug | Pred: feature | Pred: support |
|---|---|---|---|
| **True: bug** (19) | 16 | 3 | 0 |
| **True: feature** (10) | 1 | 9 | 0 |
| **True: support** (1) | 0 | 1 | 0 |

See also `confusion_matrix_after.png`.

### Run 1 collapse (before class weights)

|  | Pred: bug | Pred: feature | Pred: support |
|---|---|---|---|
| **True: bug** (19) | 19 | 0 | 0 |
| **True: feature** (10) | 10 | 0 | 0 |
| **True: support** (1) | 1 | 0 | 0 |

See `confusion_matrix_before.png`.

---

## Error analysis — 3 misclassified examples

### 1. Bug → feature_request

**Snippet:** Issue describes `run_batched_evaluation` mapper protocol mismatch but also asks *"Is this intended behavior?"* and mentions confusing documentation.

**True:** `bug_report` | **Predicted:** `feature_request`

**Why wrong:** Model latched onto question/doc-confusion language instead of repro steps, env (self-hosted v3.150.0), and bug template.

---

### 2. Bug → feature_request (pattern)

**Snippet:** Self-hosting issue with `### Describe the bug` but title starts with `feat(...)` or discusses expected SDK behavior as a design question.

**True:** `bug_report` | **Predicted:** `feature_request`

**Why wrong:** `feat` prefix and capability language in title outweighed bug template in truncated token window (256–512 tokens).

---

### 3. Support → feature_request

**Snippet:** *"How should Langfuse traces be structured for an agent using MCP tools, OpenRouter, RAG…"*

**True:** `support_question` | **Predicted:** `feature_request`

**Why wrong:** Only 10 support examples in training; model never predicted `support_question` on test. Question reads like a feature design ask.

**Systematic pattern:** Errors cluster on **bug ↔ feature** boundary when issues mix templates, and **support** is invisible due to class imbalance (n=10).

---

## Sample classifications (fine-tuned model, test set)

| Issue (short) | True label | Predicted | Confidence | Correct? |
|---|---|---|---|---|
| `bug: OTEL ingestion replay fails when S3 prefix set` + repro steps | bug_report | bug_report | 0.94 | ✓ |
| `[Feature Request] Instance-wide model cost configuration` | feature_request | feature_request | 0.88 | ✓ |
| `Feat: Allow multi-modal in Playground` + PRD body | feature_request | feature_request | 0.91 | ✓ |
| `bug(sdk-python): LangChain tool observations should use structured inputs` | bug_report | bug_report | 0.87 | ✓ |
| `How should traces be structured for MCP agent` | support_question | feature_request | 0.72 | ✗ |

_Confidence = softmax probability of predicted class from fine-tuned DistilBERT on test inference._

---

## Fine-tuning config (best run)

| Setting | Value |
|---|---|
| Base model | `distilbert-base-uncased` |
| Platform | Google Colab T4 GPU |
| Epochs | 5 (was 3) |
| Learning rate | 3e-5 (was 2e-5) |
| Class weights | `WeightedTrainer` — sklearn `balanced` weights in `CrossEntropyLoss` |
| Best checkpoint | `metric_for_best_model="eval_loss"` |

---

## AI usage

### Instance 1 — Label design (Cursor)

**Directed:** Asked Cursor to stress-test label boundaries with synthetic posts between `bug_report` and `support_question`.

**Revised:** Tightened rule — repro + env → bug; how-to without failure → support. Rejected overlapping "opinion" style labels.

### Instance 2 — Inter-annotator check (Claude)

**Directed:** Had Claude independently label 30 issues using my definitions.

**Revised:** Agreement ~90%+ with my labels; disagreements were on self-hosting edge cases. Did not change training data — confirmed GitHub label + rule approach was consistent.

### Instance 3 — Failure analysis (Cursor)

**Directed:** Exported misclassified test rows; asked for error clusters.

**Revised:** Confirmed pattern (bug/feature template confusion, support invisible). Drove decision to add `WeightedTrainer` and increase epochs.

---

## Spec reflection

**What planning.md got right:** Community choice, label definitions, macro F1 as primary metric, documenting class imbalance risk.

**What surprised me:** Validation accuracy stuck at 0.633 during training even as test accuracy improved to 0.83 — accuracy alone is misleading. Default training collapsed completely; I did not anticipate needing class weights until Run 1 failed.

**What I'd change:** Collect 50+ `support_question` examples before training; use `macro_f1` for checkpoint selection; try `roberta-base` with `max_length=512`.

---

## Repo files

| File | Purpose |
|---|---|
| `planning.md` | Design spec, experiments, edge cases |
| `data/labeled_issues.csv` | 200 labeled issues |
| `evaluation_results_after.json` | Baseline vs fine-tuned metrics |
| `confusion_matrix_before.png` | Run 1 collapse |
| `confusion_matrix_after.png` | Best fine-tuned run |
