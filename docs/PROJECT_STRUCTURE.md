# Project Structure

This document defines the target repository layout for the E-Commerce Delivery Intelligence project. New work should follow this structure as the project progresses through data validation, feature engineering, model development, serving, and operations.

Create files and directories only when the assigned phase needs them. Do not add empty implementation modules merely to reproduce the full tree.

```text
ecommerce-delivery-intelligence/
├── ai-workflows/
│   ├── CODEX_IMPLEMENTATION_WORKFLOW.md
│   └── CLAUDE_REVIEW_TEST_WORKFLOW.md
├── data/
│   ├── raw/
│   ├── interim/
│   └── processed/
├── notebooks/
│   ├── 01_data_understanding.ipynb
│   ├── 02_data_quality.ipynb
│   ├── 03_exploratory_analysis.ipynb
│   └── 04_model_experiments.ipynb
├── src/
│   └── delivery_intelligence/
│       ├── __init__.py
│       ├── config.py
│       ├── data/
│       │   ├── ingestion.py
│       │   ├── validation.py
│       │   ├── aggregation.py
│       │   └── preparation.py
│       ├── features/
│       │   ├── builder.py
│       │   └── definitions.py
│       ├── models/
│       │   ├── classification.py
│       │   ├── regression.py
│       │   └── training.py
│       ├── pipelines/
│       │   ├── prepare_data.py
│       │   ├── train.py
│       │   └── evaluate.py
│       ├── evaluation/
│       │   ├── classification.py
│       │   ├── regression.py
│       │   └── error_analysis.py
│       ├── inference/
│       │   ├── predictor.py
│       │   └── artifacts.py
│       ├── api/
│       │   ├── main.py
│       │   ├── routes/
│       │   └── schemas/
│       └── monitoring/
│           ├── drift.py
│           └── metrics.py
├── dashboard/
│   └── app.py
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── api/
│   └── fixtures/
├── configs/
├── models/
├── reports/
├── docs/
├── scripts/
├── .github/
│   └── workflows/
│       └── ci.yml
├── .gitignore
├── .env.example
├── Dockerfile
├── pyproject.toml
├── README.md
└── Makefile
```

## Directory responsibilities

| Path | Responsibility |
|---|---|
| `ai-workflows/` | Instructions governing Codex implementation and independent Claude review. |
| `data/raw/` | Immutable source data as downloaded. Raw datasets must not be committed to Git. |
| `data/interim/` | Intermediate outputs created during validation, cleaning, and order-level aggregation. |
| `data/processed/` | Final model-ready datasets and chronological data splits. |
| `notebooks/` | Numbered, phase-specific exploration and experimentation. Reusable logic belongs in `src/`. |
| `src/delivery_intelligence/config.py` | Shared configuration, paths, constants, and environment-driven settings. |
| `src/delivery_intelligence/data/` | Data ingestion, validation, aggregation, and preparation at the required order grain. |
| `src/delivery_intelligence/features/` | Purchase-time feature definitions and deterministic feature construction. |
| `src/delivery_intelligence/models/` | Classification and regression model definitions plus shared training logic. |
| `src/delivery_intelligence/pipelines/` | Executable orchestration for preparation, training, and evaluation. |
| `src/delivery_intelligence/evaluation/` | Metrics, diagnostics, and segment-level error analysis. |
| `src/delivery_intelligence/inference/` | Model artifact loading and reusable prediction behavior. |
| `src/delivery_intelligence/api/` | FastAPI application, routes, and validated request/response schemas. |
| `src/delivery_intelligence/monitoring/` | Data drift, prediction quality, and operational metric calculations. |
| `dashboard/` | Interactive application for demonstrations and business-facing exploration. |
| `tests/unit/` | Focused tests for individual functions and modules. |
| `tests/integration/` | Tests covering interactions across data, feature, model, and artifact components. |
| `tests/api/` | API contract, validation, and endpoint tests. |
| `tests/fixtures/` | Small synthetic datasets and reusable test inputs. |
| `configs/` | Version-controlled configuration files for pipelines, models, and evaluation. |
| `models/` | Locally generated model artifacts and metadata. Binary artifacts must not be committed unless explicitly approved. |
| `reports/` | Generated analyses, evaluation summaries, plots, and review handoffs. |
| `docs/` | Project architecture, data contracts, decisions, and operating documentation. |
| `scripts/` | Small maintenance and developer utility scripts that do not belong in the package. |
| `.github/workflows/` | Continuous integration and other GitHub Actions workflows. |

## Structural conventions

- Keep production and reusable logic in `src/delivery_intelligence/`; notebooks should call package code rather than duplicate it.
- Preserve one row per `order_id` in prepared modeling data. Aggregate one-to-many sources before joining them to orders.
- Restrict model features to information available at `order_purchase_timestamp`. Delivery outcomes and later events may be used only to construct labels or evaluation results.
- Keep configuration separate from implementation and avoid hardcoded machine-specific paths.
- Mirror package behavior with focused tests under the corresponding test category.
- Keep datasets, generated reports, trained model binaries, credentials, and local environment files out of Git through `.gitignore` rules.
- Add package `__init__.py` files when package directories are introduced so imports behave consistently across supported environments.

## Phase mapping

| Project phase | Primary locations |
|---|---|
| 1. Understand and validate | `notebooks/01_*`, `notebooks/02_*`, `src/delivery_intelligence/data/validation.py`, `docs/`, `tests/unit/` |
| 2. Prepare and explore | `src/delivery_intelligence/data/`, `notebooks/03_*`, `data/interim/`, `data/processed/` |
| 3. Engineer features and baselines | `src/delivery_intelligence/features/`, `src/delivery_intelligence/models/`, `tests/unit/` |
| 4. Train and evaluate | `src/delivery_intelligence/pipelines/`, `src/delivery_intelligence/evaluation/`, `notebooks/04_*`, `reports/` |
| 5. Serve and test | `src/delivery_intelligence/inference/`, `src/delivery_intelligence/api/`, `tests/integration/`, `tests/api/` |
| 6. Operationalize | `src/delivery_intelligence/monitoring/`, `dashboard/`, `.github/workflows/`, `Dockerfile`, `Makefile` |

The workflow documents in `ai-workflows/` remain the authority for phase scope, acceptance criteria, testing, documentation, and review requirements.
