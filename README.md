# High-Throughput EMS Telemetry Platform (Go + Python)

Build a mini system that **ingests high-rate sensor telemetry**, stores it in **TDengine (preferred)** via an adapter, exposes **query APIs with caching**, and provides a **Python forecasting service**.

The focus is **correctness, performance, and resilience under realistic failure modes**.

---

## Deliverables

* Git repository with:

  * `ingester/` (Go)
  * `api/` (Go)
  * `forecaster/` (Python)
  * `docker-compose.yml`
  * `README.md`
* Tests
* **Engineering Notes** (1–2 pages):

  * architecture & tradeoffs
  * bottlenecks and tuning
  * benchmarks (machine specs included)
  * failure scenarios tested

---

## A) Ingestion Service (Go)

### Input

Generated telemetry stream (you implement generator):

```
sensor_id, ts(ms), power_w, voltage_v, current_a, site_id, tags, tenant_id
```

### Output

Write to **TDengine** (local fallback allowed via adapter).

### Must Handle

* Out-of-order timestamps (1–3%)
* Duplicates (0.5–1%)
* Bursts (10× rate for 10s every 60s)
* DB brownouts (writes stall or time out)

### Requirements

* Sustain **N events/sec** locally (you choose N and justify)
* Batching: size- and time-based
* Backpressure with bounded memory
* Connection pooling, timeouts, retries w/ jitter
* Delivery semantics:

  * **at-least-once + dedupe**, or
  * **exactly-once** (explain)

---

## Required Engineering Challenges

### 1) Concurrency correctness

Enforce **one per-sensor rule**:

* strictly increasing timestamps **or**
* latest-write-wins

Must hold under concurrency and out-of-order input.
Include at least one deterministic correctness test.

---

### 2) Backpressure & memory safety

Under sustained DB slowdown:

* ingestion slows or sheds load
* memory remains bounded
* no goroutine leaks

Provide evidence (metrics + profiling notes).

---

### 3) Poison events & schema drift

≥0.1% malformed events (missing fields, bad values, unknown tags).

Requirements:

* system must not crash
* quarantine invalid events
* include rejection reason codes

---

### 4) Adaptive batching

Automatically adjust batch size or flush interval based on:

* DB latency
* error/timeout rate
* internal queue depth

Must be stable (no oscillation); explain logic.

---

### 5) Multi-tenant fairness

One noisy tenant must not starve others.

Implement:

* weighted fair scheduling **or**
* per-tenant rate limiting

Demonstrate with a test or load scenario.

---

### 6) Restart & recovery behavior

On restart:

* ingestion resumes safely
* duplicates don’t violate correctness guarantees
* partially flushed batches are handled explicitly

Document guarantees and limitations.

---

## B) Query API (Go)

### Endpoints

* `GET /sensors/{id}/recent?limit=...`
* `GET /sites/{site}/agg?window=1m&from=...&to=...`
* `GET /healthz`
* `GET /metrics`

### Caching

* Bounded LRU + TTL
* Expose metrics: hit rate, evictions, stale reads (if any)

**Correctness constraint:** cached aggregates must never return impossible values
(e.g. negative energy, decreasing cumulative metrics).
Define a clear staleness bound.

---

## C) Forecast Service (Python)

* FastAPI
* Endpoints:

  * `POST /train?sensor_id=...`
  * `GET /predict?sensor_id=...&horizon=60m&step=1m`
* Model can be simple, but must:

  * handle missing data and outliers
  * version artifacts per sensor/site
  * produce reproducible predictions

---

## Debugging Challenge (Required)

Identify and fix **one non-trivial bug**, e.g.:

* data race
* time-unit mismatch
* retry storm
* cache inconsistency
* batching edge case

Include:

* failing test or benchmark
* short write-up: symptoms → diagnosis → fix

---

## Operational Expectations

* System starts via:

  ```
  docker compose up
  ```
* Include:

  * health checks
  * structured logs
  * Prometheus-style metrics
* Provide at least one load run with observed metrics

---

## Evaluation Focus

* Concurrency correctness
* Sustained and burst performance
* Failure-mode reasoning
* Quality of tradeoffs and documentation
* Operational maturity

**Depth and correctness matter more than feature count.**
