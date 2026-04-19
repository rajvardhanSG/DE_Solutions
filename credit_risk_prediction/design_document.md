# Solar Industry Credit Risk Prediction System
## Design Document v1.0

**Date:** 2026-04-19
**Status:** Accepted
**Authors:** AI-assisted design via structured brainstorming

---

## 1. Overview

An internal credit risk prediction tool for a solar energy company that installs panels in industrial
clients on its own investment. The system predicts whether a client (industrial company) can repay
the financing over time.

**Core user personas:**
- **Credit Analysts** — enter applicant data, receive risk score + explanations, make decisions
- **Senior Managers** — monitor portfolio health, approval trends, and credit risk exposure

---

## 2. System Context Diagram (C4 Level 1)

```mermaid
C4Context
    title System Context — Solar Credit Risk Tool

    Person(analyst, "Credit Analyst", "Enters client data, reviews risk score & SHAP explanations")
    Person(manager, "Senior Manager", "Monitors portfolio trends and approval statistics")

    System(app, "Credit Risk Prediction App", "Streamlit web app serving risk scores, verdicts, and portfolio analytics")

    System_Ext(data_source, "Internal Data Store", "Proprietary loan application records (CSV/Excel)")
    System_Ext(mlflow, "MLflow Server", "Experiment tracking & model registry (on-premise)")
    System_Ext(audit_db, "SQLite Audit Log", "Immutable log of all predictions")

    Rel(analyst, app, "Submits applicant data, views prediction")
    Rel(manager, app, "Views portfolio dashboards")
    Rel(app, mlflow, "Loads production model from registry")
    Rel(app, audit_db, "Writes prediction log entry")
    Rel(data_source, app, "Training data used offline to train model")
```

---

## 3. Container Diagram (C4 Level 2)

```mermaid
C4Container
    title Container Diagram — Solar Credit Risk System

    Person(analyst, "Credit Analyst")
    Person(manager, "Senior Manager")

    Container_Boundary(docker, "Docker Compose Stack (On-Premise Server)") {
        Container(streamlit, "Streamlit App", "Python / Streamlit", "Two-tab UI: Predict (analyst) + Dashboard (manager)")
        Container(mlflow_server, "MLflow Server", "Python / MLflow", "Stores experiment runs, model artifacts, and model registry")
        ContainerDb(sqlite, "SQLite", "SQLite3", "Prediction audit log: timestamp, inputs, score, analyst ID")
        ContainerDb(mlruns, "mlruns/ Volume", "Filesystem", "MLflow artifact store shared between containers")
    }

    Container_Boundary(offline, "Offline Training (Data Scientist Machine)") {
        Container(pipeline, "Training Pipeline", "Python / sklearn / XGBoost / LightGBM", "EDA → Preprocessing → Feature Engineering → Training → Evaluation")
        Container(optuna, "Optuna Tuner", "Python / Optuna", "Hyperparameter search logged to MLflow")
    }

    Rel(analyst, streamlit, "Opens in browser (intranet)", "HTTP")
    Rel(manager, streamlit, "Opens in browser (intranet)", "HTTP")
    Rel(streamlit, mlflow_server, "Loads model from registry", "REST / MlflowClient")
    Rel(streamlit, sqlite, "Writes audit log entry", "SQLite3 DB API")
    Rel(pipeline, mlflow_server, "Logs runs, registers model", "MLflow Tracking API")
    Rel(mlflow_server, mlruns, "Reads/writes artifacts")
    Rel(pipeline, mlruns, "Optuna logs via MLflowCallback")
```

---

## 4. Component Diagram (C4 Level 3) — Training Pipeline

```mermaid
C4Component
    title Component Diagram — Training Pipeline

    Container_Boundary(pipeline, "Training Pipeline (src/)") {
        Component(loader, "Data Loader", "src/data/loader.py", "Reads CSV/Excel, infers dtypes, validates schema")
        Component(preprocessor, "Preprocessor", "src/data/preprocessor.py", "sklearn Pipeline: imputation, encoding, scaling, SMOTE")
        Component(engineer, "Feature Engineer", "src/features/engineering.py", "Derived ratios, log transforms, binning")
        Component(selector, "Feature Selector", "src/features/selection.py", "VIF, correlation filter, mutual info, RFECV")
        Component(trainer, "Model Trainer", "src/models/train.py", "Cross-validates LR, RF, XGBoost, LightGBM; logs to MLflow")
        Component(tuner, "Optuna Tuner", "src/models/tune.py", "Bayesian search on best model; MLflowCallback")
        Component(evaluator, "Evaluator", "src/models/evaluate.py", "ROC-AUC, PR-AUC, KS, Gini, SHAP, calibration plots")
        Component(config, "Config", "src/config.py", "Paths, column names, hyperparameter search spaces, seeds")
    }

    Rel(loader, preprocessor, "Raw DataFrame")
    Rel(preprocessor, engineer, "Clean DataFrame")
    Rel(engineer, selector, "Enriched DataFrame")
    Rel(selector, trainer, "Selected features")
    Rel(trainer, tuner, "Best model type")
    Rel(tuner, evaluator, "Tuned model + test set")
    Rel(config, loader, "Config")
    Rel(config, preprocessor, "Column lists")
    Rel(config, trainer, "Search spaces")
```

---

## 5. Component Diagram — Streamlit App

```mermaid
C4Component
    title Component Diagram — Streamlit App

    Container_Boundary(streamlit_app, "Streamlit App (app/)") {
        Component(auth, "Analyst Login", "Simple name/ID selector (v1)", "Captures analyst identity for audit log")
        Component(predict_tab, "Predict Tab", "Streamlit form", "Input form → model inference → score + verdict + SHAP waterfall")
        Component(dashboard_tab, "Manager Dashboard Tab", "Plotly charts", "Portfolio stats: approval rate, score distribution, time trends")
        Component(model_loader, "Model Loader", "mlflow.pyfunc", "Loads Production model from MLflow registry on startup")
        Component(shap_renderer, "SHAP Renderer", "shap + Plotly", "Generates waterfall plot for individual predictions")
        Component(audit_logger, "Audit Logger", "SQLite3", "Writes immutable prediction record on every inference")
        Component(threshold_ctrl, "Threshold Control", "Streamlit slider", "Configurable PD% cutoffs for Approve/Refer/Reject bands")
    }

    Rel(auth, predict_tab, "Analyst ID")
    Rel(predict_tab, model_loader, "Feature vector")
    Rel(model_loader, predict_tab, "PD probability")
    Rel(predict_tab, shap_renderer, "Prediction + features")
    Rel(predict_tab, audit_logger, "Full prediction record")
    Rel(predict_tab, threshold_ctrl, "Reads cutoffs")
    Rel(dashboard_tab, audit_logger, "Reads prediction history")
```

---

## 6. Sequence Diagram — Analyst Making a Prediction

```mermaid
sequenceDiagram
    actor Analyst
    participant App as Streamlit App
    participant Model as MLflow Model (cached)
    participant SHAP as SHAP Explainer
    participant DB as SQLite Audit Log

    Analyst->>App: Opens browser, selects analyst name
    Analyst->>App: Fills in client application form
    App->>Model: predict_proba(feature_vector)
    Model-->>App: PD probability (e.g. 0.72)
    App->>SHAP: shap_values(feature_vector)
    SHAP-->>App: Feature contribution values
    App->>DB: INSERT INTO predictions (timestamp, analyst_id, inputs, pd_score, verdict)
    DB-->>App: OK
    App-->>Analyst: Risk Score: 72 | PD: 72% | 🔴 REJECT
    App-->>Analyst: SHAP Waterfall: Top 5 driving factors
```

---

## 7. Data Flow Diagram

```mermaid
flowchart TD
    A[Raw CSV/Excel\nLoan Application Records] --> B[Data Loader]
    B --> C[train/test split\nStratifiedKFold]
    C --> D[Preprocessing Pipeline\nImputer → Encoder → Scaler]
    D --> E[SMOTE\nClass Balancing]
    E --> F[Feature Engineering\nDerived Ratios / Log Transforms]
    F --> G[Feature Selection\nVariance → Correlation → RFECV]
    G --> H{Model Training\nLR / RF / XGBoost / LightGBM}
    H --> I[Optuna Tuning\nBayesian CV Optimization]
    I --> J[Model Evaluation\nROC-AUC / KS / SHAP]
    J --> K[MLflow Model Registry\nPromote to Production]
    K --> L[Streamlit App\nLoads Production Model]
    L --> M[Analyst Input\nClient Application Form]
    M --> N[Inference\nPD Score + SHAP]
    N --> O[SQLite Audit Log]
    N --> P[Analyst Screen\nScore + Verdict + Reasons]
```

---

## 8. Entity-Relationship Diagram — Audit Log

```mermaid
erDiagram
    PREDICTION_LOG {
        int id PK
        datetime timestamp
        varchar analyst_id
        varchar client_name
        float pd_score
        float risk_score_0_100
        varchar verdict
        float threshold_approve
        float threshold_refer
        json input_features
        json shap_values
        varchar model_version
    }
```

---

## 9. Architecture Decision Records (ADRs)

---

### ADR-001: Streamlit over FastAPI + React

**Status:** Accepted

**Context:**
Internal tool for ~10 concurrent analysts. Need rapid delivery with rich UI including forms, charts, and SHAP plots.

**Decision:** Use Streamlit

**Rationale:**
- Python-native — same language as ML pipeline, no context switching
- SHAP + Plotly integrate natively
- Two-tab layout satisfies both analyst and manager personas
- No frontend engineer needed

**Trade-offs:**
- Not suitable if > 100 concurrent users are needed
- Limited custom UI flexibility compared to React

**Mitigation:** Migrate to FastAPI + React if user count scales beyond 50 concurrent users.

**Revisit Trigger:** User count > 50 concurrent, or mobile access required.

---

### ADR-002: SQLite for Prediction Audit Log

**Status:** Accepted

**Context:**
Every prediction must be logged with full inputs + score + analyst ID. Dataset is small (< 50K predictions/year). No existing DB infrastructure.

**Decision:** SQLite3 file on the same server as the app

**Rationale:**
- Zero configuration, no DB server admin required
- Supports < 10 concurrent writes (sufficient for this team size)
- File-based backup is trivial
- Audit queries are infrequent (manager dashboard)

**Trade-offs:**
- Not suitable for multi-server deployments
- No built-in access control at DB level

**Mitigation:** Migrate to PostgreSQL if team scales or multi-region deployment needed.

**Revisit Trigger:** > 3 concurrent write transactions per second sustained, or multi-server deployment.

---

### ADR-003: XGBoost / LightGBM as Primary Model Family

**Status:** Accepted

**Context:**
< 50K tabular records, binary classification, class imbalance expected, explainability required.

**Decision:** Train LR, RF, XGBoost, LightGBM — promote best on ROC-AUC + KS statistic

**Rationale:**
- Gradient boosted trees consistently win on tabular data at this scale
- Native SHAP support in both XGBoost and LightGBM (TreeExplainer — fast)
- Handles missing values well (important since columns are TBD)
- sklearn API compatibility with MLflow autolog

**Trade-offs:**
- Logistic Regression (simpler, more interpretable) trained as baseline/fallback
- Neural networks excluded (overkill for < 50K records, no image/text data)

**Revisit Trigger:** Dataset grows > 500K records, or time-series/sequential patterns identified.

---

### ADR-004: MLflow for Experiment Tracking and Model Registry

**Status:** Accepted

**Context:**
One-time training initially, manual retraining when needed. Need reproducibility and model versioning.

**Decision:** MLflow with local filesystem artifact store, promoted via Model Registry stages

**Rationale:**
- De facto standard for Python ML tracking
- Model Registry provides Staging → Production promotion workflow
- Streamlit loads model via `mlflow.pyfunc.load_model("models:/CreditRiskModel/Production")`
- Free, open-source, Docker-deployable

**Trade-offs:**
- No automated drift detection (acceptable for v1 manual retraining)
- S3/GCS remote can be added later without code changes

**Revisit Trigger:** Automated retraining needed, or multi-team model sharing required.

---

### ADR-005: Docker Compose for Deployment

**Status:** Accepted

**Context:**
Hybrid deployment — MLflow on-premise, Streamlit accessible on internal browser network.

**Decision:** Docker Compose with two services: `mlflow` + `streamlit_app`, sharing a `mlruns/` volume

**Rationale:**
- Single `docker-compose up` command for full stack
- Shared volume ensures model artifacts are accessible to both containers
- Port 5000 (MLflow UI) and 8501 (Streamlit) exposed on intranet

**Trade-offs:**
- No HA / load balancing (single server)
- Manual container restart on failure

**Mitigation:** Add `restart: always` policy in docker-compose. Acceptable for internal tool.

**Revisit Trigger:** Uptime SLA required, or traffic exceeds single-server capacity.

---

## 10. Non-Functional Requirements

| Requirement | Specification | Rationale |
|---|---|---|
| **Prediction latency** | < 2 seconds end-to-end | Acceptable for manual underwriting workflow |
| **Availability** | Business hours only (9am–6pm) | Internal tool, no 24×7 SLA needed |
| **Data privacy** | All data stays on-premise | Proprietary client financial records |
| **Auditability** | 100% prediction logging | Regulatory / compliance requirement |
| **Concurrency** | Up to 10 simultaneous analysts | Internal team size |
| **Explainability** | SHAP waterfall per prediction | Regulatory justification + analyst trust |
| **Reproducibility** | All experiments versioned in MLflow | Model governance and retraining baseline |

---

## 11. Project Folder Structure

```
credit_risk_prediction/
├── data/
│   ├── raw/                        # Original dataset (gitignored)
│   └── processed/                  # Transformed datasets
├── docs/
│   └── architecture/
│       └── design_document.md      # This file
├── src/
│   ├── config.py                   # Central config (paths, columns, seeds)
│   ├── data/
│   │   ├── loader.py               # Data loading & schema validation
│   │   └── preprocessor.py         # sklearn Pipeline builder + SMOTE
│   ├── features/
│   │   ├── engineering.py          # Derived features
│   │   └── selection.py            # Feature selection pipeline
│   ├── models/
│   │   ├── train.py                # Training + MLflow logging
│   │   ├── tune.py                 # Optuna study
│   │   └── evaluate.py             # Metrics + SHAP plots
│   └── pipeline.py                 # End-to-end runner
├── app/
│   └── streamlit_app.py            # Two-tab Streamlit UI
├── tests/
│   ├── test_loader.py
│   ├── test_preprocessor.py
│   ├── test_engineering.py
│   ├── test_selection.py
│   └── test_evaluate.py
├── docker/
│   ├── Dockerfile.app
│   └── Dockerfile.mlflow
├── docker-compose.yml
├── pyproject.toml
└── README.md
```

---

## 12. Decision Log Summary

| Decision | Chosen | Key Reason |
|---|---|---|
| UI Framework | Streamlit | Python-native, SHAP-compatible, rapid delivery |
| Audit Store | SQLite | Zero-config, on-premise, sufficient for team scale |
| Model Family | XGBoost / LightGBM | Best tabular performance + native SHAP TreeExplainer |
| ML Tracking | MLflow | Industry standard, Docker-deployable, free |
| Deployment | Docker Compose | Single-command full stack, shared volume for mlruns |
| Auth (v1) | Analyst name selector | Captures ID without SSO complexity; upgrade later |
| Retraining | Manual / on-demand | One-time training for now; automate when data grows |
