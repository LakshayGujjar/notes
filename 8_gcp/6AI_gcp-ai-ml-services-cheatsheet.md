# 🤖 GCP AI & ML Services — Comprehensive Cheatsheet
## AutoML · Cloud Vision API · Natural Language API · Speech-to-Text · Translation API · Document AI

> **Audience:** Developers, ML engineers, and solution architects building intelligent applications on GCP.
> **Last updated:** March 2026 | Covers all six pre-trained and customizable AI/ML APIs with Python SDK and REST examples.

---

## Table of Contents

1. [Overview & Service Comparison](#1-overview--service-comparison)
2. [AutoML (Vertex AI AutoML)](#2-automl-vertex-ai-automl)
3. [Cloud Vision API](#3-cloud-vision-api)
4. [Cloud Natural Language API](#4-cloud-natural-language-api)
5. [Speech-to-Text API](#5-speech-to-text-api)
6. [Cloud Translation API](#6-cloud-translation-api)
7. [Document AI](#7-document-ai)
8. [Common Patterns & Integration Architectures](#8-common-patterns--integration-architectures)
9. [Error Handling, Quotas & Best Practices](#9-error-handling-quotas--best-practices)
10. [gcloud CLI & REST API Quick Reference](#10-gcloud-cli--rest-api-quick-reference)
11. [Pricing Summary](#11-pricing-summary)
12. [Quick Reference & Comparison Tables](#12-quick-reference--comparison-tables)

---

## 1. Overview & Service Comparison

### Pre-trained APIs vs. Custom Training

```
Pre-trained APIs (no ML expertise, instant use):
  Vision API → General image understanding
  Natural Language API → Text sentiment, entities, syntax
  Speech-to-Text → Audio transcription
  Translation API → Text translation
  Document AI → Document OCR and parsing

Customizable (train on your data):
  AutoML → Train custom models visually (no code ML)
  Vertex AI Custom Training → Full ML engineering control
```

> 💡 **Rule of thumb:** Start with pre-trained APIs. Use AutoML when accuracy on your domain-specific data is insufficient. Use Vertex AI custom training when you need full architectural control.

---

### Service Comparison Table

| Service | Input | Output | Customizable? | Pricing Unit | Primary Use Cases |
|---|---|---|---|---|---|
| **AutoML** | Images, text, tabular, video | Predictions, labels | ✅ Custom models | Node-hours + predictions | Custom classifiers, detectors |
| **Vision API** | Images (JPEG, PNG, GIF, etc.) | Labels, text, faces, objects | ❌ (Product Search ✅) | Per 1K feature calls | Image moderation, OCR, object detection |
| **Natural Language API** | Text (any language) | Sentiment, entities, syntax | ❌ | Per 1K characters | Review analysis, NER, content tagging |
| **Speech-to-Text** | Audio (WAV, FLAC, MP3, etc.) | Transcript text + metadata | ✅ (adaptation) | Per 15-sec increment | Call transcription, voice commands |
| **Translation API** | Text or HTML | Translated text | ✅ (glossaries, AutoML) | Per character | Localization, multilingual content |
| **Document AI** | PDFs, images of documents | Structured data, entities | ✅ (custom processors) | Per page | Invoice/receipt parsing, form extraction |

---

### Decision Guide

| Problem | Best Service |
|---|---|
| Classify my own product images | AutoML Vision |
| Detect objects in any general image | Vision API (`OBJECT_LOCALIZATION`) |
| Extract text from a photo | Vision API (`DOCUMENT_TEXT_DETECTION`) |
| Analyze customer review sentiment | Natural Language API (`analyzeSentiment`) |
| Extract people/companies from articles | Natural Language API (`analyzeEntities`) |
| Transcribe a call center recording | Speech-to-Text (`phone_call` model) |
| Translate support tickets to English | Translation API (v2 or v3) |
| Extract fields from invoices at scale | Document AI (Invoice Parser) |
| Extract text from complex PDFs | Document AI (Document OCR) |
| Custom tabular ML without code | AutoML Tabular |
| Moderate user-generated text | Natural Language API (`moderateText`) |

---

### Required APIs & IAM Roles

```bash
# Enable all six service APIs
gcloud services enable \
  vision.googleapis.com \
  language.googleapis.com \
  speech.googleapis.com \
  translate.googleapis.com \
  documentai.googleapis.com \
  aiplatform.googleapis.com \
  --project=MY_PROJECT

# Common IAM roles
# roles/ml.developer          → AutoML training and prediction
# roles/aiplatform.user       → Vertex AI / AutoML operations
# roles/documentai.editor     → Document AI processor management
# roles/documentai.apiUser    → Call Document AI process endpoints
# roles/storage.objectViewer  → Read GCS input files (all services)
# roles/storage.objectCreator → Write GCS output files (batch jobs)
```

---

### Client Library Installation

```bash
# Install all six client libraries
pip install \
  google-cloud-vision \
  google-cloud-language \
  google-cloud-speech \
  google-cloud-translate \
  google-cloud-documentai \
  google-cloud-aiplatform \
  vertexai

# Common imports pattern
from google.cloud import vision
from google.cloud import language_v1
from google.cloud import speech
from google.cloud import translate_v2 as translate
from google.cloud import translate_v3
from google.cloud import documentai
from google.cloud import aiplatform
```

---

### Authentication Patterns

```python
# Option 1: Application Default Credentials (ADC) — recommended
# Run: gcloud auth application-default login
# Client libraries pick up ADC automatically
client = vision.ImageAnnotatorClient()

# Option 2: Service account key file
import os
os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = "/path/to/key.json"
client = vision.ImageAnnotatorClient()

# Option 3: Explicit credentials
from google.oauth2 import service_account
credentials = service_account.Credentials.from_service_account_file(
    "/path/to/key.json",
    scopes=["https://www.googleapis.com/auth/cloud-platform"]
)
client = vision.ImageAnnotatorClient(credentials=credentials)
```

---

## 2. 🤖 AutoML (Vertex AI AutoML)

### AutoML Service Types

| AutoML Type | Task | Vertex AI Job Class | Export Formats |
|---|---|---|---|
| **AutoML Vision** | Image classification | `AutoMLImageTrainingJob` | TFLite, Core ML, Edge TPU, TF SavedModel |
| **AutoML Vision** | Object detection | `AutoMLImageTrainingJob` | TFLite, Edge TPU, TF SavedModel |
| **AutoML NL** | Text classification | `AutoMLTextTrainingJob` | TF SavedModel |
| **AutoML NL** | Entity extraction | `AutoMLTextTrainingJob` | TF SavedModel |
| **AutoML NL** | Sentiment analysis | `AutoMLTextTrainingJob` | TF SavedModel |
| **AutoML Tabular** | Classification | `AutoMLTabularTrainingJob` | — |
| **AutoML Tabular** | Regression | `AutoMLTabularTrainingJob` | — |
| **AutoML Tabular** | Forecasting | `AutoMLForecastingTrainingJob` | — |
| **AutoML Video** | Classification | `AutoMLVideoTrainingJob` | TF SavedModel |

---

### AutoML Workflow

```
1. Prepare Data
   ├── Images: GCS folder + CSV import file with labels
   ├── Text:   GCS JSONL file with text + label pairs
   ├── Tabular: GCS CSV or BigQuery table
   └── Video:  GCS folder + CSV import file

2. Create Dataset    → aiplatform.ImageDataset.create()
3. Train Model       → AutoMLImageTrainingJob.run(dataset=...)
4. Evaluate          → model.list_model_evaluations()
5. Deploy            → model.deploy(endpoint=..., machine_type=...)
6. Predict           → endpoint.predict(instances=[...])
```

---

### Dataset Format Examples

```
# Image classification CSV import file (GCS):
gs://my-bucket/images/cat1.jpg,cat
gs://my-bucket/images/dog1.jpg,dog
gs://my-bucket/images/cat2.jpg,cat,VALIDATION
gs://my-bucket/images/dog2.jpg,dog,TEST

# Text classification JSONL import file:
{"text_snippet": {"content": "Love this product!"}, "annotations": [{"display_name": "positive"}]}
{"text_snippet": {"content": "Terrible quality."}, "annotations": [{"display_name": "negative"}]}

# Tabular CSV (first row = header, target column specified during training):
feature1,feature2,feature3,label
0.5,blue,3,churn
0.2,red,7,retained
```

---

### Python SDK: AutoML Vision

```python
from google.cloud import aiplatform

aiplatform.init(project="my-project", location="us-central1",
                staging_bucket="gs://my-staging-bucket")

# ── 1. Create Image Classification Dataset ────────────────────────────
dataset = aiplatform.ImageDataset.create(
    display_name      = "product-images-dataset",
    gcs_source        = ["gs://my-bucket/import.csv"],
    import_schema_uri = aiplatform.schema.dataset.ioformat.image.single_label_classification,
    labels            = {"env": "prod"},
)
print(f"Dataset: {dataset.resource_name}")

# ── 2. Train AutoML Vision Model ──────────────────────────────────────
job = aiplatform.AutoMLImageTrainingJob(
    display_name    = "product-classifier-v1",
    prediction_type = "classification",          # or "object_detection"
    multi_label     = False,                     # True for multi-label
    model_type      = "CLOUD",                   # CLOUD or MOBILE_TF_LOW_LATENCY_1
    base_model      = None,
)

model = job.run(
    dataset                   = dataset,
    model_display_name        = "product-classifier-model-v1",
    training_fraction_split   = 0.8,
    validation_fraction_split = 0.1,
    test_fraction_split       = 0.1,
    budget_milli_node_hours   = 8000,    # 8 node-hours
    disable_early_stopping    = False,
    sync                      = True,
)
print(f"Model: {model.resource_name}")

# ── 3. Evaluate Model ─────────────────────────────────────────────────
evaluations = model.list_model_evaluations()
for eval_obj in evaluations:
    metrics = eval_obj.metrics
    print(f"AUC-ROC:   {metrics.get('auRoc', 'N/A'):.4f}")
    print(f"Precision: {metrics.get('precision', 'N/A'):.4f}")
    print(f"Recall:    {metrics.get('recall', 'N/A'):.4f}")
    print(f"F1 Score:  {metrics.get('f1Score', 'N/A'):.4f}")

# ── 4. Deploy to Endpoint ─────────────────────────────────────────────
endpoint = aiplatform.Endpoint.create(display_name="product-classifier-endpoint")
model.deploy(
    endpoint          = endpoint,
    machine_type      = "n1-standard-4",
    min_replica_count = 1,
    max_replica_count = 5,
)

# ── 5. Online Prediction ──────────────────────────────────────────────
import base64
with open("product_image.jpg", "rb") as f:
    image_bytes = base64.b64encode(f.read()).decode("utf-8")

response = endpoint.predict(
    instances = [{"content": image_bytes}]
)
for pred in response.predictions:
    for label, score in zip(pred["displayNames"], pred["confidences"]):
        print(f"  {label}: {score:.3f}")

# ── 6. Batch Prediction from GCS ─────────────────────────────────────
batch_job = model.batch_predict(
    job_display_name       = "product-classifier-batch-001",
    gcs_source             = ["gs://my-bucket/batch_input/*.jsonl"],
    gcs_destination_prefix = "gs://my-bucket/batch_output/",
    machine_type           = "n1-standard-4",
    instances_format       = "jsonl",
    predictions_format     = "jsonl",
    sync                   = True,
)
print(f"Batch output: {batch_job.output_info.gcs_output_directory}")

# ── 7. Export Edge Model (for mobile/IoT) ────────────────────────────
model.export_model(
    export_format_id = "tflite",            # tflite, core-ml, edgetpu-tflite
    artifact_destination = "gs://my-bucket/exported-model/",
)
```

---

### gcloud: AutoML

```bash
# Create a dataset
gcloud ai datasets create \
  --display-name="product-images" \
  --metadata-schema-uri="gs://google-cloud-aiplatform/schema/dataset/metadata/image_1.0.0.yaml" \
  --region=us-central1

# List datasets
gcloud ai datasets list --region=us-central1

# Train AutoML model (via training pipeline config)
gcloud ai training-pipelines create \
  --display-name="product-automl-train" \
  --pipeline-spec-uri="gs://google-cloud-aiplatform/schema/trainingjob/definition/automl_image_classification_1.0.0.yaml" \
  --training-task-inputs-file=training_config.json \
  --region=us-central1

# Deploy model to endpoint
gcloud ai endpoints deploy-model ENDPOINT_ID \
  --region=us-central1 \
  --model=MODEL_ID \
  --display-name="deployed-product-classifier" \
  --machine-type=n1-standard-4 \
  --min-replica-count=1

# Online predict
gcloud ai endpoints predict ENDPOINT_ID \
  --region=us-central1 \
  --json-request=predict_request.json
```

---

### AutoML vs. Custom Training Decision Guide

| Criterion | AutoML | Custom Training |
|---|---|---|
| **ML expertise needed** | Low | High |
| **Time to first model** | Hours | Days–weeks |
| **Custom architecture** | ❌ | ✅ |
| **Accuracy ceiling** | Good (~85–92% typical) | Higher (SOTA possible) |
| **Edge export** | ✅ (TFLite, CoreML) | ✅ (any format) |
| **Cost for small datasets** | ✅ Lower | ❌ Higher (dev time) |
| **Hyperparameter control** | ❌ | ✅ |
| **Best for** | Proof-of-concept, domain-specific classification | Production, custom architectures |

---

## 3. 👁️ Cloud Vision API

### Feature Types Reference

| Feature Type | Description | Response Key |
|---|---|---|
| `LABEL_DETECTION` | General image content labels | `label_annotations` |
| `OBJECT_LOCALIZATION` | Detect + locate multiple objects with bounding boxes | `localized_object_annotations` |
| `TEXT_DETECTION` | OCR for sparse text (signs, labels) | `text_annotations` |
| `DOCUMENT_TEXT_DETECTION` | Dense OCR with structural hierarchy | `full_text_annotation` |
| `FACE_DETECTION` | Face bounding boxes + emotions + landmarks | `face_annotations` |
| `LANDMARK_DETECTION` | Identify geographic landmarks | `landmark_annotations` |
| `LOGO_DETECTION` | Identify brand logos | `logo_annotations` |
| `SAFE_SEARCH_DETECTION` | Adult/violence/medical likelihood | `safe_search_annotation` |
| `IMAGE_PROPERTIES` | Dominant colors with fractions | `image_properties_annotation` |
| `CROP_HINTS` | Best crop coordinates for given aspect ratio | `crop_hints_annotation` |
| `WEB_DETECTION` | Matching web pages, similar images | `web_detection` |
| `PRODUCT_SEARCH` | Find similar products in a product catalog | `product_search_results` |

---

### Python: Vision API Examples

```python
from google.cloud import vision
import base64

client = vision.ImageAnnotatorClient()

# ── Label Detection from local file ───────────────────────────────────
with open("image.jpg", "rb") as f:
    content = f.read()

image   = vision.Image(content=content)
response = client.label_detection(image=image, max_results=10)

for label in response.label_annotations:
    print(f"Label: {label.description:30s} Score: {label.score:.3f}")

# ── Label Detection from GCS URI ──────────────────────────────────────
image_gcs = vision.Image(source=vision.ImageSource(
    gcs_image_uri="gs://my-bucket/product.jpg"
))
response = client.label_detection(image=image_gcs)

# ── DOCUMENT_TEXT_DETECTION: Full OCR with structure ─────────────────
response = client.document_text_detection(image=image)
full_text = response.full_text_annotation

print(f"Full text:\n{full_text.text}\n")

for page in full_text.pages:
    for block in page.blocks:
        block_text = ""
        for para in block.paragraphs:
            for word in para.words:
                word_text = "".join([s.text for s in word.symbols])
                block_text += word_text + " "
        print(f"Block [{block.block_type.name}]: {block_text.strip()}")
        # Block has bounding_box.vertices for position

# ── Object Localization with Bounding Boxes ───────────────────────────
response = client.object_localization(image=image)
for obj in response.localized_object_annotations:
    print(f"Object: {obj.name:20s} Confidence: {obj.score:.3f}")
    verts = [(v.x, v.y) for v in obj.bounding_poly.normalized_vertices]
    print(f"  Bounding box (normalized): {verts}")

# ── Face Detection with Emotions ──────────────────────────────────────
response = client.face_detection(image=image)
for i, face in enumerate(response.face_annotations):
    likelihood = vision.Likelihood
    print(f"Face {i+1}:")
    print(f"  Joy:      {likelihood(face.joy_likelihood).name}")
    print(f"  Sorrow:   {likelihood(face.sorrow_likelihood).name}")
    print(f"  Anger:    {likelihood(face.anger_likelihood).name}")
    print(f"  Surprise: {likelihood(face.surprise_likelihood).name}")
    print(f"  Detection confidence: {face.detection_confidence:.3f}")
    bbox = face.bounding_poly.vertices
    print(f"  Bounding box: ({bbox[0].x},{bbox[0].y}) to ({bbox[2].x},{bbox[2].y})")

# ── Safe Search Detection ─────────────────────────────────────────────
response = client.safe_search_detection(image=image)
safe = response.safe_search_annotation
names = ("UNKNOWN", "VERY_UNLIKELY", "UNLIKELY", "POSSIBLE", "LIKELY", "VERY_LIKELY")
print(f"Adult:    {names[safe.adult]}")
print(f"Spoof:    {names[safe.spoof]}")
print(f"Medical:  {names[safe.medical]}")
print(f"Violence: {names[safe.violence]}")
print(f"Racy:     {names[safe.racy]}")

# ── Batched Multi-Feature Request ────────────────────────────────────
from google.cloud.vision_v1 import AnnotateImageRequest, Feature

requests = [
    AnnotateImageRequest(
        image    = image,
        features = [
            Feature(type_=Feature.Type.LABEL_DETECTION,    max_results=5),
            Feature(type_=Feature.Type.SAFE_SEARCH_DETECTION),
            Feature(type_=Feature.Type.IMAGE_PROPERTIES),
        ],
    )
]
batch_response = client.batch_annotate_images(requests=requests)
result = batch_response.responses[0]
print(f"Labels: {[l.description for l in result.label_annotations]}")

# ── Async Batch Annotation (GCS → GCS) ────────────────────────────────
from google.cloud.vision_v1 import (
    AsyncAnnotateFileRequest, AsyncBatchAnnotateFilesRequest,
    InputConfig, OutputConfig, GcsSource, GcsDestination
)

async_requests = [
    AsyncAnnotateFileRequest(
        features    = [Feature(type_=Feature.Type.DOCUMENT_TEXT_DETECTION)],
        input_config = InputConfig(
            gcs_source   = GcsSource(uri="gs://my-bucket/docs/invoice.pdf"),
            mime_type    = "application/pdf",
        ),
        output_config = OutputConfig(
            gcs_destination = GcsDestination(uri="gs://my-bucket/ocr-output/invoice/"),
            batch_size      = 2,
        ),
    )
]
operation = client.async_batch_annotate_files(requests=async_requests)
result = operation.result(timeout=300)
print(f"Output written to GCS")
```

---

### REST API: Vision API

```bash
TOKEN=$(gcloud auth print-access-token)

# Label detection via REST
curl -X POST \
  "https://vision.googleapis.com/v1/images:annotate" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "requests": [{
      "image": {"source": {"imageUri": "gs://my-bucket/image.jpg"}},
      "features": [
        {"type": "LABEL_DETECTION", "maxResults": 10},
        {"type": "SAFE_SEARCH_DETECTION"}
      ]
    }]
  }'

# TEXT_DETECTION from base64 image
curl -X POST \
  "https://vision.googleapis.com/v1/images:annotate" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "requests": [{
      "image": {"content": "'$(base64 -w0 image.jpg)'"},
      "features": [{"type": "DOCUMENT_TEXT_DETECTION"}]
    }]
  }'
```

---

### gcloud: Vision API

```bash
# Detect labels
gcloud ml vision detect-labels gs://my-bucket/image.jpg

# Detect text
gcloud ml vision detect-text gs://my-bucket/sign.jpg

# Detect objects
gcloud ml vision detect-objects gs://my-bucket/photo.jpg

# Detect faces
gcloud ml vision detect-faces gs://my-bucket/portrait.jpg

# Safe search
gcloud ml vision detect-safe-search gs://my-bucket/image.jpg

# Document OCR (dense text)
gcloud ml vision detect-document gs://my-bucket/document.pdf
```

---

### Vision API Limits & Notes

| Limit | Value |
|---|---|
| Max image size (REST) | 20 MB |
| Max base64 image size | 75 MB |
| Max requests per minute | 1,800 |
| Max features per request | 50 |
| Max images per batch request | 16 |
| PDF max pages (`DOCUMENT_TEXT_DETECTION`) | 2,000 pages |

> 💡 **TEXT_DETECTION vs. DOCUMENT_TEXT_DETECTION:** Use `TEXT_DETECTION` for sparse text (street signs, product labels). Use `DOCUMENT_TEXT_DETECTION` for dense documents — it returns a full hierarchy (page → block → paragraph → word → symbol) with bounding boxes.

---

## 4. 📊 Cloud Natural Language API

### Feature Types Reference

| Method | Input | Returns | Min Length |
|---|---|---|---|
| `analyze_sentiment` | Text | Document + sentence sentiment (score, magnitude) | Any |
| `analyze_entities` | Text | Named entities with type, salience, Wikipedia link | Any |
| `analyze_entity_sentiment` | Text | Entities + per-entity sentiment | Any |
| `analyze_syntax` | Text | Tokens with POS tags, dependency parse, lemmas | Any |
| `classify_text` | Text | IAB content categories with confidence | 20+ words |
| `moderate_text` | Text | Harm categories with confidence scores | Any |

**Sentiment score:** -1.0 (very negative) to +1.0 (very positive)
**Magnitude:** 0 to ∞ — strength of emotion regardless of sign

---

### Python: Natural Language API

```python
from google.cloud import language_v1

client = language_v1.LanguageServiceClient()

def make_doc(text, type_="PLAIN_TEXT", language=""):
    return language_v1.Document(
        content       = text,
        type_         = language_v1.Document.Type[type_],
        language      = language,
    )

# ── Sentiment Analysis ────────────────────────────────────────────────
text = """
The product quality exceeded my expectations. The delivery was fast
and the customer service team was incredibly helpful. Minor issue with
the packaging, but overall an outstanding experience.
"""
doc      = make_doc(text)
response = client.analyze_sentiment(
    request={"document": doc, "encoding_type": language_v1.EncodingType.UTF8}
)
ds = response.document_sentiment
print(f"Document sentiment → score: {ds.score:.2f}, magnitude: {ds.magnitude:.2f}")

for sentence in response.sentences:
    ss = sentence.sentiment
    print(f"  '{sentence.text.content[:60]}...'")
    print(f"  Score: {ss.score:.2f}, Magnitude: {ss.magnitude:.2f}")

# ── Entity Analysis ───────────────────────────────────────────────────
text2 = "Apple Inc. CEO Tim Cook announced iPhone 16 at their Cupertino headquarters."
response = client.analyze_entities(
    request={"document": make_doc(text2), "encoding_type": language_v1.EncodingType.UTF8}
)
for entity in sorted(response.entities, key=lambda e: e.salience, reverse=True):
    print(f"Entity: {entity.name:25s} Type: {language_v1.Entity.Type(entity.type_).name:20s} Salience: {entity.salience:.3f}")
    if entity.metadata.get("wikipedia_url"):
        print(f"  Wikipedia: {entity.metadata['wikipedia_url']}")
    for mention in entity.mentions:
        print(f"  Mention: '{mention.text.content}' ({language_v1.EntityMention.Type(mention.type_).name})")

# ── Entity Sentiment Analysis (product reviews) ───────────────────────
review = "The battery life is amazing but the camera quality is disappointing."
response = client.analyze_entity_sentiment(
    request={"document": make_doc(review), "encoding_type": language_v1.EncodingType.UTF8}
)
for entity in response.entities:
    es = entity.sentiment
    print(f"Entity: {entity.name:20s} Sentiment: {es.score:+.2f} (mag: {es.magnitude:.2f})")

# ── Syntax Analysis ────────────────────────────────────────────────────
text3 = "The quick brown fox jumps over the lazy dog."
response = client.analyze_syntax(
    request={"document": make_doc(text3), "encoding_type": language_v1.EncodingType.UTF8}
)
from collections import Counter
pos_counts = Counter()
print(f"{'Token':15s} {'Lemma':15s} {'POS':12s} {'Dep Label'}")
print("-" * 55)
for token in response.tokens:
    pos  = language_v1.PartOfSpeech.Tag(token.part_of_speech.tag).name
    dep  = language_v1.DependencyEdge.Label(token.dependency_edge.label).name
    print(f"{token.text.content:15s} {token.lemma:15s} {pos:12s} {dep}")
    pos_counts[pos] += 1
print(f"\nPOS distribution: {dict(pos_counts)}")

# ── Content Classification ────────────────────────────────────────────
article = """
Machine learning is transforming the healthcare industry.
Neural networks now detect cancer from medical imaging with
accuracy surpassing human radiologists in several studies.
"""
response = client.classify_text(request={"document": make_doc(article)})
for category in response.categories:
    print(f"Category: {category.name:50s} Confidence: {category.confidence:.3f}")

# ── Moderation (Toxic Content Detection) ─────────────────────────────
ugc_text = "This content may contain harmful material for testing moderation."
response = client.moderate_text(request={"document": make_doc(ugc_text)})
print("Moderation results:")
for category in response.moderation_categories:
    if category.confidence > 0.5:
        print(f"  ⚠️  {category.name}: {category.confidence:.3f}")
```

---

### REST API: Natural Language

```bash
TOKEN=$(gcloud auth print-access-token)

# Sentiment analysis
curl -X POST \
  "https://language.googleapis.com/v1/documents:analyzeSentiment" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "document": {"type": "PLAIN_TEXT", "content": "I love this product!"},
    "encodingType": "UTF8"
  }'

# Entity analysis
curl -X POST \
  "https://language.googleapis.com/v1/documents:analyzeEntities" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "document": {"type": "PLAIN_TEXT", "content": "Google was founded by Larry Page in 1998."},
    "encodingType": "UTF8"
  }'
```

> 💡 **Entity types:** `PERSON`, `LOCATION`, `ORGANIZATION`, `EVENT`, `WORK_OF_ART`, `CONSUMER_GOOD`, `OTHER`, `PHONE_NUMBER`, `ADDRESS`, `DATE`, `NUMBER`, `PRICE`

> ⚠️ `classifyText` requires **at least 20 words**. Shorter text returns an error. Use `analyzeSentiment` or `analyzeEntities` for short strings.

---

## 5. 🎤 Speech-to-Text API

### Recognition Models Reference

| Model | Best For | Enhanced Available |
|---|---|---|
| `latest_long` | Long audio files (>1 min), general | ✅ |
| `latest_short` | Short commands and queries (<1 min) | ✅ |
| `phone_call` | Telephone audio (8kHz), noisy | ✅ |
| `video` | Video soundtracks, meetings | ✅ |
| `medical_dictation` | Clinical notes, medical terms | ✅ |
| `medical_conversation` | Doctor-patient conversations | ✅ |
| `command_and_search` | Search queries, voice commands | ❌ |

---

### RecognitionConfig Parameters

| Parameter | Type | Description |
|---|---|---|
| `encoding` | Enum | Audio encoding (LINEAR16, FLAC, MP3, etc.) |
| `sample_rate_hertz` | int | Sample rate (8000, 16000, 44100, etc.) |
| `language_code` | str | BCP-47 language code (e.g., "en-US") |
| `alternative_language_codes` | list | Up to 3 alternative languages |
| `max_alternatives` | int | Number of transcript alternatives (1–30) |
| `profanity_filter` | bool | Filter offensive content |
| `enable_word_time_offsets` | bool | Per-word start/end timestamps |
| `enable_word_confidence` | bool | Per-word confidence scores |
| `enable_automatic_punctuation` | bool | Auto-add punctuation and capitalization |
| `model` | str | Recognition model name |
| `use_enhanced` | bool | Use enhanced (paid) model variant |
| `enable_speaker_diarization` | bool | Identify multiple speakers |
| `min_speaker_count` | int | Minimum expected speakers |
| `max_speaker_count` | int | Maximum expected speakers |

---

### Python: Speech-to-Text

```python
from google.cloud import speech
import io

client = speech.SpeechClient()

# ── Synchronous Recognition (local file, <1 min) ──────────────────────
with open("audio.wav", "rb") as f:
    audio_content = f.read()

audio  = speech.RecognitionAudio(content=audio_content)
config = speech.RecognitionConfig(
    encoding                   = speech.RecognitionConfig.AudioEncoding.LINEAR16,
    sample_rate_hertz          = 16000,
    language_code              = "en-US",
    enable_automatic_punctuation = True,
    max_alternatives           = 2,
    model                      = "latest_long",
    use_enhanced               = True,
)

response = client.recognize(config=config, audio=audio)
for result in response.results:
    alt = result.alternatives[0]
    print(f"Transcript: {alt.transcript}")
    print(f"Confidence: {alt.confidence:.3f}")

# ── Asynchronous Recognition (GCS, long audio) ───────────────────────
audio_gcs = speech.RecognitionAudio(
    uri="gs://my-bucket/long_audio/meeting_recording.flac"
)
config_async = speech.RecognitionConfig(
    encoding                   = speech.RecognitionConfig.AudioEncoding.FLAC,
    sample_rate_hertz          = 44100,
    language_code              = "en-US",
    enable_automatic_punctuation = True,
    enable_word_time_offsets   = True,
    enable_word_confidence     = True,
    model                      = "video",
)

operation = client.long_running_recognize(config=config_async, audio=audio_gcs)
print("Waiting for operation to complete...")
response = operation.result(timeout=3600)   # Wait up to 1 hour

full_transcript = ""
for result in response.results:
    alt = result.alternatives[0]
    full_transcript += alt.transcript + " "

    # Word-level timestamps
    for word_info in alt.words:
        start = word_info.start_time.total_seconds()
        end   = word_info.end_time.total_seconds()
        conf  = word_info.confidence
        print(f"  [{start:.1f}s–{end:.1f}s] {word_info.word:20s} ({conf:.3f})")

print(f"\nFull transcript:\n{full_transcript}")

# ── Speaker Diarization ───────────────────────────────────────────────
diarization_config = speech.SpeakerDiarizationConfig(
    enable_speaker_diarization = True,
    min_speaker_count          = 2,
    max_speaker_count          = 4,
)
config_diar = speech.RecognitionConfig(
    encoding                   = speech.RecognitionConfig.AudioEncoding.LINEAR16,
    sample_rate_hertz          = 8000,
    language_code              = "en-US",
    enable_automatic_punctuation = True,
    diarization_config         = diarization_config,
    model                      = "phone_call",
    use_enhanced               = True,
)

response = client.long_running_recognize(config=config_diar, audio=audio_gcs).result()
# Speaker tags are on the final result's last alternative
result   = response.results[-1]
words    = result.alternatives[0].words
current_speaker = None
current_text    = []
for word_info in words:
    if word_info.speaker_tag != current_speaker:
        if current_text:
            print(f"Speaker {current_speaker}: {' '.join(current_text)}")
        current_speaker = word_info.speaker_tag
        current_text    = []
    current_text.append(word_info.word)
if current_text:
    print(f"Speaker {current_speaker}: {' '.join(current_text)}")

# ── Speech Adaptation (Custom Vocabulary) ─────────────────────────────
speech_context = speech.SpeechContext(
    phrases = ["Vertex AI", "BigQuery", "Cloud Spanner", "Dataplex"],
    boost   = 15.0,    # 1–20; higher = stronger boost
)
config_adapted = speech.RecognitionConfig(
    encoding        = speech.RecognitionConfig.AudioEncoding.LINEAR16,
    sample_rate_hertz = 16000,
    language_code   = "en-US",
    speech_contexts = [speech_context],
    model           = "latest_long",
)

# ── Streaming Recognition (real-time microphone) ─────────────────────
import pyaudio
import queue
import threading

RATE       = 16000
CHUNK      = int(RATE / 10)   # 100ms chunks
audio_queue = queue.Queue()

def mic_callback(in_data, frame_count, time_info, status):
    audio_queue.put(in_data)
    return (None, pyaudio.paContinue)

def generate_requests():
    yield speech.StreamingRecognizeRequest(
        streaming_config = speech.StreamingRecognitionConfig(
            config        = speech.RecognitionConfig(
                encoding          = speech.RecognitionConfig.AudioEncoding.LINEAR16,
                sample_rate_hertz = RATE,
                language_code     = "en-US",
                enable_automatic_punctuation = True,
            ),
            interim_results = True,
        )
    )
    while True:
        chunk = audio_queue.get()
        if chunk is None:
            break
        yield speech.StreamingRecognizeRequest(audio_content=chunk)

p   = pyaudio.PyAudio()
stream = p.open(format=pyaudio.paInt16, channels=1, rate=RATE,
                input=True, frames_per_buffer=CHUNK, stream_callback=mic_callback)
stream.start_stream()

try:
    responses = client.streaming_recognize(generate_requests())
    for response in responses:
        for result in response.results:
            alt = result.alternatives[0]
            if result.is_final:
                print(f"Final:   {alt.transcript}")
            else:
                print(f"Interim: {alt.transcript}", end="\r")
except KeyboardInterrupt:
    pass
finally:
    stream.stop_stream()
    stream.close()
    p.terminate()
    audio_queue.put(None)
```

---

### REST API: Speech-to-Text

```bash
TOKEN=$(gcloud auth print-access-token)

# Synchronous recognition
curl -X POST \
  "https://speech.googleapis.com/v1/speech:recognize" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "config": {
      "encoding": "LINEAR16",
      "sampleRateHertz": 16000,
      "languageCode": "en-US",
      "enableAutomaticPunctuation": true,
      "model": "latest_long"
    },
    "audio": {
      "uri": "gs://my-bucket/audio/short_clip.wav"
    }
  }'

# Long-running async recognition
curl -X POST \
  "https://speech.googleapis.com/v1/speech:longrunningrecognize" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "config": {
      "encoding": "FLAC",
      "sampleRateHertz": 44100,
      "languageCode": "en-US",
      "enableWordTimeOffsets": true
    },
    "audio": {"uri": "gs://my-bucket/audio/long_recording.flac"}
  }'
```

---

## 6. 🌐 Cloud Translation API

### v2 (Basic) vs. v3 (Advanced)

| Feature | Basic (v2) | Advanced (v3) |
|---|---|---|
| Simple translation | ✅ | ✅ |
| Language detection | ✅ | ✅ |
| HTML translation | ✅ | ✅ |
| Glossaries | ❌ | ✅ |
| Batch translation | ❌ | ✅ |
| Custom AutoML models | ❌ | ✅ |
| Project/location scope | ❌ | ✅ |
| Best for | Simple scripts, prototyping | Production, enterprise |

---

### Python: Translation API

```python
from google.cloud import translate_v2 as translate
from google.cloud import translate_v3

# ────────────────────────── v2 (Basic) ───────────────────────────────

client_v2 = translate.Client()

# Basic translation
result = client_v2.translate("Hello, how are you?", target_language="es")
print(f"Original:   {result['input']}")
print(f"Translated: {result['translatedText']}")
print(f"Detected language: {result.get('detectedSourceLanguage', 'N/A')}")

# Auto-detect and translate
texts = ["Bonjour le monde", "Hola mundo", "Ciao mondo", "Hello world"]
results = client_v2.translate(
    texts,
    target_language = "en",
    format_          = "text",    # "text" or "html"
)
for r in results:
    print(f"[{r.get('detectedSourceLanguage', '?')}] {r['input']:20s} → {r['translatedText']}")

# Language detection only
detection = client_v2.detect_language("Guten Morgen")
print(f"Language: {detection['language']}, Confidence: {detection['confidence']:.3f}")

# List supported languages
languages = client_v2.get_languages(target_language="en")
for lang in languages[:10]:
    print(f"  {lang['language']:8s} {lang['name']}")

# HTML translation (tags preserved)
html_text = "<b>Welcome</b> to our <em>website</em>!"
result = client_v2.translate(html_text, target_language="ja", format_="html")
print(f"HTML translated: {result['translatedText']}")

# ────────────────────────── v3 (Advanced) ────────────────────────────

PROJECT  = "my-project"
LOCATION = "global"          # or "us-central1", "europe-west1"
parent   = f"projects/{PROJECT}/locations/{LOCATION}"

client_v3 = translate_v3.TranslationServiceClient()

# Translate with v3
response = client_v3.translate_text(
    request = {
        "parent":              parent,
        "contents":            ["I am learning machine learning."],
        "mime_type":           "text/plain",   # "text/plain" or "text/html"
        "source_language_code":"en",
        "target_language_code":"fr",
    }
)
for translation in response.translations:
    print(f"Translated: {translation.translated_text}")

# ── Create and Use a Glossary ─────────────────────────────────────────
# Upload glossary file to GCS first:
# en,es
# BigQuery,BigQuery
# Cloud Storage,Cloud Storage
# Vertex AI,Vertex AI

glossary_id = "gcp-technical-terms"
glossary = translate_v3.Glossary(
    name = f"{parent}/glossaries/{glossary_id}",
    language_codes_set = translate_v3.Glossary.LanguageCodesSet(
        language_codes = ["en", "es"]
    ),
    input_config = translate_v3.GlossaryInputConfig(
        gcs_source = translate_v3.GcsSource(
            input_uri = "gs://my-bucket/glossaries/gcp_terms_en_es.csv"
        )
    ),
)
operation = client_v3.create_glossary(parent=parent, glossary=glossary)
created_glossary = operation.result(timeout=90)
print(f"Glossary created: {created_glossary.name}")

# Translate using glossary
response = client_v3.translate_text(
    request = {
        "parent":              parent,
        "contents":            ["Store data in Cloud Storage and analyze with BigQuery."],
        "source_language_code":"en",
        "target_language_code":"es",
        "glossary_config": translate_v3.TranslateTextGlossaryConfig(
            glossary = created_glossary.name
        ),
    }
)
for t in response.glossary_translations:
    print(f"With glossary: {t.translated_text}")

# ── Batch Translation (GCS → GCS) ─────────────────────────────────────
operation = client_v3.batch_translate_text(
    request = {
        "parent":              f"projects/{PROJECT}/locations/us-central1",
        "source_language_code":"en",
        "target_language_codes": ["es", "fr", "de", "ja"],
        "input_configs": [{
            "gcs_source":  {"input_uri": "gs://my-bucket/docs_to_translate/*.txt"},
            "mime_type":   "text/plain",
        }],
        "output_config": {
            "gcs_destination": {"output_uri_prefix": "gs://my-bucket/translated/"},
        },
        "glossaries": {
            "es": translate_v3.TranslateTextGlossaryConfig(
                glossary = created_glossary.name
            )
        },
    }
)
result = operation.result(timeout=300)
print(f"Total characters: {result.total_characters}")
print(f"Translated chars: {result.translated_characters}")
```

---

### REST API: Translation

```bash
TOKEN=$(gcloud auth print-access-token)
API_KEY="YOUR_API_KEY"

# v2: Basic translation (API key)
curl "https://translation.googleapis.com/language/translate/v2?key=${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"q": "Hello World", "target": "es"}'

# v2: Language detection
curl "https://translation.googleapis.com/language/translate/v2/detect?key=${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"q": "Hola, ¿cómo estás?"}'

# v3: Advanced translation (service account)
curl -X POST \
  "https://translation.googleapis.com/v3/projects/my-project/locations/global:translateText" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": ["Good morning!"],
    "mimeType": "text/plain",
    "sourceLanguageCode": "en",
    "targetLanguageCode": "zh"
  }'
```

---

## 7. 📄 Document AI

### Processor Types Reference

| Category | Processor | Key Output | Best For |
|---|---|---|---|
| **General** | Document OCR | Full text, layout, tables | Any document OCR |
| **General** | Form Parser | Key-value pairs, checkboxes | Generic forms |
| **General** | Layout Parser | Sections, tables, headers | Complex layouts |
| **Finance** | Invoice Parser | Vendor, line items, totals, tax | Invoices, bills |
| **Finance** | Receipt Parser | Merchant, items, total, date | Receipts |
| **Finance** | Bank Statement Parser | Transactions, balances, dates | Bank statements |
| **Tax** | W2 Parser | Wages, tax withheld, employer | US W-2 forms |
| **Tax** | 1099 Parser | Income types, payer, amounts | US 1099 forms |
| **HR** | Payslip Parser | Salary, deductions, net pay | Pay stubs |
| **ID** | Identity Document Parser | Name, DOB, ID number, expiry | Passports, licenses |
| **Custom** | Custom Extractor | User-defined entities | Domain-specific docs |
| **Custom** | Custom Classifier | Document type label | Multi-doc routing |
| **Custom** | Custom Splitter | Page-level boundaries | Multi-doc PDFs |

---

### Python: Document AI

```python
from google.cloud import documentai
from google.cloud.documentai_toolbox import document as toolbox_doc

PROJECT    = "my-project"
LOCATION   = "us"           # "us" or "eu"
PROCESSOR_ID = "abc123def456"   # Get from Console

client = documentai.DocumentProcessorServiceClient(
    client_options={"api_endpoint": f"{LOCATION}-documentai.googleapis.com"}
)
processor_name = client.processor_path(PROJECT, LOCATION, PROCESSOR_ID)

# ── Online Processing (single document) ──────────────────────────────
with open("invoice.pdf", "rb") as f:
    raw_document = documentai.RawDocument(
        content   = f.read(),
        mime_type = "application/pdf",
    )

request  = documentai.ProcessRequest(name=processor_name, raw_document=raw_document)
result   = client.process_document(request=request)
document = result.document

# ── Get Full Text ─────────────────────────────────────────────────────
print(f"Full text (first 500 chars):\n{document.text[:500]}")

# ── Entity Extraction (Invoice Parser example) ────────────────────────
print("\nExtracted entities:")
for entity in document.entities:
    entity_type = entity.type_
    mention     = entity.mention_text.strip()
    confidence  = entity.confidence

    # Normalized value (for dates, money, etc.)
    norm_value = ""
    if entity.normalized_value:
        nv = entity.normalized_value
        if nv.HasField("date_value"):
            d = nv.date_value
            norm_value = f"{d.year}-{d.month:02d}-{d.day:02d}"
        elif nv.HasField("money_value"):
            m = nv.money_value
            norm_value = f"{m.currency_code} {m.units}.{m.nanos // 1e7:02.0f}"
        else:
            norm_value = nv.text

    print(f"  [{confidence:.2f}] {entity_type:30s}: {mention:30s}  Normalized: {norm_value}")

    # Sub-entities (e.g., line items within an invoice)
    for prop in entity.properties:
        print(f"    └─ {prop.type_:25s}: {prop.mention_text.strip()}")

# ── Form Field Extraction (Form Parser) ───────────────────────────────
print("\nForm fields (key-value pairs):")
for page in document.pages:
    for field in page.form_fields:
        # Get field name and value text
        field_name  = get_text(field.field_name,  document)
        field_value = get_text(field.field_value, document)
        confidence  = field.field_value.confidence
        print(f"  [{confidence:.2f}] {field_name:30s} = {field_value}")

def get_text(layout, document):
    """Helper: extract text from a Layout object using text anchors."""
    response = ""
    for segment in layout.text_anchor.text_segments:
        start = int(segment.start_index)
        end   = int(segment.end_index)
        response += document.text[start:end]
    return response.strip()

# ── Table Extraction ──────────────────────────────────────────────────
print("\nTables:")
for page_num, page in enumerate(document.pages):
    for table_num, table in enumerate(page.tables):
        print(f"  Page {page_num+1}, Table {table_num+1}:")
        # Header rows
        for row in table.header_rows:
            cells = [get_text(cell.layout, document) for cell in row.cells]
            print(f"  HEADERS: {' | '.join(cells)}")
        # Body rows
        for row in table.body_rows:
            cells = [get_text(cell.layout, document) for cell in row.cells]
            print(f"  ROW:     {' | '.join(cells)}")

# ── Page Layout (blocks, paragraphs, lines) ───────────────────────────
for page in document.pages[:1]:   # First page only
    print(f"\nPage {page.page_number}: {page.dimension.width}x{page.dimension.height}")
    for block in page.blocks:
        btext = get_text(block.layout, document)
        verts = [(v.x, v.y) for v in block.layout.bounding_poly.normalized_vertices]
        print(f"  Block: '{btext[:50]}' at {verts}")

# ── Batch Processing (GCS folder → GCS output) ───────────────────────
gcs_input_uri    = "gs://my-bucket/invoices/"
gcs_output_uri   = "gs://my-bucket/processed-invoices/"

input_config = documentai.BatchDocumentsInputConfig(
    gcs_prefix = documentai.GcsPrefix(gcs_uri_prefix=gcs_input_uri)
)
output_config = documentai.DocumentOutputConfig(
    gcs_output_config = documentai.DocumentOutputConfig.GcsOutputConfig(
        gcs_uri = gcs_output_uri
    )
)

request = documentai.BatchProcessRequest(
    name         = processor_name,
    input_documents = input_config,
    document_output_config = output_config,
)
operation = client.batch_process_documents(request=request)
print("Batch processing started... waiting for completion")
operation.result(timeout=300)
print(f"Batch complete. Results in: {gcs_output_uri}")

# ── Document AI Toolbox (convenience wrapper) ─────────────────────────
# Wrap the result document for easier access
wrapped = toolbox_doc.Document.from_document_path(
    document_path = "gs://my-bucket/processed-invoices/invoice_001/"
)
# Extract tables as pandas DataFrames
tables_df = wrapped.convert_document_to_dataframe(table_index=0)
print(tables_df)

# Get entities as a dict
entities_dict = {e.type_: e.mention_text for e in wrapped.entities}
print(f"Invoice total: {entities_dict.get('total_amount', 'N/A')}")
print(f"Invoice date:  {entities_dict.get('invoice_date', 'N/A')}")
print(f"Vendor name:   {entities_dict.get('supplier_name', 'N/A')}")
```

---

### gcloud: Document AI

```bash
# List available processors in a location
gcloud ai document-processors list --location=us

# Get processor details
gcloud ai document-processors describe PROCESSOR_ID --location=us

# Process a document online
gcloud ai document-processors process PROCESSOR_ID \
  --location=us \
  --file=invoice.pdf \
  --mime-type=application/pdf \
  --format=json \
  | jq '.document.entities[] | {type: .type, text: .mentionText, confidence: .confidence}'
```

---

### REST API: Document AI

```bash
TOKEN=$(gcloud auth print-access-token)
PROJECT="my-project"
LOCATION="us"
PROCESSOR_ID="abc123"

# Online process (base64 PDF)
curl -X POST \
  "https://${LOCATION}-documentai.googleapis.com/v1/projects/${PROJECT}/locations/${LOCATION}/processors/${PROCESSOR_ID}:process" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "rawDocument": {
      "content": "'$(base64 -w0 invoice.pdf)'",
      "mimeType": "application/pdf"
    }
  }'
```

---

### Document AI vs. Vision OCR

| Feature | Document AI | Vision API (`DOCUMENT_TEXT_DETECTION`) |
|---|---|---|
| **Text extraction** | ✅ | ✅ |
| **Form fields (key-value)** | ✅ Form Parser | ❌ |
| **Table extraction** | ✅ Structured | ⚠️ Basic |
| **Entity extraction** | ✅ Domain-specific | ❌ |
| **Specialized parsers** | ✅ Invoice, Receipt, W2 | ❌ |
| **Multi-page PDF support** | ✅ Unlimited (batch) | ✅ Up to 2,000 pages |
| **Price** | Per page (~$0.0015–$0.065) | Per 1K features ($1.50–$3.50) |
| **Best for** | Structured document extraction | General image text OCR |

---

## 8. 🚀 Common Patterns & Integration Architectures

### Pattern 1: Event-Driven Document Processing

```
Cloud Storage (PDF uploaded)
       │  trigger
       ▼
Cloud Functions / Cloud Run
       │  process_document()
       ▼
Document AI (Invoice Parser)
       │  structured entities
       ▼
BigQuery (invoice_entities table)
       │  optional
       ▼
Looker Studio (dashboard)
```

```python
# Cloud Function: process invoice on GCS upload
import functions_framework
from google.cloud import documentai, bigquery

PROCESSOR_NAME = "projects/proj/locations/us/processors/PROC_ID"

@functions_framework.cloud_event
def process_invoice(cloud_event):
    data   = cloud_event.data
    bucket = data["bucket"]
    name   = data["name"]

    if not name.endswith(".pdf"):
        return

    # Process with Document AI
    dai_client = documentai.DocumentProcessorServiceClient(
        client_options={"api_endpoint": "us-documentai.googleapis.com"}
    )
    request = documentai.ProcessRequest(
        name = PROCESSOR_NAME,
        raw_document = documentai.RawDocument(
            content   = __import__('google.cloud.storage', fromlist=['Client']).Client().bucket(bucket).blob(name).download_as_bytes(),
            mime_type = "application/pdf",
        )
    )
    result   = dai_client.process_document(request=request)
    entities = {e.type_: e.mention_text for e in result.document.entities}

    # Write to BigQuery
    bq_client = bigquery.Client()
    bq_client.insert_rows_json("proj.invoices.extracted", [{
        "gcs_path":       f"gs://{bucket}/{name}",
        "invoice_date":   entities.get("invoice_date", ""),
        "total_amount":   entities.get("total_amount", ""),
        "supplier_name":  entities.get("supplier_name", ""),
        "processed_at":   __import__('datetime').datetime.utcnow().isoformat(),
    }])
    print(f"Processed: {name} → {entities}")
```

---

### Pattern 2: Audio Transcription + Sentiment Pipeline

```python
from google.cloud import speech, language_v1, bigquery
from datetime import datetime

def transcribe_and_analyze(gcs_audio_uri: str, call_id: str) -> dict:
    """Transcribe a call recording and analyze sentiment."""

    # Step 1: Transcribe audio
    speech_client = speech.SpeechClient()
    audio  = speech.RecognitionAudio(uri=gcs_audio_uri)
    config = speech.RecognitionConfig(
        encoding          = speech.RecognitionConfig.AudioEncoding.FLAC,
        sample_rate_hertz = 8000,
        language_code     = "en-US",
        enable_automatic_punctuation = True,
        model             = "phone_call",
        use_enhanced      = True,
    )
    operation  = speech_client.long_running_recognize(config=config, audio=audio)
    transcript_parts = [
        r.alternatives[0].transcript
        for r in operation.result(timeout=300).results
    ]
    full_transcript = " ".join(transcript_parts)

    # Step 2: Analyze sentiment
    nl_client = language_v1.LanguageServiceClient()
    doc = language_v1.Document(content=full_transcript,
                                type_=language_v1.Document.Type.PLAIN_TEXT)
    sentiment = nl_client.analyze_sentiment(
        request={"document": doc}
    ).document_sentiment

    # Step 3: Write to BigQuery
    bq_client = bigquery.Client()
    bq_client.insert_rows_json("proj.calls.transcripts", [{
        "call_id":     call_id,
        "transcript":  full_transcript,
        "sentiment_score":     sentiment.score,
        "sentiment_magnitude": sentiment.magnitude,
        "processed_at": datetime.utcnow().isoformat(),
    }])

    return {
        "transcript": full_transcript,
        "sentiment_score": sentiment.score,
        "sentiment_magnitude": sentiment.magnitude,
    }
```

---

### Pattern 3: Multi-API Document Enrichment Pipeline

```python
from google.cloud import documentai, translate_v3, language_v1

def enrich_document(pdf_path: str, target_language: str = "en") -> dict:
    """Extract → Translate → Analyze pipeline."""

    # 1. OCR with Document AI
    dai_client = documentai.DocumentProcessorServiceClient(
        client_options={"api_endpoint": "us-documentai.googleapis.com"}
    )
    with open(pdf_path, "rb") as f:
        result = dai_client.process_document(
            request=documentai.ProcessRequest(
                name         = "projects/proj/locations/us/processors/OCR_PROC_ID",
                raw_document = documentai.RawDocument(content=f.read(), mime_type="application/pdf"),
            )
        )
    extracted_text = result.document.text

    # 2. Detect language and translate if needed
    trans_client = translate_v3.TranslationServiceClient()
    detect_resp  = trans_client.detect_language(
        request={"parent": "projects/proj/locations/global", "content": extracted_text[:500]}
    )
    source_lang  = detect_resp.languages[0].language_code

    translated_text = extracted_text
    if source_lang != target_language:
        trans_resp = trans_client.translate_text(
            request={
                "parent":               "projects/proj/locations/global",
                "contents":             [extracted_text],
                "source_language_code": source_lang,
                "target_language_code": target_language,
                "mime_type":            "text/plain",
            }
        )
        translated_text = trans_resp.translations[0].translated_text

    # 3. Entity analysis in target language
    nl_client = language_v1.LanguageServiceClient()
    entities_resp = nl_client.analyze_entities(
        request={
            "document":      language_v1.Document(
                content=translated_text, type_=language_v1.Document.Type.PLAIN_TEXT
            ),
            "encoding_type": language_v1.EncodingType.UTF8,
        }
    )
    entities = [
        {"name": e.name, "type": language_v1.Entity.Type(e.type_).name, "salience": e.salience}
        for e in sorted(entities_resp.entities, key=lambda x: x.salience, reverse=True)[:10]
    ]

    return {
        "source_language": source_lang,
        "original_text":   extracted_text[:200],
        "translated_text": translated_text[:200],
        "top_entities":    entities,
    }
```

---

### Pattern 4: Content Moderation Pipeline (Cloud Run)

```python
# cloud_run_app.py — Vision + NL moderation endpoint
from flask import Flask, request, jsonify, abort
from google.cloud import vision, language_v1

app    = Flask(__name__)
vis_cl = vision.ImageAnnotatorClient()
nl_cl  = language_v1.LanguageServiceClient()

@app.post("/moderate")
def moderate_content():
    data     = request.get_json()
    image_b64 = data.get("image_base64")
    text      = data.get("text", "")
    flags     = {"approved": True, "reasons": []}

    # Check image with Vision Safe Search
    if image_b64:
        import base64
        image = vision.Image(content=base64.b64decode(image_b64))
        ss    = vis_cl.safe_search_detection(image=image).safe_search_annotation
        THRESHOLD = vision.Likelihood.POSSIBLE
        if ss.adult   >= THRESHOLD: flags["reasons"].append("adult_content")
        if ss.violence >= THRESHOLD: flags["reasons"].append("violent_content")
        if ss.medical  >= THRESHOLD: flags["reasons"].append("medical_content")

    # Check text with NL Moderation API
    if text and len(text) > 5:
        doc   = language_v1.Document(content=text, type_=language_v1.Document.Type.PLAIN_TEXT)
        resp  = nl_cl.moderate_text(request={"document": doc})
        for category in resp.moderation_categories:
            if category.confidence > 0.7:
                flags["reasons"].append(f"text_{category.name.lower().replace(' ', '_')}")

    flags["approved"] = len(flags["reasons"]) == 0
    return jsonify(flags)
```

---

## 9. ⚠️ Error Handling, Quotas & Best Practices

### Common Error Codes

| Error Code | HTTP | Cause | Fix |
|---|---|---|---|
| `INVALID_ARGUMENT` | 400 | Bad request format, wrong field type | Check request schema and field values |
| `RESOURCE_EXHAUSTED` | 429 | Quota exceeded (RPM, chars, pages) | Implement exponential backoff; request quota increase |
| `PERMISSION_DENIED` | 403 | Missing IAM role or API not enabled | Grant appropriate role; enable API |
| `NOT_FOUND` | 404 | Processor/model/resource doesn't exist | Verify resource name and project ID |
| `UNAUTHENTICATED` | 401 | Invalid/expired credentials | Refresh token; check key file path |
| `DEADLINE_EXCEEDED` | 504 | Operation timed out | Use async/long-running variants; increase timeout |
| `INTERNAL` | 500 | Google-side error | Retry with backoff |
| `UNIMPLEMENTED` | 501 | Feature not available in region | Use supported region |

---

### Quota Limits Reference

| Service | Limit Type | Default Limit |
|---|---|---|
| Vision API | Requests per minute | 1,800 |
| Vision API | Max image size | 20 MB (REST) / 75 MB (base64) |
| Vision API | Max features per request | 50 |
| Natural Language | Requests per minute | 600 |
| Natural Language | Max document size | 1 MB (1,000,000 characters) |
| Speech-to-Text | Max audio duration (sync) | 60 seconds |
| Speech-to-Text | Max audio duration (async) | 480 minutes (8 hours) |
| Speech-to-Text | Concurrent streaming requests | 6 |
| Translation | Characters per request | 30,000 |
| Translation | Requests per minute | 600 |
| Document AI | Online: max pages | 15 pages |
| Document AI | Batch: max file size | 40 MB |
| Document AI | Concurrent batch operations | 5 |

---

### Retry with Exponential Backoff

```python
from google.api_core import retry, exceptions
from google.cloud import vision

client = vision.ImageAnnotatorClient()

# Define a custom retry predicate
def is_retryable(exc):
    retryable_codes = {
        exceptions.ResourceExhausted,
        exceptions.ServiceUnavailable,
        exceptions.DeadlineExceeded,
        exceptions.InternalServerError,
    }
    return any(isinstance(exc, code) for code in retryable_codes)

# Configure retry policy
my_retry = retry.Retry(
    predicate    = is_retryable,
    initial      = 1.0,       # First wait: 1 second
    multiplier   = 2.0,       # Double each time
    maximum      = 60.0,      # Max wait: 60 seconds
    deadline     = 300.0,     # Give up after 5 minutes total
)

# Use retry in API call
image = vision.Image(source=vision.ImageSource(gcs_image_uri="gs://bucket/img.jpg"))

@my_retry
def detect_labels_with_retry():
    return client.label_detection(image=image)

response = detect_labels_with_retry()
```

---

### Best Practices

| Practice | Recommendation |
|---|---|
| **Auth** | Use ADC / Workload Identity Federation; never commit service account keys |
| **Batching** | Combine multiple images/texts in a single API request to reduce overhead |
| **Caching** | Cache API responses (Redis, Firestore) for identical inputs |
| **Async** | Use long-running operations for audio >60s, docs >15 pages |
| **Feature selection** | Request only needed Vision features (each counts toward quota) |
| **Preprocessing** | Resize images to <1 MB before Vision API (min quality for accuracy) |
| **Audio quality** | Resample audio to 16kHz LINEAR16 for best Speech-to-Text accuracy |
| **Sampling** | Use `samplingPercent` in scans; use `row_filter` to scan only new data |
| **Logging** | Log all API responses to Cloud Logging for debugging and audit |
| **Regional endpoints** | Use regional endpoints when data residency matters |

---

## 10. 📋 gcloud CLI & REST API Quick Reference

### gcloud Commands

```bash
# ── VISION API ────────────────────────────────────────────────────────
gcloud ml vision detect-labels       IMAGE_URI
gcloud ml vision detect-text         IMAGE_URI
gcloud ml vision detect-document     IMAGE_URI    # Dense OCR
gcloud ml vision detect-objects      IMAGE_URI
gcloud ml vision detect-faces        IMAGE_URI
gcloud ml vision detect-safe-search  IMAGE_URI
gcloud ml vision detect-landmarks    IMAGE_URI
gcloud ml vision detect-logos        IMAGE_URI
gcloud ml vision detect-web          IMAGE_URI
gcloud ml vision suggest-crops       IMAGE_URI

# ── NATURAL LANGUAGE API ──────────────────────────────────────────────
gcloud ml language analyze-sentiment  --content="Great product!"
gcloud ml language analyze-entities   --content="Apple was founded by Steve Jobs."
gcloud ml language analyze-syntax     --content="The dog runs fast."
gcloud ml language classify-text      --content-file=article.txt

# ── SPEECH-TO-TEXT ────────────────────────────────────────────────────
gcloud ml speech recognize audio.flac \
  --language-code=en-US \
  --encoding=FLAC \
  --sample-rate=16000

gcloud ml speech recognize-long-running gs://bucket/audio.flac \
  --language-code=en-US \
  --encoding=FLAC \
  --sample-rate=44100 \
  --async

gcloud ml speech operations describe OPERATION_ID

# ── TRANSLATION ───────────────────────────────────────────────────────
gcloud ml translate detect-language --content="Bonjour monde"

# ── DOCUMENT AI ───────────────────────────────────────────────────────
gcloud ai document-processors list --location=us
gcloud ai document-processors describe PROCESSOR_ID --location=us

# ── AUTOML / VERTEX AI ────────────────────────────────────────────────
gcloud ai datasets list            --region=us-central1
gcloud ai models list              --region=us-central1
gcloud ai endpoints list           --region=us-central1
gcloud ai custom-jobs list         --region=us-central1
```

---

### REST API Base URLs

| Service | Base URL |
|---|---|
| Vision API | `https://vision.googleapis.com/v1/images:annotate` |
| Natural Language API | `https://language.googleapis.com/v1/documents:{method}` |
| Speech-to-Text v1 | `https://speech.googleapis.com/v1/speech:{method}` |
| Speech-to-Text v2 | `https://speech.googleapis.com/v2/projects/{project}/locations/{location}/recognizers/-:recognize` |
| Translation v2 | `https://translation.googleapis.com/language/translate/v2` |
| Translation v3 | `https://translation.googleapis.com/v3/projects/{project}/locations/{location}:translateText` |
| Document AI | `https://{location}-documentai.googleapis.com/v1/projects/{project}/locations/{location}/processors/{processor}:process` |
| Vertex AI AutoML | `https://{region}-aiplatform.googleapis.com/v1/projects/{project}/locations/{region}/` |

---

### Python ADC Initialization

```python
# All clients use ADC automatically when no credentials are specified
# Run: gcloud auth application-default login

from google.cloud import vision, language_v1, speech
from google.cloud import translate_v2 as translate
from google.cloud import documentai
from google.cloud import aiplatform

# Standard client initialization (picks up ADC)
vision_client   = vision.ImageAnnotatorClient()
language_client = language_v1.LanguageServiceClient()
speech_client   = speech.SpeechClient()
translate_client = translate.Client()
translate_v3_client = __import__('google.cloud.translate_v3', fromlist=['TranslationServiceClient']).TranslationServiceClient()
docai_client    = documentai.DocumentProcessorServiceClient(
    client_options={"api_endpoint": "us-documentai.googleapis.com"}
)
aiplatform.init(project="my-project", location="us-central1")
```

---

## 11. 💰 Pricing Summary

> ⚠️ Prices are approximate as of early 2026. Verify at [cloud.google.com/pricing](https://cloud.google.com/pricing).

### Free Tier Summary

| Service | Free Monthly Quota | Paid Beyond Free |
|---|---|---|
| **Vision API** | 1,000 units/feature | $1.50–$3.50 per 1K units |
| **Natural Language API** | 5,000 units/feature | $0.50–$2.00 per 1K units |
| **Speech-to-Text** | 60 minutes audio | $0.004–$0.009 per 15 sec |
| **Translation v2** | 500,000 characters | $20 per 1M characters |
| **Translation v3** | 500,000 characters | $20 per 1M characters |
| **Document AI** | 300 pages (general OCR) | $0.0015–$0.065 per page |
| **AutoML** | $300 free credit (new) | $1.60–$27/node-hr by type |

---

### Per-Service Pricing Details

**Vision API (per 1K units):**
| Feature | Price |
|---|---|
| Label, Object, Logo, Landmark, Properties | $1.50 |
| OCR, Face, Safe Search, Crop Hints | $1.50 |
| Document Text Detection | $1.50 |
| Web Detection | $3.50 |

**Natural Language API (per 1K units):**
| Feature | Price |
|---|---|
| Sentiment, Entity, Syntax, Classification | $1.00 |
| Entity Sentiment | $2.00 |

**Speech-to-Text (per 15-second increment):**
| Model | Standard | Enhanced |
|---|---|---|
| Default | $0.004 | $0.009 |
| Medical | — | $0.012 |
| Multi-channel | $0.008 | $0.018 |

**Document AI (per page):**
| Processor | Price |
|---|---|
| Document OCR | $0.0015 |
| Form Parser | $0.0015 |
| Invoice / Receipt Parser | $0.030 |
| Identity Document Parser | $0.065 |
| W2 / 1099 Parser | $0.030 |

---

### Cost Optimization Tips

| Service | Tip |
|---|---|
| Vision API | Batch up to 16 images per request; request only needed features |
| Natural Language | Cache results for repeated identical text; use smallest document chunks |
| Speech-to-Text | Use FLAC compression; avoid `use_enhanced` for non-critical audio |
| Translation | Detect language first; skip translation if already in target language |
| Document AI | Use online mode only for <15 pages; batch process for bulk jobs |
| AutoML | Use `disable_early_stopping=False`; start with small budgets |

---

### Monthly Cost Examples

```
Content Moderation Pipeline (10K images/day × 30 days = 300K images):
  Vision Safe Search: 300K units × $1.50/1K = $450/month
  NL Moderation (captions): 300K × $1.00/1K = $300/month
  Total: ~$750/month (after free tier)

Document Processing System (5K invoices/day × 30 days = 150K pages):
  Document AI Invoice Parser: 150K pages × $0.030 = $4,500/month
  Translation (non-English invoices, 20% = 30K): 30K × 200 chars × $20/1M = $120/month
  Total: ~$4,620/month

Audio Transcription (100 hrs of calls/day × 30 days = 3,000 hrs):
  Speech-to-Text (phone_call, enhanced): 3,000 × 60 × 60 / 15 × $0.009 = $6,480/month
  With 60 min free: negligible discount on this scale
  Total: ~$6,480/month
```

---

## 12. Quick Reference & Comparison Tables

### Service Feature Matrix

| Problem | Vision API | NL API | Speech-to-Text | Translation | Document AI | AutoML |
|---|---|---|---|---|---|---|
| Image classification | ✅ General | ❌ | ❌ | ❌ | ❌ | ✅ Custom |
| Image OCR | ✅ | ❌ | ❌ | ❌ | ✅ Better | ❌ |
| Object detection | ✅ General | ❌ | ❌ | ❌ | ❌ | ✅ Custom |
| Text sentiment | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ Custom |
| Named entity extraction | ❌ | ✅ General | ❌ | ❌ | ✅ Domain | ✅ Custom |
| Audio transcription | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Language translation | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Invoice/form parsing | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Custom tabular model | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Content moderation | ✅ Visual | ✅ Text | ❌ | ❌ | ❌ | ✅ Custom |

---

### Async vs. Sync Availability

| Service | Sync | Async / Long-running | Streaming |
|---|---|---|---|
| Vision API | ✅ `annotateImages` | ✅ `asyncBatchAnnotateFiles` | ❌ |
| Natural Language API | ✅ All methods | ❌ | ❌ |
| Speech-to-Text | ✅ `recognize` (<60s) | ✅ `longRunningRecognize` | ✅ `streamingRecognize` |
| Translation API | ✅ `translateText` | ✅ `batchTranslateText` | ❌ |
| Document AI | ✅ `processDocument` (<15 pages) | ✅ `batchProcessDocuments` | ❌ |
| AutoML | ✅ Online prediction | ✅ Batch prediction | ❌ |

---

### Vision API Feature One-Liner Reference

```
LABEL_DETECTION          → ["cat", "animal", "pet"] — general tags
OBJECT_LOCALIZATION      → [{name, score, boundingBox}] — multiple objects with locations
TEXT_DETECTION           → string of all detected text (sparse text)
DOCUMENT_TEXT_DETECTION  → structured page/block/para/word/symbol hierarchy
FACE_DETECTION           → [{boundingPoly, emotions, landmarks, headwear}]
LANDMARK_DETECTION       → [{description, score, boundingPoly, locations}]
LOGO_DETECTION           → [{description, score, boundingPoly}] — brand logos
SAFE_SEARCH_DETECTION    → {adult, spoof, medical, violence, racy} likelihoods
IMAGE_PROPERTIES         → {dominantColors: [{color, score, pixelFraction}]}
CROP_HINTS               → [{boundingPoly, confidence, importanceFraction}]
WEB_DETECTION            → {webEntities, fullMatchingImages, bestGuessLabels}
```

---

### Natural Language API Feature Comparison

| Method | Returns | Use Case |
|---|---|---|
| `analyzeSentiment` | `{score, magnitude}` per doc + sentence | Customer review analysis |
| `analyzeEntities` | `[{name, type, salience, metadata, mentions}]` | NER, knowledge graph |
| `analyzeEntitySentiment` | Entities + `{score, magnitude}` per entity | Product feature feedback |
| `analyzeSyntax` | Tokens with POS, lemma, dependency | Linguistic analysis, parsing |
| `classifyText` | `[{name, confidence}]` IAB categories | Content tagging, routing |
| `moderateText` | `[{name, confidence}]` harm categories | UGC moderation pipeline |

---

### Speech-to-Text Model Comparison

| Model | Best For | Audio Length | Enhanced Available |
|---|---|---|---|
| `latest_long` | General long audio, meetings | >1 min | ✅ |
| `latest_short` | Queries, commands | <1 min | ✅ |
| `phone_call` | 8kHz telephone audio | Any | ✅ |
| `video` | Video soundtracks, web audio | Any | ✅ |
| `medical_dictation` | Clinical notes, monologue | Any | ✅ |
| `medical_conversation` | Doctor-patient dialog | Any | ✅ |
| `command_and_search` | Short voice commands | <1 min | ❌ |

---

### AutoML Model Type Reference

| AutoML Type | Data Format | Optimization Target | Export Options |
|---|---|---|---|
| Image Classification | CSV + GCS images | Precision/Recall/AUC | TFLite, CoreML, EdgeTPU, TF SavedModel |
| Object Detection | CSV + GCS images | mAP | TFLite, EdgeTPU, TF SavedModel |
| Text Classification | JSONL | Precision/Recall | TF SavedModel |
| Text Entity Extraction | JSONL | F1 | TF SavedModel |
| Text Sentiment | JSONL | MAE | TF SavedModel |
| Tabular Classification | CSV or BQ | AUC-ROC | — |
| Tabular Regression | CSV or BQ | RMSE, MAE | — |
| Tabular Forecasting | CSV or BQ | RMSE, WAPE | — |
| Video Classification | CSV + GCS videos | AUC | TF SavedModel |

---

### Python Client Library Import Reference

```python
# Vision API
from google.cloud import vision
client = vision.ImageAnnotatorClient()

# Natural Language API
from google.cloud import language_v1
client = language_v1.LanguageServiceClient()

# Speech-to-Text (v1)
from google.cloud import speech
client = speech.SpeechClient()

# Translation v2 (Basic)
from google.cloud import translate_v2 as translate
client = translate.Client()

# Translation v3 (Advanced)
from google.cloud import translate_v3
client = translate_v3.TranslationServiceClient()

# Document AI
from google.cloud import documentai
client = documentai.DocumentProcessorServiceClient(
    client_options={"api_endpoint": "us-documentai.googleapis.com"}
)
# Document AI Toolbox
from google.cloud.documentai_toolbox import document as toolbox

# AutoML / Vertex AI
from google.cloud import aiplatform
aiplatform.init(project="my-project", location="us-central1")
# Job classes:
# aiplatform.AutoMLImageTrainingJob
# aiplatform.AutoMLTabularTrainingJob
# aiplatform.AutoMLTextTrainingJob
# aiplatform.AutoMLVideoTrainingJob
```

---

*Generated for GCP AI & ML Services | Official Docs: [cloud.google.com/products/ai](https://cloud.google.com/products/ai)*
