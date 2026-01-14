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
* Concurrency Correctness:
  * strictly increasing timestamps **or**
  * latest-write-wins
* Delivery semantics:
  * **at-least-once + dedupe**, or
  * **exactly-once** (explain)

---

## B) Query API (Go)

### Endpoints

* `GET /sensors/{id}/recent?limit=...`
* `GET /sites/{site}/agg?window=1m&from=...&to=...`
* `GET /healthz`
* `GET /metrics`

### Caching

* Bounded LRU + TTL
* Correctness: cached aggregates must never return impossible values
(e.g. negative energy, decreasing cumulative metrics).
* Define a clear staleness bound.

---

## C) Forecast Service (Python)
Provide a small forecasting service that trains per sensor (or per site) and produces short-horizon forecasts from historical telemetry stored in TDengine (or via the shared DB adapter).

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

Include a short write-up that explains the symptoms, the process for diagnosising, and steps taken to fix.

---

## Operational Expectations

* System starts via:

  ```
  docker compose up
  ```

---

## Evaluation Focus

* Concurrency correctness
* Sustained and burst performance
* Failure-mode reasoning
* Quality of tradeoffs and documentation

**Depth and correctness matter more than feature count.**
