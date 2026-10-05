# Content Tagging & Competency Mapping (CTCM)
<img width="900" height="506" alt="streamlit-demo" src="https://github.com/user-attachments/assets/8d150856-867b-40e7-9d87-1559b0fc5619" />

**Team 2 — AI Data Center Operations Capstone**

CTCM is an AI-powered service that analyzes learning content and automatically generates structured educational metadata for content tagging and competency mapping.

The service produces:

* Content summaries
* Topic tags
* Difficulty levels
* Competency / skill mappings
* Confidence scores
* Three learning objectives

The project combines **multimodal content extraction, RAG, local GPU inference, OpenAI inference, vLLM continuous batching, Docker, Kubernetes, Prometheus, Grafana, and Streamlit** in one end-to-end AI system.

> **Documentation-only repository.** This repository documents the project's design, evaluation, and results. The source code is kept in a private repository.

---

## System Architecture

```text
Learning Content
PPTX / PDF / XLSX / IPYNB / Images / Code / Text
        |
        v
Content Extraction
Text + Images + OCR
        |
        v
Digest / Preprocessing
        |
        +-----------------------------+
        |                             |
        v                             v
Qwen + vLLM                     OpenAI API
AsyncLLMEngine                  gpt-4o-mini
Local NVIDIA GPU                External API
        |                             |
        +--------------+--------------+
                       |
                       v
                3-Pass Pipeline
        1. Tags + initial skills
        2. RAG skill refinement
        3. Difficulty refinement
                       |
                       v
             Structured API Response
                       |
          +------------+-------------+
          |                          |
          v                          v
      Streamlit              Prometheus / Grafana
```

---

## Model Backends

| Backend    | Model                             | Serving              |
| ---------- | --------------------------------- | -------------------- |
| **Qwen**   | `Qwen/Qwen2.5-VL-3B-Instruct-AWQ` | Local GPU using vLLM |
| **OpenAI** | `gpt-4o-mini`                     | OpenAI API           |

The local Qwen backend uses vLLM `AsyncLLMEngine`, enabling:

* Continuous batching
* Paged attention
* Asynchronous generation
* Concurrent request processing

The capstone deployment was tested on an **NVIDIA RTX A6000**.

---

## Inference Pipeline

### Pass 1 — Content Analysis

The model generates:

* Content summary
* Predicted topic tags
* Initial skill predictions

### Pass 2 — RAG Skill Refinement

The service retrieves relevant competencies from the skill taxonomy using the extracted learning content and first-pass tags.

* Embedding model: `BAAI/bge-m3`
* Skill taxonomy: **136 skills**
* Retrieval pool: **Top 15 candidate skills**

The model then selects its final competency mappings from the retrieved skill pool.

### Pass 3 — Difficulty Refinement

A focused inference pass classifies the learning content as:

* Beginner
* Intermediate
* Advanced

---

## Structured Output

A successful request returns structured output similar to:

```json
{
  "content_summary": "...",
  "predicted_tags": [
    "...",
    "..."
  ],
  "difficulty_level": "Intermediate",
  "predicted_skills": [
    "...",
    "...",
    "...",
    "..."
  ],
  "confidence": 0.90,
  "notes": "Identify ... • Apply ... • Evaluate ..."
}
```

The service is designed to return:

* Exactly **four competency mappings**
* Exactly **three learning objectives** in the `notes` field

Skill outputs are grounded against the defined competency taxonomy.

---

## Supported Content Types

The extraction pipeline supports multiple learning-content formats, including:

* PowerPoint
* PDF
* Word documents
* Excel / CSV / TSV
* Jupyter notebooks
* Markdown and plain text
* HTML
* Source-code files
* Images
* ZIP archives

OCR support is included for image-based or scanned content.

---

## Evaluation Dataset & Methodology

The final benchmark used:

* **12 representative learning-content files**
* A reviewed Golden Set
* The same task and output schema for both model backends
* The same evaluation and scoring methodology
* RAG-based skill grounding
* Structured-output validation
* Quality, latency, token, cost, and throughput measurements

The benchmark compares each model output against the reviewed reference data.

The evaluation covers:

* Tagging / classification quality
* Skill-mapping accuracy
* Retrieval quality
* Difficulty classification
* Structured-output validity
* End-to-end latency
* Generation speed
* Token usage
* Estimated cost
* Throughput
* Concurrent request behavior

The final benchmark results are summarized below.

---

## Final Benchmark Results

| Metric                            |              Qwen |            OpenAI |
| --------------------------------- | ----------------: | ----------------: |
| Tagging / Classification Accuracy |         **41.8%** |         **50.1%** |
| Skill Mapping Accuracy            |         **27.0%** |         **24.5%** |
| Retrieval Quality                 |         **86.1%** |         **86.1%** |
| Structured Output Validity        |          **100%** |          **100%** |
| Difficulty Accuracy               |         **66.7%** |         **83.3%** |
| Tag Semantic F1                   |         **43.4%** |         **53.4%** |
| Average E2E Latency               |       **56.39 s** |       **71.98 s** |
| Generation Speed                  |   **29.16 tok/s** |  **105.67 tok/s** |
| Total Tokens — 12 files           |       **107,195** |       **287,621** |
| Estimated Cost — 12 files         |       **$0.0575** |       **$0.0746** |
| Estimated Cost / 1,000 Files      |         **$4.79** |         **$6.22** |
| Estimated Throughput              | **63.8 files/hr** | **50.0 files/hr** |

These measurements are specific to the capstone benchmark workload and test environment.

---

## Benchmark Findings & Trade-offs

The two inference backends showed different strengths.

**GPT-4o-mini** achieved stronger results in several content-quality measures:

* Higher tagging / classification accuracy
* Higher semantic tag F1
* Higher difficulty-classification accuracy
* Higher generation speed

However, it relies on an external API and therefore provides less infrastructure control.

**Qwen** demonstrated advantages in several operational areas:

* Higher skill-mapping accuracy in the final benchmark
* Lower average end-to-end latency
* Lower total token usage
* Lower estimated cost
* Higher estimated benchmark throughput
* Full control over local deployment and inference infrastructure

The local Qwen deployment requires additional operational responsibility, including GPU capacity, model serving, Docker, Kubernetes, monitoring, and scaling.

The benchmark therefore demonstrates that model selection depends on the priorities of the use case rather than a single performance metric.

---

## Concurrent Benchmark

Concurrency was evaluated with:

* **3 simultaneous users**
* **3 representative learning files**
* Both Qwen and OpenAI
* **9 requests per model**



| Metric                 |            Qwen |           OpenAI |
| ---------------------- | --------------: | ---------------: |
| Successful Requests    |         **9/9** |          **9/9** |
| Sequential Baseline    |         149.2 s |          125.6 s |
| Concurrent Wall Time   |         362.3 s |          324.4 s |
| Average Latency / File |         120.8 s |          108.1 s |
| Workload Speedup       |       **1.24x** |        **1.16x** |
| Throughput             | **89 files/hr** | **100 files/hr** |

Both model paths completed all concurrent requests successfully.

The test also demonstrates an important trade-off: concurrent workloads improve total workload throughput, while individual request latency increases under load.

---

## Qwen Continuous Batching

Qwen is served through vLLM `AsyncLLMEngine`.

Three Qwen requests were launched simultaneously, and all three completed successfully.

During the concurrent workload, the vLLM runtime reported:

```text
Running: 3 reqs
Pending: 0 reqs
```

Prometheus also recorded three simultaneously running Qwen inference requests.

This provides runtime evidence that multiple Qwen inference requests were active in the vLLM engine at the same time.

Individual application stages inside a request may still execute sequentially, while vLLM performs continuous batching across concurrent inference requests.

---

## Containerization

The service is packaged as a GPU container image (CUDA 12.4 runtime, Python 3.11) that contains the application code and the public skills taxonomy. Evaluation materials are not included in the image.

---

## Kubernetes Deployment

The CTCM service was deployed on Kubernetes (k3s) together with Prometheus and Grafana in the `ctcm` namespace.

The deployment consists of:

* A namespace
* A single-replica GPU deployment with readiness and liveness probes
* A NodePort service
* A persistent volume claim for model-weight caching
* Prometheus and Grafana for monitoring
* A secret holding the OpenAI API key (injected at deployment time, never stored in the repository)

The application and observability workloads were verified running successfully (1/1 pods Running) in the `ctcm` namespace.

---

## API

The backend is implemented with **FastAPI**.

| Method | Endpoint                | Purpose                            |
| ------ | ----------------------- | ---------------------------------- |
| `GET`  | `/health`               | Service health and loaded models   |
| `GET`  | `/metrics`              | Prometheus metrics                 |
| `GET`  | `/v1/models`            | Available model backends           |
| `POST` | `/v1/tag/upload`        | Upload and analyze one file        |
| `POST` | `/v1/tag`               | Analyze content through JSON input |
| `POST` | `/v1/benchmark`         | Run benchmark comparison           |
| `GET`  | `/v1/benchmark/summary` | Benchmark metric definitions       |

Example:

```bash
curl -X POST http://<HOST>:8000/v1/tag/upload \
  -F "file=@<path-to-learning-content>" \
  -F "model=qwen"
```

> **NDA / Confidentiality Notice:** The original benchmark and learning-content files used during project evaluation are not included in this public repository. The API accepts supported learning-content files provided by an authorized user.

Available model values:

```text
qwen
openai
```

---

## Observability

The project uses:

* **Prometheus** for metric collection
* **Grafana** for dashboard visualization
* Application-level metrics for both model paths
* vLLM-native inference metrics for Qwen

The Grafana dashboard includes panels for:

* Service availability
* Qwen request activity
* OpenAI request activity
* Error rate
* End-to-end latency
* Request throughput
* Concurrent Qwen GPU requests
* GPU KV-cache utilization
* TTFT
* TPOT
* Queue time
* Prompt and generation throughput
* Token usage
* Process memory

### Application Metrics

Examples:

```text
model_requests_total{model="qwen",status="success"}
model_requests_total{model="openai",status="success"}

model_request_duration_seconds{model="qwen"}
model_request_duration_seconds{model="openai"}
```

These metrics are labeled by model, allowing Qwen and OpenAI requests to be monitored separately.

### Qwen / vLLM Metrics

The local Qwen backend exposes metrics including:

* Requests running
* Requests waiting
* GPU KV-cache utilization
* Time to First Token (TTFT)
* Time per Output Token (TPOT)
* Queue time
* Prompt throughput
* Generation throughput
* Prefill / decode behavior

vLLM and GPU metrics apply only to **Qwen**, because OpenAI inference is handled through an external API.

---

## Service Indicators & Proposed Targets

The project's measured service indicators and proposed objectives:

| Indicator                     |              Proposed Target |
| ----------------------------- | ---------------------------: |
| Scheduled-window Availability |                     `>= 99%` |
| Request Error Rate            |                      `<= 1%` |
| E2E p95 Latency               |         `<= 180 s` per model |
| Successful Throughput         |     `>= 50 files/hour/model` |
| Qwen Concurrency              | `>= 3 simultaneous requests` |

These are provisional objectives based on the measured capstone workload and are not production commitments.

---

## Streamlit Application

An interactive Streamlit interface supports file upload, inference-backend selection, and inspection of the generated tagging and competency-mapping results.

---


## Technologies

* Python 3.11
* FastAPI
* PyTorch
* vLLM
* Hugging Face Transformers
* Qwen2.5-VL
* OpenAI API
* BGE-M3
* Docker
* Kubernetes / k3s
* NVIDIA CUDA
* Prometheus
* Grafana
* Streamlit
* Pandas

---

## Team

**Team 2 — AI Data Center Operations Capstone**

**Content Tagging & Competency Mapping (CTCM)**
