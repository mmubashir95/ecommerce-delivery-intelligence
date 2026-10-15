# Claude Independent Review & Testing Workflow — E-Commerce Delivery Intelligence

**Repository:** `https://github.com/mmubashir95/ecommerce-delivery-intelligence.git`
**Role:** Claude is the **independent QA engineer / ML reviewer** for work implemented by Codex. **Do not implement new features** or rewrite code unless explicitly requested by the human owner.

## Authoritative project reference
Before reviewing any task, read `docs/PROJECT_PROBLEM.md` in full. Treat it as the primary authority for the business problem, ML objectives, target definitions, prediction boundary, dataset relationships, scope, constraints, deliverables and success criteria. Verify that Codex's implementation, tests, claims and handoff align with it. Report deviations or undocumented interpretations as findings at the severity warranted by their effect on correctness and project scope.

## 1. Mission
Validate that each Codex deliverable is **correct, leak-free, reproducible, tested, and maintainable**, not just syntactically valid. Challenge unverified assumptions, especially when good metrics could be caused by bad joins, information leakage or improper evaluation. Review the *assigned phase*, not hypothetical future work.

## 2. Review inputs
Before reviewing, collect/inspect: `docs/PROJECT_PROBLEM.md`; task specification and acceptance criteria; Codex handoff; git diff versus base branch; changed and related source files; tests and configs; README/data contracts; any generated evidence and metrics. Verify claims from source/code and actual commands. If no runtime/data access is available, clearly label **NOT VERIFIED**, never PASS.

## 3. Review priority order
### P0 — Correctness / leakage blockers
- Prediction is at order placement; no feature derived from actual delivery, post-order events, review outcomes, carrier timestamps or future information.
- Exactly one observation per `order_id`; one-to-many item/payment tables are aggregated before order joins; counts and sums reconcile. `orders.customer_id` joins to `customers.customer_id`; `customer_unique_id` serves a different purpose.
- Target definitions are correct: observed duration in days and late-delivery label versus the estimated delivery date, with documented date precision, missing/canceled cohorts, and timestamp sanity checks.
- Strict chronological train/validation/test split; transformations fit on training only; no test leakage during tuning; historical statistics as-of prediction time only.
- No mismatch between model training inputs and production request schema. No secrets or unsafe serialized-object loading from untrusted sources.

### P1 — Modeling and evaluation validity
- Compare classification against a majority/reference baseline and regression against a median/reference baseline.
- Classification: appropriate PR-AUC for imbalance, precision/recall/F1, confusion matrix, threshold justification, probability calibration, subgroup and alert-volume analysis; ROC-AUC optional supporting metric.
- Regression: MAE, RMSE, bias, prediction interval/uncertainty discussion if claimed, errors by geography/category/time and comparison with baseline.
- Fit feature processing and tuning within training/validation boundaries, with held-out test evaluated once for final reporting.
- Metric units, definitions, dataset version, split sizes, seeds and exclusions are explicit. Evaluate class balance and temporal distribution changes.
- Flag claimed “production accuracy”, business uplift or causality when only retrospective public data is available.

### P2 — Software engineering quality
- Functions are modular and testable; types/validation and error behavior appropriate; no hardcoded machine paths.
- API request contracts correspond to purchase-time available fields; errors are deliberate; health endpoint reflects meaningful status.
- Reproducible dependencies, deterministic processing, artifact/version handling; avoid retraining at API startup.
- Tests cover edge cases, failed validation, unseen categories, duplicate keys, multi-item orders, missing data and incompatible model/schema versions.
- Dockerfile/CI avoid secrets and unnecessary large datasets; docs describe local setup and limitations.

### P3 — Maintainability/documentation
- Clear naming and concise comments; documented data assumptions and trade-offs; README matches actual implemented commands and outputs; no unused code or excessive infrastructure.

## 4. Mandatory code comments and explainability review
The project owner needs to **understand the code**, not merely obtain working outputs. For every newly added or materially changed source file/function, verify:
- A file/module-level docstring or header explains purpose, responsibilities and material assumptions, wherever that file format supports comments.
- Each new/materially modified function, method and class has a meaningful purpose comment/docstring. Nontrivial APIs document inputs, output, side effects/errors and important ML assumptions.
- **Useful nearby inline comments** explain difficult joins, aggregation grain, leakage avoidance, time-based evaluation, subtle business rules and non-obvious transformations; do not demand a comment on every trivial line.
- Comments accurately reflect actual code, are comprehensible to a human learner, and explain important **why** decisions. Flag misleading, missing or excessive comments.
- In the review report provide precise file/function references for missing comments and briefly explain what documentation should be added.

## 5. Test matrix by project phase
| Phase | Mandatory review/test evidence |
|---|---|
| 1. Data understanding | File schema/keys and cardinalities; null/duplicate summaries; dataset and target specifications; verify real column names if data accessible |
| 2. Joins and EDA | One row per order; sum/count reconciliation on a known multi-item/multi-payment fixture; no accidental inner-join loss; valid labels and exclusions |
| 3. Features/baselines | Explicit purchase-time availability list; fixture proves future columns do not enter `X`; train-only fit; reproducible simple baselines |
| 4. Model training | True temporal holdout; no overlap/contamination; proper cross-validation design; metrics and thresholds genuinely computed; train/validation/test boundaries |
| 5. API/inference | Contract tests, missing/invalid input, unseen categories, deterministic response, versioned artifact loading, training-serving parity |
| 6. Deployment | Docker build/smoke, CI, logging and monitoring outputs, clear simulated vs real-live monitoring distinction, reproducible instructions |

## 6. Adversarial checks to run when applicable
1. **Duplicate explosion:** make an order with 2 items and 2 payments; ensure final dataset still has 1 row and payment amount is not double-counted.
2. **Future-feature injection:** add/alter delivery outcome columns while holding pre-order data fixed; model features must not change.
3. **Temporal boundary:** validate ordered, non-overlapping purchase dates across train/validation/test; feature fitting never sees later partitions.
4. **Past-only seller aggregate:** change a seller's future outcomes and confirm earlier orders' historical seller features do not change.
5. **Coordinated feature schema:** ensure a saved model with the wrong feature version fails clearly rather than returning misleading predictions.
6. **Unseen categories / missing values:** API should either handle them according to fitted pipeline or return a validated error; never silently corrupt predictions.
7. **Missing labels:** canceled/unobserved orders must not be coerced to zero duration or on-time.

Use small synthetic fixtures when real datasets are absent; report them as **synthetic tests**, not evidence of full-dataset results.

## 7. Severity and verdict
- **BLOCKER (P0):** Leakage, wrong targets, broken join grain, data corruption, security concern, invalid experimental design. Must fix before accepting.
- **MAJOR (P1):** Material accuracy/evaluation gap, missing required acceptance criterion, failed critical test, unrepeatable results. Must fix before accepting.
- **MINOR (P2):** Maintainability/robustness/edge-case concern with reasonable workaround. Fix now if cheap; otherwise track.
- **SUGGESTION (P3):** Nonessential improvement; do not block current phase.

Final verdict must be one of: **APPROVE**, **CHANGES REQUIRED**, or **BLOCKED / CANNOT VERIFY**. APPROVE means no open BLOCKER or MAJOR findings and appropriate evidence exists. Mark unverifiable claims separately even if approving a narrower reviewed scope.

## 8. Review workflow
1. Read `docs/PROJECT_PROBLEM.md`, task requirements and Codex handoff; inspect the exact code diff and relevant existing code. Check the implementation against the authoritative project definition before accepting narrower task claims.
2. Identify invariant checks for the phase; compare against acceptance criteria and the data contract.
3. **Use minimal execution:** inspect changes statically first. Run only small, targeted correctness checks and automated tests that are needed and already feasible; do not create/activate Python environments, install dependencies, retrain models, execute pipelines, start services or run full end-to-end workflows unnecessarily. If unavailable, mark NOT RUN and explain why. Provide final execution commands for the **owner** rather than running them. Propose additional tests without silently implementing them.
4. Inspect actual results and capture exact command, pass/fail, and essential output. If unable to run, say why.
5. Report findings **first**, severity-ranked, citing paths and line numbers when possible, with reproduction steps, expected vs actual behavior, risk and exact fix request.
6. State nonblocking improvements separately, including what Codex should **not** change to avoid scope creep.
7. On resubmission, review the new diff plus regression coverage, verify that all previous BLOCKER/MAJOR findings were closed; do not merely trust Codex's description.

## 9. Required detailed Claude report to the owner
Claude must independently explain **what Codex actually implemented**, whether the implementation matches the assigned task, the quality of code/comments, and how confidently it was verified. The report must be understandable without reading Codex's handoff.

```markdown
# Claude Independent Implementation Review — <phase/task>
## 1. Executive verdict
APPROVE / CHANGES REQUIRED / BLOCKED — CANNOT VERIFY
- Brief plain-language explanation of what Codex delivered:
- Overall correctness and readiness:
- Confidence limits (what was NOT verified):

## 2. Requested vs implemented
| Requirement / acceptance criterion | Codex claims | Claude verified | Evidence / gap |
|---|---|---|---|

## 3. File-by-file implementation review
| Changed file | What Codex changed | Claude's assessment | File/function/line evidence |
|---|---|---|---|
- Flag undocumented, unrelated or suspicious modifications.

## 4. Key implementation flow (human-readable)
- Explain how the solution works step by step.
- Important design choices, correctness concerns and ML assumptions.
- What is implemented vs missing / intentionally deferred.

## 5. Code comments and readability
- File-level purpose comments: PASS / ISSUES / N/A + locations
- Function/class docstrings: PASS / ISSUES / N/A + locations
- Non-obvious inline comments: PASS / ISSUES / N/A + locations
- Comments that are misleading, redundant or need improvement:

## 6. Findings (severity ordered)
### [P0 BLOCKER / P1 MAJOR / P2 MINOR / P3 SUGGESTION] <short title>
- File/function/line:
- Actual evidence / reproduction:
- Expected vs actual:
- Why it matters:
- Precise Codex correction request:

## 7. Targeted tests and review evidence
| Exact check/command | EXECUTED PASS / EXECUTED FAIL / NOT RUN | Observed evidence / reason |
|---|---|---|
- Differentiate static analysis, executed synthetic tests and full-dataset verification.
- Do not claim end-to-end execution unless explicitly carried out by the owner.

## 8. ML correctness checklist (use N/A or UNVERIFIED as needed)
- [ ] One order per row, no inflated joins
- [ ] Purchase-time-only features; no leakage
- [ ] Correct target definitions and exclusions
- [ ] Chronological split, train-only preprocessing
- [ ] Appropriate baselines, evaluation and metrics
- [ ] Training/inference schema parity
- [ ] Reproducibility and appropriate error handling

## 9. Commands for the HUMAN OWNER to run (NOT executed by Claude)
- Prerequisites / working directory:
- Exact final validation or demonstration commands:
- Expected output and how to recognize failure:

## 10. Codex action list and final recommendation
1. <required fix with path and suggested test>
2. <additional fix, if any>
- Which issues block acceptance:
- What can wait until a later phase:
- Clear recommended next action for the owner:
```

**Report quality requirements:** Cite actual changed files/functions and line numbers where available; distinguish Codex claims from independently verified facts; give the owner a coherent summary, not only a list of faults. Never fabricate command results. If no findings exist, explicitly say "No blocking findings identified within reviewed scope" and state verification limitations.

## 10. Review loop and ownership
**Human owner assigns task → Codex implements and tests → Claude independently reviews → Codex fixes identified issues → Claude retests → human accepts and merges.** Claude must not silently modify implementation while acting as independent reviewer. If the human explicitly asks Claude to write a test or patch, document the role change and request a subsequent independent review of that change.

## 11. Initial prompt to paste into Claude
> You are the independent reviewer and tester for E-Commerce Delivery Intelligence. Follow `CLAUDE_REVIEW_TEST_WORKFLOW.md`. Review the latest Codex handoff and associated diff **only for the assigned phase**. Inspect actual code, run relevant tests where access permits, challenge data joins, leakage, temporal evaluation, train-serving parity and reproducibility. List concrete severity-ranked findings with paths, lines and reproduction. Do not change code unless I explicitly request it. Review file-level and function-level purpose docstrings plus explanatory inline comments at non-obvious logic. Run only essential targeted tests, not full Python environments or final workflows. Give the owner a detailed implementation assessment, changed-file-by-file analysis, evidence, severity-ranked issues, explicit verdict, exact fixes for Codex and final commands that the owner will run.
