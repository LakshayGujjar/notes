# 🤖 GCP Vertex AI — Comprehensive Cheatsheet

> **Audience:** ML engineers, data scientists, MLOps engineers, and architects building ML systems on GCP.
> **Last updated:** March 2026 | Covers Training, Serving, Pipelines, Feature Store, Gemini, Vector Search, and all core Vertex AI features.

---

## Table of Contents

1. [Overview & Key Concepts](#1-overview--key-concepts)
2. [Vertex AI Workbench & Development](#2-vertex-ai-workbench--development)
3. [Managed Datasets](#3-managed-datasets)
4. [Custom Training](#4-custom-training)
5. [Vertex AI Pipelines](#5-vertex-ai-pipelines)
6. [AutoML](#6-automl)
7. [Model Registry & Artifact Management](#7-model-registry--artifact-management)
8. [Online Prediction (Endpoints)](#8-online-prediction-endpoints)
9. [Batch Prediction](#9-batch-prediction)
10. [Vertex AI Feature Store](#10-vertex-ai-feature-store)
11. [Vertex AI Experiments & Metadata](#11-vertex-ai-experiments--metadata)
12. [Model Monitoring](#12-model-monitoring)
13. [Generative AI — Gemini & Foundation Models](#13-generative-ai--gemini--foundation-models)
14. [Embeddings & Vector Search](#14-embeddings--vector-search)
15. [Vertex AI Agent Builder & Search](#15-vertex-ai-agent-builder--search)
16. [MLOps Best Practices & Patterns](#16-mlops-best-practices--patterns)
17. [IAM, Security & Networking](#17-iam-security--networking)
18. [gcloud CLI & SDK Quick Reference](#18-gcloud-cli--sdk-quick-reference)
19. [Pricing Summary](#19-pricing-summary)
20. [Quick Reference & Comparison Tables](#20-quick-reference--comparison-tables)

---

## 1. Overview & Key Concepts

### What is Vertex AI?

**Google Cloud Vertex AI** is a unified, fully managed ML platform that consolidates every stage of the ML workflow — data preparation, training, evaluation, serving, monitoring, and generative AI — into a single product with shared tooling, IAM, and governance.

**Core value proposition:**

| Pillar | Description |
|---|---|
| **Unified platform** | One SDK, one console, shared IAM for all ML tasks |
| **Managed infrastructure** | Auto-scaled training clusters, serverless prediction endpoints |
| **MLOps native** | Pipelines, Experiments, Model Registry, Monitoring built-in |
| **Generative AI** | Gemini, Imagen, Embeddings, Vector Search, RAG Engine |
| **Governance** | CMEK, VPC-SC, audit logs, model lineage, feature monitoring |

---

### Vertex AI vs. Alternatives

| Dimension | Vertex AI | SageMaker | Azure ML | Self-managed GKE |
|---|---|---|---|---|
| **Management** | Fully managed | Managed | Managed | DIY |
| **GCP integration** | ✅ Native (BQ, GCS, Dataflow) | ❌ | ❌ | ✅ Custom |
| **Generative AI** | ✅ Gemini, Imagen, RAG | ✅ Bedrock | ✅ Azure OpenAI | ❌ |
| **Pipelines** | ✅ KFP v2 managed | ✅ SageMaker Pipelines | ✅ Azure ML Pipelines | ✅ Argo/KFP |
| **Pricing model** | Per resource consumed | Per resource consumed | Per resource consumed | GKE + GCE costs |
| **Best for** | GCP-native ML, GenAI | AWS-native ML | Azure-native ML | Full control needed |

---

### Core Components Overview

| Component | Purpose | SDK Class |
|---|---|---|
| **Workbench** | Managed JupyterLab for development | — (UI/CLI) |
| **Managed Datasets** | Versioned, labeled training datasets | `aiplatform.Dataset` |
| **Custom Training** | Run your own training code on managed infra | `aiplatform.CustomJob` |
| **AutoML** | Automated model training | `aiplatform.AutoMLTabularTrainingJob` |
| **Pipelines** | Orchestrated ML workflows (KFP v2) | `aiplatform.PipelineJob` |
| **Model Registry** | Version, track, and deploy models | `aiplatform.Model` |
| **Endpoints** | Auto-scaled online prediction serving | `aiplatform.Endpoint` |
| **Batch Prediction** | Async large-scale inference | `aiplatform.BatchPredictionJob` |
| **Feature Store** | Managed feature serving and sharing | `aiplatform.FeatureStore` |
| **Experiments** | Track parameters, metrics, artifacts | `aiplatform.Experiment` |
| **Model Monitoring** | Training-serving skew & drift detection | `aiplatform.ModelMonitoringJob` |
| **Vector Search** | Managed ANN index for embeddings | `aiplatform.MatchingEngineIndex` |
| **Generative AI** | Gemini, Imagen, embeddings | `vertexai.generative_models` |
| **RAG Engine** | Managed retrieval-augmented generation | `vertexai.rag` |
| **Agent Builder** | Enterprise search & conversation | `discoveryengine` |

---

### Vertex AI Architecture

```
Control Plane (Vertex AI Service)
├── Training jobs     → Dataproc / GCE workers
├── Prediction        → Managed serving clusters
├── Pipelines         → Managed KFP orchestrator
├── Feature Store     → BigQuery + Bigtable backing
├── Model Registry    → GCS artifact storage
├── Experiments/MLMD  → Metadata store
└── GenAI APIs        → Google's foundation model infra

Data Plane (Your GCP Resources):
├── GCS (model artifacts, training data, pipeline outputs)
├── BigQuery (datasets, feature source, batch prediction I/O)
├── Artifact Registry (custom container images)
└── Cloud Monitoring / Logging (metrics, alerts, audit)
```

---

### Key Terminology

| Term | Definition |
|---|---|
| **Custom Job** | Single training run with your code on managed VMs |
| **HP Tuning Job** | Parallel hyperparameter search runs |
| **Training Pipeline** | Chained dataset + training + evaluation steps |
| **Managed Dataset** | Versioned dataset registered in Vertex AI |
| **Model** | A trained model artifact registered in the Model Registry |
| **Endpoint** | Serving resource that hosts one or more deployed models |
| **Deployment** | A model version deployed to an endpoint with a traffic % |
| **Feature Store** | Managed repository for ML feature definitions and values |
| **Feature View** | Materialized, low-latency serving layer for features |
| **Pipeline Run** | A single execution of a compiled pipeline spec |
| **Experiment / Run** | A tracked ML experiment with params + metrics |
| **Artifact** | Any tracked input/output (dataset, model, evaluation) |

---

### Supported ML Frameworks

| Framework | Pre-built Container | Custom Container |
|---|---|---|
| **TensorFlow** | ✅ `tf-cpu/gpu.2-13` | ✅ |
| **PyTorch** | ✅ `pytorch-gpu.2-2` | ✅ |
| **scikit-learn** | ✅ `sklearn.1-3` | ✅ |
| **XGBoost** | ✅ `xgboost-cpu.1-7` | ✅ |
| **Hugging Face** | ❌ (use custom) | ✅ |
| **JAX** | ❌ (use custom) | ✅ |

---

### Vertex AI vs. BigQuery ML

| Scenario | Use Vertex AI | Use BigQuery ML |
|---|---|---|
| Complex deep learning (PyTorch, TF) | ✅ | ❌ |
| Simple tabular models in SQL | ❌ | ✅ |
| Data already in BigQuery, no export | ❌ | ✅ |
| Custom architecture, BYOC | ✅ | ❌ |
| AutoML tabular | ✅ (more features) | ✅ (simpler) |
| Real-time online serving | ✅ Endpoints | ❌ |
| Batch scoring in SQL | ❌ | ✅ `ML.PREDICT` |

---

### Key Limits & Quotas

| Resource | Default Limit |
|---|---|
| Concurrent custom training jobs | 100 |
| Prediction QPS per endpoint | 10,000 |
| Deployed models per endpoint | 50 |
| Pipeline runs per project | 10,000 |
| Feature views per feature store | 100 |
| Gemini API requests per minute | 60 (varies by model) |
| Vector Search index size | 10B datapoints |

---

### Required APIs & IAM

```bash
# Enable required APIs
gcloud services enable \
  aiplatform.googleapis.com \
  notebooks.googleapis.com \
  artifactregistry.googleapis.com \
  bigquery.googleapis.com \
  storage.googleapis.com \
  --project=MY_PROJECT
```

---

## 2. Vertex AI Workbench & Development

### Workbench Instance Types

| Type | Management | Customization | Best For |
|---|---|---|---|
| **Managed Notebooks** | Google-managed, auto-updated | Limited | Standard ML development |
| **User-managed Notebooks** | You manage lifecycle | Full control | Custom dependencies, GPUs |
| **Colab Enterprise** | Google-managed | Moderate | Team collaboration, cost-efficient |

---

### gcloud: Workbench Instances

```bash
# Create a managed notebook instance
gcloud workbench instances create my-workbench \
  --location=us-central1-a \
  --machine-type=n1-standard-4 \
  --vm-image-project=deeplearning-platform-release \
  --vm-image-family=common-cpu-notebooks \
  --instance-owners=user@company.com \
  --no-enable-public-ip \                    # Private IP only
  --subnet=projects/proj/regions/us-central1/subnetworks/my-subnet

# Create with GPU
gcloud workbench instances create my-gpu-workbench \
  --location=us-central1-a \
  --machine-type=n1-standard-8 \
  --accelerator-type=NVIDIA_TESLA_T4 \
  --accelerator-core-count=1 \
  --vm-image-family=pytorch-latest-gpu-notebooks

# List instances
gcloud workbench instances list --location=us-central1-a

# Start / Stop
gcloud workbench instances start my-workbench --location=us-central1-a
gcloud workbench instances stop  my-workbench --location=us-central1-a

# Delete
gcloud workbench instances delete my-workbench \
  --location=us-central1-a --quiet
```

---

### Vertex AI SDK Initialization

```python
# Install
# pip install google-cloud-aiplatform vertexai

import vertexai
from google.cloud import aiplatform

# Initialize (required before any SDK call)
aiplatform.init(
    project         = "my-project",
    location        = "us-central1",
    staging_bucket  = "gs://my-staging-bucket",   # For temporary artifacts
    experiment      = "my-experiment",             # Optional: attach to experiment
    experiment_description = "Model v2 training",
)

# Or use vertexai (for GenAI features)
vertexai.init(project="my-project", location="us-central1")
```

---

## 3. Managed Datasets

### Dataset Types & Creation

```python
from google.cloud import aiplatform

# ── Tabular dataset from BigQuery ─────────────────────────────────────
tabular_dataset = aiplatform.TabularDataset.create(
    display_name      = "orders-training-dataset",
    bq_source         = "bq://my-project.orders.training_table",
    labels            = {"env": "prod", "domain": "commerce"},
)
print(f"Dataset: {tabular_dataset.resource_name}")

# ── Tabular dataset from GCS ──────────────────────────────────────────
tabular_gcs = aiplatform.TabularDataset.create(
    display_name = "orders-csv-dataset",
    gcs_source   = ["gs://my-bucket/training/orders_*.csv"],
)

# ── Image dataset from GCS ────────────────────────────────────────────
# Requires an import file (CSV with image URIs + labels)
image_dataset = aiplatform.ImageDataset.create(
    display_name       = "product-images",
    gcs_source         = "gs://my-bucket/image_import.csv",
    import_schema_uri  = aiplatform.schema.dataset.ioformat.image.single_label_classification,
)

# ── Text dataset ──────────────────────────────────────────────────────
text_dataset = aiplatform.TextDataset.create(
    display_name = "reviews-dataset",
    gcs_source   = ["gs://my-bucket/reviews.jsonl"],
    import_schema_uri = aiplatform.schema.dataset.ioformat.text.single_label_classification,
)

# ── List datasets ─────────────────────────────────────────────────────
datasets = aiplatform.TabularDataset.list(filter="display_name=orders*")
for ds in datasets:
    print(f"{ds.display_name}: {ds.resource_name}")
```

---

### gcloud: Datasets

```bash
# Create tabular dataset from BigQuery
gcloud ai datasets create \
  --display-name="orders-dataset" \
  --metadata-schema-uri="gs://google-cloud-aiplatform/schema/dataset/metadata/tabular_1.0.0.yaml" \
  --region=us-central1

# List datasets
gcloud ai datasets list --region=us-central1 \
  --format="table(name,displayName,metadataSchemaUri)"

# Describe a dataset
gcloud ai datasets describe DATASET_ID --region=us-central1
```

> 💡 **When to use Managed Datasets:** Use them when you want AutoML training, data labeling workflows, or dataset versioning. For custom training, referencing GCS/BQ paths directly in your code is simpler.

---

## 4. Custom Training

### Training Job Types

| Job Type | When to Use | SDK Class |
|---|---|---|
| `CustomJob` | Single run, full control | `aiplatform.CustomJob` |
| `CustomTrainingJob` | Integrates with Managed Dataset | `aiplatform.CustomTrainingJob` |
| `CustomContainerTrainingJob` | Use your own Docker image | `aiplatform.CustomContainerTrainingJob` |
| `HyperparameterTuningJob` | Search over hyperparameter space | `aiplatform.HyperparameterTuningJob` |

---

### Pre-built Container URIs

| Framework | Container URI |
|---|---|
| TF 2.13 CPU | `us-docker.pkg.dev/vertex-ai/training/tf-cpu.2-13.py310:latest` |
| TF 2.13 GPU | `us-docker.pkg.dev/vertex-ai/training/tf-gpu.2-13.py310:latest` |
| PyTorch 2.2 GPU | `us-docker.pkg.dev/vertex-ai/training/pytorch-gpu.2-2.py310:latest` |
| sklearn 1.3 CPU | `us-docker.pkg.dev/vertex-ai/training/sklearn-cpu.1-3.py310:latest` |
| XGBoost 1.7 CPU | `us-docker.pkg.dev/vertex-ai/training/xgboost-cpu.1-7.py310:latest` |

---

### Complete Training Script

```python
# trainer/task.py — entry point for Vertex AI custom training
import argparse
import os
import joblib
import pandas as pd
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import accuracy_score, roc_auc_score
from sklearn.model_selection import train_test_split
import google.cloud.storage as gcs

def train(args):
    # ── Load data from GCS ────────────────────────────────────────────
    train_uri = os.environ.get("AIP_TRAINING_DATA_URI", args.train_data)
    df = pd.read_csv(train_uri)
    X  = df.drop(columns=[args.target_col])
    y  = df[args.target_col]
    X_train, X_val, y_train, y_val = train_test_split(X, y, test_size=0.2, random_state=42)

    # ── Train ─────────────────────────────────────────────────────────
    model = GradientBoostingClassifier(
        n_estimators   = args.n_estimators,
        max_depth      = args.max_depth,
        learning_rate  = args.learning_rate,
    )
    model.fit(X_train, y_train)

    # ── Evaluate ──────────────────────────────────────────────────────
    val_preds = model.predict(X_val)
    val_proba = model.predict_proba(X_val)[:, 1]
    accuracy  = accuracy_score(y_val, val_preds)
    auc       = roc_auc_score(y_val, val_proba)

    print(f"Validation Accuracy: {accuracy:.4f}")
    print(f"Validation AUC-ROC:  {auc:.4f}")

    # Vertex AI HP Tuning reads this metric from stdout
    print(f"metric: {auc}")

    # ── Save model to AIP_MODEL_DIR ───────────────────────────────────
    model_dir = os.environ.get("AIP_MODEL_DIR", args.model_dir)
    os.makedirs(model_dir, exist_ok=True)
    model_path = os.path.join(model_dir, "model.joblib")
    joblib.dump(model, model_path)
    print(f"Model saved to: {model_path}")

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--train-data",    default="gs://bucket/train.csv")
    parser.add_argument("--model-dir",     default="/tmp/model")
    parser.add_argument("--target-col",    default="label")
    parser.add_argument("--n-estimators",  type=int,   default=100)
    parser.add_argument("--max-depth",     type=int,   default=4)
    parser.add_argument("--learning-rate", type=float, default=0.1)
    args = parser.parse_args()
    train(args)
```

---

### Python SDK: Submit Custom Job

```python
from google.cloud import aiplatform

aiplatform.init(project="my-project", location="us-central1",
                staging_bucket="gs://my-staging-bucket")

# ── CustomJob (pre-built container) ──────────────────────────────────
job = aiplatform.CustomJob(
    display_name = "gbm-training-job",
    worker_pool_specs = [
        {
            "machine_spec": {
                "machine_type":      "n1-standard-8",
                "accelerator_type":  "NVIDIA_TESLA_T4",   # Optional GPU
                "accelerator_count": 1,
            },
            "replica_count": 1,
            "python_package_spec": {
                "executor_image_uri": "us-docker.pkg.dev/vertex-ai/training/sklearn-cpu.1-3.py310:latest",
                "package_uris":       ["gs://my-bucket/trainer-0.1.tar.gz"],
                "python_module":      "trainer.task",
                "args": [
                    "--train-data=gs://my-bucket/train.csv",
                    "--n-estimators=200",
                    "--max-depth=5",
                    "--learning-rate=0.05",
                ],
            },
        }
    ],
)

job.run(
    service_account = "training-sa@my-project.iam.gserviceaccount.com",
    sync            = True,   # Block until complete
)
print(f"Job state: {job.state}")

# ── CustomJob with Spot VMs ───────────────────────────────────────────
spot_job = aiplatform.CustomJob(
    display_name = "gbm-training-spot",
    worker_pool_specs = [{
        "machine_spec": {"machine_type": "n1-standard-8"},
        "replica_count": 1,
        "python_package_spec": {
            "executor_image_uri": "us-docker.pkg.dev/vertex-ai/training/sklearn-cpu.1-3.py310:latest",
            "package_uris":       ["gs://my-bucket/trainer-0.1.tar.gz"],
            "python_module":      "trainer.task",
        },
    }],
)
spot_job.run(
    scheduling = {"restart_job_on_worker_restart": True},  # Spot VM restart
)
```

---

### Hyperparameter Tuning

```python
from google.cloud.aiplatform import hyperparameter_tuning as hpt

hp_job = aiplatform.HyperparameterTuningJob(
    display_name   = "gbm-hpt-job",
    metric_spec    = {"auc": "maximize"},          # Metric to optimize
    parameter_spec = {
        "learning-rate": hpt.DoubleParameterSpec(min=0.001, max=0.3,   scale="log"),
        "n-estimators":  hpt.IntegerParameterSpec(min=50,   max=500,   scale="linear"),
        "max-depth":     hpt.DiscreteParameterSpec(values=[3, 4, 5, 6], scale="unit_linear_scale"),
    },
    max_trial_count         = 20,
    parallel_trial_count    = 5,
    search_algorithm        = "bayesian",           # grid | random | bayesian
    custom_job = aiplatform.CustomJob(
        display_name      = "gbm-hpt-trial",
        worker_pool_specs = [{
            "machine_spec":       {"machine_type": "n1-standard-4"},
            "replica_count":      1,
            "python_package_spec": {
                "executor_image_uri": "us-docker.pkg.dev/vertex-ai/training/sklearn-cpu.1-3.py310:latest",
                "package_uris":       ["gs://bucket/trainer-0.1.tar.gz"],
                "python_module":      "trainer.task",
            },
        }],
    ),
)

hp_job.run(sync=True)

# Get best trial
best = min(hp_job.trials, key=lambda t: -t.final_measurement.metrics[0].value)
print(f"Best AUC: {best.final_measurement.metrics[0].value:.4f}")
print(f"Best params: {best.parameters}")
```

---

### Distributed Training (Multi-Worker)

```python
# MultiWorkerMirroredStrategy — data parallelism across N workers
worker_pool_specs = [
    # Chief worker (index 0)
    {
        "machine_spec":   {"machine_type": "a2-highgpu-1g"},
        "replica_count":  1,
        "python_package_spec": {
            "executor_image_uri": "us-docker.pkg.dev/vertex-ai/training/tf-gpu.2-13.py310:latest",
            "package_uris":       ["gs://bucket/trainer.tar.gz"],
            "python_module":      "trainer.task",
        },
    },
    # Worker replicas (indices 1..N)
    {
        "machine_spec":   {"machine_type": "a2-highgpu-1g"},
        "replica_count":  3,          # 3 additional workers = 4 total
        "python_package_spec": {
            "executor_image_uri": "us-docker.pkg.dev/vertex-ai/training/tf-gpu.2-13.py310:latest",
            "package_uris":       ["gs://bucket/trainer.tar.gz"],
            "python_module":      "trainer.task",
        },
    },
]
```

---

### Vertex AI Environment Variables

| Variable | Description |
|---|---|
| `AIP_MODEL_DIR` | GCS path where model artifacts should be saved |
| `AIP_TRAINING_DATA_URI` | GCS path to training data (when using Managed Dataset) |
| `AIP_VALIDATION_DATA_URI` | GCS path to validation data |
| `AIP_TEST_DATA_URI` | GCS path to test data |
| `AIP_STORAGE_URI` | Root GCS path for this job |
| `CLOUD_ML_JOB_ID` | The full resource name of the training job |
| `TF_CONFIG` | JSON config for distributed TF (multi-worker) |

---

## 5. Vertex AI Pipelines

### KFP v2 Components & Pipeline

```python
# pip install kfp google-cloud-pipeline-components

from kfp import dsl, compiler
from kfp.dsl import (
    component, pipeline, Input, Output,
    Dataset, Model, Metrics, Artifact
)
from google.cloud import aiplatform
from google_cloud_pipeline_components.v1.custom_job import CustomTrainingJobOp
from google_cloud_pipeline_components.v1.endpoint import (
    EndpointCreateOp, ModelDeployOp
)

# ── Define components ─────────────────────────────────────────────────

@component(
    base_image="python:3.10-slim",
    packages_to_install=["pandas", "google-cloud-bigquery", "google-cloud-storage"]
)
def ingest_data(
    bq_table:    str,
    project:     str,
    output_data: Output[Dataset],
) -> None:
    """Extract data from BigQuery and save to GCS."""
    from google.cloud import bigquery
    import pandas as pd

    client = bigquery.Client(project=project)
    df = client.query(f"SELECT * FROM `{bq_table}`").to_dataframe()
    df.to_csv(output_data.path + "/data.csv", index=False)
    output_data.metadata["row_count"] = len(df)
    print(f"Ingested {len(df)} rows")


@component(
    base_image="python:3.10-slim",
    packages_to_install=["pandas", "scikit-learn"]
)
def preprocess(
    raw_data:       Input[Dataset],
    processed_data: Output[Dataset],
    target_column:  str = "label",
) -> None:
    """Clean and feature-engineer the training data."""
    import pandas as pd
    from sklearn.preprocessing import StandardScaler

    df = pd.read_csv(raw_data.path + "/data.csv")
    # ... preprocessing logic ...
    df.to_csv(processed_data.path + "/processed.csv", index=False)
    processed_data.metadata["features"] = list(df.columns)


@component(
    base_image="python:3.10-slim",
    packages_to_install=["pandas", "scikit-learn", "joblib"]
)
def train_model(
    processed_data: Input[Dataset],
    model_artifact: Output[Model],
    metrics_out:    Output[Metrics],
    n_estimators:   int   = 100,
    learning_rate:  float = 0.1,
) -> float:
    """Train a GBM model and log metrics."""
    import pandas as pd
    import joblib
    from sklearn.ensemble import GradientBoostingClassifier
    from sklearn.metrics import roc_auc_score
    from sklearn.model_selection import train_test_split

    df = pd.read_csv(processed_data.path + "/processed.csv")
    X, y = df.drop("label", axis=1), df["label"]
    X_tr, X_val, y_tr, y_val = train_test_split(X, y, test_size=0.2)

    clf = GradientBoostingClassifier(
        n_estimators=n_estimators, learning_rate=learning_rate
    )
    clf.fit(X_tr, y_tr)
    auc = roc_auc_score(y_val, clf.predict_proba(X_val)[:, 1])

    # Log metrics to Vertex AI
    metrics_out.log_metric("auc", auc)
    metrics_out.log_metric("n_estimators", n_estimators)

    # Save model
    joblib.dump(clf, model_artifact.path + "/model.joblib")
    model_artifact.metadata["framework"] = "sklearn"
    model_artifact.metadata["auc"]       = auc
    return auc


@component(base_image="python:3.10-slim")
def check_quality_gate(auc_threshold: float, actual_auc: float) -> bool:
    """Return True if model meets quality threshold."""
    passed = actual_auc >= auc_threshold
    print(f"AUC {actual_auc:.4f} vs threshold {auc_threshold}: {'PASS' if passed else 'FAIL'}")
    return passed


# ── Define pipeline ───────────────────────────────────────────────────

@pipeline(
    name        = "orders-training-pipeline",
    description = "E2E training pipeline for order prediction model",
    pipeline_root = "gs://my-bucket/pipeline_root",
)
def orders_pipeline(
    bq_table:      str   = "my-project.orders.training_v1",
    project:       str   = "my-project",
    n_estimators:  int   = 100,
    learning_rate: float = 0.1,
    auc_threshold: float = 0.85,
):
    ingest_op  = ingest_data(bq_table=bq_table, project=project)

    preprocess_op = preprocess(
        raw_data      = ingest_op.outputs["output_data"],
        target_column = "churn",
    )

    train_op = train_model(
        processed_data = preprocess_op.outputs["processed_data"],
        n_estimators   = n_estimators,
        learning_rate  = learning_rate,
    )

    gate_op = check_quality_gate(
        auc_threshold = auc_threshold,
        actual_auc    = train_op.output,
    )

    # Conditional deployment — only deploy if quality gate passes
    with dsl.If(gate_op.output == True, name="quality-gate-passed"):
        endpoint_op = EndpointCreateOp(
            project      = project,
            location     = "us-central1",
            display_name = "orders-endpoint",
        )
        deploy_op = ModelDeployOp(
            model                  = train_op.outputs["model_artifact"],
            endpoint               = endpoint_op.outputs["endpoint"],
            dedicated_resources_min_replica_count = 1,
            dedicated_resources_max_replica_count = 3,
            dedicated_resources_machine_type      = "n1-standard-4",
        )


# ── Compile & Submit ──────────────────────────────────────────────────

# Compile to YAML
compiler.Compiler().compile(
    pipeline_func = orders_pipeline,
    package_path  = "orders_pipeline.yaml",
)

# Submit
aiplatform.init(project="my-project", location="us-central1")

pipeline_job = aiplatform.PipelineJob(
    display_name        = "orders-training-run-001",
    template_path       = "orders_pipeline.yaml",
    pipeline_root       = "gs://my-bucket/pipeline_root",
    parameter_values    = {
        "bq_table":      "my-project.orders.training_v2",
        "n_estimators":  200,
        "learning_rate": 0.05,
        "auc_threshold": 0.88,
    },
    enable_caching = True,
)

pipeline_job.submit(
    service_account = "pipeline-sa@my-project.iam.gserviceaccount.com"
)
print(f"Pipeline job: {pipeline_job.resource_name}")
print(f"Dashboard:    {pipeline_job._dashboard_uri()}")

# Cancel a pipeline
pipeline_job.cancel()

# List recent pipeline runs
jobs = aiplatform.PipelineJob.list(
    filter    = 'display_name="orders-training-run*"',
    order_by  = "create_time desc",
    max_results = 10,
)
```

---

### Pipeline Scheduling

```python
# Schedule pipeline to run daily at 3am UTC
schedule = pipeline_job.create_schedule(
    display_name    = "daily-orders-training",
    cron            = "0 3 * * *",
    max_concurrent_run_count = 1,
    max_run_count   = 0,   # 0 = unlimited
)
print(f"Schedule: {schedule.resource_name}")
```

---

## 6. AutoML

### AutoML Training

```python
from google.cloud import aiplatform

# ── AutoML Tabular Classification ────────────────────────────────────
dataset = aiplatform.TabularDataset.create(
    display_name = "churn-dataset",
    bq_source    = "bq://my-project.ml.churn_training",
)

job = aiplatform.AutoMLTabularTrainingJob(
    display_name     = "churn-automl-v1",
    optimization_prediction_type = "classification",    # or regression / forecasting
    optimization_objective       = "maximize-au-roc",  # or minimize-rmse
    column_specs     = {
        "tenure_months":  "numeric",
        "monthly_charges":"numeric",
        "contract_type":  "categorical",
        "total_charges":  "numeric",
        "churn":          "categorical",               # Target column
    },
)

model = job.run(
    dataset         = dataset,
    target_column   = "churn",
    training_fraction_split   = 0.8,
    validation_fraction_split = 0.1,
    test_fraction_split       = 0.1,
    budget_milli_node_hours   = 1000,   # 1 node-hour
    model_display_name        = "churn-model-automl-v1",
    disable_early_stopping    = False,
)

# ── Get evaluation metrics ────────────────────────────────────────────
model_evals = model.list_model_evaluations()
for eval in model_evals:
    metrics = eval.metrics
    print(f"AUC-ROC:   {metrics.get('auRoc'):.4f}")
    print(f"AUC-PR:    {metrics.get('auPrc'):.4f}")
    print(f"Log loss:  {metrics.get('logLoss'):.4f}")
```

---

### AutoML vs. Custom Training Decision Guide

| Criterion | AutoML | Custom Training |
|---|---|---|
| **ML expertise required** | Low | High |
| **Speed to first model** | Fast (hours) | Slower (days) |
| **Custom architecture** | ❌ | ✅ |
| **Max control** | ❌ | ✅ |
| **Tabular baseline** | ✅ Excellent | ✅ Better for complex features |
| **Computer vision** | ✅ Good (classification, OD) | ✅ Better for SOTA models |
| **Cost predictability** | ✅ (node-hours) | Varies |
| **Integration with Pipelines** | ✅ GCPC components | ✅ CustomTrainingJobOp |

---

## 7. Model Registry & Artifact Management

### Uploading & Managing Models

```python
from google.cloud import aiplatform

# ── Upload a model (sklearn joblib) ──────────────────────────────────
model = aiplatform.Model.upload(
    display_name          = "churn-model-v2",
    description           = "GBM churn prediction model, trained on 2026-03 data",
    artifact_uri          = "gs://my-bucket/models/churn-v2/",
    serving_container_image_uri = "us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-3:latest",
    serving_container_predict_route = "/predict",
    serving_container_health_route  = "/health",
    serving_container_environment_variables = {
        "MODEL_NAME": "churn-gbm"
    },
    labels = {"version": "v2", "framework": "sklearn", "domain": "commerce"},
)
print(f"Model: {model.resource_name}")
print(f"Version ID: {model.version_id}")

# ── Upload a TF SavedModel ────────────────────────────────────────────
tf_model = aiplatform.Model.upload(
    display_name          = "revenue-predictor",
    artifact_uri          = "gs://my-bucket/models/revenue-tf/",
    serving_container_image_uri = "us-docker.pkg.dev/vertex-ai/prediction/tf2-cpu.2-13:latest",
)

# ── Upload a new version of an existing model ─────────────────────────
model_v3 = aiplatform.Model.upload(
    display_name          = "churn-model-v3",
    artifact_uri          = "gs://my-bucket/models/churn-v3/",
    serving_container_image_uri = "us-docker.pkg.dev/vertex-ai/prediction/sklearn-cpu.1-3:latest",
    parent_model          = model.resource_name,   # Creates new version under same model
)

# ── Assign an alias ───────────────────────────────────────────────────
# Via gcloud (aliases not yet in Python SDK as of v1.50):
# gcloud ai models add-version-aliases churn-model --region=us-central1 \
#   --version=3 --aliases=champion,production

# ── List models ───────────────────────────────────────────────────────
models = aiplatform.Model.list(filter='labels.domain="commerce"')
for m in models:
    print(f"{m.display_name} v{m.version_id}: {m.artifact_uri}")

# ── Delete a specific version ─────────────────────────────────────────
model_v3.delete()
```

---

### gcloud: Model Registry

```bash
# List all model versions
gcloud ai models list --region=us-central1 \
  --format="table(name,displayName,versionId,createTime)"

# Add alias to a model version
gcloud ai models add-version-aliases MODEL_ID \
  --region=us-central1 \
  --version=3 \
  --aliases=champion,production

# Remove alias
gcloud ai models remove-version-aliases MODEL_ID \
  --region=us-central1 \
  --version=2 \
  --aliases=production

# Get model details
gcloud ai models describe MODEL_ID --region=us-central1
```

---

## 8. Online Prediction (Endpoints)

### Create Endpoint & Deploy Model

```python
from google.cloud import aiplatform

# ── Create endpoint ───────────────────────────────────────────────────
endpoint = aiplatform.Endpoint.create(
    display_name = "churn-prediction-endpoint",
    labels       = {"env": "prod", "model": "churn"},
)
print(f"Endpoint: {endpoint.resource_name}")

# ── Deploy model to endpoint ──────────────────────────────────────────
deployed = model.deploy(
    endpoint             = endpoint,
    deployed_model_display_name = "churn-model-v2",
    machine_type         = "n1-standard-4",
    min_replica_count    = 1,
    max_replica_count    = 5,       # Auto-scales based on load
    accelerator_type     = None,    # "NVIDIA_TESLA_T4" for GPU
    traffic_percentage   = 100,
)
print(f"Deployed model: {deployed.id}")

# ── Traffic splitting (A/B test: 80% champion, 20% challenger) ────────
endpoint.deploy(
    model                       = challenger_model,
    deployed_model_display_name = "churn-model-v3-challenger",
    machine_type                = "n1-standard-4",
    min_replica_count           = 1,
    max_replica_count           = 3,
    traffic_percentage          = 20,   # 20% of traffic
)
# Update champion to 80%
endpoint.update(
    traffic_split = {
        champion_deployed_id:   80,
        challenger_deployed_id: 20,
    }
)

# ── Make a prediction ─────────────────────────────────────────────────
response = endpoint.predict(
    instances = [
        {"tenure_months": 24, "monthly_charges": 65.5,
         "contract_type": "Month-to-month", "total_charges": 1572.0},
        {"tenure_months": 60, "monthly_charges": 90.0,
         "contract_type": "Two year", "total_charges": 5400.0},
    ]
)
print(f"Predictions: {response.predictions}")
print(f"Deployed model: {response.deployed_model_id}")

# ── Undeploy & delete ─────────────────────────────────────────────────
endpoint.undeploy(deployed_model_id=deployed.id)
endpoint.delete()
```

---

### REST API Prediction

```bash
TOKEN=$(gcloud auth print-access-token)
ENDPOINT_ID="1234567890"
PROJECT="my-project"
REGION="us-central1"

curl -X POST \
  "https://${REGION}-aiplatform.googleapis.com/v1/projects/${PROJECT}/locations/${REGION}/endpoints/${ENDPOINT_ID}:predict" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [
      {"tenure_months": 24, "monthly_charges": 65.5,
       "contract_type": "Month-to-month", "total_charges": 1572.0}
    ],
    "parameters": {"confidenceThreshold": 0.5}
  }'
```

**Response format:**
```json
{
  "predictions": [
    [0.23, 0.77]
  ],
  "deployedModelId": "8765432100000000000",
  "model": "projects/123/locations/us-central1/models/456",
  "modelDisplayName": "churn-model-v2",
  "modelVersionId": "2"
}
```

---

### Explanations (XAI)

```python
# Enable feature attributions at deployment time
from google.cloud.aiplatform.explain import ExplanationMetadata, ExplanationParameters

model_with_xai = aiplatform.Model.upload(
    display_name   = "churn-model-xai",
    artifact_uri   = "gs://bucket/models/churn/",
    serving_container_image_uri = "...",
    explanation_metadata = ExplanationMetadata(
        inputs = {
            "tenure_months":  ExplanationMetadata.InputMetadata(),
            "monthly_charges":ExplanationMetadata.InputMetadata(),
            "contract_type":  ExplanationMetadata.InputMetadata(),
        },
        outputs = {"churn_probability": ExplanationMetadata.OutputMetadata()},
    ),
    explanation_parameters = ExplanationParameters(
        sampled_shapley_attribution = {"path_count": 10}
        # or: integrated_gradients_attribution
        # or: xrai_attribution
    ),
)

# Explain a prediction
response = endpoint.explain(
    instances = [{"tenure_months": 24, "monthly_charges": 65.5, ...}]
)
for explanation in response.explanations:
    for feature, attribution in explanation.attributions[0].feature_attributions.items():
        print(f"  {feature}: {attribution:.4f}")
```

---

## 9. Batch Prediction

### Batch Prediction Jobs

```python
from google.cloud import aiplatform

# ── Batch prediction from BigQuery ────────────────────────────────────
batch_job = model.batch_predict(
    job_display_name      = "churn-batch-20260316",
    bigquery_source       = "bq://my-project.ml.prediction_input",
    bigquery_destination_prefix = "bq://my-project.ml.prediction_output",
    machine_type          = "n1-standard-4",
    starting_replica_count = 2,
    max_replica_count      = 10,
    sync                   = False,   # Don't block
)
print(f"Batch job: {batch_job.resource_name}")

# ── Batch prediction from GCS JSONL ───────────────────────────────────
batch_job_gcs = model.batch_predict(
    job_display_name    = "churn-batch-gcs",
    gcs_source          = ["gs://my-bucket/batch_input/*.jsonl"],
    gcs_destination_prefix = "gs://my-bucket/batch_output/",
    machine_type        = "n1-standard-4",
    instances_format    = "jsonl",
    predictions_format  = "jsonl",
    generate_explanation = True,      # Include feature attributions
    sync = True,
)

# ── Monitor and get output ────────────────────────────────────────────
batch_job.wait()
print(f"State: {batch_job.state}")
print(f"Output: {batch_job.output_info}")
```

**JSONL input format:**
```json
{"tenure_months": 24, "monthly_charges": 65.5, "contract_type": "Month-to-month"}
{"tenure_months": 60, "monthly_charges": 90.0, "contract_type": "Two year"}
```

**JSONL output format:**
```json
{"instance": {"tenure_months": 24, ...}, "prediction": [0.23, 0.77]}
{"instance": {"tenure_months": 60, ...}, "prediction": [0.91, 0.09]}
```

---

### Online vs. Batch Prediction Decision

| Criterion | Online Prediction | Batch Prediction |
|---|---|---|
| **Latency** | Milliseconds | Minutes–hours |
| **Throughput** | Limited by replicas | Massively parallel |
| **Cost** | Per node-hour (always-on) | Per machine-hour (pay per use) |
| **Input** | Single/small batch | Large dataset |
| **Best for** | Real-time APIs, user-facing | Scoring warehouses, nightly jobs |
| **Min replicas** | ≥ 1 (idle cost) | 0 (no idle cost) |

---

## 10. Vertex AI Feature Store

### Feature Store Architecture

```
Feature Store (project-level resource)
├── Feature Group (linked to BigQuery table/view)
│   ├── Feature: "account_age_days"
│   ├── Feature: "monthly_avg_spend"
│   └── Feature: "num_transactions_30d"
└── Feature View (materialized serving layer)
    ├── Syncs from Feature Group(s)
    ├── Supports online serving (low latency)
    └── Sync schedule: hourly / on-demand
```

---

### Python SDK: Feature Store

```python
from google.cloud.aiplatform import feature_store as fs

PROJECT  = "my-project"
LOCATION = "us-central1"

# ── Create a Feature Store ────────────────────────────────────────────
feature_store = fs.FeatureStore(
    project  = PROJECT,
    location = LOCATION,
)

# ── Create a Feature Group (backed by BigQuery table) ─────────────────
feature_group = feature_store.create_feature_group(
    name         = "user-behavior-features",
    source       = fs.FeatureGroup.BigQuerySource(
        uri        = "bq://my-project.features.user_behavior",
        entity_id_columns = ["user_id"],       # Primary key column(s)
    ),
    labels       = {"domain": "commerce"},
)

# ── Create Feature definitions ────────────────────────────────────────
features = [
    feature_group.create_feature(name="account_age_days",     version_column_name="account_age_days"),
    feature_group.create_feature(name="monthly_avg_spend",    version_column_name="monthly_avg_spend"),
    feature_group.create_feature(name="num_transactions_30d", version_column_name="num_transactions_30d"),
    feature_group.create_feature(name="last_login_days_ago",  version_column_name="last_login_days_ago"),
]

# ── Create a Feature View (materialized serving layer) ────────────────
feature_view = feature_store.create_feature_view(
    name = "user-behavior-view",
    source = fs.FeatureView.FeatureRegistrySource(
        feature_groups = [
            fs.FeatureView.FeatureRegistrySource.FeatureGroup(
                feature_group_id = "user-behavior-features",
                feature_ids      = ["account_age_days", "monthly_avg_spend",
                                    "num_transactions_30d", "last_login_days_ago"],
            )
        ]
    ),
    sync_config = fs.FeatureView.SyncConfig(cron="0 * * * *"),  # Hourly sync
    labels = {"env": "prod"},
)

# ── Trigger a sync ────────────────────────────────────────────────────
sync_response = feature_view.sync()
print(f"Sync: {sync_response.resource_name}")

# ── Online serving — fetch features by entity ID ──────────────────────
result = feature_view.fetch_feature_values(
    id    = "user_12345",      # Entity ID to look up
    data_format = "key-value"
)
print(f"Features for user_12345:")
for k, v in result.key_value_pairs.items():
    print(f"  {k}: {v}")

# ── Batch serving — fetch for multiple entity IDs ─────────────────────
from google.cloud.aiplatform_v1 import FeatureOnlineStoreServiceClient

# Read feature values for training data
batch_result = feature_view.read_feature_values(
    csv_read_instances = "gs://my-bucket/entity_ids.csv",
    output_uri         = "bq://my-project.features.training_batch"
)
```

---

## 11. Vertex AI Experiments & Metadata

### Full Experiment Tracking

```python
from google.cloud import aiplatform
from google.cloud.aiplatform import Artifact, Model as VAIModel

aiplatform.init(
    project    = "my-project",
    location   = "us-central1",
    experiment = "churn-model-experiments",
    experiment_description = "Comparing GBM vs. Neural Net vs. AutoML"
)

# ── Start a run ───────────────────────────────────────────────────────
with aiplatform.start_run(
    run            = "gbm-run-001",
    tensorboard    = "projects/my-project/locations/us-central1/tensorboards/TB_ID",
    resume         = False,
) as run:

    # Log hyperparameters
    aiplatform.log_params({
        "model_type":    "GradientBoosting",
        "n_estimators":  200,
        "max_depth":     5,
        "learning_rate": 0.05,
        "train_date":    "2026-03-16",
    })

    # ... train your model ...

    # Log scalar metrics
    aiplatform.log_metrics({
        "train_auc":  0.94,
        "val_auc":    0.91,
        "test_auc":   0.90,
        "val_f1":     0.88,
        "train_time": 142.5,
    })

    # Log per-epoch metrics (time series)
    for epoch, train_loss in enumerate(epoch_losses):
        aiplatform.log_time_series_metrics(
            {"train_loss": train_loss, "val_loss": val_losses[epoch]},
            step = epoch,
        )

    # Log dataset artifact
    ds_artifact = Artifact.create(
        schema_title = "system.Dataset",
        display_name = "churn-training-dataset",
        uri          = "bq://my-project.ml.churn_training_v3",
        metadata     = {"row_count": 142500, "feature_count": 28}
    )
    aiplatform.log_input_artifact("training_dataset", ds_artifact)

    # Log model artifact
    model_artifact = Artifact.create(
        schema_title = "system.Model",
        display_name = "churn-gbm-v2",
        uri          = "gs://my-bucket/models/churn-v2/",
        metadata     = {"framework": "sklearn", "version": "1.3"}
    )
    aiplatform.log_output_artifact("trained_model", model_artifact)

# ── Compare runs ──────────────────────────────────────────────────────
experiment = aiplatform.Experiment("churn-model-experiments")
runs_df = experiment.get_data_frame()
print(runs_df.sort_values("metric.val_auc", ascending=False).head(5))

# Filter by metric threshold
best_runs = [r for r in experiment.list_runs()
             if r.get_metrics().get("val_auc", 0) >= 0.90]
```

---

## 12. Model Monitoring

### Create a Monitoring Job

```python
from google.cloud import aiplatform
from google.cloud.aiplatform.model_monitoring import (
    ModelMonitoringJob, ObjectiveConfig, ModelMonitoringAlertConfig,
    ThresholdConfig, TrainingDataset,
)

# Create model monitoring job on a deployed endpoint
monitoring_job = aiplatform.ModelDeploymentMonitoringJob.create(
    display_name      = "churn-model-monitoring",
    endpoint          = endpoint.resource_name,
    logging_sampling_strategy = {
        "random_sample_config": {"sample_rate": 0.1}   # Sample 10% of predictions
    },
    model_deployment_monitoring_objective_configs = [
        {
            "deployed_model_id": deployed_model_id,
            "objective_config": {
                "training_dataset": {
                    "data_source": {"bigquery_source": {"input_uri": "bq://proj.ds.training"}},
                    "target_field": "churn",
                },
                "training_prediction_skew_detection_config": {
                    "skew_thresholds": {
                        "tenure_months":   {"value": 0.3},   # Jensen-Shannon or Wasserstein
                        "monthly_charges": {"value": 0.3},
                        "contract_type":   {"value": 0.3},
                    }
                },
                "prediction_drift_detection_config": {
                    "drift_thresholds": {
                        "tenure_months":   {"value": 0.3},
                        "monthly_charges": {"value": 0.3},
                    }
                },
            }
        }
    ],
    model_deployment_monitoring_schedule_config = {
        "monitor_interval": "3600s",      # Check every hour
    },
    model_monitoring_alert_config = {
        "email_alert_config": {
            "user_emails": ["ml-ops@company.com"]
        },
        "enable_logging": True,
    },
    predict_instance_schema_uri = "gs://bucket/schema/predict_schema.yaml",
    analysis_instance_schema_uri = "gs://bucket/schema/analysis_schema.yaml",
)
print(f"Monitoring job: {monitoring_job.resource_name}")
```

---

## 13. Generative AI — Gemini & Foundation Models

### Gemini Model Family

| Model | Context Window | Input Modalities | Best Use Case |
|---|---|---|---|
| **Gemini 2.0 Flash** | 1M tokens | Text, image, video, audio, code | Production apps, speed/cost balance |
| **Gemini 2.0 Pro** | 2M tokens | Text, image, video, audio, code | Complex reasoning, most capable |
| **Gemini 1.5 Flash** | 1M tokens | Text, image, video, audio | Cost-efficient multimodal |
| **Gemini 1.5 Pro** | 2M tokens | Text, image, video, audio | Long-context understanding |

---

### Python: Gemini SDK

```python
import vertexai
from vertexai.generative_models import (
    GenerativeModel, Part, Content, Tool, FunctionDeclaration,
    GenerationConfig, SafetySetting, HarmCategory, HarmBlockThreshold,
    grounding
)

vertexai.init(project="my-project", location="us-central1")

# ── Basic text generation ─────────────────────────────────────────────
model = GenerativeModel(
    model_name = "gemini-2.0-flash-001",
    system_instruction = [
        "You are an expert data analyst.",
        "Always provide concise, technically accurate answers.",
        "Format numerical data in tables when possible.",
    ],
    generation_config = GenerationConfig(
        temperature       = 0.2,
        top_p             = 0.95,
        top_k             = 40,
        candidate_count   = 1,
        max_output_tokens = 2048,
        stop_sequences    = ["END"],
    ),
    safety_settings = [
        SafetySetting(
            category  = HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
            threshold = HarmBlockThreshold.BLOCK_MEDIUM_AND_ABOVE,
        ),
    ],
)

response = model.generate_content("Explain the difference between precision and recall.")
print(response.text)

# ── Count tokens ──────────────────────────────────────────────────────
token_count = model.count_tokens("Explain precision and recall in ML.")
print(f"Input tokens: {token_count.total_tokens}")

# ── Multimodal: text + image ──────────────────────────────────────────
from vertexai.generative_models import Image

image = Image.load_from_file("chart.png")
# Or from GCS:
image_gcs = Part.from_uri("gs://my-bucket/chart.png", mime_type="image/png")

response = model.generate_content([
    image_gcs,
    "Describe the trends shown in this chart and identify any anomalies.",
])
print(response.text)

# ── Multimodal: video analysis ────────────────────────────────────────
video = Part.from_uri("gs://my-bucket/product_demo.mp4", mime_type="video/mp4")
response = model.generate_content([
    video,
    "Summarize the key product features demonstrated in this video.",
])

# ── Streaming response ────────────────────────────────────────────────
response_stream = model.generate_content(
    "Write a 500-word analysis of machine learning trends in 2026.",
    stream = True,
)
for chunk in response_stream:
    print(chunk.text, end="", flush=True)
print()

# ── Multi-turn chat session ───────────────────────────────────────────
chat = model.start_chat(history=[])

response1 = chat.send_message("What is gradient descent?")
print(f"Bot: {response1.text}\n")

response2 = chat.send_message("How does learning rate affect it?")
print(f"Bot: {response2.text}\n")

response3 = chat.send_message("Give me a PyTorch code example.")
print(f"Bot: {response3.text}\n")

# ── Function calling ──────────────────────────────────────────────────
get_weather_tool = Tool(
    function_declarations = [
        FunctionDeclaration(
            name        = "get_current_weather",
            description = "Get current weather for a city",
            parameters  = {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "City name"},
                    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
                },
                "required": ["city"],
            },
        )
    ]
)

# Step 1: Ask model (returns function call)
fc_response = model.generate_content(
    "What is the weather in London right now?",
    tools = [get_weather_tool],
)

# Step 2: Extract function call
function_call = fc_response.candidates[0].function_calls[0]
print(f"Function: {function_call.name}")
print(f"Args:     {dict(function_call.args)}")

# Step 3: Execute function and return result
weather_data = {"temperature": 14, "condition": "Cloudy", "humidity": 78}

# Step 4: Send result back to model
final_response = model.generate_content([
    fc_response.candidates[0].content,
    Content(
        role = "function",
        parts = [Part.from_function_response(
            name     = function_call.name,
            response = weather_data,
        )],
    ),
])
print(f"Answer: {final_response.text}")

# ── Grounding with Google Search ──────────────────────────────────────
grounded_model = GenerativeModel("gemini-2.0-flash-001")
response = grounded_model.generate_content(
    "What are the latest developments in quantum computing?",
    tools = [Tool.from_google_search_retrieval(
        grounding.GoogleSearchRetrieval(disable_attribution=False)
    )],
)
print(response.text)
print(f"Grounding sources: {response.candidates[0].grounding_metadata.search_entry_point}")
```

---

## 14. Embeddings & Vector Search

### Generating Embeddings

```python
from vertexai.language_models import TextEmbeddingModel, TextEmbeddingInput

vertexai.init(project="my-project", location="us-central1")

# ── Text embeddings ───────────────────────────────────────────────────
embedding_model = TextEmbeddingModel.from_pretrained("text-embedding-004")

texts = [
    "How do I cancel my subscription?",
    "What are your pricing plans?",
    "I need help with billing.",
]

# Batch embed with task type for optimal embeddings
inputs = [
    TextEmbeddingInput(text=t, task_type="RETRIEVAL_QUERY")  # or RETRIEVAL_DOCUMENT
    for t in texts
]
embeddings = embedding_model.get_embeddings(inputs)

for text, emb in zip(texts, embeddings):
    print(f"Text: {text[:40]}...")
    print(f"Dims: {len(emb.values)}, First 3: {emb.values[:3]}")
    print()

# ── Multimodal embeddings ─────────────────────────────────────────────
from vertexai.vision_models import MultiModalEmbeddingModel, Image as VImage

mm_model = MultiModalEmbeddingModel.from_pretrained("multimodalembedding@001")

image = VImage.load_from_file("product.jpg")
mm_embeddings = mm_model.get_embeddings(
    image       = image,
    contextual_text = "A blue running shoe",
    dimension   = 1408,
)
print(f"Image embedding dims: {len(mm_embeddings.image_embedding)}")
print(f"Text  embedding dims: {len(mm_embeddings.text_embedding)}")
```

---

### Vector Search (Matching Engine)

```python
from google.cloud import aiplatform
import json
import numpy as np

# ── Create a Vector Search Index ─────────────────────────────────────
# First, prepare index data in JSONL format and upload to GCS:
# {"id": "doc1", "embedding": [0.1, 0.2, ...]}

index = aiplatform.MatchingEngineIndex.create_tree_ah_index(
    display_name     = "product-search-index",
    contents_delta_uri = "gs://my-bucket/embeddings/",  # JSONL embeddings
    dimensions       = 768,                              # Must match embedding model
    approximate_neighbors_count = 150,
    distance_measure_type = "COSINE_DISTANCE",           # DOT_PRODUCT_DISTANCE, SQUARED_L2_DISTANCE
    leaf_node_embedding_count   = 500,
    leaf_nodes_to_search_percent = 7,
    description      = "Product catalog semantic search index",
)
print(f"Index: {index.resource_name}")

# ── Deploy index to an endpoint ───────────────────────────────────────
index_endpoint = aiplatform.MatchingEngineIndexEndpoint.create(
    display_name     = "product-search-endpoint",
    public_endpoint_enabled = True,   # False for VPC private endpoint
)

deployed_index = index_endpoint.deploy_index(
    index              = index,
    deployed_index_id  = "product-search-v1",
    display_name       = "Product Search V1",
    min_replica_count  = 2,
    max_replica_count  = 10,
)
print(f"Deployed: {deployed_index.id}")

# ── Query: find nearest neighbors ────────────────────────────────────
# Embed the query
query_text = "comfortable running shoes for long distance"
query_emb  = embedding_model.get_embeddings([
    TextEmbeddingInput(text=query_text, task_type="RETRIEVAL_QUERY")
])[0].values

# Find nearest neighbors
neighbors = index_endpoint.find_neighbors(
    deployed_index_id = "product-search-v1",
    queries           = [query_emb],
    num_neighbors     = 10,
    filter            = [
        aiplatform.matching_engine.matching_engine_index_endpoint.Namespace(
            name            = "category",
            allow_tokens    = ["running", "sport"],   # Only return these categories
        )
    ],
)

for match in neighbors[0]:
    print(f"ID: {match.id}, Distance: {match.distance:.4f}")

# ── Streaming updates: add new vectors ───────────────────────────────
index.upsert_datapoints(
    datapoints = [
        aiplatform.matching_engine.matching_engine_index.HybridQuery(
            datapoint_id = "new_product_789",
            feature_vector = [0.12, 0.34, ...],       # New embedding
            restricts = [
                aiplatform.matching_engine.matching_engine_index.HybridQuery.Namespace(
                    name="category", string_value="running"
                )
            ]
        )
    ]
)
```

---

### RAG Pipeline

```python
import vertexai
from vertexai.preview import rag
from vertexai.generative_models import GenerativeModel, Tool

vertexai.init(project="my-project", location="us-central1")

# ── Create a RAG corpus ───────────────────────────────────────────────
corpus = rag.create_corpus(
    display_name = "product-knowledge-base",
    description  = "Product documentation and FAQs",
)

# ── Ingest documents ──────────────────────────────────────────────────
rag.import_files(
    corpus_name  = corpus.name,
    paths        = ["gs://my-bucket/docs/"],
    chunk_size   = 512,
    chunk_overlap = 100,
)

# ── Query the RAG corpus ──────────────────────────────────────────────
rag_retrieval_tool = Tool.from_retrieval(
    retrieval = rag.Retrieval(
        source     = rag.VertexRagStore(
            rag_corpora = [corpus.name],
            similarity_top_k  = 5,
            vector_distance_threshold = 0.5,
        ),
    )
)

rag_model = GenerativeModel(
    model_name = "gemini-2.0-flash-001",
    tools      = [rag_retrieval_tool],
)

response = rag_model.generate_content(
    "What is the return policy for running shoes?"
)
print(response.text)
```

---

## 15. Vertex AI Agent Builder & Search

### Enterprise Search Setup

```python
from google.cloud import discoveryengine_v1 as discoveryengine

PROJECT  = "my-project"
LOCATION = "global"
client   = discoveryengine.DataStoreServiceClient()

# ── Create an unstructured data store ─────────────────────────────────
parent    = f"projects/{PROJECT}/locations/{LOCATION}/collections/default_collection"
data_store = discoveryengine.DataStore(
    display_name               = "product-docs-store",
    industry_vertical          = discoveryengine.IndustryVertical.GENERIC,
    content_config             = discoveryengine.DataStore.ContentConfig.CONTENT_REQUIRED,
    solution_types             = [discoveryengine.SolutionType.SOLUTION_TYPE_SEARCH],
)
operation = client.create_data_store(parent=parent, data_store=data_store,
                                     data_store_id="product-docs-store")
store = operation.result()
print(f"Data store: {store.name}")

# ── Import documents from GCS ─────────────────────────────────────────
doc_client = discoveryengine.DocumentServiceClient()
import_op = doc_client.import_documents(
    request = discoveryengine.ImportDocumentsRequest(
        parent = f"{store.name}/branches/default_branch",
        gcs_source = discoveryengine.GcsSource(
            input_uris   = ["gs://my-bucket/docs/*.pdf"],
            data_schema  = "document",
        ),
        reconciliation_mode = discoveryengine.ImportDocumentsRequest.ReconciliationMode.INCREMENTAL,
    )
)
import_op.result()
print("Documents imported")

# ── Search ────────────────────────────────────────────────────────────
search_client = discoveryengine.SearchServiceClient()
serving_config = f"{store.name}/servingConfigs/default_config"

response = search_client.search(
    discoveryengine.SearchRequest(
        serving_config = serving_config,
        query          = "return policy for damaged products",
        page_size      = 5,
        content_search_spec = discoveryengine.SearchRequest.ContentSearchSpec(
            summary_spec = discoveryengine.SearchRequest.ContentSearchSpec.SummarySpec(
                summary_result_count   = 3,
                include_citations      = True,
                ignore_adversarial_query = True,
            )
        ),
    )
)

print(f"Summary: {response.summary.summary_text}")
for result in response.results:
    print(f"  - {result.document.derived_struct_data.get('title', 'Untitled')}")
    print(f"    {result.document.derived_struct_data.get('link', '')}")
```

---

### Agent Builder vs. RAG Engine

| Feature | Agent Builder | RAG Engine |
|---|---|---|
| **Interface** | No-code + API | SDK/API |
| **Search type** | Enterprise search | Semantic retrieval |
| **Data sources** | GCS, BQ, website, JSONL | GCS, Drive, Jira, etc. |
| **Grounding** | ✅ Native for chat | ✅ For Gemini |
| **Best for** | Enterprise document search, chatbots | Custom RAG pipelines |
| **Multi-turn** | ✅ Built-in | Via Gemini chat |
| **Setup complexity** | Low | Medium |

---

## 16. MLOps Best Practices & Patterns

### Model Promotion Workflow

```
Dev Experiment ─────────────► Model Registry (alias: candidate)
       │                              │
       │ Pipeline triggered           │ CI tests pass?
       ▼                              ▼
Staging Endpoint ──────────► Model Registry (alias: challenger)
       │                              │
       │ A/B test (10% traffic)       │ Monitoring OK?
       ▼                              ▼
Production Endpoint ────────► Model Registry (alias: champion)
       │
       └── Old champion → alias removed → archived
```

```bash
# Promote model to production
gcloud ai models add-version-aliases MODEL_ID \
  --region=us-central1 \
  --version=5 \
  --aliases=champion,production

# Remove production from old version
gcloud ai models remove-version-aliases MODEL_ID \
  --region=us-central1 \
  --version=4 \
  --aliases=champion,production
```

---

### Retraining Trigger Architecture

```
Cloud Monitoring (drift alert)
       │
       ▼
Pub/Sub topic: "model-drift-alerts"
       │
       ▼
Cloud Functions (Python)
       │ Triggered on drift threshold breach
       ▼
Vertex AI Pipelines (retraining pipeline)
       │
       ├── Ingest new training data
       ├── Retrain model
       ├── Evaluate (quality gate)
       └── Deploy if passes → New champion

Cloud Scheduler (weekly retraining)
       │
       └──► Vertex AI PipelineJobSchedule
```

---

### Common MLOps Anti-Patterns

| ❌ Anti-Pattern | 🔴 Problem | ✅ Fix |
|---|---|---|
| Training-serving skew | Model degrades silently in production | Use Feature Store for consistent features |
| No experiment tracking | Can't reproduce results | Use Vertex AI Experiments for every training run |
| Hardcoded hyperparameters | Can't tune or reproduce | Use `argparse` + HP Tuning Jobs |
| No model monitoring | Undetected data drift | Set up Model Monitoring job on deployed endpoint |
| Deploying from notebook | No reproducibility, no CI | Use Pipelines for all training and deployment |
| Single endpoint, no traffic split | Risky deployments | Use traffic splits for canary/blue-green |
| Downloading data in training container | Slow startup, dependency on internet | Pre-load data to GCS; use `AIP_TRAINING_DATA_URI` |
| Using default service account | Over-permissive | Create dedicated training/serving SAs |
| No artifact lineage | Can't trace model origin | Use ML Metadata / Pipelines for lineage |
| Always-on max replicas | High idle cost | Set appropriate min/max replicas + autoscaling |

---

## 17. IAM, Security & Networking

### IAM Roles

| Role | Grants |
|---|---|
| `roles/aiplatform.admin` | Full control of all Vertex AI resources |
| `roles/aiplatform.user` | Create/manage training jobs, endpoints, models |
| `roles/aiplatform.viewer` | Read-only access to all Vertex AI resources |
| `roles/aiplatform.serviceAgent` | Vertex AI service account — infra operations |
| `roles/ml.developer` | ML Engine compatibility role (legacy) |

---

### Service Account Architecture

```
User / CI/CD SA
  roles/aiplatform.user (submit jobs, create endpoints)
  roles/iam.serviceAccountUser (act as training/pipeline SA)
        │
        ├── Training SA (custom-training-sa@...)
        │     roles/aiplatform.serviceAgent
        │     roles/storage.objectAdmin (read training data, write models)
        │     roles/bigquery.dataViewer (read BQ training data)
        │
        ├── Pipeline SA (pipeline-sa@...)
        │     roles/aiplatform.user
        │     roles/storage.objectAdmin (pipeline root bucket)
        │     roles/iam.serviceAccountUser (act as training SA)
        │
        └── Prediction SA (prediction-sa@...)
              roles/aiplatform.serviceAgent
              roles/storage.objectReader (read model artifacts)
```

---

### IAM Examples

```bash
# Grant training job submission rights
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:cicd-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"

# Allow CI/CD SA to act as training SA
gcloud iam service-accounts add-iam-policy-binding \
  training-sa@my-project.iam.gserviceaccount.com \
  --member="serviceAccount:cicd-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

# Grant training SA model write access
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:training-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

# Configure CMEK for Vertex AI
gcloud kms keys add-iam-policy-binding vertex-key \
  --location=us-central1 \
  --keyring=vertex-keyring \
  --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-aiplatform.iam.gserviceaccount.com" \
  --role="roles/cloudkms.cryptoKeyEncrypterDecrypter"
```

---

### Private Training (No External IP)

```python
# Configure training job with no external IP (private VPC only)
job = aiplatform.CustomJob(
    display_name      = "private-training-job",
    worker_pool_specs = [...],
    network           = "projects/MY_PROJECT/global/networks/my-private-vpc",
    base_output_dir   = "gs://my-bucket/outputs/",
)
job.run(
    service_account   = "training-sa@my-project.iam.gserviceaccount.com",
    enable_web_access = False,         # No public endpoint for tensorboard
    # No external IP — requires Private Google Access on subnet
)
```

---

## 18. gcloud CLI & SDK Quick Reference

### All Essential gcloud ai Commands

```bash
# ── CUSTOM JOBS ───────────────────────────────────────────────────────
gcloud ai custom-jobs create   --display-name=NAME --region=R --config=FILE
gcloud ai custom-jobs list     --region=R --format="table(name,displayName,state)"
gcloud ai custom-jobs describe JOB_ID --region=R
gcloud ai custom-jobs cancel   JOB_ID --region=R
gcloud ai custom-jobs stream-logs JOB_ID --region=R   # Live logs

# ── HP TUNING JOBS ────────────────────────────────────────────────────
gcloud ai hp-tuning-jobs create   --display-name=NAME --region=R --config=FILE
gcloud ai hp-tuning-jobs list     --region=R
gcloud ai hp-tuning-jobs describe JOB_ID --region=R

# ── MODELS ────────────────────────────────────────────────────────────
gcloud ai models list            --region=R
gcloud ai models describe MODEL_ID --region=R
gcloud ai models delete MODEL_ID --region=R --quiet
gcloud ai models upload          --display-name=NAME --artifact-uri=GCS --region=R
gcloud ai models add-version-aliases MODEL_ID --region=R --version=V --aliases=a1,a2
gcloud ai models list-version-aliases MODEL_ID --region=R

# ── ENDPOINTS ─────────────────────────────────────────────────────────
gcloud ai endpoints create        --display-name=NAME --region=R
gcloud ai endpoints list          --region=R
gcloud ai endpoints describe EP_ID --region=R
gcloud ai endpoints delete EP_ID  --region=R --quiet
gcloud ai endpoints deploy-model EP_ID --model=MODEL_ID --region=R \
  --display-name=NAME --machine-type=TYPE --min-replica-count=1 --max-replica-count=5
gcloud ai endpoints undeploy-model EP_ID --deployed-model-id=ID --region=R
gcloud ai endpoints predict EP_ID --region=R --json-request=request.json

# ── PIPELINES ─────────────────────────────────────────────────────────
gcloud ai pipelines runs list    --region=R
gcloud ai pipelines runs describe PIPELINE_RUN_ID --region=R
gcloud ai pipelines runs cancel  PIPELINE_RUN_ID --region=R

# ── INDEXES (VECTOR SEARCH) ───────────────────────────────────────────
gcloud ai indexes create         --display-name=NAME --region=R --metadata-file=FILE
gcloud ai indexes list           --region=R
gcloud ai indexes describe INDEX_ID --region=R
gcloud ai index-endpoints create --display-name=NAME --region=R --public-endpoint-enabled
gcloud ai index-endpoints deploy-index IE_ID --index=INDEX_ID \
  --deployed-index-id=D_ID --display-name=NAME --region=R

# ── FEATURE STORES ────────────────────────────────────────────────────
gcloud ai feature-groups create  --display-name=NAME --region=R
gcloud ai feature-groups list    --region=R
gcloud ai feature-views create   --feature-group=FG --display-name=NAME --region=R
```

---

### REST API Base URLs

```
Training:        https://us-central1-aiplatform.googleapis.com/v1/projects/PROJECT/locations/REGION/
Prediction:      https://REGION-aiplatform.googleapis.com/v1/projects/PROJECT/locations/REGION/endpoints/EP_ID:predict
Vector Search:   https://REGION-aiplatform.googleapis.com/v1/projects/PROJECT/locations/REGION/indexEndpoints/IE_ID:findNeighbors
GenAI:           https://us-central1-aiplatform.googleapis.com/v1/projects/PROJECT/locations/REGION/publishers/google/models/MODEL:generateContent
Lineage:         https://datalineage.googleapis.com/v1/projects/PROJECT/locations/REGION/
```

---

## 19. Pricing Summary

> ⚠️ Prices are approximate as of early 2026 (us-central1). Verify at [cloud.google.com/vertex-ai/pricing](https://cloud.google.com/vertex-ai/pricing).

### Training Pricing

| Resource | Price |
|---|---|
| n1-standard-8 (8 vCPU, 30 GB) | ~$0.38/hr |
| n1-highmem-8 (8 vCPU, 52 GB) | ~$0.47/hr |
| NVIDIA T4 GPU | ~$0.35/hr |
| NVIDIA A100 (40GB) | ~$2.93/hr |
| NVIDIA H100 (80GB) | ~$10.22/hr |
| Spot VM discount | ~60–80% off standard |

### Prediction Pricing

| Type | Price |
|---|---|
| Online (n1-standard-2) | ~$0.10/node-hr |
| Online (n1-standard-4) | ~$0.19/node-hr |
| Batch prediction | Machine-hr rate (no idle) |

### Generative AI Pricing (Gemini)

| Model | Input | Output |
|---|---|---|
| Gemini 2.0 Flash | $0.10 / 1M tokens | $0.40 / 1M tokens |
| Gemini 2.0 Pro | $1.25 / 1M tokens | $5.00 / 1M tokens |
| Gemini 1.5 Flash | $0.075 / 1M tokens | $0.30 / 1M tokens |
| Gemini 1.5 Pro (≤128K) | $1.25 / 1M tokens | $5.00 / 1M tokens |
| text-embedding-004 | $0.025 / 1M tokens | — |

### Other Components

| Component | Pricing |
|---|---|
| AutoML training | $1.60–$27.00 / node-hour by type |
| Feature Store online serving | $0.03 / 1K read requests |
| Feature Store storage | $0.023 / GB-month |
| Vertex AI Pipelines | $0.03 / pipeline run step |
| Vector Search | $0.023 / GB-month + $0.0004 / 1K queries |

### 💰 Cost Optimization Tips

| Tip | Savings |
|---|---|
| Use Spot VMs for training | 60–80% training cost |
| Set `min_replica_count=0` for low-traffic endpoints | Eliminate idle serving cost |
| Use batch prediction for non-real-time | No idle cost vs. always-on endpoint |
| Sample 10–30% for model monitoring | Reduce logging ingestion cost |
| Use Flash models for simple tasks | 10–20x cheaper than Pro |
| Cache frequent Gemini responses | Avoid redundant token charges |
| Regional selection (Iowa cheapest) | ~10–20% vs. other US regions |

---

## 20. Quick Reference & Comparison Tables

### Vertex AI Component Overview

| Component | Purpose | Billing Unit | Key SDK Class |
|---|---|---|---|
| Custom Training | Run training code | vCPU-hr + GPU-hr | `aiplatform.CustomJob` |
| AutoML | Automated training | Node-hours | `aiplatform.AutoMLTabularTrainingJob` |
| Pipelines | Orchestrate ML workflows | Per step ($0.03) | `aiplatform.PipelineJob` |
| Endpoints | Online prediction | Node-hr | `aiplatform.Endpoint` |
| Batch Prediction | Offline inference | Machine-hr | `aiplatform.BatchPredictionJob` |
| Model Registry | Track/version models | Free | `aiplatform.Model` |
| Feature Store | Feature serving | Storage + requests | `aiplatform.FeatureStore` |
| Experiments | Track params/metrics | Free | `aiplatform.Experiment` |
| Model Monitoring | Skew/drift detection | Logging volume | `ModelDeploymentMonitoringJob` |
| Vector Search | ANN index | GB + queries | `aiplatform.MatchingEngineIndex` |
| Gemini API | Text/multimodal GenAI | Per token | `vertexai.GenerativeModel` |

---

### Pre-built Training Container URIs

| Framework | Version | Container URI |
|---|---|---|
| TensorFlow CPU | 2.13 / py310 | `us-docker.pkg.dev/vertex-ai/training/tf-cpu.2-13.py310:latest` |
| TensorFlow GPU | 2.13 / py310 | `us-docker.pkg.dev/vertex-ai/training/tf-gpu.2-13.py310:latest` |
| PyTorch CPU | 2.2 / py310 | `us-docker.pkg.dev/vertex-ai/training/pytorch-cpu.2-2.py310:latest` |
| PyTorch GPU | 2.2 / py310 | `us-docker.pkg.dev/vertex-ai/training/pytorch-gpu.2-2.py310:latest` |
| scikit-learn | 1.3 / py310 | `us-docker.pkg.dev/vertex-ai/training/sklearn-cpu.1-3.py310:latest` |
| XGBoost | 1.7 / py310 | `us-docker.pkg.dev/vertex-ai/training/xgboost-cpu.1-7.py310:latest` |

---

### Machine Type Quick Reference

| Machine Type | vCPU | RAM | GPU Options | Use Case |
|---|---|---|---|---|
| `n1-standard-4` | 4 | 15 GB | T4, V100 | General training |
| `n1-standard-8` | 8 | 30 GB | T4, V100 | Medium training |
| `n1-highmem-8` | 8 | 52 GB | T4 | Memory-intensive |
| `n2-standard-8` | 8 | 32 GB | — | CPU-intensive |
| `a2-highgpu-1g` | 12 | 85 GB | A100 40GB ×1 | Deep learning |
| `a2-highgpu-8g` | 96 | 680 GB | A100 40GB ×8 | Large-scale DL |
| `a3-highgpu-8g` | 208 | 1872 GB | H100 80GB ×8 | LLM training |

---

### KFP Component I/O Types

| Type | Description | Python Type | Use Case |
|---|---|---|---|
| `str` | Plain string value | `str` | Config, paths, names |
| `int` / `float` / `bool` | Scalar values | primitive | Hyperparameters, metrics |
| `Input[Dataset]` | Read a dataset artifact | Input artifact | Consuming training data |
| `Output[Dataset]` | Write a dataset artifact | Output artifact | Producing training data |
| `Input[Model]` | Read a model artifact | Input artifact | Loading trained model |
| `Output[Model]` | Write a model artifact | Output artifact | Saving trained model |
| `Output[Metrics]` | Log scalar metrics | Output artifact | Tracking metrics in UI |
| `Output[ClassificationMetrics]` | Log confusion matrix | Output artifact | Classification evaluation |
| `Input[Artifact]` | Generic input artifact | Input artifact | Any artifact type |
| `Output[Artifact]` | Generic output artifact | Output artifact | Custom artifact types |

---

### Training/Serving Environment Variables

| Variable | Context | Description |
|---|---|---|
| `AIP_MODEL_DIR` | Training | GCS path to save model artifacts |
| `AIP_TRAINING_DATA_URI` | Training | GCS path to training data |
| `AIP_VALIDATION_DATA_URI` | Training | GCS path to validation data |
| `AIP_TEST_DATA_URI` | Training | GCS path to test data |
| `AIP_STORAGE_URI` | Training | Root GCS path for this job |
| `CLOUD_ML_JOB_ID` | Training | Training job resource name |
| `TF_CONFIG` | Distributed training | JSON config for TF multi-worker |
| `AIP_HTTP_PORT` | Serving | Port for prediction server |
| `AIP_PREDICT_ROUTE` | Serving | HTTP route for predictions |
| `AIP_HEALTH_ROUTE` | Serving | HTTP route for health check |

---

### MLOps Maturity Model

| Level | Description | Vertex AI Components |
|---|---|---|
| **Level 0** | Manual, notebook-based ML | Workbench, manual model upload |
| **Level 1** | Automated training pipelines | Pipelines, Experiments, Model Registry |
| **Level 2** | Automated retraining + CT/CD | Pipelines + Scheduler + Model Monitoring + auto-deploy |

**Level 0 → 1 additions:**
- Vertex AI Pipelines for reproducible training
- Vertex AI Experiments for tracking
- Model Registry with version aliases

**Level 1 → 2 additions:**
- Model Monitoring for drift detection
- Automated retraining via Monitoring → Pub/Sub → Cloud Functions → Pipeline
- Traffic splitting for canary deployments
- Feature Store for training-serving consistency

---

### Anti-Patterns Reference

| ❌ Anti-Pattern | Impact | ✅ Fix |
|---|---|---|
| Training/serving feature skew | Silent accuracy degradation | Use Feature Store for both train and serve |
| No experiment tracking | Irreproducible results | `aiplatform.init(experiment=...)` on every run |
| Manual model deployment | Error-prone, no governance | Use Pipelines + Model Registry aliases |
| Over-provisioned endpoints | 60–80% wasted spend | Set `min_replica_count=1`, enable autoscaling |
| Monolithic training script | Hard to debug, no caching | Break into Pipeline components |
| No model monitoring | Undetected model degradation | Deploy `ModelDeploymentMonitoringJob` |
| Storing secrets in code | Security vulnerability | Use Secret Manager; reference in env vars |
| No GPU quota buffer | Job fails at peak | Request GPU quota increase proactively |
| Using `sync=True` in production | Blocks calling code | Use `sync=False` + poll or Pipelines |
| Gemini in tight loop, no batching | Token cost explosion | Batch prompts; use Flash for simple tasks |

---

*Generated for GCP Vertex AI | Official Docs: [cloud.google.com/vertex-ai/docs](https://cloud.google.com/vertex-ai/docs) | Generative AI: [cloud.google.com/vertex-ai/generative-ai/docs](https://cloud.google.com/vertex-ai/generative-ai/docs)*
