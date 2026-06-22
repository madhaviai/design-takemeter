# TakeMeter — planning.md

## Community

I chose the **Langfuse OSS GitHub community** ([langfuse/langfuse issues](https://github.com/langfuse/langfuse/issues)) because I already use Langfuse for LLM observability and know the issue types firsthand. It's a good fit for classification: text-heavy posts, varied quality (detailed bug reports vs thin complaints vs feature asks), and 2,600+ public open/closed issues.

**Data collection:** I fetched issues as JSON via the public GitHub Search API (`repo:langfuse/langfuse is:issue`), combined title + body into a `text` field, assigned labels using GitHub labels plus title/body rules, and saved the result as `data/labeled_issues.csv` (200 rows).

| Label | Count |
|---|---|
| `bug_report` | 127 |
| `feature_request` | 63 |
| `support_question` | 10 |

No label exceeds 70%. If a label were underrepresented, I'd pull more pages from the API filtered by `label:documentation` or question keywords.

---

## Labels

| Label | Definition | Examples |
|---|---|---|
| `bug_report` | Reports broken behavior with repro steps, env, or expected vs actual. | #14427 (OTEL replay + S3 prefix); #14420 (heredoc bug in dev-tables.sh) |
| `feature_request` | Requests new capability; describes desired behavior, not a failure. | #14265 (global model cost config); #14238 (Playground import/export) |
| `support_question` | How-to or config question without clear bug evidence. | "How should traces be structured for MCP agents?"; docs typo / setup questions |

---

## Hard edge cases

**Ambiguous:** Self-hosting posts saying "X doesn't work" — misconfiguration or real bug?

**Rule:** Repro + env + expected/actual → `bug_report`. "How do I configure…" / "Is this expected?" → `support_question`.

---

## Baseline (Milestone 4)

Zero-shot Groq (`llama-3.3-70b-versatile`) on locked test set (30 examples):

| Metric | Score |
|---|---|
| Accuracy | **0.90** |
| Macro F1 | 0.60 |

| Label | F1 | Support |
|---|---|---|
| `bug_report` | 0.95 | 19 |
| `feature_request` | 0.86 | 10 |
| `support_question` | 0.00 | 1 |

**Note:** Baseline metrics also recorded in `evaluation_results_after.json` (Section 6 comparison).

---

## Fine-tuning experiments (Milestone 5)

**Goal:** Beat baseline accuracy (0.90) via DistilBERT + class-imbalance fixes.  
**Dataset:** `data/labeled_issues.csv` — no CSV edits; truncation handled in notebook (`max_length` in Section 2).

### Run 1 — Default `Trainer` (before)

| Setting | Value |
|---|---|
| Epochs | 3 |
| Learning rate | 2e-5 |
| Class weights | None |

| Metric | Test |
|---|---|
| Accuracy | **0.63** |
| Macro F1 | 0.26 |

**What happened:** Majority-class collapse — model predicted `bug_report` for every example. Val accuracy stuck at 0.633 all 3 epochs (19/30 bugs in val ≈ always-guess-bug).

**Artifacts:** `evaluation_results_before.json`, `confusion_matrix_before.png`

---

### Run 2 — `WeightedTrainer` only

| Setting | Value |
|---|---|
| Change | `CrossEntropyLoss` with sklearn balanced class weights |
| Epochs | 3 |

| Metric | Test |
|---|---|
| Accuracy | **~0.70** |

**What happened:** Partial recovery — model started predicting `feature_request`, but still weak on minority classes.

---

### Run 3 — Weighted + hyperparameters (best run)

| Setting | Value |
|---|---|
| `WeightedTrainer` | Yes |
| Epochs | **5** (was 3) |
| Learning rate | **3e-5** (was 2e-5) |
| `metric_for_best_model` | `eval_loss` |

| Metric | Test |
|---|---|
| Accuracy | **0.83** (25/30) |
| Macro F1 | ~0.56 |

| Label | Correct | Total |
|---|---|---|
| `bug_report` | 16 | 19 |
| `feature_request` | 9 | 10 |
| `support_question` | 0 | 1 |

**What happened:** Features largely fixed (9/10). Bugs good (16/19). Support still missed — only 10 training examples; model never predicts `support_question`. Close to baseline on features, below baseline on overall accuracy.

**Artifacts:** `confusion_matrix_after.png`, `evaluation_results_after.json`

---

### Run 4 — Oversampling (no gain)

| Setting | Value |
|---|---|
| Change | `balance_train()` — equal class counts in train split via `sklearn.resample` |
| Also tried | Combined with weighted trainer |

| Metric | Test |
|---|---|
| Accuracy | **~0.83** (no meaningful change) |

**Conclusion:** Oversampling alone did not beat Run 3. Diminishing returns without more data or a different model.

---

## Before vs after summary

| Run | Approach | Test accuracy | vs baseline (0.90) |
|---|---|---|---|
| Baseline | Groq zero-shot | **0.90** | — |
| Run 1 | Default Trainer | 0.63 | −27pp |
| Run 2 | WeightedTrainer | ~0.70 | −20pp |
| Run 3 | Weighted + 5ep + 3e-5 | **0.83** | −7pp |
| Run 4 | + oversampling | ~0.83 | −7pp |

---

## What we learned

1. **Accuracy misleads** — val stuck at 0.633 during training even as test improved; macro F1 is the honest metric.
2. **Class imbalance caused collapse** — 127/200 `bug_report` → default loss favors always guessing bug.
3. **Class weights fixed the collapse** — biggest single improvement (0.63 → ~0.70 → 0.83).
4. **Hyperparameters mattered** — 5 epochs + 3e-5 pushed test to 0.83; oversampling did not add much on top.
5. **Baseline still wins on accuracy** — Groq reads full issue text + GitHub templates (`### Describe the bug`, `[Feature Request]`). DistilBERT on 200 truncated tokens is near but not past 0.90.
6. **`support_question` may be unlearnable at n=10** — neither baseline nor fine-tuned model got the 1 test example.

**Next levers to beat 0.90:** more data (especially support), `roberta-base` / `deberta-v3-base`, `max_length=512`, macro-F1 early stopping — not more oversampling alone.

---

## Fine-tuning plan (final config)

- **Model:** `distilbert-base-uncased`
- **Best config:** `WeightedTrainer` + 5 epochs + lr 3e-5 + `eval_loss` checkpoint
- **Input:** `text` → **Target:** `label` via `LABEL_MAP`
- **Split:** 70 / 15 / 15 (train ~140, val ~30, test ~30)

---

## Evaluation metrics

- **Macro F1** — primary; treats all labels equally despite imbalance
- **Per-class precision & recall** — accuracy alone hides weak classes
- **Confusion matrix** — focus on `bug_report` ↔ `support_question` errors

**Priority:** High recall on `bug_report` — missing real bugs is worse than misrouting a question.

---

## Definition of success

| Tier | Threshold |
|---|---|
| **Useful** | Macro F1 ≥ 0.75, `bug_report` recall ≥ 0.80 |
| **Good enough** | Macro F1 ≥ 0.65, no class recall < 0.55 |
| **Fail** | Macro F1 < 0.60 |

---

## Committed artifacts

| File | Contents |
|---|---|
| `data/labeled_issues.csv` | 200 labeled issues (train/val/test source) |
| `evaluation_results_before.json` | Run 1 — default Trainer, 0.63 test accuracy |
| `confusion_matrix_before.png` | Run 1 confusion matrix (majority-class collapse) |
| `evaluation_results_after.json` | Run 3 — WeightedTrainer + 5ep, 0.83 test accuracy (+ baseline comparison) |
| `confusion_matrix_after.png` | Run 3 confusion matrix (best fine-tuned run) |

---

## AI tool plan

| Phase | Plan |
|---|---|
| **Label stress-testing** | Used Cursor to generate boundary cases before annotation; tightened rules where labels overlapped. |
| **Annotation assistance** | GitHub labels used as pre-label signal via script; every row reviewed; rule recorded in `notes` column. |
| **Failure analysis** | After training, export misclassified rows to Cursor for pattern clustering; verify each pattern manually on 5–10 examples. |
