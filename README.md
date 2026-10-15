# E-Commerce Delivery Intelligence

An ML project for predicting late deliveries and delivery duration from information available when an order is placed.

The system addresses two purchase-time prediction tasks: estimating the probability that an order will arrive late and estimating its delivery duration in days. The authoritative business problem, ML definitions, scope, constraints, roadmap, and success criteria are documented in [Project Problem Definition](docs/PROJECT_PROBLEM.md).

## Dataset

This project uses the [Olist Brazilian E-Commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce?resource=download) from Kaggle.

Download and extract the dataset files into `data/raw/`. Treat these files as immutable source inputs: do not edit them in place or commit them to Git. Store cleaned or aggregated outputs in `data/interim/`, and place final model-ready datasets in `data/processed/`.
