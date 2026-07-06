# Network Security — Phishing Website Detection

An end-to-end, production-style MLOps pipeline that classifies websites as **phishing** or **legitimate** from URL and domain-based features. The project covers the full lifecycle — data ingestion from MongoDB, automated validation and drift detection, feature transformation, multi-model training with hyperparameter tuning, experiment tracking, and a containerized REST API — deployed to AWS via a GitHub Actions CI/CD pipeline.

## Objectives

- Build a reliable, reproducible pipeline for detecting phishing websites from lexical and host-based URL features, rather than a one-off notebook model.
- Treat the ML workflow as a software system: modular components, typed configs/artifacts, centralized logging and exception handling, and automated tests-in-CI hooks.
- Track experiments and model versions so any run is auditable and reproducible.
- Ship the trained model behind a real API and automate build → push → deploy so a new model can go from training to production with a single `git push`.

## Problem Statement

Phishing sites imitate legitimate ones to steal credentials and financial information. Rather than relying on blocklists (which fail against unseen domains), this project trains a **supervised binary classifier** on 30 structural and behavioral features extracted from a URL/page (e.g. IP-address usage, URL length, sub-domain count, SSL state, domain age, DNS record presence, web traffic rank) to flag phishing attempts, including on domains not seen during training.

## Dataset

- **Source**: [UCI Phishing Websites dataset](Network_Data/phisingData.csv), staged in MongoDB (`DheerajChoudhary.NetworkData`) as the system of record.
- **Size**: 11,055 labeled records, 30 features + binary target (`Result`).
- **Features**: URL-based (`having_IP_Address`, `URL_Length`, `Shortining_Service`, `having_At_Symbol`, `Prefix_Suffix`, `having_Sub_Domain` …), domain-based (`SSLfinal_State`, `Domain_registeration_length`, `age_of_domain`, `DNSRecord`), and traffic/reputation-based (`web_traffic`, `Page_Rank`, `Google_Index`, `Links_pointing_to_page`, `Statistical_report`).
- Feature contract enforced via [data_schema/schema.yaml](data_schema/schema.yaml) — the pipeline fails fast if incoming data doesn't match the expected 30-column schema.

## Architecture

```
MongoDB Atlas
     │
     ▼
┌─────────────────┐    ┌──────────────────┐    ┌────────────────────┐    ┌───────────────────┐
│ Data Ingestion   │───▶│ Data Validation  │───▶│ Data Transformation│───▶│  Model Training    │
│ (pull, split)    │    │ (schema + drift) │    │ (KNN imputation)   │    │ (GridSearchCV x5)  │
└─────────────────┘    └──────────────────┘    └────────────────────┘    └─────────┬──────────┘
                                                                                     │
                                                                     MLflow / DagsHub tracking
                                                                                     │
                                                                                     ▼
                                                                         final_model/*.pkl
                                                                                     │
                                                                                     ▼
                                                                        FastAPI (/train, /predict)
                                                                                     │
                                                                                     ▼
                                                              Docker → Amazon ECR → EC2 (self-hosted runner)
```

Each stage is a self-contained component (`networksecurity/components/`) wired together by typed **config** and **artifact** dataclasses (`networksecurity/entity/`), so every stage's inputs/outputs are explicit and independently testable.

## Pipeline Stages

1. **Data Ingestion** ([data_ingestion.py](networksecurity/components/data_ingestion.py)) — pulls records from MongoDB, exports a raw feature store CSV, and performs an 80/20 train/test split.
2. **Data Validation** ([data_validation.py](networksecurity/components/data_validation.py)) — verifies column count/schema against `schema.yaml` and runs a **Kolmogorov–Smirnov test** per column between train and test splits to detect distribution drift, writing a YAML drift report.
3. **Data Transformation** ([data_transformation.py](networksecurity/components/data_transformation.py)) — imputes missing values with a `KNNImputer` (k=3) wrapped in a scikit-learn `Pipeline`, persists the fitted preprocessor, and serializes transformed arrays to `.npy`.
4. **Model Training** ([model_trainer.py](networksecurity/components/model_trainer.py)) — trains and tunes five candidate classifiers, selects the best performer, wraps it with the preprocessor into a single deployable `NetworkModel`, and logs metrics/artifacts to MLflow (via DagsHub).
5. **Serving** ([app.py](app.py)) — FastAPI app exposing `/train` (triggers the pipeline on demand) and `/predict` (batch CSV upload → predictions rendered as an HTML table).

## Modeling

Five candidate models are trained and hyperparameter-tuned with `GridSearchCV` (3-fold CV) over the transformed feature set:

| Model | Tuned hyperparameters |
|---|---|
| Random Forest | `n_estimators` |
| Gradient Boosting | `learning_rate`, `subsample`, `n_estimators` |
| Decision Tree | `criterion` |
| AdaBoost | `learning_rate`, `n_estimators` |
| Logistic Regression | baseline (no tuning) |

The best model is chosen by held-out test score, then repackaged with its fitted preprocessor into a single `NetworkModel` object ([estimator.py](networksecurity/utils/ml_utils/model/estimator.py)) so inference is a single `predict(df)` call — no separate preprocessing step for consumers of the model.

## Results

Trained on an 8,844 / 2,211 (80/20) train/test split, with each model's hyperparameters selected via 3-fold `GridSearchCV`:

| Model | Accuracy | F1 | Precision | Recall |
|---|---|---|---|---|
| **Random Forest** | **97.51%** | **0.9780** | **0.9683** | **0.9879** |
| Decision Tree | 96.56% | 0.9695 | 0.9633 | 0.9757 |
| Gradient Boosting | 95.61% | 0.9612 | 0.9509 | 0.9717 |
| Logistic Regression | 92.04% | 0.9298 | 0.9167 | 0.9434 |
| AdaBoost | 91.54% | 0.9263 | 0.9032 | 0.9506 |

**Best model: Random Forest** (`n_estimators=128`, Gini criterion) — **97.5% test accuracy**, **F1 = 0.978**, correctly flagging **98.8% of actual phishing sites** (recall) while keeping false positives low (96.8% precision). Train/test F1 gap (0.992 → 0.978) is small, indicating limited overfitting.

All runs (params, metrics, and the serialized model) are versioned in [MLflow via DagsHub](https://dagshub.com/dheerajchoudhary2002/networksecurity.mlflow).

## Experiment Tracking & Reproducibility

- Every training run logs F1, precision, and recall for both train and test splits, plus the fitted model artifact, to **MLflow** backed by **DagsHub** — giving a full history of experiments without running local tracking infrastructure.
- Data drift between train/test is checked automatically on every run and recorded to `drift_report/report.yaml`; the pipeline can be gated on this in the future to block training on drifted data.
- Config/artifact dataclasses (`config_entity.py`, `artifact_entity.py`) timestamp every pipeline run into its own `Artifacts/<timestamp>/` directory, so historical runs are never overwritten.

## Serving & API

Built with **FastAPI**:

- `GET /train` — runs the full training pipeline on demand and persists the new model to `final_model/`.
- `GET /predict` — upload form for batch scoring.
- `POST /predict` — accepts a CSV upload, runs it through the saved preprocessor + model, and returns predictions rendered as an HTML table (also saved to `prediction_output/output.csv`).
- Interactive API docs auto-generated at `/docs` (Swagger UI).

## Deployment & CI/CD

- **Containerization**: [Dockerfile](Dockerfile) packages the FastAPI app, AWS CLI, and all dependencies into a slim Python 3.10 image.
- **CI/CD**: [.github/workflows/main.yml](.github/workflows/main.yml) — on every push to `main`:
  1. **Continuous Integration** — checkout and lint/test hooks.
  2. **Continuous Delivery** — builds the Docker image and pushes it to **Amazon ECR**.
  3. **Continuous Deployment** — a self-hosted runner pulls the latest image from ECR and redeploys the container on an EC2 instance, with old images pruned automatically.
- **Artifact sync**: an [S3Sync](networksecurity/cloud/s3_syncer.py) utility pushes/pulls pipeline artifacts and models to/from S3 for backup and cross-environment reuse.

## Tech Stack

**ML/Data**: scikit-learn, pandas, NumPy, SciPy · **Tracking**: MLflow, DagsHub · **Data store**: MongoDB · **API**: FastAPI, Uvicorn · **Infra**: Docker, AWS (ECR, EC2, S3), GitHub Actions

## Project Structure

```
networksecurity/
├── components/        # Pipeline stages: ingestion, validation, transformation, training
├── entity/             # Config & artifact dataclasses shared across stages
├── pipeline/           # Orchestration (training_pipeline.py)
├── utils/
│   ├── main_utils/     # YAML/pickle/numpy I/O, GridSearchCV model evaluation
│   └── ml_utils/        # Classification metrics, deployable NetworkModel wrapper
├── cloud/              # S3 sync utility
├── exception/          # Centralized custom exception with file/line traceback
└── logging/            # Centralized logging
data_schema/schema.yaml  # Expected feature contract
final_model/              # Serialized preprocessor + model used by the API
app.py                    # FastAPI app (train/predict endpoints)
main.py                   # CLI entry point to run the full training pipeline
push_data.py              # One-off script to load the source CSV into MongoDB
Dockerfile
.github/workflows/main.yml # CI/CD pipeline
```

## Reproducing Results

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Configure environment
echo "MONGO_DB_URL=<your-mongodb-connection-string>" > .env

# 3. Load the dataset into MongoDB (one-time)
python push_data.py

# 4. Run the full training pipeline
python main.py

# 5. Serve the trained model
python app.py   # Swagger UI at http://localhost:8000/docs
```

## Author

**Dheeraj Choudhary** — [dheerajchoudhary2002@gmail.com](mailto:dheerajchoudhary2002@gmail.com)
