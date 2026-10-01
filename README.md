# KMRL Intelligent Document Load Balancer Service

An intelligent, workload-predictive load balancer service positioned between **FastAPI** and **Celery Workers** over **RabbitMQ / Redis**.

---

## Key Technical Innovation

Unlike conventional round-robin or simple queue-depth load balancers, this system balances **predicted document processing cost**, business/safety priorities, language/OCR capabilities, worker resource utilization, and estimated completion times.

```text
 ┌─────────────────────────┐     ┌─────────────────────────┐
 │   PHP Web Frontend UI   │ ──► │  Streamlit Monitoring   │
 │   (http://localhost:8080)│     │  (http://localhost:8501)│
 └────────────┬────────────┘     └────────────┬────────────┘
              │                               │
              └───────────────┬───────────────┘
                              │
                              ▼
                 ┌─────────────────────────┐
                 │       FastAPI API       │
                 │   (http://localhost:8000)│
                 └────────────┬────────────┘
                              │
                              ▼
              ┌────────────────────────────────┐
              │   Intelligent Load Balancer    │
              │ 1. Feature Extraction          │
              │ 2. Workload Prediction (ML)    │
              │ 3. 5-Factor Priority & Aging   │
              │ 4. Capability & Health Filter  │
              │ 5. Circuit Breaker & Routing   │
              └───────────────┬────────────────┘
                              │
                              ▼
                      ┌───────────────┐
                      │   RabbitMQ    │
                      │ Priority Queue│
                      └───────┬───────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    ┌───────────┐       ┌───────────┐       ┌───────────┐
    │ Worker 1  │       │ Worker 2  │       │ Worker 3  │
    │ OCR/NLP   │       │ Malayalam │       │ High Cap. │
    └─────┬─────┘       └─────┬─────┘       └─────┬─────┘
          │                   │                   │
          └───────────────────┼───────────────────┘
                              ▼
                  ┌───────────────────────┐
                  │ PostgreSQL / pgvector │
                  └───────────────────────┘
```

---

## Core Algorithmic Components

### 1. Document Workload Prediction (Two-Stage Adaptive Model)
- **Stage A (Heuristic Linear Model)**:
  $$C = 0.35 P + 0.15 S + 0.25 O + 0.15 I + 0.10 L$$
  Where $P$=Pages, $S$=File Size, $O$=OCR Required, $I$=Image/Scan Ratio, $L$=Language Complexity.
- **Stage B (Random Forest Regressor)**:
  Trained automatically on stored `ProcessingHistory` data when $>20$ samples accumulate.
- **Adaptive Selector**: Uses ML prediction when sufficient history exists; falls back seamlessly to Heuristic scoring during cold-starts.

### 2. Task Prioritization & Starvation Prevention (Aging)
- **5-Factor Priority Formula**:
  $$P = 0.30 S_{safety} + 0.25 R_{regulatory} + 0.20 B_{business} + 0.15 D_{deadline} + 0.10 W_{waiting}$$
- **Starvation Prevention (Aging)**:
  $$P_{effective} = P_{base} + \text{AgingFactor} \times \text{WaitingTime}$$

### 3. Capability-Aware Worker Selection & Completion Time Estimation
- Filters out workers lacking language (e.g. Malayalam) or task capabilities (e.g. OCR).
- Computes predicted completion time for each eligible candidate worker:
  $$\text{PredictedCompletionTime}_i = \text{CurrentWorkload}_i + \text{NewTaskWorkload}$$
- Evaluates multi-factor score $W_s = 0.30 R + 0.25 Q + 0.20 C + 0.15 P + 0.10 F$.

### 4. Comprehensive Fault Tolerance
- **Exponential Backoff Retries**: 5s $\rightarrow$ 15s $\rightarrow$ 45s.
- **Worker Health State Machine**: `STARTING` $\rightarrow$ `HEALTHY` $\leftrightarrow$ `DEGRADED` $\rightarrow$ `UNHEALTHY`.
- **Circuit Breaker**: Trips to `OPEN` after 3 consecutive worker failures; recovers via `HALF_OPEN` trial state.
- **Dead Letter Queue (DLQ)** & **Idempotency Guard**: (`document_id` + `processing_version`).

---

## Quick Start & Running

### 1. Install Dependencies
```bash
pip install -r requirements.txt
```

### 2. Run FastAPI Load Balancer API
```bash
uvicorn app.main:app --reload --port 8000
```

### 3. Run Web Operations Dashboard (Python or PHP)
```bash
# Option A: Using Python (No PHP required)
python -m http.server 8080 -d frontend

# Option B: Using PHP (if PHP is installed on your machine)
php -S 127.0.0.1:8080 -t frontend
```

### 4. Run Real-Time Streamlit Monitoring Dashboard
```bash
streamlit run dashboard/app.py
```

### 5. Run Automated Test Suite
```bash
pytest tests/test_load_balancer.py -v
```

### 6. Run Comparative Evaluation Benchmark (Round-Robin vs. Intelligent Load Balancer)
```bash
python -m benchmarking.benchmark
```

### 7. Full Docker Compose Deployment (PostgreSQL + RabbitMQ + Redis + FastAPI + PHP Frontend + Celery + Dashboard)
```bash
docker-compose up --build
```
Access points when running Docker Compose:
- **PHP Operations UI**: http://localhost:8080
- **FastAPI Endpoint & Docs**: http://localhost:8000 / http://localhost:8000/docs
- **Streamlit Dashboard**: http://localhost:8501
