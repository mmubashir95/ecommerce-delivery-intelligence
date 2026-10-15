# Olist Data Dictionary — E-Commerce Delivery Intelligence

## 1. Purpose, scope, and evidence

This document is the Phase 1 data dictionary for the E-Commerce Delivery Intelligence project. It describes the actual CSV files inspected in `data/raw/` and defines their schema, grain, keys, relationships, data-quality findings, prediction-time eligibility, target sources, and proposed order-level modeling contract.

The business and ML definitions in [PROJECT_PROBLEM.md](PROJECT_PROBLEM.md) remain authoritative. This dictionary adds dataset evidence; it does not implement data preparation, features, or models.

### Inspection basis

- **Inspected source:** nine CSV files present in `data/raw/` on 2026-10-15.
- **Method:** read-only full-file scans using Python 3 standard-library `csv`, `datetime`, counters, sets, and compact row hashes. Pandas was not installed and no package was installed.
- **Dtype convention:** “Inferred dtype” is the Pandas-compatible dtype expected from lexical inspection, not the output of `pandas.read_csv`. Timestamps remain `object` until deliberately parsed. Nullable numeric columns are shown as `float64`, which is the conventional inferred dtype when nulls are present.
- **Null convention:** an empty CSV field is counted as null. Literal category values such as `not_defined` are not counted as null.
- **Duplicate-row convention:** repeated complete parsed records were counted with 128-bit row digests. The observed collision risk is negligible, but this is not a byte-for-byte forensic duplicate test.
- **Encoding:** all files decoded with UTF-8. `product_category_name_translation.csv` begins with a UTF-8 byte-order mark (BOM), so readers should use `utf-8-sig` or remove the BOM from the first header. CSV quoting and newline handling should be delegated to a CSV parser.

No source file was modified. No replacement data was downloaded, and no modeling pipeline was run.

## 2. Dataset overview

**Dataset:** Brazilian E-Commerce Public Dataset by Olist

**Source:** [Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

**Purpose in this project:** support purchase-time prediction of late delivery and actual delivery duration.

All nine expected source files were present as uncompressed CSV files.

| File | Size (bytes; approximate) | Rows | Columns | Complete duplicate rows | Observed grain |
|---|---:|---:|---:|---:|---|
| `olist_orders_dataset.csv` | 17,654,914; 16.8 MiB | 99,441 | 8 | 0 | One row per order |
| `olist_customers_dataset.csv` | 9,033,957; 8.6 MiB | 99,441 | 5 | 0 | One row per order-specific customer record |
| `olist_order_items_dataset.csv` | 15,438,671; 14.7 MiB | 112,650 | 7 | 0 | One row per `(order_id, order_item_id)` |
| `olist_products_dataset.csv` | 2,379,446; 2.3 MiB | 32,951 | 9 | 0 | One row per product |
| `olist_sellers_dataset.csv` | 174,703; 170.6 KiB | 3,095 | 4 | 0 | One row per seller |
| `olist_order_payments_dataset.csv` | 5,777,138; 5.5 MiB | 103,886 | 5 | 0 | One row per payment sequence within an order |
| `olist_order_reviews_dataset.csv` | 14,451,670; 13.8 MiB | 99,224 | 7 | 0 | One row per review record; order and review IDs are not individually unique |
| `olist_geolocation_dataset.csv` | 61,273,883; 58.4 MiB | 1,000,163 | 5 | 261,831 | One raw coordinate/locality observation per row; ZIP prefix repeats |
| `product_category_name_translation.csv` | 2,613; 2.6 KiB | 71 | 2 | 0 | One row per translated Portuguese category |

### Time coverage

| Timestamp | Minimum observed | Maximum observed | Invalid non-null values |
|---|---|---|---:|
| `order_purchase_timestamp` | 2016-09-04 21:15:19 | 2018-10-17 17:30:18 | 0 |
| `order_approved_at` | 2016-09-15 12:16:38 | 2018-09-03 17:40:06 | 0 |
| `order_delivered_carrier_date` | 2016-10-08 10:34:01 | 2018-09-11 19:48:28 | 0 |
| `order_delivered_customer_date` | 2016-10-11 13:46:32 | 2018-10-17 13:22:46 | 0 |
| `order_estimated_delivery_date` | 2016-09-30 00:00:00 | 2018-11-12 00:00:00 | 0 |
| `shipping_limit_date` | 2016-09-19 00:15:34 | 2020-04-09 22:35:08 | 0 |
| `review_creation_date` | 2016-10-02 00:00:00 | 2018-08-31 00:00:00 | 0 |
| `review_answer_timestamp` | 2016-10-07 18:32:28 | 2018-10-29 12:27:35 | 0 |

The purchase coverage is not the same as the outcome or estimated-date coverage. Chronological modeling splits must use purchase time and report the exact cohort-specific ranges.

### Dataset limitations

- The data is a historical marketplace snapshot, not a live production feed.
- Timestamps contain no explicit timezone offset; their timezone semantics are unverified.
- The data shows completed outcomes but does not establish what every operational field looked like at the exact purchase-time inference boundary.
- The product table contains metadata and length counts, not product titles or full descriptions.
- Geolocation is a noisy repeated reference table rather than one canonical coordinate per ZIP prefix.
- Historical correlations cannot establish causal operational or customer outcomes.

## 3. Field classification legend

| Availability label | Meaning |
|---|---|
| **Purchase candidate** | Plausibly known at checkout, but still requires an approved serving contract and transformation definition |
| **Post-purchase — exclude** | Created or finalized after order placement; using it as a model feature would leak future information |
| **Historical only** | May support a point-in-time feature only from strictly earlier, already-observed records |
| **Identifier/join only** | Used for lineage, joins, grouping, or integrity checks; not a direct model feature |
| **Target only** | Used to construct or validate a label; prohibited as an input feature |
| **Static reference candidate** | Reference data that may be joined after duplicate-safe validation |
| **Undetermined** | Availability or operational meaning must be validated before feature approval |

Examples below are representative values observed in the files. Identifier examples are opaque public-dataset identifiers.

## 4. Complete column-level dictionary

### 4.1 `olist_orders_dataset.csv`

**Observed grain:** 99,441 rows and 99,441 unique `order_id` values. Both `order_id` and `customer_id` are unique and non-null in this table.

| Column | Inferred dtype / semantic type | Meaning | Example | Nulls (%) | Unique | Role | Availability and validation |
|---|---|---|---|---:|---:|---|---|
| `order_id` | `object` / opaque identifier | Marketplace order identifier | `e481f51cbdc54678b7cc49136f2d6af7` | 0 (0.0000%) | 99,441 | Primary key | Identifier/join only; must remain unique and non-null |
| `customer_id` | `object` / opaque identifier | Order-specific customer record referenced by this order | `9ef432eb6251297304e76186b10a928d` | 0 (0.0000%) | 99,441 | Foreign key to customers | Identifier/join only; do not substitute `customer_unique_id` in this join |
| `order_status` | `object` / categorical | Recorded lifecycle status | `delivered` | 0 (0.0000%) | 8 | Outcome/status | Post-purchase — exclude; validate against observed status set |
| `order_purchase_timestamp` | `object` / datetime | Order-placement event and prediction timestamp | `2017-10-02 10:56:33` | 0 (0.0000%) | 98,875 | Timestamp | Purchase candidate; chronological split anchor and regression target source |
| `order_approved_at` | `object` / datetime | Payment/order approval event | `2017-10-02 11:07:15` | 160 (0.1609%) | 90,733 | Timestamp | Post-purchase — exclude; never assume approval exists at placement |
| `order_delivered_carrier_date` | `object` / datetime | Handover or delivery to the carrier | `2017-10-04 19:55:00` | 1,783 (1.7930%) | 81,018 | Timestamp | Post-purchase — exclude; leakage if used as a feature |
| `order_delivered_customer_date` | `object` / datetime | Actual customer-delivery timestamp | `2017-10-10 21:25:13` | 2,965 (2.9817%) | 95,664 | Target source | Target only; required by both targets and never an input feature |
| `order_estimated_delivery_date` | `object` / datetime-like promise | Promised/estimated delivery value stored at midnight | `2017-10-18 00:00:00` | 0 (0.0000%) | 459 | Promise and target source | Purchase candidate and classification target source; validate that the same promise is available to serving |

Observed statuses: `delivered` 96,478; `shipped` 1,107; `canceled` 625; `unavailable` 609; `invoiced` 314; `processing` 301; `created` 5; `approved` 2.

### 4.2 `olist_customers_dataset.csv`

**Observed grain:** one row per `customer_id`. There are 99,441 `customer_id` values but 96,096 `customer_unique_id` values; 2,997 unique-customer identities occur in more than one customer record, with a maximum of 17 records.

| Column | Inferred dtype / semantic type | Meaning | Example | Nulls (%) | Unique | Role | Availability and validation |
|---|---|---|---|---:|---:|---|---|
| `customer_id` | `object` / opaque identifier | Order-level customer key used by orders | `06b8999e2fba1a1fbc88172c00ba8bc7` | 0 (0.0000%) | 99,441 | Primary key | Identifier/join only; exact parent of `orders.customer_id` |
| `customer_unique_id` | `object` / cross-order entity identifier | Stable identity that can link customer records across orders | `861eff4711a542e4b93843c6dd7febb0` | 0 (0.0000%) | 96,096 | Entity identifier | Historical only; prior-customer features need point-in-time logic and cold-start handling |
| `customer_zip_code_prefix` | `int64` lexically / postal-code prefix | Customer ZIP prefix | `14409` | 0 (0.0000%) | 14,994 | Geographic join key | Purchase candidate; treat as a string/code to preserve leading zeros |
| `customer_city` | `object` / categorical text | Customer city name | `franca` | 0 (0.0000%) | 4,119 | Categorical | Purchase candidate; normalize cautiously because spelling/diacritics may vary |
| `customer_state` | `object` / two-letter category | Customer state code | `SP` | 0 (0.0000%) | 27 | Categorical | Purchase candidate; validate accepted Brazilian state codes |

`customer_id` is not the long-lived customer identity: it is unique per order-linked customer record in this extract. `customer_unique_id` links repeat buyers. Orders therefore join directly through `orders.customer_id = customers.customer_id`; joining orders on `customer_unique_id` would be structurally wrong and can create duplicates.

### 4.3 `olist_order_items_dataset.csv`

**Observed grain:** one row per `(order_id, order_item_id)`. The composite key has zero duplicates. The file covers 98,666 orders, 32,951 products, and 3,095 sellers.

| Column | Inferred dtype / semantic type | Meaning | Example | Nulls (%) | Unique | Role | Availability and validation |
|---|---|---|---|---:|---:|---|---|
| `order_id` | `object` / opaque identifier | Parent order | `00010242fe8c5a6d1ba2dd792cb16214` | 0 (0.0000%) | 98,666 | Foreign key | Identifier/join only; valid many-to-one relation to orders |
| `order_item_id` | `int64` / within-order sequence | Item-row sequence within an order | `1` | 0 (0.0000%) | 21 | Composite-key component | Identifier/join only; range 1–21 and unique only with `order_id` |
| `product_id` | `object` / opaque identifier | Purchased product | `4244733e06e7ecb4970a6e2683c13e61` | 0 (0.0000%) | 32,951 | Foreign key | Identifier/join only; joins products before order aggregation |
| `seller_id` | `object` / opaque identifier | Seller fulfilling this item | `48436dade18ac8b2bce089ec2a041202` | 0 (0.0000%) | 3,095 | Foreign key | Identifier/join only; joins sellers before order aggregation |
| `shipping_limit_date` | `object` / datetime | Seller shipping deadline recorded for the item | `2017-09-19 09:45:35` | 0 (0.0000%) | 93,318 | Timestamp | Undetermined; exclude until purchase-time availability and semantics are proven |
| `price` | `float64` / currency amount | Item price in dataset currency | `58.90` | 0 (0.0000%) | 5,968 | Numeric candidate | Purchase candidate; range 0.85–6,735.00 and no negatives observed |
| `freight_value` | `float64` / currency amount | Item-row freight charge | `13.29` | 0 (0.0000%) | 6,999 | Numeric candidate | Purchase candidate; range 0–409.68, with 383 zero values and no negatives |

The table does not contain `customer_id`. The valid route is `order_items.order_id → orders.order_id → orders.customer_id → customers.customer_id`. Joining items directly to customers is impossible without orders and should not be approximated.

### 4.4 `olist_products_dataset.csv`

**Observed grain:** one row per `product_id`; all 32,951 products appear in order items.

The source column names intentionally retain the dataset misspelling `lenght`. Code and contracts must use the exact source spelling or rename it explicitly in a controlled preparation step.

| Column | Inferred dtype / semantic type | Meaning | Example | Nulls (%) | Unique | Role | Availability and validation |
|---|---|---|---|---:|---:|---|---|
| `product_id` | `object` / opaque identifier | Product key | `1e9e8ef04dbcff4541ed26657ea517e5` | 0 (0.0000%) | 32,951 | Primary key | Identifier/join only |
| `product_category_name` | `object` / categorical text | Portuguese category code/name | `perfumaria` | 610 (1.8512%) | 73 | Categorical candidate / translation FK | Purchase candidate; missing category needs explicit handling |
| `product_name_lenght` | `float64` / character-count measure | Length of the unavailable product-name text | `40` | 610 (1.8512%) | 66 | Numeric candidate | Purchase candidate if serving can reproduce it; range 5–76 |
| `product_description_lenght` | `float64` / character-count measure | Length of the unavailable product description | `287` | 610 (1.8512%) | 2,960 | Numeric candidate | Purchase candidate if serving can reproduce it; range 4–3,992 |
| `product_photos_qty` | `float64` / count | Number of listed product photos | `1` | 610 (1.8512%) | 19 | Numeric candidate | Purchase candidate; observed range 1–20 |
| `product_weight_g` | `float64` / grams | Product weight | `225` | 2 (0.0061%) | 2,204 | Numeric candidate | Purchase candidate; range 0–40,425 and four non-null zero weights require handling |
| `product_length_cm` | `float64` / centimeters | Product length | `16` | 2 (0.0061%) | 99 | Numeric candidate | Purchase candidate; observed positive range 7–105 |
| `product_height_cm` | `float64` / centimeters | Product height | `10` | 2 (0.0061%) | 102 | Numeric candidate | Purchase candidate; observed positive range 2–105 |
| `product_width_cm` | `float64` / centimeters | Product width | `14` | 2 (0.0061%) | 95 | Numeric candidate | Purchase candidate; observed positive range 6–118 |

This table does not contain readable product names or product descriptions. It contains only their length measures. Category translation cannot reconstruct missing names, descriptions, or titles.

### 4.5 `olist_sellers_dataset.csv`

**Observed grain:** one row per `seller_id`; all 3,095 sellers appear in order items.

| Column | Inferred dtype / semantic type | Meaning | Example | Nulls (%) | Unique | Role | Availability and validation |
|---|---|---|---|---:|---:|---|---|
| `seller_id` | `object` / opaque identifier | Seller key | `3442f8959a84dea7ee197c632cb2df15` | 0 (0.0000%) | 3,095 | Primary key | Identifier/join only; raw identity should not be treated as a stable generalizable numeric signal |
| `seller_zip_code_prefix` | `int64` lexically / postal-code prefix | Seller ZIP prefix | `13023` | 0 (0.0000%) | 2,246 | Geographic join key | Purchase candidate; treat as string/code to preserve leading zeros |
| `seller_city` | `object` / categorical text | Seller city | `campinas` | 0 (0.0000%) | 611 | Categorical candidate | Purchase candidate; normalize cautiously |
| `seller_state` | `object` / two-letter category | Seller state | `SP` | 0 (0.0000%) | 23 | Categorical candidate | Purchase candidate; validate accepted values |

Historical seller performance is not stored as a ready-to-use field. It must be computed point in time from strictly earlier outcomes that were already observable before the current purchase.

### 4.6 `olist_order_payments_dataset.csv`

**Observed grain:** one row per `(order_id, payment_sequential)`. The composite key has zero duplicates. There are 103,886 records covering 99,440 orders.

| Column | Inferred dtype / semantic type | Meaning | Example | Nulls (%) | Unique | Role | Availability and validation |
|---|---|---|---|---:|---:|---|---|
| `order_id` | `object` / opaque identifier | Parent order | `b81ef226f3fe1789b1e8b2acac839d17` | 0 (0.0000%) | 99,440 | Foreign key | Identifier/join only |
| `payment_sequential` | `int64` / within-order sequence | Sequence number of a payment record | `1` | 0 (0.0000%) | 29 | Composite-key component | Identifier/join only; range 1–29 |
| `payment_type` | `object` / categorical | Payment method category | `credit_card` | 0 (0.0000%) | 5 | Categorical candidate | Undetermined; must be finalized and reproducible at the prediction boundary |
| `payment_installments` | `int64` / count | Installment count | `8` | 0 (0.0000%) | 24 | Numeric candidate | Undetermined; range 0–24, including two zero-installment records |
| `payment_value` | `float64` / currency amount | Amount represented by the payment row | `99.33` | 0 (0.0000%) | 29,077 | Numeric candidate | Undetermined; range 0–13,664.08, including nine zero-value records |

Observed payment types: `credit_card` 76,795; `boleto` 19,784; `voucher` 5,775; `debit_card` 1,529; `not_defined` 3. Payment fields must not be approved merely because they appear in the historical extract; the inference contract must establish whether they are known and final at order placement.

### 4.7 `olist_order_reviews_dataset.csv`

**Observed grain:** one review record per row. Neither `review_id` nor `order_id` is a primary key: 98,410 distinct review IDs appear in 99,224 rows, and 98,673 distinct orders are represented. There are 547 orders with multiple review rows, with a maximum of three. There are 789 review IDs that appear more than once.

| Column | Inferred dtype / semantic type | Meaning | Example | Nulls (%) | Unique | Role | Availability and validation |
|---|---|---|---|---:|---:|---|---|
| `review_id` | `object` / opaque identifier | Review identifier | `7bc2406110b926393aa56f80a40eba40` | 0 (0.0000%) | 98,410 | Non-unique identifier | Identifier/join only; do not enforce individual uniqueness without a documented composite grain |
| `order_id` | `object` / opaque identifier | Reviewed order | `73fc7af87114b39712e6da79b0a377eb` | 0 (0.0000%) | 98,673 | Foreign key | Identifier/join only; valid many-to-one relationship to orders |
| `review_score` | `int64` / ordinal category | Customer rating from 1 to 5 | `4` | 0 (0.0000%) | 5 | Outcome | Post-purchase — exclude |
| `review_comment_title` | `object` / free text | Optional review title | `recomendo` | 87,656 (88.3415%) | 4,527 | Outcome text | Post-purchase — exclude; highly sparse |
| `review_comment_message` | `object` / free text | Optional review message | `Recebi bem antes do prazo estipulado.` | 58,247 (58.7025%) | 36,159 | Outcome text | Post-purchase — exclude; can directly reveal delivery outcome |
| `review_creation_date` | `object` / datetime | Review creation date/time value | `2018-01-18 00:00:00` | 0 (0.0000%) | 636 | Timestamp | Post-purchase — exclude |
| `review_answer_timestamp` | `object` / datetime | Review response/answer timestamp | `2018-01-18 21:46:59` | 0 (0.0000%) | 98,248 | Timestamp | Post-purchase — exclude |

Review scores are distributed as follows: score 1: 11,424; 2: 3,151; 3: 8,179; 4: 19,142; 5: 57,328. These are retrospective outcomes and must not become purchase-time features or historical aggregates unless strict outcome-availability timing is enforced.

### 4.8 `olist_geolocation_dataset.csv`

**Observed grain:** raw geographic observation, not ZIP-prefix dimension. There are 1,000,163 rows but only 19,015 distinct ZIP prefixes. Of those prefixes, 17,972 have multiple rows, and one prefix has 1,146 rows.

| Column | Inferred dtype / semantic type | Meaning | Example | Nulls (%) | Unique | Role | Availability and validation |
|---|---|---|---|---:|---:|---|---|
| `geolocation_zip_code_prefix` | `int64` lexically / postal-code prefix | ZIP-prefix lookup key | `01037` | 0 (0.0000%) | 19,015 | Non-unique geographic key | Static reference candidate; parse as string/code to retain leading zero |
| `geolocation_lat` | `float64` / decimal degrees | Latitude observation | `-23.54562128115268` | 0 (0.0000%) | 717,372 | Numeric reference | Static reference candidate after aggregation; range −36.6054 to 45.0659 |
| `geolocation_lng` | `float64` / decimal degrees | Longitude observation | `-46.63929204800168` | 0 (0.0000%) | 717,615 | Numeric reference | Static reference candidate after aggregation; range −101.4668 to 121.1054 |
| `geolocation_city` | `object` / categorical text | City attached to the observation | `sao paulo` | 0 (0.0000%) | 8,011 | Geographic category | Static reference candidate after normalization and aggregation |
| `geolocation_state` | `object` / two-letter category | State attached to the observation | `SP` | 0 (0.0000%) | 27 | Geographic category | Static reference candidate; validate ZIP consistency |

All coordinates fall within global latitude/longitude validity ranges, but 31 rows fall outside a rough Brazil bounding box of latitude −34 to 6 and longitude −74 to −32. This is a screening rule, not proof that each row is wrong. Eight ZIP prefixes span multiple state codes and 8,556 span multiple city strings, which may reflect boundary issues, spelling variants, locality aliases, or noisy records.

A direct customer-to-geolocation join on ZIP prefix would expand 99,441 customer rows to 15,083,455 rows before accounting for 278 customer records whose 157 distinct ZIP prefixes have no geolocation match. A direct seller join would expand 3,095 sellers to 435,087 rows, with seven unmatched seller ZIP prefixes. The geolocation table must therefore be aggregated first—for example, to one robust median coordinate and a documented canonical locality per ZIP prefix. State-aware grouping, trimming outliers, coordinate medians, and explicit unmatched indicators are candidate strategies. The selected rule must be tested and versioned.

### 4.9 `product_category_name_translation.csv`

**Observed grain:** one row per Portuguese category name. Both fields are unique and non-null. The file begins with a UTF-8 BOM.

| Column | Inferred dtype / semantic type | Meaning | Example | Nulls (%) | Unique | Role | Availability and validation |
|---|---|---|---|---:|---:|---|---|
| `product_category_name` | `object` / categorical text | Portuguese category key | `beleza_saude` | 0 (0.0000%) | 71 | Primary key / product FK target | Static reference candidate; strip BOM from header during parsing |
| `product_category_name_english` | `object` / categorical text | English category translation | `health_beauty` | 0 (0.0000%) | 71 | Reference attribute | Static reference candidate |

Two observed product categories lack translations: `pc_gamer` and `portateis_cozinha_e_preparadores_de_alimentos`. Translation provides category labels only; it does not supply product titles or descriptions.

## 5. Keys, cardinalities, and referential integrity

| Relationship | Expected cardinality | Observed evidence | Unmatched/problem records | Join guidance |
|---|---|---|---|---|
| Orders `customer_id` → Customers `customer_id` | Many orders conceptually to customers; one-to-one in this extract's order-specific key | 99,441 distinct keys on each side | 0 unmatched either direction | Join directly on `customer_id`; use `customer_unique_id` only for cross-order identity |
| Items `order_id` → Orders `order_id` | Many-to-one | 112,650 items across 98,666 orders | 0 orphan item orders; 775 orders have no items | Aggregate items before joining to order grain; missing items occur mainly in unavailable/canceled orders |
| Items `product_id` → Products `product_id` | Many-to-one | 32,951 item product keys and 32,951 product keys | 0 unmatched; 0 unreferenced products | Enrich items before order-level aggregation |
| Items `seller_id` → Sellers `seller_id` | Many-to-one | 3,095 seller keys on both sides | 0 unmatched; 0 unreferenced sellers | Enrich items before aggregation; preserve multi-seller counts |
| Payments `order_id` → Orders `order_id` | Many-to-one | 103,886 rows across 99,440 orders | 0 orphan payment orders; 1 delivered order has no payment row | Aggregate payment rows independently before joining |
| Reviews `order_id` → Orders `order_id` | Many-to-one | 99,224 rows across 98,673 orders | 0 orphan review orders; 768 orders have no review | Reviews are post-purchase and excluded from model inputs |
| Customer ZIP → Geolocation ZIP | Many-to-many if raw | 14,994 customer ZIP prefixes; 19,015 geolocation ZIP prefixes | 157 unmatched prefixes affecting 278 customers | Aggregate geolocation to one documented row per ZIP before joining |
| Seller ZIP → Geolocation ZIP | Many-to-many if raw | 2,246 seller ZIP prefixes | 7 unmatched prefixes affecting 7 sellers | Apply the same canonical geolocation contract |
| Products category → Translation category | Many-to-one | 73 non-null product categories; 71 translations | 2 untranslated category values | Left join and preserve original category/fallback label |

### Orders without child records

- The 775 orders without items comprise 603 `unavailable`, 164 `canceled`, five `created`, two `invoiced`, and one `shipped` order.
- The single order without payments is marked `delivered`; it requires investigation before financial reconciliation.
- The 768 orders without reviews include 646 delivered orders and 122 orders in other statuses. A missing review is not itself a data error because reviews are optional.

### One-to-many behavior

- 9,803 orders contain more than one item row; the maximum is 21.
- 3,236 orders contain more than one distinct product; the maximum is eight.
- 1,278 orders involve more than one seller; the maximum is five.
- 2,961 orders contain multiple payment rows; the maximum is 29.
- 547 orders contain multiple review rows; the maximum is three.

Joining raw items, payments, and reviews together on `order_id` would multiply rows. Each source must be aggregated independently to exactly one order row before the aggregates are joined.

## 6. Data-quality assessment

### 6.1 Findings

| Severity | Observed finding | Evidence | Modeling risk and required treatment |
|---|---|---|---|
| Critical | Raw geolocation is non-unique and highly duplicated | 1,000,163 rows, 19,015 ZIP prefixes, 261,831 repeated full rows; raw customer join would produce 15,083,455 rows | Direct joins corrupt grain and weights; deduplicate/aggregate geolocation first and test uniqueness |
| High | Estimated delivery values have date-like midnight granularity | All 99,441 values are `00:00:00`; 1,292 delivered orders arrived later on the same calendar date and are late under strict timestamp comparison | Target semantics materially affect labels; retain the authoritative strict rule unless governance deliberately changes it, and report sensitivity |
| High | Lifecycle sequences contain contradictions or nonstandard ordering | 1,359 carrier timestamps precede approval; 23 customer deliveries precede carrier timestamps | Do not silently repair; investigate source semantics and exclude or flag invalid sequences according to a documented cohort rule |
| High | Four item rows have shipping deadlines after 2018, including dates in 2020 | Two orders have one such row and one order has two; the maximum is 2020-04-09 although its purchase was in 2017 | Treat `shipping_limit_date` as undetermined and anomalous until its source semantics and correction policy are established |
| High | Reviews contain non-unique identifiers and multiple rows per order | 99,224 rows, 98,410 review IDs, 98,673 order IDs; 789 review IDs repeat; 547 orders have multiple reviews | Never treat `review_id` or `order_id` alone as a review-table primary key; reviews are excluded from features regardless |
| Medium | Some parent orders lack item or payment children | 775 orders have no items; one delivered order has no payment | Left joins and explicit missing-child flags are required; inner joins would silently drop orders |
| Medium | Product metadata has missing and invalid physical attributes | 610 missing category/name-length/description-length/photo fields; two rows missing every physical measurement; four non-null weights are zero | Define imputation/unknown-category policy using training data only; zero weight requires validation |
| Medium | Geography contains unmatched and suspicious records | 278 customer rows and seven seller rows have unmatched ZIP prefixes; 31 geolocation rows fall outside a rough Brazil bounding box; eight prefixes span states | Preserve unmatched flags, use robust coordinate aggregation, and avoid treating screening outliers as automatically invalid |
| Medium | Product translation is incomplete | Two observed categories have no English mapping | Preserve the Portuguese value and use a documented fallback rather than dropping rows |
| Low | Payment edge values require interpretation | Nine zero-value payments, two zero-installment rows, three `not_defined` payment types | Validate business meaning before aggregation or feature approval |
| Low | Optional review text is sparse | Title 88.3415% null; message 58.7025% null | Expected for optional reviews and irrelevant to purchase-time modeling; document rather than impute for this project |

### 6.2 Validity checks with no observed failures

- No complete duplicate rows were observed outside geolocation.
- Orders, customers, products, sellers, item composite keys, payment composite keys, and translation keys satisfy their proposed uniqueness rules.
- No orphan foreign keys were observed for orders-to-customers, items-to-orders/products/sellers, payments-to-orders, or reviews-to-orders.
- No malformed non-null order timestamps were found using `%Y-%m-%d %H:%M:%S`.
- No purchase-after-delivery, approval-before-purchase, or estimated-before-purchase record was observed.
- No negative item price, freight value, payment value, or installment count was observed.
- No nonpositive product length, height, or width was observed.
- No coordinate falls outside global latitude/longitude bounds.

These checks establish properties of this inspected snapshot only. They should become repeatable validation tests in the preparation pipeline rather than permanent assumptions.

## 7. Timestamp meanings and leakage classification

| Timestamp | Event | Relative to purchase | Feature eligibility | Target/evaluation use |
|---|---|---|---|---|
| `order_purchase_timestamp` | Order placed | Prediction boundary | Available; derive calendar features and chronological splits | Regression start time |
| `order_approved_at` | Approval recorded | Usually after purchase | Exclude; future information | Cohort quality checks only |
| `shipping_limit_date` | Item-level seller shipping deadline | Meaning/creation timing not proven | Undetermined; exclude until contract validated | Possible operational analysis only |
| `order_delivered_carrier_date` | Carrier handover | After purchase | Exclude; future information | Sequence/data-quality analysis only |
| `order_delivered_customer_date` | Actual customer delivery | After purchase | Exclude; direct outcome leakage | Required for both targets |
| `order_estimated_delivery_date` | Delivery promise/estimate | Intended to be known at purchase | Purchase candidate | Required for `is_late`; derive promised duration |
| `review_creation_date` | Review created | Post-delivery experience | Exclude | Retrospective analysis only |
| `review_answer_timestamp` | Review answer submitted | Post-purchase | Exclude | Retrospective analysis only |

The CSV timestamps contain no timezone offset. Parsing should initially preserve their common naive convention, while the data contract explicitly records that timezone is unknown. Converting only some fields to a timezone would create artificial differences.

## 8. Target definitions and observed eligibility

### 8.1 Classification target: `is_late`

For an eligible row:

```text
is_late = 1 if order_delivered_customer_date > order_estimated_delivery_date
          0 otherwise
```

Required source fields are `order_delivered_customer_date` and `order_estimated_delivery_date`. The comparison is strict and timestamp-based, consistent with `PROJECT_PROBLEM.md`; equality is on time. The actual delivery timestamp is label-only, while the estimated value may also support purchase-time promise features.

**Observed eligibility:** 96,476 orders have both parseable timestamps. Under the strict convention, 7,827 are late (8.1129%). Orders without an actual delivery must remain unlabeled rather than being coerced to on-time. Eight records marked `delivered` lack the actual delivery timestamp, while six records with other statuses contain one; status alone is therefore insufficient for label creation.

**Granularity issue:** every estimated delivery value is stored at `00:00:00`. Consequently, a delivery later on the estimated calendar date counts as late. This affects 1,292 observed eligible orders. The project authority currently chooses strict timestamp comparison. Reports must disclose this implication; changing to an end-of-day or calendar-date convention would be a governance change requiring an update to `PROJECT_PROBLEM.md` and a label-sensitivity comparison.

### 8.2 Regression target: `delivery_days`

```text
delivery_days = (
    order_delivered_customer_date - order_purchase_timestamp
).total_seconds() / 86,400
```

Required fields are `order_purchase_timestamp` and `order_delivered_customer_date`. The unit is elapsed fractional days; values must not be rounded during construction. The inspected snapshot contains 96,476 rows with both parseable fields and no observed negative delivery duration.

Orders missing actual delivery are ineligible for supervised regression labels. Any future negative or malformed duration must be quarantined or excluded with an explicit reason and count, never clipped silently. Both source timestamps and the resulting label are prohibited model inputs.

### 8.3 Cohort principle

Target eligibility is narrower than deployment eligibility. Canceled, unavailable, undelivered, missing-label, or timestamp-invalid rows may be excluded from supervised training, but their exclusion must be quantified and must not imply that the deployed system will never encounter such orders.

## 9. Purchase-time feature eligibility

| Source fields or concept | Classification | Rationale |
|---|---|---|
| Purchase timestamp and derived weekday/month/hour | Purchase candidate | Known at prediction time; derive consistently in training and serving |
| Estimated delivery date and promised duration | Purchase candidate | Intended purchase-time promise; validate serving availability |
| Customer/seller city, state, ZIP prefix | Purchase candidate | Checkout/static attributes; ZIP must be treated as a code |
| Geolocation coordinates | Static reference candidate | Aggregate to a unique ZIP lookup before joining |
| Item price, freight, product count, seller count | Purchase candidate | Likely checkout-known; aggregate without row multiplication |
| Product category, weight, dimensions, photo/name/description length metadata | Purchase candidate | Catalog attributes; validate snapshot and serving consistency |
| Payment type/value/installments | Undetermined | May be entered at checkout but finalization timing and serving availability are not proven |
| Order, customer, product, seller IDs | Identifier/join only | Useful for joins and grouping; raw IDs risk memorization and weak generalization |
| `customer_unique_id` | Historical only / identifier | Enables prior-customer history only with strict point-in-time logic |
| Historical seller/customer/route results | Historical only | Outcomes must have been known strictly before the current purchase |
| Order status, approval, carrier, actual delivery | Post-purchase — exclude | Direct or indirect future outcome information |
| Review score/text/timestamps | Post-purchase — exclude | Created after the order experience and can reveal delivery outcome |
| `is_late`, `delivery_days` | Target only | Labels, never input features |

No candidate becomes an approved model input until its lineage, purchase-time availability, aggregation, missing-value handling, and serving parity are documented and tested.

## 10. Candidate engineered features

These features are proposed for later phases; none is implemented or approved here.

| Candidate feature | Business meaning | Source and proposed calculation | Availability / missing treatment | Leakage risk |
|---|---|---|---|---|
| `total_item_value` | Merchandise value of the order | Sum item `price` by `order_id` | Purchase candidate; distinguish missing item rows from zero | Multiplication if joined to payments before aggregation |
| `total_freight_value` | Quoted freight burden | Sum item `freight_value` by order | Purchase candidate; preserve legitimate zero freight | Same join-multiplication risk |
| `item_count` | Number of purchased item rows | Count item rows by order | Purchase candidate; missing items need explicit flag | Do not count after one-to-many joins |
| `distinct_product_count` | Product variety | Count distinct `product_id` by order | Purchase candidate; zero only if truly no items | Duplicate item joins can inflate count without `distinct` |
| `distinct_seller_count` | Fulfillment complexity | Count distinct `seller_id` by order | Purchase candidate | Seller history must remain point in time |
| `total_product_weight_g` | Approximate shipment weight | Sum item-level product weight, respecting item quantity represented by rows | Purchase candidate; impute/flag two missing and four zero weights | Incorrect product/item grain can under- or over-count |
| `product_volume_cm3_*` | Shipment size profile | `length × height × width` per item, then sum/max/mean by order | Purchase candidate; missing dimensions need flags and train-only imputation | Unit assumptions and outliers require validation |
| `dominant_product_category` / category counts | Order mix | Translate if available, then deterministic order-level category aggregation | Purchase candidate; unknown fallback for 610 missing and two untranslated categories | Arbitrary tie-breaking or high cardinality |
| `customer_seller_distance_km_*` | Geographic delivery complexity | Haversine distance from canonical ZIP coordinates for each seller-item pair, then aggregate | Static/purchase candidate; unmatched ZIP flags required | Raw geolocation join explosion; noisy coordinates |
| `purchase_weekday` | Weekly order timing | Derive from purchase timestamp | Available; timezone semantics unverified | Low leakage risk if derived only from purchase |
| `purchase_month` | Seasonal timing | Derive from purchase timestamp | Available | Temporal drift and year effects need evaluation |
| `promised_delivery_days` | Length of customer-facing promise | `(estimated_delivery - purchase).total_seconds()/86400` | Purchase candidate; validate nonnegative and midnight convention | Must use the promise available for that order, not revised future values |
| `payment_*` aggregates | Checkout payment structure | Count methods, sum values, max installments after independent payment aggregation | Undetermined pending availability contract | Later payment finalization and row multiplication |
| `seller_prior_late_rate` | Seller's previously observed reliability | Prior late labels for seller, using only outcomes known before current purchase | Historical only; cold-start fallback required | Severe leakage if future/current outcomes enter the window |
| `seller_prior_delivery_days` | Seller's prior delivery-duration history | Point-in-time aggregate of already-delivered earlier orders | Historical only | Outcome availability time, not purchase order alone, governs inclusion |

## 11. Proposed future order-level modeling contract

This is a proposed schema, not an implemented dataset. Final feature approval belongs to later phase data and feature contracts.

**Grain:** exactly one row per `order_id`.

**Uniqueness:** `order_id` must be non-null and unique after every preparation step.

**Lineage:** each field must retain source columns, aggregation rule, availability classification, and transformation version.

| Proposed field/group | Type | Source lineage | Missing-value requirement |
|---|---|---|---|
| `order_id` | String identifier | Orders | Never null; retained for lineage, excluded from ordinary model inputs |
| `order_purchase_timestamp` | Datetime | Orders | Never null; split anchor and feature derivation source |
| `is_late` | Nullable integer/bool target | Actual delivery and estimated delivery | Null for ineligible/unobserved orders; never impute |
| `delivery_days` | Nullable float target | Actual delivery and purchase timestamps | Null for ineligible/unobserved orders; never impute or round |
| Purchase-calendar features | Numeric/categorical | Purchase timestamp | Deterministic derivation; document timezone convention |
| Promised-delivery duration | Float | Purchase and estimated timestamps | Flag missing/invalid promise; do not use revised outcomes |
| Item/value/freight counts and sums | Numeric | Order items aggregated independently | Missing-child indicator; do not silently equate missing with zero |
| Product physical aggregates | Numeric | Products joined to items, then aggregated | Training-only imputation plus explicit missing/invalid flags |
| Product/category aggregates | Categorical/numeric | Products and translation lookup | Stable unknown category; preserve untranslated source values |
| Seller-count and seller-geography aggregates | Numeric/categorical | Sellers joined to items | Handle multi-seller orders explicitly |
| Customer geography | Categorical/numeric | Customers and canonical geolocation | Preserve unmatched ZIP indicator |
| Distance aggregates | Numeric | Canonical customer/seller coordinates | Missing when either coordinate is unavailable; impute only in fitted preprocessing |
| Approved payment aggregates | Numeric/categorical | Payments aggregated independently | Include only after availability is validated |
| Approved historical features | Numeric | Point-in-time prior observable outcomes | Cold-start flag/default fitted without validation/test leakage |

### Required preparation invariants

1. Orders remain the left-side spine unless a documented eligibility filter is being applied.
2. Every child table is reduced to one row per order before it joins the spine.
3. Row counts and key uniqueness are asserted before and after joins.
4. Item, freight, and payment totals reconcile to their independently computed source aggregates.
5. Features exclude post-purchase timestamps, reviews, statuses, current outcomes, and future historical outcomes.
6. Preprocessing is fit only within the training boundary and reused unchanged for validation, test, and inference.

## 12. Recommended Phase 1 validation tests

The following stable rules are strong candidates for automated validation in the data pipeline:

- Required file and exact-column presence, including the source `lenght` spellings.
- Primary/composite-key uniqueness for orders, customers, products, sellers, item rows, payments, and category translations.
- Referential integrity for the relationships in Section 5.
- One-row-per-order assertion for every prepared modeling dataset.
- Accepted order-status and payment-type monitoring with explicit handling for new categories.
- Timestamp parsing and ordering checks, with known anomalies reported rather than silently rewritten.
- Nonnegative price, freight, payment, and dimension checks; positive item sequence values.
- Label eligibility and target-formula tests, including equality at the lateness boundary.
- Geolocation uniqueness after canonical aggregation and explicit unmatched ZIP rates.
- Adversarial join fixture with two items and two payments to prove totals are not multiplied.
- Future-outcome perturbation test proving purchase-time features do not change.

Observed counts should be monitored as snapshot evidence, not hard-coded forever as universal rules.

## 13. Decisions and questions requiring owner review

1. **Lateness deadline semantics:** retain the authoritative strict timestamp comparison despite midnight estimates, or formally change the project definition after reviewing the 1,292 same-calendar-day cases.
2. **Timestamp timezone:** confirm whether naive source timestamps should remain naive or be assigned a documented timezone.
3. **Eligibility cohort:** decide whether supervised training requires `order_status == delivered` in addition to valid timestamps, given eight delivered rows without actual timestamps and six non-delivered-status rows with them.
4. **Sequence anomalies:** decide whether carrier-before-approval and delivery-before-carrier cases are excluded, retained with flags, or treated under clarified source semantics.
5. **Payment availability:** confirm which payment fields, if any, are finalized at order placement and available to the inference API.
6. **Shipping deadline availability:** confirm whether `shipping_limit_date` is known at purchase and whether it represents a usable promise or a later operational field.
7. **Geolocation contract:** approve a canonical ZIP aggregation and outlier policy before distance features are built.
8. **Historical features:** define minimum history, cold-start behavior, and outcome-availability timing before seller or customer history is implemented.

Until these decisions are recorded, affected fields remain candidate, historical-only, or undetermined rather than approved model inputs.
