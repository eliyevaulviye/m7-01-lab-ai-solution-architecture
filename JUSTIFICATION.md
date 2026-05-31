# JUSTIFICATION.md — Real-Time Fraud Scoring (Scenario A)

## 1. Serving Pattern: Online (Synchronous)

The scenario specifies a p95 latency budget of **80 ms end-to-end** and the requirement that the score is available *before the user sees payment confirmation*. That single constraint eliminates both alternatives:

- **Batch** is ruled out because scores must be computed per-transaction at the moment of submission, not pre-computed on a schedule. There is no population to score in advance — each transaction is novel.
- **Streaming (async)** is ruled out because a Kafka-based async pipeline would decouple the score from the user response, meaning we cannot hold the confirmation screen while the score arrives. Streaming is used in this design, but only for near-line feature population, not for the inference decision itself.

**Online serving** is therefore the only pattern that satisfies the synchronous decision contract. At **300 transactions/second peak**, a horizontally scaled scoring service with in-process model inference is well within commodity infrastructure limits (a single 8-core node running ONNX Runtime can handle ~2,000 lightweight tabular inferences per second).

---

## 2. Where Inference Runs: Cloud, In-Process

Inference runs **in the cloud, in-process within the Fraud Scoring Service**, co-located with the feature fetch logic.

**Edge inference is rejected** for this scenario. The model requires account history and device fingerprint features that live in a shared Redis cluster — features that cannot be replicated to the edge without introducing consistency hazards and prohibitive sync latency. Edge would also complicate model versioning: deploying to thousands of edge nodes on a weekly retraining cycle is an operational burden with no latency payoff when the feature fetch already requires a round-trip to cloud Redis.

**Separate inference microservice is rejected** in favor of in-process ONNX Runtime. An additional network hop to a dedicated model server (e.g., Triton) would consume 10–20 ms of the 80 ms budget and add a failure domain. For a single tabular model, the operational simplicity of in-process loading outweighs the flexibility of a separate server.

---

## 3. Latency, Throughput, and Cost Targets

| Dimension | Target | Priority |
|---|---|---|
| Latency | p95 ≤ 80 ms end-to-end | **Optimize** |
| Throughput | ≥ 300 TPS sustained, ≥ 500 TPS burst | **Optimize** |
| Cost | Cloud compute for scoring fleet | Budget constraint |

**Latency and throughput are the two optimized dimensions.** Cost is the budget: we accept higher compute spend to meet the latency SLA rather than batching requests to improve GPU utilization. Practically, this means running CPU-based ONNX inference (cheaper per-node, lower cold-start) rather than GPU serving, and right-sizing the Redis cluster for sub-5 ms reads even at peak.

The latency budget breakdown: ~5 ms network ingress, ~20 ms feature fetch (parallel Redis fan-out with timeout), ~10 ms inference, ~5 ms decision + response serialization, ~40 ms remaining as headroom and p95 tail buffer.

---

## 4. Fallback Strategy

The model can fail in two distinct ways:

**Model unavailable (inference service crash / Redis timeout):** The Decision Engine falls back to a **rules-only path** — a hard-coded rule set covering the highest-signal features (transaction amount > $5,000, new device + new geography, velocity > N in 1 hour). This path has no ML dependency and executes in < 5 ms. The fallback is not as accurate as the model but is preferable to blocking all transactions or passing everything through.

**Model wrong (false negatives / false positives):** The step-up-auth action serves as the third tier between allow and block. Borderline scores route to step-up rather than a hard block, reducing the cost of false positives (legitimate transactions blocked) while still raising friction for suspicious ones. The Prediction Logger captures every decision; confirmed fraud labels flow back through the Label Pipeline within 72 hours, feeding weekly retraining.

**Stale features:** If a feature key is missing from Redis (e.g., new account with no history), the Feature Fetch returns a zero-vector with a `cold_start` flag. The model was trained with cold-start examples and degrades gracefully to transaction-level features only.
