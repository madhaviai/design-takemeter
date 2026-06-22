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

Zero-shot Groq baseline on locked test set (30 examples, 30/30 parseable):

| Metric | Score |
|---|---|
| Accuracy | 0.90 |
| **Macro F1** | **0.60** |
| Weighted F1 | 0.89 |

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `bug_report` | 0.95 | 0.95 | 0.95 | 19 |
| `feature_request` | 0.82 | 0.90 | 0.86 | 10 |
| `support_question` | 0.00 | 0.00 | 0.00 | 1 |

**Reflection:** 90% accuracy is misleading — macro F1 (0.60) is the honest score. Baseline handles `bug_report` and `feature_request` well (clear GitHub templates). It missed the only `support_question` in test (0/1), likely because that class is rare (10/200) and overlaps with bugs that mention confusing docs. **Hypothesis:** fine-tuning should help most on `support_question` and bug/support boundary cases; bug vs feature may not improve much over baseline.

**Targets to beat:** macro F1 > 0.60, `support_question` recall > 0.00, keep `bug_report` recall ≥ 0.80.

---

## Fine-tuning plan

- **Input:** `text` (title + body) → **Target:** `label` via `LABEL_MAP`
- **Split:** Notebook handles 70% / 15% / 15% (train ~140, val ~30, test ~30)
- **Watch:** Class imbalance — only 10 `support_question` examples; monitor per-class recall
- **Goal:** Beat baseline macro F1 (0.60), not just accuracy (0.90)

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

## AI tool plan

| Phase | Plan |
|---|---|
| **Label stress-testing** | Used Cursor to generate boundary cases before annotation; tightened rules where labels overlapped. |
| **Annotation assistance** | GitHub labels used as pre-label signal via script; every row reviewed; rule recorded in `notes` column. |
| **Failure analysis** | After training, export misclassified rows to Cursor for pattern clustering; verify each pattern manually on 5–10 examples. |
