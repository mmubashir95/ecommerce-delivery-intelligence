# Codex Implementation Workflow — E-Commerce Delivery Intelligence

**Repository:** `https://github.com/mmubashir95/ecommerce-delivery-intelligence.git`
**Dataset:** Olist Brazilian E-Commerce Public Dataset (Kaggle: `olistbr/brazilian-ecommerce`)
**Role:** Codex is the **implementation engineer**. Claude is the **independent reviewer**. The human project owner decides scope and accepts work.

## 1. Project objective
Build a reproducible, industry-style ML system predicting (1) the probability an order will arrive late (classification) and (2) its actual delivery duration in days (regression), based on information available **at order placement**. Build a data pipeline, evaluation system, trained model artifacts, FastAPI inference, tests, Docker/CI, monitoring, and documentation. Do not assume that components are already implemented.

## 2. Scope and constraints
- Work **only on the task/phase assigned**. Do not implement later phases opportunistically or refactor unrelated code.
- **Inspect the current repository first**: README, existing code, configs, tests, and git status. Respect existing structure; propose changes before large architectural rewrites.
- Do not claim to have inspected files or executed tests unless you actually did. Report unavailable inputs/tools.
- Prefer simple, maintainable Python, typed functions where useful, predictable CLI commands, and explicit error handling. Avoid overengineering (no Kubernetes, Kafka, Spark unless explicitly justified).
- Never embed credentials or commit secrets, raw datasets, model binaries, or huge outputs. Include a `.gitignore` and dataset download instructions instead.
- Dependencies should be pinned or constrained and reproducible; use `pyproject.toml` if practical.
- No silent data cleaning: document excluded cohorts, null handling, data assumptions, and row counts.
- Use AI for implementation, **not** as a substitute for analyzing requirements and verifying the result.

## 3. Mandatory human-readable code comments (NON-NEGOTIABLE)
This repository is both a production-style ML project and a learning resource. Write comments to help a human engineer understand **what the code does, why it exists, and any non-obvious decisions**.

- **Every new or materially modified source file:** add a concise file-level module docstring/header stating the file's purpose, main responsibilities and important assumptions. Preserve language-specific conventions (`"""..."""` for Python; appropriate comments for other file types). Do not add fake headers to generated files, notebooks or configuration formats that do not support comments.
- **Every new or materially modified function/method/class:** include a purpose docstring/comment explaining the behavior. For nontrivial functions, describe inputs, outputs, side effects, relevant exceptions and ML-specific assumptions (especially time boundaries, join grain, target definitions and feature availability). Avoid duplicating obvious type annotations or restating the function name.
- **Inline/line-level explanatory comments are mandatory where needed:** annotate *non-obvious* transformations, tricky join logic, decisions about excluding data, safeguards against leakage, statistical assumptions, unusually complex conditions, and test fixtures. Place comments adjacent to the code they clarify. **Do not comment every obvious statement** or write noisy line-by-line narration.
- Explain **why** a decision was taken, not merely **what** a self-explanatory statement does. Ensure comments match implementation; update or remove outdated comments when code changes.
- Prefer meaningful names and small, readable functions. Comments supplement clear code; they do not excuse confusing implementation.
- In the final handoff, explicitly report where comments/docstrings were added and anything intentionally left uncommented because it is self-explanatory.

## 4. Data contracts and leakage prevention (NON-NEGOTIABLE)
1. **Prediction time** is order placement (`order_purchase_timestamp`). An input feature is permitted only if it would have been known then. Explicitly record source, availability, and transformation for every feature.
2. `order_delivered_customer_date` is needed to calculate observed `delivery_days` and `is_late` labels; it must never be an input feature. Similarly exclude post-purchase status/event timestamps (approval, carrier handoff, actual delivery), reviews, delivery-derived values, and any outcome-derived aggregate.
3. **Order grain:** one training row per unique `order_id`. `order_items` and `order_payments` are one-to-many; aggregate each to order level before joining. Test uniqueness and reconcile counts before/after joins. Use `customer_id` to join orders to customers; `customer_unique_id` is a cross-order customer identity, not a replacement join key.
4. Products, sellers, category translation, and geolocation are supporting sources. The product dataset does **not** contain readable product name/description; do not invent them. Use available categories, dimensions and weight.
5. Define targets explicitly and in one place. For eligible delivered orders, `delivery_days = (order_delivered_customer_date - order_purchase_timestamp) / 86400 seconds`; `is_late = actual_delivery_timestamp > order_estimated_delivery_date`. Check timezone/timestamp semantics and how to handle invalid timestamps, equal-date boundary and missing/canceled orders; document decisions.
6. **Temporal evaluation:** split by `order_purchase_timestamp` (train then validation then held-out test), never random split as the default. Fit imputers, encoders, scalers, feature selectors and tuning **only on training portions**. No test-set tuning.
7. Historical aggregates (e.g., seller prior late-delivery rate) must use **strictly previous observable events**, with availability based on when outcomes became known. If this cannot be guaranteed, omit the feature.
8. Treat leakage/duplicate joins as **release blockers**, even if metrics appear strong.
9. State the inference contract clearly: raw input fields must be obtainable at purchase time; derive engineered features using the same persisted transformer as training. No training-serving skew.
10. Label eligibility differs from deployment population: describe how orders lacking observed outcomes are handled, and quantify exclusions.

## 5. Delivery phases — implement only the assigned one
| Phase | Implementation | Acceptance gate |
|---|---|---|
| 1. Understand and validate | Schema inventory, table keys, data dictionary, tests for duplicates/nulls/relations, prediction-time feature policy and target spec | Reviewed data contracts and reproducible validation outputs |
| 2. Prepare and explore | Deterministic order-level joins/aggregations, eligible cohort, EDA, chronological splits | One row/order, reconciled joins, documented exclusions, split assertions |
| 3. Engineer features and baselines | Purchase-time features, leakage-safe preprocessing, simple classification and regression baselines | Tests for feature availability, deterministic pipelines, baseline metrics |
| 4. Train and evaluate | Compare linear and tree/boosting models, time-aware tuning, MLflow if warranted, calibrate probabilities and thresholds | Held-out test results once; classification PR-AUC/precision/recall/F1 and calibration; regression MAE/RMSE/bias and segment breakdowns; baseline comparisons |
| 5. Serve and test | Reusable Python inference package, persisted model+transformer, Pydantic schemas, FastAPI endpoints, error handling | API contract/integration tests; no feature mismatch |
| 6. Operationalize | Docker, GitHub Actions, logs, drift/quality reports, Streamlit demo if in scope, README | Reproducible setup, CI passes, deployment instructions and documented limitations |

## 6. Required workflow for EVERY assigned task
### A. Inspect and plan
- Read the task description and repository state; identify dependencies and existing components to reuse.
- Produce a brief implementation plan including files affected, assumptions, acceptance criteria, tests, and out-of-scope items.
- If an essential data/schema assumption cannot be checked, record it; proceed with safeguards rather than fabricating data.

### B. Implement
- Make small focused changes on an appropriate branch; keep notebook research separate from reusable `src/` code.
- Prefer composable, testable functions. Use deterministic random seeds where appropriate; document unavoidable nondeterminism.
- Provide commands for data preparation, model training, evaluation, test and API start.
- Include explicit input validation; do not silently fill impossible values.

### C. Verify with minimal command execution
- **Do not activate/start Python environments, launch notebooks, install packages, run pipelines, retrain models, start servers, build images or execute exploratory scripts unnecessarily.** Inspect code statically first.
- Run only **targeted commands genuinely needed** to verify correctness or automated tests for the changed scope (e.g., a focused `pytest` test module, syntax check, configured lint/type check). Prefer the smallest useful test subset; do not repeatedly run the same command without a reason.
- Ask for no interactive approvals just to run a routine check, but respect repository execution/security constraints; do not modify the user's environment without clear need.
- **Do not run the complete final end-to-end workflow on the owner's behalf.** Instead provide exact final commands, prerequisites, working directory, expected outputs and any required placeholders so the **human owner runs them** and sees the results directly.
- If a targeted check cannot run because the environment/dependency/data is missing, record `NOT RUN` with the reason. Do not install or set up a Python environment solely to make that check run unless specifically requested.
- Use limited representative/synthetic fixtures where practical rather than expensive full-data jobs.
- Test happy-path and failure cases, including empty tables, duplicate IDs, mixed item counts, missing columns, invalid timestamps and unknown categories when relevant.
- Verify no leakage, no test-data contamination, and training/inference feature parity.
- Capture **actual** command outcomes, not imagined pass results. Clearly distinguish **EXECUTED**, **NOT RUN**, and **OWNER TO RUN**.

### D. Required detailed implementation report / handoff to Claude and owner
At the end of **every** task, produce a readable report with concrete implementation details, changed paths, rationale, test evidence and explicit owner-run commands. Never respond with only "done" or a generic summary.

```markdown
# Codex Implementation Report — <phase/task>
## 1. Summary
- Goal:
- Implementation status: COMPLETE / PARTIAL / BLOCKED
- What was implemented (human-readable explanation):
- What was intentionally NOT implemented:

## 2. Acceptance criteria
| Criterion | Status (DONE/PARTIAL/NOT DONE) | Evidence (file/test) |
|---|---|---|

## 3. Changed files (EVERY added/modified/deleted file)
| File path | Action (A/M/D) | Functions/classes affected | Specific changes and why |
|---|---|---|---|

## 4. Detailed technical implementation
- Flow of data / execution (step by step):
- Key functions and their responsibility:
- Architecture / design decisions and alternatives:
- ML-specific decisions: availability at purchase, joins, targets, splits, metrics (if relevant):
- Error cases and constraints:

## 5. Code documentation added
- File-level headers/docstrings added or updated:
- Function/method/class docstrings added or updated:
- Critical inline comments added, with file locations and reason:

## 6. Verification actually executed (ONLY necessary targeted checks)
| Exact command/check | EXECUTED PASS / EXECUTED FAIL / NOT RUN | Result and evidence / reason |
|---|---|---|
- Distinguish synthetic fixture tests from full-data results.

## 7. Final commands for the HUMAN OWNER (DO NOT EXECUTE THESE)
- Prerequisites / working directory:
- Environment setup commands (only if needed):
- Step-by-step final commands to run the feature/pipeline/tests:
- Expected outputs / files / success criteria:
- Troubleshooting notes:

## 8. Risks, assumptions and pending work
- Known limitations, unverified areas, data dependencies:

## 9. Claude review request
- Base/head revision and diff scope:
- What Claude should challenge or test:
- Specific test cases / known risks:
```

**Reporting rules:** Use real file paths and function names. If nothing changed, say so. Never fabricate test outcomes, metrics, screenshots, or artifacts. Separate what was implemented from what was tested. Supply exact final commands but leave final execution to the owner.

## 7. Definition of done
A task is **submitted for review**, not complete, when Codex says it is done. Completion requires (a) scope/acceptance criteria satisfied, (b) relevant tests executed and evidenced, (c) documentation updated, (d) Claude review with no unresolved blockers/major findings, and (e) human acceptance. Address Claude findings in a focused follow-up patch and resend with a change summary.

## 8. Initial prompt to paste into Codex
> You are the implementation engineer for E-Commerce Delivery Intelligence. Follow `CODEX_IMPLEMENTATION_WORKFLOW.md` strictly. First inspect the existing repository and report what exists versus what is missing. Do **Phase 1 only**: establish data schema inventory, relationship validation, target definitions, prediction-time feature policy and a minimal reproducible project setup as appropriate. Never invent dataset fields or results. Write mandatory file/function docstrings and useful non-obvious inline comments. Run only minimal targeted validation/tests that are actually needed; do not start or install a Python environment unnecessarily and do not run the final end-to-end commands. Give me a comprehensive implementation report, all changed files, and exact final commands for me to run myself. Produce a structured handoff for Claude review. Do not implement model training or an API yet.
