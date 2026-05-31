# Architecture: Real-Time Fraud Scoring

## Serving Boundary Legend
- 🟥 **Offline** — batch jobs, training, registry
- 🟨 **Near-line** — feature store population, async enrichment
- 🟩 **Online** — synchronous request path (latency-critical)

---

```mermaid
flowchart TD
    %% ── ONLINE PATH (p95 ≤ 80 ms) ──────────────────────────────
    subgraph online["🟩 Online — Synchronous Request Path"]
        A["Payments API Gateway\n(REST / gRPC)"]
        B["Fraud Scoring Service\n(stateless, horizontally scaled)"]
        C["Feature Fetch\n(parallel fan-out, timeout 20 ms)"]
        D["Model Inference Engine\n(ONNX Runtime, in-process)"]
        E["Decision Engine\n(threshold + rules overlay)"]
    end

    %% ── NEAR-LINE ────────────────────────────────────────────────
    subgraph nearline["🟨 Near-line — Feature Store Population"]
        F["Account Feature Writer\n(Flink streaming job)"]
        G["Feature Store\n(Redis Cluster, TTL-keyed)"]
        H["Device Fingerprint Cache\n(Redis, per-device key)"]
    end

    %% ── OFFLINE ──────────────────────────────────────────────────
    subgraph offline["🟥 Offline — Training & Registry"]
        I["Raw Event Lake\n(S3 / GCS, Parquet)"]
        J["Label Pipeline\n(confirmed fraud labels, T+3d)"]
        K["Training Job\n(XGBoost / LightGBM, weekly)"]
        L["Model Registry\n(MLflow, versioned artifacts)"]
    end

    %% ── DOWNSTREAM ACTIONS ───────────────────────────────────────
    subgraph downstream["Downstream Actions"]
        M["allow → Transaction Processor"]
        N["block → User Notification Service"]
        O["step-up-auth → 2FA Orchestrator"]
    end

    %% ── MONITORING ───────────────────────────────────────────────
    subgraph monitoring["Monitoring & Feedback Loop"]
        P["Prediction Logger\n(async, Kafka topic)"]
        Q["Drift Monitor\n(PSI / KL divergence, daily)"]
        R["Retraining Trigger\n(threshold breach → CI pipeline)"]
    end

    %% ── FLOWS ────────────────────────────────────────────────────
    A -- "txn request (JSON, <1 KB)" --> B
    B -- "account_id, device_id" --> C
    C -- "account features (velocity, history)" --> G
    C -- "device fingerprint vector" --> H
    G -- "feature vector" --> D
    H -- "feature vector" --> D
    D -- "fraud score (float 0–1)" --> E
    E -- "allow / block / step-up" --> M & N & O

    %% near-line feed
    A -- "txn event (Kafka)" --> F
    F -- "account feature writes (batch micro)" --> G

    %% offline feed
    A -- "raw txn events" --> I
    I -- "historical features + labels" --> J
    J -- "labeled dataset" --> K
    K -- "model artifact + metadata" --> L
    L -- "model version deploy (canary)" --> D

    %% monitoring / feedback
    B -- "score + features (async)" --> P
    P -- "prediction stream" --> Q
    Q -- "drift signal" --> R
    R -- "retrain trigger" --> K
    J -- "confirmed fraud labels" --> Q
```

---

## Component Glossary

| Component | Role | SLA / Notes |
|---|---|---|
| Payments API Gateway | Entry point; routes txn to scoring service | — |
| Fraud Scoring Service | Orchestrates feature fetch + inference + decision | p95 ≤ 80 ms owner |
| Feature Fetch | Parallel Redis calls; 20 ms timeout with stale-fallback | Fan-out, not sequential |
| Model Inference Engine | ONNX Runtime, model loaded in-process | ~5–10 ms per score |
| Decision Engine | Converts score to action using threshold + rules | Rules can override model |
| Feature Store (Redis) | Account velocity, history; TTL-keyed per account | < 5 ms p99 read |
| Device Fingerprint Cache | Per-device feature vectors pre-computed | < 5 ms p99 read |
| Flink Streaming Job | Consumes txn events; updates account features near-real-time | Lag < 2 s |
| Raw Event Lake | Immutable audit log; source of truth for retraining | S3 / GCS |
| Label Pipeline | Joins fraud labels (T+3d) with prediction records | Weekly batch |
| Training Job | Retrains model on labeled data | Weekly or on drift trigger |
| Model Registry | Versioned artifacts; gates canary deploys | MLflow |
| Prediction Logger | Async Kafka write from scoring service | No latency impact |
| Drift Monitor | Computes PSI / KL divergence between score distributions | Daily job |
| Retraining Trigger | Fires CI pipeline if drift exceeds threshold | Automated |
