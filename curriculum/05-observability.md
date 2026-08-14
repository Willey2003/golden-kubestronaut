# Module 5 — Observability (PCA · OTCA)

Module 5 makes the cluster measurable. It targets **PCA** (Prometheus
metrics, PromQL, exporters, alerting) and **OTCA** (OpenTelemetry traces,
metrics, and logs, and the collector that connects everything). Observability
is the only topic every other certification assumes: CKA troubleshooting,
CKS runtime detection, and GitOps health checks all rest on it. The PCA and
OTCA banks are knowledge-heavy, but the exercises below give you real
queries and pipelines to run.

## Learning objectives

1. Explain observability's three pillars — metrics, logs, traces — and how
   OpenTelemetry standardises them.
2. Operate Prometheus: scrape configuration, service discovery, targets,
   and metric types.
3. Write PromQL: selectors, functions, rate/irate, histogram_quantile, and
   recording rules.
4. Instrument and export: client libraries, exporters, and Prometheus
   remote write.
5. Configure alerting: rules, Alertmanager routing, grouping, and
   inhibition.
6. Assemble an OpenTelemetry pipeline: SDKs, auto-instrumentation, the
   collector (receivers, processors, exporters), and backends.

## Key concepts

- **The three pillars**: metrics (numeric state over time — count and rate),
  logs (discrete events with timestamps), traces (request lifecycles across
  services). They answer *is it down*, *what happened*, and *where does it
  slow down* respectively.
- **Prometheus model**: every metric has a name and labels. The classic
  quartet: Counter (monotonic — use `rate()`), Gauge (can go down), Histogram
  (bucketed observations → `histogram_quantile`), Summary (client-side
  quantiles). `http_requests_total{code="500"}` is a series.
- **Scraping**: Prometheus pulls `/metrics` on an interval
  (`scrape_interval: 15s`). Static targets or `kubernetes_sd_configs` for
  service discovery; `relabel_configs` filter by labels; `metric_relabel_configs`
  mutate series. Exporters (node_exporter, kube-state-metrics) translate
  other systems into Prometheus format.
- **PromQL**: selectors (`rate(http_requests_total[5m])`), binary operators
  (`avg by (job) (rate(...[5m]))`), and aggregations. `rate()` is the correct
  function for counters; `irate()` for spiky short windows.
  `histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))`
  is the latency-percentile workhorse.
- **Recording rules**: precompute expensive queries (`job:http_requests:rate5m`),
  stored back into Prometheus so dashboards and alerting stay cheap.
- **Alerting**: `rules.yml` evaluates `expr` every `evaluation_interval`;
  `for:` prevents flapping; Alertmanager groups and routes by labels,
  deduplicates, and supports inhibition (a "page" suppresses the "incident
  page").
- **OpenTelemetry**: a single standard API + SDK + protocol for metrics,
  logs, and traces. `otelcol` is the collector; signals flow
  app → SDK → collector → backend.
- **Signals**: spans (trace segments with parent/child IDs), events, metrics
  (OTLP), and logs (log records with resource/scope). Context propagation
  (W3C `traceparent`) is what links spans across services.
- **Instrumentation**: auto (language SDKs attach to libraries — agent-based
  or `auto_instrumentation`), manual (`tracer.start_as_current_span`), and
  `instrumentation-library` metadata. Zero-code is fastest; manual spans are
  where the business context lives.
- **The collector**: receivers (OTLP, Prometheus, filelog, hostmetrics) →
  processors (batch, memory-limiter, tail_sampling, resource, attributes) →
  exporters (OTLP, Prometheus, Jaeger, Loki, Splunk). Pipelines are named,
  wired as `receivers:processors:exporters` per signal.
- **Backends**: Prometheus/Grafana, Tempo, Loki, Jaeger, Zipkin. The
  collector's job is decoupling — apps export OTLP, the platform owns
  where it goes.
- **Reliability vs. cost**: sampling (tail sampling drops low-value traces),
  `memory_limiter` (protect the collector), and cardinality control (labels
  must not include request IDs) are the operational OTCA questions.
- **Dashboards and the human loop**: Grafana renders PromQL as dashboards
  and fronts Alertmanager; the PCA bank asks which panel type fits which
  query, and how to build a dashboard *from* recording rules so panels and
  alerts read the same numbers. The three pillars matter here: the graph
  shows the metric, the log says what happened, the trace says where.
- **Federation and remote write**: `remote_write` streams series to a long-
  term store (Cortex, Thanos, Mimir) so the scraper stays stateless;
  `federation` pulls selected series from other Prometheus servers. Both
  answer "where do the numbers go when the local store is not enough".

## Hands-on exercises

Exercises 1–4 run against a live cluster (or `docker run prom/prometheus` for
a scratch target). Exercise 5 needs a small HTTP service to instrument.

### Exercise 1 — A running Prometheus and its targets

- Task: stand up Prometheus scraping itself, then confirm the `up` metric.
- Expected outcome: `up == 1` for the self-scrape target.

```sh
kubectl create namespace observability
helm install prometheus prometheus-community/kube-prometheus-stack --namespace observability
kubectl port-forward -n observability svc/prometheus-operated 9090 &
```

- Verification:

```sh
kubectl get svc -n observability
curl -s 'http://localhost:9090/api/v1/targets' | head -c 300
```

### Exercise 2 — PromQL on real metrics

- Task: from the stack's own metrics, compute the per-node memory
  utilisation and a 5-minute request rate for the API server.
- Expected outcome: both queries return one series per node / per instance.

```sh
curl -sG 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)' | head -c 400
curl -sG 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=rate(apiserver_request_total[5m])' | head -c 400
```

- Verification:

```sh
curl -s 'http://localhost:9090/api/v1/status/rules' | head -c 300
```

### Exercise 3 — Recording rule and a firing alert

- Task: define a recording rule for the request rate and an alert that
  fires when it drops below a floor; verify both appear in `/rules`.
- Expected outcome: the recording rule is queryable and the alert is listed
  with `state: pending` (or `firing` after `for:`).

```yaml
groups:
  - name: app.rules
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
      - alert: NoRequests
        expr: sum by (job) (rate(http_requests_total[5m])) < 1
        for: 10m
```

- Verification:

```sh
curl -sG 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=job:http_requests:rate5m'
```

### Exercise 4 — An OpenTelemetry collector pipeline

- Task: deploy the collector with a `hostmetrics → batch → prometheus/otlp`
  pipeline, and confirm metrics appear at the Prometheus endpoint.
- Expected outcome: `/metrics` on the collector exposes host metrics from
  `hostmetrics` receiver after processing.

```yaml
receivers:
  hostmetrics: {scrape_interval: 15s}
processors:
  batch: {}
exporters:
  prometheus: {endpoint: 0.0.0.0:8889}
service:
  pipelines:
    metrics:
      receivers: [hostmetrics]
      processors: [batch]
      exporters: [prometheus]
```

- Verification:

```sh
curl -s localhost:8889/metrics | grep system_cpu_utilization | head -3
```

### Exercise 5 — Instrument a service and see a trace (needs an HTTP app)

- Task: add the OTel SDK with auto-instrumentation to a small HTTP service,
  export via OTLP to the collector, and query spans from the backend.
- Expected outcome: a request through the service produces one trace with
  two spans (client + server) visible in the backend UI or `curl` of its
  search API.

```sh
otelcol-contrib run --config config.yaml &   # collector with otlp receiver
kubectl run app --image=your-instrumented-app:tag
```

- Verification:

```sh
curl -s 'http://localhost:16686/api/traces?service=your-app' | head -c 400
```

## Test yourself

- **Bank**: `banks/pca` — **Training**, `focus_domain = promql`, then
  `scraping-targets`; re-sit `alerting-recording-rules` as **Mastery**.
- **Bank**: `banks/otca` — **Training**, `focus_domain = instrumentation`,
  then **Mastery** on `collector-configuration`. Weak `promql` → Exercise 2;
  weak `collector-configuration` → Exercise 4.

## Self-check quiz

1. **Why `rate()` and not the counter itself?** — counters are monotonic; a
   bare value is meaningless without a time window. `rate(x[5m])` is per-
   second growth over five minutes.
2. **Histogram vs Summary — which can you aggregate across instances?** —
   histogram; client-side summaries cannot be summed meaningfully across
   replicas.
3. **What is `for:` in an alert rule for?** — it requires the condition to
   hold continuously for the duration before the alert fires, suppressing
   blips.
4. **Why does the OTel collector sit between apps and backends?** — apps
   export OTLP once; the collector owns sampling, batching, enrichment, and
   routing, so backends and SDK versions stay swappable.
5. **A trace is missing one hop. What to suspect first?** — propagation: the
   intermediate service wasn't instrumented or dropped the `traceparent`
   header.

## See also

- PCA and OTCA pages on linuxfoundation.org / cncf.io and prometheus.io
  (referenced by name; objectives change between releases).
- [Module 6 — Mesh & GitOps (ICA · CAPA · CGOA)](06-mesh-and-gitops.md) —
  next: connecting services at the mesh layer and shipping them declaratively.
