# Observability for the Data Access Building Block

> **Status: Draft for discussion.** This is a proposal/RFC, not committed scope. It tracks
> [EOEPCA/data-access#202](https://github.com/EOEPCA/data-access/issues/202) and
> [developmentseed/eoAPI#193](https://github.com/developmentseed/eoAPI/issues/193). The
> cross-team dependencies in the *Assumptions* table (A1–A9) are not yet confirmed; feedback is
> welcome inline.

## Context

EOEPCA+ requires the Data Access Building Block (built on **eoAPI**) to expose monitoring &
logging through standards-based observability. Two issues drive the work:

- **[EOEPCA/data-access#202](https://github.com/EOEPCA/data-access/issues/202)** — "Support
  implementation of Monitoring & Logging in the Operations Building Block." Acceptance: clarify
  the dependency between the Operations BB and eoAPI's monitoring/logging, and **make eoAPI
  metrics part of the BB**. EOEPCA deploys eoAPI on **Kubernetes** (eoapi-k8s Helm chart). The
  Operations BB owner also asked that **stac-auth-proxy** be covered alongside STAC.
- **[developmentseed/eoAPI#193](https://github.com/developmentseed/eoAPI/issues/193)** — add
  OpenTelemetry/Prometheus observability to the STAC, Raster (and Vector) APIs and visualize in
  Grafana (VEDA-style per-collection analytics).

The architecture follows
[`developmentseed/titiler-observability`](https://github.com/developmentseed/titiler-observability):
apps emit OTLP → an **OpenTelemetry Collector** fans out to **Prometheus** (metrics), **Tempo**
(traces) and **Loki** (logs) → **Grafana** dashboards.

**Scope:** metrics + logs across **four** FastAPI services (Raster/titiler-pgstac,
STAC/stac-fastapi-pgstac, Vector/tipg, **stac-auth-proxy**) as the baseline; **tracing** and
**per-collection analytics** as follow-ups; targeting **both** a docker-compose reference (this
repo) **and** the eoapi-k8s deployment integrating with EOEPCA's existing stack. A
productionization layer (image build/publish pipeline, CI smoke test, data scrubbing, dashboard
portability) is treated as first-class.

## Key design decisions (agreed with the Operations BB)

1. **Integrate with EOEPCA's existing k8s stack — do not duplicate it.** EOEPCA runs
   kube-prometheus-stack, Grafana, Loki, **Alloy (logs)**, dashboards and PrometheusRules. The
   k8s deliverable integrates via **ServiceMonitor/PodMonitor, dashboard ConfigMaps,
   PrometheusRules and OTEL env** — no parallel stack.
2. **Traces backend = Tempo (Grafana-native), and tracing is a later milestone.** The app side
   stays **backend-neutral via OTLP**.
3. **Metrics topology differs by target, by design.** OTLP → collector for compose; Prometheus
   Operator **pull** scraping (ServiceMonitor/PodMonitor) on k8s.
4. **Per-collection analytics ship as a follow-up, with carefully bounded labels** to avoid
   high-cardinality Prometheus metrics.
5. **Logs path differs by target.** Compose uses the **OTel collector → Loki**; on k8s the app
   emits structured, trace-correlated logs to **stdout**, shipped by the existing
   **Alloy → Loki** pipeline (no OTLP log push on k8s).

## Assumptions & dependencies to confirm before starting

Each item is a hard dependency on the Operations BB team or on upstream image internals; an
unconfirmed assumption invalidates a phase estimate.

| # | Assumption / dependency | Owner to confirm | Risk if wrong |
|---|---|---|---|
| A1 | **Tempo is deployed** in the EOEPCA cluster (Prometheus/Grafana/Loki/Alloy are confirmed; Tempo is not) | Operations BB | No trace backend on k8s; tracing milestone blocked or needs Tempo install |
| A2 | An **OTLP trace ingest path exists** on k8s (Alloy OTLP receiver, or a small collector we deploy) | Operations BB | Apps emit OTLP traces to nothing |
| A3 | **Container registry** the cluster can pull from, + permission to publish our instrumented images | Data Access / Ops | k8s path cannot run (see Phase 3) |
| A4 | **Grafana dashboard sidecar** label/folder convention, and **datasource UIDs/names** for Prometheus/Loki/Tempo | Operations BB | Dashboard ConfigMaps don't import / panels show "datasource not found" |
| A5 | **Dev/staging cluster access** (kubectl + helm + a reconciling Prometheus Operator) for testing the wiring | Operations BB | Phase 4 cannot be verified |
| A6 | **Worker model** of the upstream images (uvicorn single vs `--workers` vs gunicorn pre-fork) | Data Access (read images) | Auto-instrumentation drops/duplicates metrics |
| A7 | **PrometheusRules / cardinality budget** and label conventions the cluster enforces | Operations BB | Our metrics/labels get dropped or rejected |
| A8 | A **demo dataset** for end-to-end verification (collections + items + a COG) | Data Access | Cannot validate signals end-to-end |
| A9 | The upstream images can **emit JSON logs to stdout with `trace_id`/`span_id`** (or Alloy can parse their default format) | Data Access / Ops | k8s log↔trace correlation silently fails (see note below) |

## Architecture

The reference repo is a docker-compose orchestration layer — no Python source and no image
build/publish pipeline today. Services start via a `command:` that runs `uvicorn <module>:app`;
this is the injection point.

**Instrumentation strategy — zero-code OTel auto-instrumentation:**

- Build a thin image per service: `FROM <upstream image>`, install a **pinned** set of OTel
  packages (`opentelemetry-distro`, `opentelemetry-exporter-otlp`,
  `opentelemetry-exporter-prometheus`) and run `opentelemetry-bootstrap -a install`. **Pin
  versions and snapshot the bootstrap output** for reproducible builds and to avoid pulling
  instrumentation that conflicts with the upstream image's pinned `fastapi`/`starlette`/
  `httpx`/`psycopg` (validated in the spike).
- Change each `command:` from `uvicorn ...` to `opentelemetry-instrument uvicorn ...` — or use
  a gunicorn `post_fork` hook if the image runs a pre-fork server, so instrumentation
  initializes per worker.
- Configure via `OTEL_*` env. The **exporter env differs by target** (see the matrix). Common
  across targets: `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`,
  `OTEL_PYTHON_LOG_CORRELATION=true`, `OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true`.

This gives, for all services: HTTP server traces, request-duration/active-request metrics,
DB-query spans, and trace-correlated logs — uniformly, with zero upstream code changes.

### Per-signal exporter configuration (compose vs k8s)

The two targets use different exporters because k8s integrates with a pull-based stack and a
separate log pipeline:

| Signal | docker-compose | eoapi-k8s | Notes |
|---|---|---|---|
| **Metrics** | `OTEL_METRICS_EXPORTER=otlp` → collector; Prometheus scrapes the collector | **Recommended:** `otlp` → a **namespaced OTel Collector** that exposes Prometheus; **ServiceMonitor scrapes the collector** (Operator pull). *Alt:* `OTEL_METRICS_EXPORTER=prometheus` in-app (opens :9464), ServiceMonitor scrapes each pod | OTLP push has no scrapeable endpoint; pull needs either a collector or the in-app Prometheus exporter + a named Service port. The in-app OTel Python Prometheus exporter is comparatively limited/experimental — prefer the collector path |
| **Logs** | `OTEL_LOGS_EXPORTER=otlp` → collector → Loki | `OTEL_LOGS_EXPORTER=none`; emit **structured JSON to stdout** with `trace_id`/`span_id`; **Alloy → Loki** | Not purely zero-code (A9): OTel only *injects* trace IDs into log records — emitting them as JSON to stdout depends on the image's logging formatter, so we may need a logging-config override or Alloy-side parsing. Correlation also depends on Loki/Alloy promoting the trace ID to a derived field (A4) |
| **Traces** (follow-up) | `OTEL_TRACES_EXPORTER=otlp` → collector → Tempo | `OTEL_TRACES_EXPORTER=otlp` → OTLP receiver (Alloy or collector) → Tempo | Depends on A1/A2; needs a head-sampling default |

> **Logs caveat (A9):** on the **compose** target the collector receives OTLP logs directly, so
> formatting is a non-issue. On **k8s** the stdout path means trace↔log correlation only works
> if the emitted log lines actually carry the trace IDs in a parseable form — confirm the
> upstream images' log format and add a small logging-config override (or Alloy pipeline stage)
> if needed. Treat this as part of Phase 4, not free.

**Recommendation:** run a **lightweight per-namespace OTel Collector on k8s too** (a single
Deployment/DaemonSet owned by the Data Access namespace — not a parallel stack). It (a) keeps
**app config uniform across targets** (always OTLP push), (b) provides a clean **OTLP ingress
for traces** (A2), and (c) is scraped by a single **ServiceMonitor** for metrics (Operator
pull). The direct in-app Prometheus-exporter path is the documented fallback if the Operations
BB prefers no collector. **This choice needs Operations BB sign-off.**

**Backends:**

- **docker-compose:** ported configs from `titiler-observability` (`otel-collector-config.yml`,
  `prometheus.yml`, `loki-config.yaml`, Grafana provisioning); the collector fans out to
  **Prometheus / Tempo / Loki** → Grafana.
- **eoapi-k8s:** integrate with EOEPCA's **existing** kube-prometheus-stack (pull), Grafana
  (ConfigMap dashboards), Loki fed by **Alloy**, and PrometheusRules — plus the recommended
  namespaced collector for OTLP ingress.

**Per-collection analytics are not free from auto-instrumentation.** OTel HTTP server metrics
are labeled by **route _template_** (`/collections/{collection_id}`), not the concrete
collection ID — deliberate, to bound cardinality. The VEDA-style per-collection breakdown
eoAPI#193 headlines therefore needs **custom instrumentation**: a thin ASGI middleware (or
`SpanProcessor`) lifting the collection identifier into a **bounded** span/metric attribute. It
lives in the overlay (no upstream PRs) but is real code, and ships as a follow-up with bounded
labels only (collection IDs, never per-tile — per-tile detail belongs in traces/logs).

**Raster richer metrics.** titiler-core ships a `[telemetry]` extra for tile-level and
per-collection metrics. It must not run together with `opentelemetry-instrument` (double
instrumentation duplicates spans/metrics) and likely needs a version above the pinned
`titiler-pgstac:1.7.2`. Treated as a follow-up.

## Security & data governance

With **stac-auth-proxy** in scope, auto-instrumented httpx/FastAPI spans and correlated logs
can capture sensitive data (URLs with tokens/query params, `Authorization` headers, user
identifiers) in a multi-tenant deployment. Baseline controls, configured in the overlay:

- **HTTP header capture stays OFF** (default — assert it; do not set
  `OTEL_INSTRUMENTATION_HTTP_CAPTURE_HEADERS_*`).
- **Scrub URLs/query strings** of secrets via a collector `attributes`/`redaction` processor,
  and avoid logging full request URLs at the proxy.
- **Never record auth tokens** as span attributes or log fields; maintain a deny-list.
- **Access control & retention** of Grafana/Loki/Tempo data is owned by the Operations BB;
  document the boundary in the #202 deliverable.

A redaction processor config + a documented attribute deny-list are part of Phase 1 (compose)
and validated on k8s in Phase 4.

## Implementation plan

Estimates are **ranges**. They credit two facts that compress the work: the
`titiler-observability` reference provides a working compose stack to **copy-adapt** (not build
from scratch), and EOEPCA's existing k8s stack means the Kubernetes milestone is
**integration, not deployment**. Hours are engineering effort; cross-team coordination latency
is called out separately as the top schedule risk.

### Milestone 1 — Compose reference (near-term, ~10–16h core)

**Phase 0 — Spike & de-risk (2–4h).**

1. Pick raster (titiler) **and** STAC to prove uniformity across two bases, not just one.
2. Add a thin instrumentation Dockerfile; switch the command to `opentelemetry-instrument`.
3. Prove the three highest risks up front: **(a) dependency-conflict** — the pinned OTel
   install doesn't break the upstream image; **(b) worker-model** behaviour (single vs
   `--workers`/gunicorn pre-fork); **(c) metric-label cardinality** — metrics are
   route-templated, and tile `z/x/y` / collection IDs appear only in traces/logs.
4. Stand up `otel-collector` + `prometheus` + `grafana`; confirm spans/metrics arrive.

**Phase 1 — docker-compose reference: metrics + logs (5–8h core; +3–5h for auth-proxy + redaction) — PRIMARY DELIVERABLE.**
Baseline = metrics + logs across the three core services; **stac-auth-proxy** and the redaction
processor add ~3–5h and are recommended but separable. Most of the backend stack is copied from
the `titiler-observability` reference. New files:

- `dockerfiles/Dockerfile.otel` — thin layer parameterized by `ARG BASE_IMAGE` (pinned OTel
  packages; bootstrap output snapshotted), or per-service Dockerfiles where bases differ
  (raster is `linux/amd64`).
- `observability/otel-collector-config.yml` (incl. the redaction/attributes processor),
  `observability/prometheus.yml`, `observability/loki-config.yaml`,
  `observability/grafana/{datasources.yml,dashboards.yml,dashboards/*.json}`.
- `docker-compose.observability.yml` (overlay, run with
  `docker compose -f docker-compose.yml -f docker-compose.observability.yml up`) adding
  `otel-collector`, `prometheus`, `loki`, `grafana` (and optionally `tempo` as plumbing only —
  see below), and overriding the three core services (plus `stac-auth-proxy` when that separable
  +3–5h is included) to build the OTel image, set the per-target `OTEL_*` env (metrics/logs =
  otlp), and use `opentelemetry-instrument`. Leaves the default `docker-compose.yml` unchanged.

**Phase 2a — Basic provisioned dashboards (2–4h).**

- Per-service overview: request rate / error rate / p50-p95 latency + a Loki logs panel.
- **Templatize datasource references** (`${datasource}` variables) from the start so the same
  JSON is portable to EOEPCA's Grafana later (avoids hardcoded datasource-UID breakage).
- Provision via `observability/grafana/dashboards/*.json`.

→ **Milestone 1 total: ~10–16h core** (~15–22h with auth-proxy + redaction). A runnable,
documented metrics+logs reference. (If the `tempo` service is wired in compose here it is
**plumbing only** — sampling, trace↔log correlation panels and the k8s trace path are all
Milestone 3.)

### Milestone 2 — Productionization & k8s integration (14–22h)

**Phase 3 — Image build & publish pipeline (3–5h).** The repo has none today and k8s needs
**published** images (a values `command:` change cannot add packages at runtime).

- CI to build, **pin/lock**, **scan**, tag and push the instrumented images to the registry
  (A3); reproducible builds (snapshot bootstrap output).

**Phase 4 — k8s integration with the existing EOEPCA stack (8–14h eng — top schedule risk).**
The artifacts are small (mostly YAML against an existing stack); the range reflects running
against a live cluster (RBAC, namespaces, selectors, conventions). Coordination latency is
separate from these hours.

- `values-observability.yaml`: per-target `OTEL_*` env + `opentelemetry-instrument` command.
- **Metrics:** the recommended **namespaced collector** scraped by a **ServiceMonitor** (or
  in-app Prometheus exporter + named Service port — per the matrix and A-confirmation).
- **Logs:** structured stdout → existing **Alloy → Loki**; coordinate the trace-ID derived
  field with the Operations BB (A4).
- **Dashboards:** ConfigMaps with the sidecar label (A4), reusing the templatized JSON.
- **PrometheusRules:** basic availability/latency/error rules wired into the existing set (A7).

**Phase 5 — Docs + #202 clarification + governance writeup (3–5h).**

- An observability page in this docs site: how to run, the per-target env matrix, what each
  signal shows, screenshots, the `OTEL_SDK_DISABLED` kill-switch, resource-overhead notes.
- **#202 deliverable:** the "Monitoring & Logging ↔ Operations Building Block" section — eoAPI
  exposes OTLP/Prometheus **metrics + logs** (traces in a later milestone); the Operations BB
  consumes them via its existing kube-prometheus-stack (pull), Grafana, and Alloy → Loki;
  ownership/retention/access boundaries. Cross-link eoAPI#193 and data-access#202.
- **CI smoke test** for the observability overlay (boots, `/metrics` responds, a sample request
  produces series) — prevents bit-rot of a reusable reference.

### Milestone 3 — Tracing end-to-end (5–10h)

Tempo (depends on A1/A2), OTLP from all services + the proxy, **context propagation through
stac-auth-proxy**, a **head-sampling default**, and trace↔log correlation panels in Grafana, on
both compose and k8s.

### Milestone 4 — Per-collection analytics (6–10h)

The ASGI/`SpanProcessor` middleware adding a **bounded** `collection` attribute so eoAPI#193's
VEDA-style per-collection metrics populate, across STAC & Vector (+ raster if native telemetry
is enabled). Bounded labels only; cardinality budget agreed with the Operations BB (A7).

### Milestone 5 — First-class platform support (15–30h)

Templated Helm support (optional collector subchart), an alerting/SLO rule library with
runbooks, native titiler `[telemetry]` for richer raster metrics, and sampling tuning.

## Success metrics

Measurable acceptance criteria for the capability. Metrics 1–7 gate Milestones 1–2; 8–9 gate
the follow-up milestones.

**Coverage & availability**

1. **100% of in-scope services** emit metrics + logs (3 core; 4 incl. stac-auth-proxy).
2. Per-service **request rate, error rate, latency p50/p95/p99** are visible in Grafana with
   **zero manual setup** (provisioned in compose / auto-imported on k8s).

**Integration — the #202 acceptance test**

3. ServiceMonitor/PodMonitor **scraped by the existing cluster Prometheus**; dashboards
   **auto-import into the existing Grafana**; logs **queryable in the existing Loki** —
   confirmed by an Operations BB sign-off that *"eoAPI metrics are part of the BB."*

**Quality guardrails**

4. **Cardinality budget respected** — added active series < an agreed N; an automated check
   asserts no unbounded label (collection ID, tile `z/x/y`) appears on a metric.
5. **Instrumentation overhead bounded** — measured against an uninstrumented baseline (see
   Verification): added **p95 latency < 5%** and added per-pod **CPU/mem < an agreed %** under
   representative load (protects the tile data plane).

**Operational outcome**

6. **Time-to-detect < 60s** — a synthetic error/latency injection appears on the dashboard
   within one scrape interval.
7. An operator can answer *"which service/route is erroring or slow right now?"* and
   *"what is request volume per service?"* **from the dashboards, without ad-hoc queries**
   (verified by a walkthrough).

**Follow-up milestones**

8. *(M3)* Clicking a trace ID **jumps between trace ↔ logs**; context **propagates through
   stac-auth-proxy** end-to-end.
9. *(M4)* Per-collection request breakdown **populates with bounded labels only**, within the
   agreed cardinality budget.

## Definition of Done (per issue)

- **eoAPI#193** (upstream developmentseed/eoAPI): the implementation lands as **overlays in the
  data-access repo, not upstream PRs**. Confirm with the #193 owner that closing #193 is
  acceptable without upstream changes **and** whether the **per-collection analytics** headline
  (Milestone 4) is required for closure or can land later. If #193 expects changes in eoAPI /
  eoapi-k8s itself, scope a separate upstream PR.
- **data-access#202**: confirm whether the docs/clarification (Phase 5) closes it, or whether
  "make eoAPI metrics part of the BB" requires the **k8s integration (Phase 4) merged and
  deployed**. Treat Phase 4 as part of #202's DoD unless the Operations BB says otherwise.

## Verification

- `docker compose -f docker-compose.yml -f docker-compose.observability.yml up --build`.
- Load demo data (A8), then hit each API:
    - STAC: `curl localhost:8081/collections` and `/collections/{id}`
    - Raster: a `/cog/tiles/...` or pgstac tile request on `:8082`
    - Vector: a tipg collection request on `:8083`
    - Auth-proxy: a request through the proxy front door
- **Cardinality gate:** in Prometheus assert metric labels are route-templated — tile `z/x/y`
  and collection IDs must **not** appear as metric labels (only in traces/logs).
- **Dependency-conflict gate:** the instrumented image starts cleanly and serves requests (no
  import/version errors from the OTel install).
- **Multi-worker gate:** under the production worker model, metrics are neither dropped nor
  duplicated across workers.
- **Overhead gate:** run the same representative load against an **uninstrumented control** and
  the instrumented build; the delta is what success metric #5 (p95 < 5%, CPU/mem < agreed %) is
  measured against — without the baseline run the "< 5%" cannot be asserted.
- **Security gate:** no `Authorization` header or token/query-secret appears in any span
  attribute or log line.
- In **Grafana** (`localhost:3000`): per-service request-rate/latency panels populate and
  **Loki** shows the structured logs; datasource variables resolve.
- k8s: `helm upgrade` with `values-observability.yaml`; confirm the **ServiceMonitor/PodMonitor**
  is scraped by the **existing cluster Prometheus** (`kubectl get servicemonitor`, series
  present), the **dashboard ConfigMap** auto-imports into the **existing Grafana**,
  **PrometheusRules** load, and logs appear in the **existing Loki via Alloy**.
- **CI smoke test** passes (overlay boots, `/metrics` responds, sample request → series).
- **Tracing (Milestone 3):** Tempo shows traces, logs are trace-correlated, clicking a trace ID
  jumps between signals, context propagates through the auth proxy.

## Effort summary

The core value (a working metrics+logs capability in compose and integrated into the EOEPCA
cluster) is **~24–38h**; the minimal "just get metrics+logs into the BB" path is **~15–25h**.
The remaining milestones are valuable but discretionary.

| Milestone | Work | Range |
|---|---|---|
| **1 — Compose reference** *(near-term)* | Spike + metrics/logs (3 core svcs; +auth-proxy/redaction) + basic dashboards | **10–16h** (15–22h w/ extras) |
| 2 — Productionization & k8s | Image build/publish + k8s integration (existing stack) + docs/#202 + CI smoke + security validation | 14–22h |
| 3 — Tracing | Tempo end-to-end, sampling, proxy propagation, correlation | 5–10h |
| 4 — Per-collection analytics | Bounded-label middleware (STAC/Vector/raster) | 6–10h |
| 5 — Platform hardening | First-class Helm, alerting/SLOs, native titiler telemetry, sampling tuning | 15–30h |
| — | **Core value (M1+M2)** | **~24–38h** |
| — | **Program total** | **~50–88h** |

**Top schedule risk:** Phase 4 (k8s integration) — it depends on cross-team access and
conventions (A1–A7) and on coordination latency that is not in the engineering hours above.
Treat A1–A9 as gating; do not start Phase 4 until they are confirmed.

## Open implementation questions

1. **Raster double-instrumentation** — `opentelemetry-instrument` (auto) and titiler-core native
   `[telemetry]` can duplicate spans/metrics. Pick one path for raster.
2. **titiler `[telemetry]` availability** — likely not in `titiler-pgstac:1.7.2` (the reference
   installs titiler from main); may need a version bump/build.
3. **Dashboard panel reuse** — the reference `grafana-dashboard.json` is titiler/tile-specific;
   STAC, tipg and the proxy need new panels (not a straight port).
4. **Cardinality budget** — agree a concrete active-series budget and bounded `collection` label
   policy with the Operations BB before Milestone 4.
