# E-Commerce Delivery Intelligence — Project Problem Definition

## 1. Document purpose and authority

This document is the single source of truth for the business problem, machine learning objectives, data boundaries, project scope, expected outcomes, and success criteria of the E-Commerce Delivery Intelligence project. Implementation plans, code, experiments, reviews, and documentation must align with it. If a later discovery conflicts with this document, the conflict must be recorded and this document must be deliberately updated before the new interpretation is treated as authoritative.

| Item | Definition |
|---|---|
| Project | E-Commerce Delivery Intelligence |
| Repository | <https://github.com/mmubashir95/ecommerce-delivery-intelligence> |
| Project type | End-to-end, production-oriented machine learning engineering project |
| Dataset | Brazilian E-Commerce Public Dataset by Olist |
| Dataset source | [Kaggle dataset page](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) |
| Prediction time | Order placement, represented by `order_purchase_timestamp` |
| Modeling grain | One record per `order_id` |

The primary goal is to build a reproducible ML system that predicts delivery risk and duration while demonstrating industry-relevant practices in data engineering, feature engineering, model development, evaluation, deployment, testing, and monitoring.

### Status language

This document uses the following labels to keep facts separate from decisions and future work:

- **Confirmed fact:** information defined by the published Olist dataset or agreed project objective.
- **Design decision:** a rule adopted for consistent implementation and evaluation.
- **Assumption to validate:** a reasonable interpretation that must be checked against the downloaded data or operational contract.
- **Future work:** work intentionally deferred beyond the initial version.

## 2. Business problem

E-commerce delivery promises influence customer expectations and operational planning. Some orders arrive after the promised delivery timestamp, while delivery duration varies across order, product, seller, freight, time, and geographic conditions. Without an early estimate, operational teams have limited ability to identify risky orders, prioritize investigation, communicate proactively, or understand recurring delivery patterns.

At the time an order is placed, the system must answer two related questions:

1. What is the probability that this order will arrive late?
2. How many elapsed days will this order take to reach the customer?

### Stakeholders and possible decisions

| Stakeholder | Information need | Possible use of model output |
|---|---|---|
| Logistics and operations teams | Early visibility into delivery risk and expected duration | Prioritize high-risk orders, investigate lanes or sellers, and plan interventions |
| Customer support teams | Consistent risk context before a delay occurs | Prioritize outreach or prepare support responses |
| Marketplace and seller-management teams | Patterns in delivery reliability | Identify segments requiring operational review |
| Product and analytics teams | Measurable delivery-risk signals | Evaluate product experiences and design decision-support tools |
| ML and platform engineers | Reliable, reproducible predictions | Operate, test, monitor, and improve the system |

The models are decision-support tools. Potential benefits include earlier risk detection, better prioritization, clearer delivery estimates, and more focused operational analysis. The historical dataset cannot by itself prove customer-satisfaction gains, cost reductions, causal business uplift, or production effectiveness. Such outcomes require a live operating context and controlled measurement.

## 3. Machine learning problem definitions

The project contains two supervised learning problems built from the same purchase-time prediction boundary. Target construction may use later outcomes, but model inputs may not.

### 3.1 Problem A — late-delivery classification

**Objective:** estimate the probability that an eligible order will be delivered after its promised delivery timestamp.

**Target:** `is_late`

For an eligible delivered order:

```text
is_late = 1 if order_delivered_customer_date > order_estimated_delivery_date
          0 otherwise
```

This is a strict timestamp comparison. Delivery exactly at the estimated timestamp is on time (`is_late = 0`). Both fields must be parsed consistently and validated before comparison. The initial implementation must document whether the source timestamps are timezone-naive or timezone-aware and must not silently mix timestamp conventions.

Eligibility requires a valid actual customer-delivery timestamp and a valid estimated-delivery timestamp. Canceled, undelivered, missing-label, or invalid-timestamp records must not be silently assigned `is_late = 0`; their exclusion and counts must be reported.

**Expected output:** a probability in the closed interval `[0, 1]`. An optional risk class may be produced only from a threshold selected and documented using validation data and an explicit operating tradeoff. The held-out test set must not be used to choose the threshold.

**Evaluation:**

- Precision, recall, and F1-score at the selected threshold
- PR-AUC as a primary ranking metric when late deliveries are imbalanced
- ROC-AUC as a supporting ranking metric
- Confusion matrix and predicted-positive alert volume
- Probability calibration, using appropriate calibration diagnostics
- Segment-level performance where sample sizes support responsible interpretation

The first benchmark must be a simple reference classifier, such as a class-prior or majority baseline. More advanced algorithms are justified only by measured improvement and operational suitability.

### 3.2 Problem B — delivery-duration regression

**Objective:** estimate elapsed time from order placement to actual customer delivery.

**Target:** `delivery_days`

For an eligible delivered order:

```text
delivery_days = (
    order_delivered_customer_date - order_purchase_timestamp
).total_seconds() / 86,400
```

The target is elapsed days and may be fractional; it must not be rounded during target construction. Rows with missing or invalid timestamps, delivery before purchase, or no observed delivery require an explicit exclusion rule and reported counts.

**Expected output:** predicted delivery duration in elapsed days. The implementation must define how impossible predictions, such as negative durations, are prevented or handled without hiding model behavior.

**Evaluation:**

- Mean absolute error (MAE), reported in days
- Root mean squared error (RMSE), reported in days
- Prediction bias, with its sign convention documented
- Segment-level error analysis across relevant and sufficiently supported groups
- Comparison with a simple reference regressor, such as the training-set median duration

### 3.3 Shared modeling requirements

- Establish reproducible simple baselines before advanced model development.
- Use chronological train, validation, and held-out test partitions based on `order_purchase_timestamp`.
- Fit preprocessing, feature selection, calibration, and tuning only within the permitted training or validation boundary.
- Evaluate the held-out test set once for final model selection reporting, after choices are fixed.
- Report cohort sizes, exclusions, class balance, time ranges, metric definitions, and relevant uncertainty or limitations.
- Persist training-consistent preprocessing with the model to prevent training-serving skew.

No numerical performance target is set before empirical baselines and business operating requirements exist. Invented accuracy or error thresholds are not success criteria.

## 4. Dataset description and relationships

The source is the public Olist relational dataset. The table descriptions and relationships below define expected use, but Phase 1 must verify actual files, columns, types, nulls, uniqueness, and cardinalities against the downloaded version.

| File | Represents | Principal relationship and modeling relevance |
|---|---|---|
| `olist_orders_dataset.csv` | Order lifecycle records, including purchase, estimated delivery, and observed status timestamps | Central order table keyed by `order_id`; links to customers through `customer_id`; supplies prediction time and target timestamps |
| `olist_customers_dataset.csv` | Order-level customer identifiers and customer location attributes | `customer_id` joins to orders; `customer_unique_id` links repeated purchases attributed to the same underlying customer |
| `olist_order_items_dataset.csv` | Items purchased within each order, with product, seller, price, freight, and item-level records | Many rows may map to one `order_id`; joins to products through `product_id` and sellers through `seller_id`; must be aggregated before the final order join |
| `olist_products_dataset.csv` | Product category and physical attributes | Keyed by `product_id`; enriches order items before order-level aggregation |
| `olist_sellers_dataset.csv` | Seller identifiers and seller location attributes | Keyed by `seller_id`; joins through order items; one order may involve multiple sellers |
| `olist_order_payments_dataset.csv` | Payment records associated with orders | Multiple payment rows may map to one `order_id`; must be aggregated independently before joining to order grain |
| `olist_order_reviews_dataset.csv` | Customer review records associated with completed order experiences | Relates reviews to orders, but review content, scores, and review timing occur after purchase and are excluded from purchase-time model inputs |
| `olist_geolocation_dataset.csv` | Geographic coordinates associated with postal-code prefixes | Postal prefixes may repeat; geolocation must be reduced to a documented lookup grain before joining to customer or seller locations |
| `product_category_name_translation.csv` | Translation mapping for product category names | Enriches product categories through the category-name field; it does not create product descriptions or new product identities |

### 4.1 Customer identity

`customer_id` and `customer_unique_id` serve different purposes. `customer_id` is the key used by the orders table to reference its customer record. `customer_unique_id` is a cross-order identity that can associate separate customer records with the same underlying buyer. It must not replace `customer_id` in the orders-to-customers join. Any feature based on prior customer behavior is a historical feature and must satisfy the same strict availability rules as seller history.

### 4.2 One-to-many tables and order grain

The final modeling dataset must contain exactly one row per `order_id`. Order items and payments are one-to-many sources, and an order may contain several products, item rows, payment records, or sellers. Each one-to-many source must be aggregated to order level independently before it is joined to the central order table. Joining raw item and payment rows together would create a multiplicative row explosion and corrupt counts and sums.

Product and seller attributes should first enrich item records and then be summarized with explicit aggregation rules. Multi-seller orders must be represented with order-level values such as seller count or defensible aggregates rather than duplicated training observations. Join counts, key uniqueness, and financial or item-count reconciliation must be tested before and after each material join.

### 4.3 Prediction-time availability by source

| Source | Purchase-time use |
|---|---|
| Orders | Purchase timestamp and the estimated delivery promise are candidates; later lifecycle status and timestamps are prohibited as features |
| Customers | Location attributes are candidates, subject to validation of inference availability |
| Order items, products, and sellers | Checkout-known item, product, seller, price, freight, and physical attributes are candidates, subject to source and inference-contract validation |
| Payments | Candidate only if the selected fields are demonstrably finalized and obtainable at prediction time; later payment events are prohibited |
| Reviews | Prohibited as features because they describe post-purchase experience |
| Geolocation | Candidate static lookup data after duplicate-safe aggregation and quality validation |
| Category translation | Candidate static reference data |

## 5. Prediction timing and leakage policy

**Design decision:** every prediction represents information available at `order_purchase_timestamp`. A feature is allowed only when its value could be reproduced at that moment in the intended inference service.

The following values may be used for label construction, eligibility analysis, or retrospective evaluation but must never enter model features:

- Actual customer-delivery timestamps
- Carrier handover or carrier-delivery timestamps recorded after purchase
- Approval, fulfillment, cancellation, or other post-purchase lifecycle outcomes
- Customer reviews, review scores, comments, or review timestamps
- Delivery-derived durations, lateness indicators, or aggregates containing the current order outcome
- Seller, customer, route, or category statistics calculated using future orders or outcomes that were not yet observable

Historical aggregates require point-in-time-correct computation. For an order at time `t`, only events strictly before `t` whose relevant outcomes were already known by `t` may contribute. A future outcome must never change an earlier order's feature value. If this guarantee cannot be implemented and tested, the historical feature must be omitted.

Static reference data also needs an availability policy. A current lookup table must not silently supply information that would have differed historically unless that limitation is documented and accepted.

Leakage, duplicate-driven target contamination, or preprocessing fitted with validation or test data is a release blocker regardless of apparent model performance.

## 6. Candidate feature framework

The features below are ideas for validation, not automatically approved model inputs. Approval requires a documented source, definition, aggregation rule, purchase-time availability argument, missing-value policy, and training/inference implementation.

| Candidate feature group | Examples | Required validation |
|---|---|---|
| Order composition | Total order value, item count, distinct product count, distinct seller count | Aggregate at order grain without multiplying item or payment rows; define the authoritative monetary source |
| Product characteristics | Weight, length, height, width, derived volume, category mix | Validate units, missingness, impossible values, and aggregation across multiple items |
| Geography | Seller and customer location attributes, approximate geographic distance | Resolve repeated postal-prefix geolocation records; document coordinate aggregation and distance approximation |
| Freight | Total freight cost, freight-to-item-value ratio | Confirm availability at purchase and guard against invalid denominators |
| Purchase calendar | Purchase weekday, month, hour, or seasonal indicators | Derive only from `order_purchase_timestamp`; document locale and timestamp semantics |
| Delivery promise | Promised delivery duration from purchase to estimated delivery | Use the estimate available at purchase; validate negative or missing promises |
| Payments | Payment value or method summaries | Confirm each selected field is finalized and available at prediction time |
| Historical behavior | Past seller delivery rate, duration, or volume | Use strictly past, already-observed outcomes with point-in-time-correct tests and cold-start handling |

Feature approval belongs in a versioned feature definition or data contract during the relevant phase. Features may be removed for leakage risk, poor data quality, unstable inference availability, excessive complexity, fairness concerns, or lack of measurable value.

## 7. Project scope

### 7.1 In scope

- Data ingestion, schema checks, and quality validation
- SQL or dataframe joins with explicit order-level aggregation and reconciliation
- Exploratory data analysis
- Leakage-safe feature engineering
- Classification and regression baselines, training, comparison, and evaluation
- Time-aware hyperparameter tuning where justified
- Reproducible experiment tracking and saved artifacts
- FastAPI inference service with validated contracts
- Unit, integration, and API testing
- Docker containerization and continuous integration/deployment workflows appropriate to the repository
- Basic data, drift, prediction, and operational monitoring
- Optional demonstration dashboard
- Complete technical and operating documentation

### 7.2 Out of scope for the initial version

- Real-time streaming infrastructure
- Kubernetes
- Large-scale distributed ML training
- Unnecessary microservice decomposition
- Deep learning without evidence that the problem and baselines justify it
- Additional ML objectives unrelated to delivery lateness or duration
- Claims of production impact without live validation

## 8. Constraints, assumptions, and unresolved decisions

### 8.1 Confirmed constraints and design decisions

- Olist is a historical public dataset, not a live production feed.
- Canceled, undelivered, missing-label, and invalid-timestamp orders require explicit eligibility rules and reported counts.
- Multi-item, multi-payment, and multi-seller orders must not create duplicate modeling rows.
- The final modeling grain is one row per `order_id`.
- Train, validation, and test partitions are chronological by purchase time.
- Imputation, encoding, scaling, feature selection, calibration, and tuning must not learn from validation or test data outside their defined role.
- Both tasks require reproducible baselines, training, evaluation, configuration, and artifact handling.
- Unknown decisions must be recorded rather than silently assumed.

### 8.2 Assumptions to validate

- Source timestamp parsing, precision, and timezone semantics
- Exact label-eligible cohort and treatment of anomalous delivered records
- Availability of payment, freight, product, seller, and customer fields in a realistic purchase-time inference request
- Geolocation aggregation rules and distance quality
- Dataset version and file integrity
- Operational probability threshold and acceptable alert volume
- Segments suitable for reliable error analysis

These items must be resolved through Phase 1 data contracts or later phase-specific decisions. Until then, they are not confirmed facts.

## 9. Expected deliverables

1. A validated, documented, and reproducible data preparation pipeline.
2. Exploratory data analysis and data quality reports with cohort and exclusion counts.
3. A reusable, leakage-safe feature engineering pipeline.
4. A trained and evaluated late-delivery classification model.
5. A trained and evaluated delivery-duration regression model.
6. Reproducible experiment records, model metadata, and saved artifacts.
7. FastAPI prediction endpoints with explicit request and response contracts.
8. Automated unit, integration, and API tests.
9. A Dockerized application with reproducible build and run instructions.
10. A CI/CD workflow appropriate to the project's deployment context.
11. Basic model, data-quality, drift, and service monitoring.
12. Complete GitHub documentation covering setup, execution, design, results, limitations, and review evidence.

## 10. Success criteria

The initial project is successful when the following criteria are supported by inspectable evidence:

| Area | Measurable acceptance evidence |
|---|---|
| Data correctness | Source schemas and relationships are validated; joins reconcile; final datasets contain one unique row per eligible order |
| Target correctness | `is_late` and `delivery_days` follow the definitions in this document; exclusions and invalid records are quantified |
| Leakage prevention | Every model feature has a documented purchase-time availability argument; adversarial or invariant tests find no known future-data leakage |
| Evaluation validity | Chronological partitions are non-overlapping; preprocessing is fit within the correct boundary; both models are compared with simple baselines using defined metrics |
| Reproducibility | Documented commands, dependencies, configurations, random seeds where applicable, and versioned artifacts reproduce training and evaluation within stated limits |
| Inference reliability | The service applies the same validated feature and preprocessing contract used in training and fails clearly on incompatible inputs or artifacts |
| Software quality | Relevant automated unit, integration, and API checks pass in the documented environment |
| Operability | Container, CI/CD, deployment, logging, and basic monitoring steps are documented and verified to the degree claimed |
| Communication | Results, business relevance, assumptions, cohort limits, ethical or operational concerns, and model limitations are stated without unsupported impact claims |

Model metrics must be reported as observed results, not predetermined claims. Promotion decisions should consider baseline improvement, calibration, segment behavior, stability, inference cost, and the eventual business operating point.

## 11. Six-phase roadmap

| Phase | Main objective | Expected deliverables |
|---|---|---|
| 1. Data Understanding and Validation | Establish trustworthy source contracts, keys, cardinalities, targets, cohort rules, and prediction-time boundaries | Schema inventory, relationship checks, data dictionary, quality report, target specification, feature-availability policy, and focused validation tests |
| 2. Data Preparation and Exploratory Analysis | Produce a deterministic one-row-per-order analytical dataset and understand distributions, missingness, exclusions, and temporal behavior | Reconciled aggregations and joins, eligible cohort, interim/processed datasets, EDA report, and chronological split assertions |
| 3. Feature Engineering and Baselines | Build reproducible purchase-time features and establish honest reference performance | Approved feature definitions, train-only preprocessing pipeline, leakage tests, classification baseline, regression baseline, and baseline evaluation report |
| 4. Classification and Regression Model Development | Compare suitable models, tune within temporal boundaries, and select evaluated candidates | Experiment records, calibrated classifier where justified, regression model, validation decisions, held-out test results, segment/error analysis, and model cards or equivalent documentation |
| 5. Production ML Engineering and API | Package training-consistent inference behind reliable contracts | Versioned artifacts, reusable predictor, FastAPI endpoints, schema validation, error handling, and unit/integration/API tests |
| 6. Deployment, Monitoring, and Documentation | Make the system reproducible, observable, and understandable to operators and reviewers | Docker image, CI/CD workflow, operational logs and monitoring reports, optional dashboard, runbooks, final README, limitations, and review evidence |

Each phase must satisfy its acceptance gate before later work is treated as complete. Work may be scaffolded only when required by the assigned phase; future-phase functionality should not be implemented opportunistically.

## 12. Documentation and governance standards

- Write in clear professional English for engineers, analysts, reviewers, and project stakeholders.
- Explain technical decisions together with their business or validity implications.
- Distinguish verified facts, design decisions, assumptions, observed results, and future work.
- Cite real source files, functions, commands, artifacts, metrics, and evidence; never fabricate execution or results.
- Keep target definitions, eligibility rules, feature availability, split rules, and inference contracts consistent across code and documentation.
- Record data version, cohort counts, exclusions, units, timestamp conventions, metric definitions, and model limitations.
- Prefer technology-neutral requirements until a phase makes an implementation choice necessary and records its rationale.
- Update this document deliberately when the authoritative project definition changes, and include that change in implementation and independent review scope.

## 13. Known limitations and future validation

The dataset reflects a particular historical marketplace and period, so results may not transfer to current operations, other countries, or different logistics networks. It does not provide a live feature feed, intervention history, controlled experiments, or proof that acting on a prediction improves a business outcome. Historical correlations may reflect operational processes that change over time.

Before any real deployment, owners must validate feature availability, data contracts, privacy and governance requirements, acceptable failure behavior, intervention capacity, threshold economics, monitoring ownership, retraining policy, and performance on current representative data. These are future operational decisions, not claims established by this project dataset.
