# eoAPI Monitoring & Logging (Operations BB integration)

> Status: draft for discussion. First increment for
> [EOEPCA/data-access#202](https://github.com/EOEPCA/data-access/issues/202), aligned with the
> [Operations BB documentation](https://eoepca.readthedocs.io/projects/operations/en/latest/)
> (STAC Scenario, ServiceMonitors, Alerting and SLOs) and the
> [EOEPCA+ demo of 8 May 2026](https://drive.google.com/drive/folders/1lvPqXoW1-fMYNZVvfJw3LPjwb38nt2BS?usp=drive_link).

## Goal

Clarify the monitoring and logging dependency between eoAPI and the Operations BB, which is
the acceptance criterion on #202, and take the first step towards the end state the BB
already specifies: native, bounded application metrics from the STAC services.

## The gap

The platform side already exists: APISIX gateway metrics, postgres-exporter, logs shipped by
Alloy to Loki, synthetic checks, and an SLO alerting workflow (99% of STAC requests under
500 ms). The application side is missing. `/metrics` returns 404 on `eoapi-stac` and
`eoapi-stac-auth-proxy`, and there is no ServiceMonitor in the `data-access` namespace. As a
result the gateway can see that a request is slow but not which STAC operation is slow.

## First delivery (STAC slice, about 15h)

The work lands upstream, in the apps and the `eoapi-k8s` chart we maintain, as a generic
opt-in capability:

- Opt-in `/metrics` on `stac-fastapi-pgstac` and `stac-auth-proxy`: request counters and
  duration histograms labelled by route template (the STAC operation), method and status
  class. No URL or per-collection labels, and the endpoint stays cluster-internal.
- An opt-in ServiceMonitor and native rate, error and latency dashboard panels in
  `eoapi-k8s`.
- EOEPCA enablement through Helm values in `eoepca-plus`, with no EOEPCA-specific code.
- A short hand-off note on #202 for Operations BB sign-off, including how the native series
  can back the existing SLO. Whether to rebase the burn-rate rules on them is the BB's
  decision.

## Split of responsibilities

eoAPI provides the native application metrics, the ServiceMonitor, the dashboard panels, and
logs written to stdout. The platform provides Prometheus, Grafana, Alertmanager, Keep and
the Alloy to Loki log pipeline, and it owns SLOs, alert routing and retention.

## Out of scope

- Other services (raster, vector, multidim), which can follow the same pattern in later
  increments.
- Tracing, since no Tempo is deployed on EOEPCA.
- Per-collection analytics
  ([eoAPI#193](https://github.com/developmentseed/eoAPI/issues/193)): a real need, but high
  cardinality, so it requires a bounded design agreed with the Operations BB.
