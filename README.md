# Simple RAG on Kubernetes (Cloud-Agnostic) — Requirements (v1)

This repo captures the requirements and initial architecture for a **simple RAG (Retrieval-Augmented Generation)** pipeline designed to be **cloud-vendor agnostic** by running key infrastructure components **inside Kubernetes** (initially on **GKE**, but portable to EKS/AKS/on-prem).

We are proceeding with **Path A**: **Kafka + OpenSearch + OCR endpoint**, and **no Postgres initially**.

---

## 1) Objective

Build a “simple RAG” platform that:

1. **Ingests PDFs** from an object-store “folder” (prefix)  
2. Uses **Kafka messages** to drive processing stages  
3. Extracts text/structure from PDFs using **DeepSeek-OCR served via vLLM**  
4. Performs **document understanding + chunking** and stores chunks + metadata in **OpenSearch**  
5. Generates **embeddings** for chunks and stores vectors in OpenSearch  
6. Exposes **retrieval APIs** (REST) for downstream consumers

Primary language: **Python**  
Primary deployment: **Kubernetes**  
Initial runtime: **GKE**, with portability as a core design constraint

---

## 2) High-Level Architecture

### Compute & orchestration
- Kubernetes cluster (initially GKE Standard; portable)

### Storage & messaging (cloud-agnostic posture)
- **Object storage** for PDFs (initially GCS; later S3/MinIO)
- **Kafka on Kubernetes** (not managed Kafka)
- **OpenSearch on Kubernetes** for:
  - chunk storage + metadata
  - vector embeddings (kNN)

### OCR
- **vLLM** deployed on GPU nodes serving **DeepSeek-OCR** behind an HTTP endpoint
- OCR service is accessed internally (ClusterIP) in v1

### SQL database
- **Not included in v1 (Path A)**
- We’ll use OpenSearch + Kafka offsets + idempotency patterns initially

---

## 3) In-Scope Functional Requirements

### 3.1 PDF ingestion
- PDFs are placed into an object store prefix (e.g., `pdfs/`)
- A Kafka message is produced containing the PDF location
- Ingestion must validate object existence before proceeding downstream

### 3.2 OCR extraction
- OCR worker consumes Kafka messages and calls the internal OCR endpoint
- OCR output includes:
  - extracted text
  - structural hints when available (pages, layout blocks, tables)
- OCR results must carry traceable metadata (`doc_id`, provider/bucket/key, page info)

### 3.3 Document understanding & chunking
- Chunking service converts OCR output into retrieval-friendly chunks
- Each chunk is stored in OpenSearch with metadata
- Chunking supports:
  - page-aware chunk boundaries
  - configurable max tokens/characters
  - configurable overlap

### 3.4 Embeddings generation
- Embeddings are created per chunk
- Stored alongside chunk record as a vector field in OpenSearch
- Embedding model is configurable (self-hosted or API-based)
- Store embedding provenance:
  - `embedding_model`, `embedding_dim`, `embedding_version`

### 3.5 Retrieval APIs
REST APIs (internal in v1; external later):
- Query by text → top-K relevant chunks (vector search + metadata filters)
- Retrieve all chunks for a `doc_id`
- Health endpoints for each service

---

## 4) Out of Scope for v1 (Path A)

- Postgres / SQL cluster
- Advanced authz / ACLs / multi-tenancy enforcement
- UI
- Formal evaluation pipelines (RAGAS, etc.)
- Full production HA sizing (we start dev-friendly and scale later)

---

## 5) Non-Functional Requirements

### 5.1 Portability
- Deploy via Helm + Kubernetes manifests
- Config via ConfigMaps/Secrets and Helm values
- Avoid cloud-specific managed services where feasible
- Object store access abstracted (GCS/S3/MinIO)

### 5.2 Reliability & idempotency
- Stable `doc_id` across all pipeline steps
- Reprocessing should not create duplicate chunk records
- Approach:
  - deterministic `chunk_id`
  - OpenSearch upsert instead of blind insert

### 5.3 Observability
- Structured (JSON) logs
- Metrics endpoint (Prometheus-friendly) preferred
- Trace correlation via `doc_id` (and optional `trace_id`) across Kafka and services

### 5.4 Security (v1)
- Secrets in Kubernetes Secrets (later integrate a secret manager)
- OCR endpoint internal-only (ClusterIP)
- Network policies are optional in v1, recommended later

### 5.5 Performance (initial targets)
- v1 target: process small PDFs (1–50 pages) end-to-end reliably
- Latency is secondary to correctness + stability in v1

---

## 6) Data Contracts

### 6.1 Kafka topics (v1)
- `pdf-ingest` — new documents
- `pdf-ocr-done` — OCR stage completion
- `pdf-chunked` — chunking completion (ready for embeddings)
- `pdf-indexed` — fully indexed completion
- `pdf-errors` — failures with error details + original reference

### 6.2 Kafka message schema (v1)
Minimal JSON message (recommended):

```json
{
  "doc_id": "string",
  "source": {
    "provider": "gcs|s3|minio",
    "bucket": "string",
    "key": "string",
    "version": "optional-string"
  },
  "ingest_ts": "ISO-8601 timestamp",
  "trace_id": "optional-string",
  "attempt": 0
}
```

Notes:
- `doc_id` must be stable and unique in the deployment.
- `attempt` supports retry tracking (even without SQL).

### 6.3 OpenSearch storage model (v1)

#### A) Document record (1 per PDF)
- `doc_id`
- `source_uri` (provider/bucket/key)
- `status` (RECEIVED / OCR_DONE / CHUNKED / EMBEDDED / INDEXED / ERROR)
- timestamps
- optional `content_hash` (future)

#### B) Chunk record (many per PDF)
- `chunk_id` (deterministic)
- `doc_id`
- `page_start`, `page_end`
- `chunk_text`
- `metadata` (layout type, section title, etc.)
- `embedding` (`knn_vector`)
- `embedding_model`, `embedding_dim`, `embedding_version`

---

## 7) Deployment Requirements (Path A)

### 7.1 Kubernetes workloads
- Kafka: **Strimzi operator** + Kafka cluster
- OpenSearch: **Helm chart** (single-node for dev; scale later)
- OCR: **vLLM** Deployment on **GPU node pool** + Service
- Python services:
  - ingest consumer
  - OCR worker
  - chunker
  - embedder
  - retrieval API

### 7.2 Node pools
- CPU pool: Kafka operator, chunking, embedding (if CPU), APIs
- GPU pool: OCR service only (taints/labels to isolate)

---

## 8) Testing Requirements

### 8.1 Component smoke tests
- Kafka:
  - create topic `pdf-ingest`
  - produce and consume a test JSON message in-cluster
- OpenSearch:
  - create index
  - upsert sample doc and query it
  - (later) vector search test
- OCR (vLLM):
  - `/v1/models` reachable
  - one OCR request returns output
- End-to-end:
  - upload 1 PDF + produce ingest message
  - chunks retrievable via retrieval API

### 8.2 CI (later)
- Unit tests for chunking and message parsing
- Integration tests using kind/minikube optional (or ephemeral GKE dev)

---

## 9) Deferred Decision: Postgres

We explicitly chose **not** to deploy Postgres in v1.

We’ll revisit Postgres if/when we need:
- stronger job tracking/audit trails
- dedup/version control beyond OpenSearch fields
- multi-tenancy/permissions/reporting

---

## 10) Technology Choices Summary

- Orchestration: Kubernetes (GKE initially)
- Kafka: in-cluster (Strimzi)
- Search + vectors: OpenSearch in-cluster
- OCR: DeepSeek-OCR served via vLLM on GPU
- Language: Python
- SQL DB: deferred (Path A)

Cloud posture:
- runs on GCP today; portable to AWS/Azure/on-prem
- object store abstracted across providers
- no managed Kafka/OpenSearch dependency

---

## 11) Next Step

Proceed to **Step 1 setup on GKE** (infra only + health checks):
1. Namespaces + Helm bootstrap
2. Strimzi + Kafka cluster + produce/consume test
3. OpenSearch install + curl/index test
4. GPU enablement + vLLM DeepSeek-OCR endpoint smoke test
