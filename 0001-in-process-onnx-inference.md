# ADR 0001: Load Model In-Process via ONNX Runtime Instead of a Separate Model-Serving Microservice

## Context

The Fraud Scoring Service must return a decision within an 80 ms p95 budget. Every millisecond of network overhead between the orchestration logic and the inference compute reduces the headroom available for feature fetching and tail-latency spikes. A separate model-serving microservice (e.g., Triton Inference Server, TorchServe, BentoML) is the conventional pattern for ML serving but introduces an additional synchronous network hop on the critical path.

## Decision

We load the fraud model directly into the Fraud Scoring Service process using **ONNX Runtime** (CPU). The model artifact is pulled from the Model Registry at service startup and held in memory. Model updates are applied via a **rolling restart** triggered by the CI/CD pipeline after each registry promotion. There is no separate inference process; the scoring service owns both feature orchestration and inference.

## Alternatives Rejected

- **Triton Inference Server (sidecar or separate pod):** Adds a loopback or inter-pod network hop of 10–20 ms at p95, consuming 12–25% of the total latency budget before any feature work is done. Triton's strengths (GPU batching, multi-framework) are irrelevant for a single lightweight tabular model at 300 TPS.

- **Separate gRPC model microservice (shared across teams):** Operationally appealing for multi-team reuse, but couples the fraud service's deployment lifecycle to a shared dependency. A shared service degrading under load from another consumer could breach the fraud SLA. Blast radius is too wide.

- **Edge inference on payment terminal / mobile SDK:** Requires replicating account-history features to the edge, which introduces consistency risk and prohibitive sync overhead. The model's highest-signal features (cross-account velocity, historical fraud rate) are inherently server-side.

## Consequences

- **Positive:** Eliminates one network hop; inference latency is bounded by CPU compute (~5–10 ms for a tabular XGBoost/LightGBM model), not network jitter. Deployment is simpler — one artifact, one service.
- **Positive:** Failure domain is contained. A model crash brings down only the instances running that version, not a shared serving fleet.
- **Negative:** Rolling restarts during model updates create a brief window (seconds) where old and new model versions are simultaneously in production across instances. Score distributions may differ slightly during this window; downstream monitoring must tolerate version-tagged predictions.
- **Negative:** If the model grows significantly (e.g., a deep neural net for richer embeddings), in-process CPU inference may not meet the latency target and this decision must be revisited with GPU serving or a sidecar.
- **Negative:** Memory footprint of the model lives inside the scoring service pod, reducing headroom for feature-cache warming and connection pools.

## Revisit If

The model architecture changes from a gradient-boosted tree to a neural network requiring > 50 ms CPU inference time, or if throughput requirements exceed 1,000 TPS sustained and batching across requests becomes necessary to achieve acceptable GPU utilization.
