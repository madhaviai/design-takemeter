# Langfuse Issue Classifier — planning.md

## 1. Community

**Community:** [Langfuse OSS GitHub issues](https://github.com/langfuse/langfuse/issues) (`langfuse/langfuse`).

**Why this community:** Active LLM-observability project with ~300+ public issues. Posts vary from detailed self-hosting bug reports (env vars, repro steps, logs) to structured feature requests to thin "doesn't work" posts. Maintainers triage by type — same task a classifier would automate.

**Why it's a good fit:** Text-heavy issue bodies, clear community norms (bug template vs feature template), and enough volume for 200+ examples via the public GitHub API.

---

## 2. Labels

| Label | Definition |
|---|---|
| `bug_report` | Reports broken behavior with enough detail to investigate (symptoms, env, repro steps, or expected vs actual). |
| `feature_request` | Asks for new capability or improvement; describes desired behavior, not a failure. |
| `support_question` | Seeks help using or configuring Langfuse; no clear broken behavior or feature proposal. |

### Examples

**bug_report**
1. [#14427](https://github.com/langfuse/langfuse/issues/14427) — OTEL replay fails when `LANGFUSE_S3_EVENT_UPLOAD_PREFIX` is set; includes env, steps, expected behavior.
2. [#14420](https://github.com/langfuse/langfuse/issues/14420) — Unquoted heredoc in `dev-tables.sh` causes command substitution; points to file and failure mode.

**feature_request**
1. [#14265](https://github.com/langfuse/langfuse/issues/14265) — Instance-wide model cost configuration with discount factor; describes problem and desired capability.
2. [#14238](https://github.com/langfuse/langfuse/issues/14238) — Import/export portable Playground draft snapshots; scoped feature ask.

**support_question**
1. "How should I configure MinIO endpoint URLs for browser-accessible media on self-hosted Langfuse?" — setup question, no repro of a specific bug.
2. "Is it expected that observation-level evaluators don't trigger on self-hosted v3?" — asks whether behavior is normal before filing a bug.

---

## 3. Hard edge cases

**Ambiguous type:** Self-hosting posts that say "X doesn't work" but may be misconfiguration vs a real bug (e.g. S3 prefix / MinIO setup issues near [#14427](https://github.com/langfuse/langfuse/issues/14427) / [#14299](https://github.com/langfuse/langfuse/issues/14299)).

**Rule during annotation:**
- Has repro steps + env + expected vs actual → `bug_report`
- Asks how to configure or whether behavior is expected → `support_question`
- Describes desired new behavior with use case, no failure → `feature_request`

**If still unclear after 30 seconds:** read first comment; if maintainer asks for repro → `bug_report`; if they point to docs → `support_question`. Log ID in a `borderline.csv` for evaluation review.

---

## 4. Data collection plan

**Source:** GitHub REST API — `GET /repos/langfuse/langfuse/issues?state=all` (issues only, skip PRs). Text field = `title + "\n\n" + body`.

**Target:** 200 labeled issues total.

| Label | Target count | How to find |
|---|---|---|
| `bug_report` | 70 | `bug` label or title starts with `bug:` |
| `feature_request` | 70 | `feature` label or `[Feature Request]` in title |
| `support_question` | 60 | No bug/feature label; title contains how/is/expected/configure |

**If underrepresented after 200:** collect more pages from API; search titles for `how`, `configure`, `self-hosting`; add [GitHub Discussions](https://github.com/langfuse/langfuse/discussions) if still short on `support_question`. Never duplicate the same issue.

**Split:** 160 train / 40 holdout (stratified by label).

---

## 5. Evaluation metrics

| Metric | Why |
|---|---|
| **Macro F1** | Primary metric — treats all 3 labels equally despite imbalance. |
| **Per-class precision & recall** | Accuracy hides weak classes; need to see each label separately. |
| **Confusion matrix** | Focus on `bug_report` ↔ `support_question` boundary — main error source. |

**Not accuracy alone:** ~60% of issues are bugs/features; a model predicting only `bug_report` could look decent on accuracy while failing on questions.

**Priority:** High **recall on `bug_report`** (≥ 0.80) — missing real bugs is worse than misrouting a question.

---

## 6. Definition of success

**Useful in production (triage bot):**
- Macro F1 ≥ **0.75** on holdout
- `bug_report` recall ≥ **0.80**
- `feature_request` precision ≥ **0.75** (don't flood feature backlog with bugs)

**Good enough to ship as assist (human reviews all):**
- Macro F1 ≥ **0.65**
- No single class recall below **0.55**

**Fail:** Macro F1 < 0.60 or `bug_report` recall < 0.65 → revisit labels or add training data before fine-tuning.

These thresholds are checkable from a sklearn classification report — no subjective "works well."

---

## 7. AI Tool Plan

### Label stress-testing
Before annotation: give Cursor/Claude my 3 definitions + edge-case rule. Ask for 10 synthetic Langfuse-style posts on the bug/support boundary. If I can't label ≥8/10 cleanly in under 30s each, tighten definitions first.

### Annotation assistance
**Yes — pre-label only.** Use Groq/Cursor to suggest a label for each scraped issue; I manually confirm or override every row. Track in CSV column `pre_labeled=true/false` and `annotator=human`. Disclose pre-label count in final writeup. Never auto-accept without review.

### Failure analysis
After evaluation: export misclassified examples to CSV; ask AI to cluster error patterns (e.g. "self-hosting config misclassified as bug"). I verify each pattern by reading 5–10 examples myself — AI suggests hypotheses, I confirm with evidence.
