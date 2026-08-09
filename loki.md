# Observability Lab Workshop
### Log Aggregation with Loki · Distributed Tracing with Tempo · Unified Correlation in Grafana

**Format:** Hands-on, self-paced
**Duration:** ~3 hours (3 modules, ~1 hour each)
**Prerequisites:** Docker & Docker Compose installed, a terminal, a text editor, port 3000/3100/3200/4317/9090 free on localhost

---

## 0. Environment Setup

All three modules share one stack: Grafana, Loki, Promtail/Alloy, Tempo, Prometheus, and a small sample app that emits logs, metrics, and traces together.

### 0.1 Create the project folder

```bash
mkdir observability-lab && cd observability-lab
mkdir -p config/loki config/tempo config/prometheus config/grafana/provisioning/datasources
```

### 0.2 `docker-compose.yml`

```yaml
version: "3.8"

services:
  grafana:
    image: grafana/grafana:11.3.0
    ports: ["3000:3000"]
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_FEATURE_TOGGLES_ENABLE=traceqlSearch
    volumes:
      - ./config/grafana/provisioning:/etc/grafana/provisioning
    depends_on: [loki, tempo, prometheus]

  loki:
    image: grafana/loki:3.2.0
    ports: ["3100:3100"]
    command: -config.file=/etc/loki/config.yaml
    volumes:
      - ./config/loki/config.yaml:/etc/loki/config.yaml

  alloy:
    image: grafana/alloy:v1.4.3
    ports: ["12345:12345"]
    command: run --server.http.listen-addr=0.0.0.0:12345 /etc/alloy/config.alloy
    volumes:
      - ./config/alloy.alloy:/etc/alloy/config.alloy
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on: [loki]

  tempo:
    image: grafana/tempo:2.6.0
    ports:
      - "3200:3200"   # Tempo query API
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    command: -config.file=/etc/tempo/config.yaml
    volumes:
      - ./config/tempo/config.yaml:/etc/tempo/config.yaml

  prometheus:
    image: prom/prometheus:v2.55.0
    ports: ["9090:9090"]
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --enable-feature=exemplar-storage
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml

  demo-app:
    image: grafana/microservices-demo-mythical-server:latest
    ports: ["4000:4000", "4001:4001"]
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://tempo:4318
    depends_on: [tempo]
```

> If `demo-app` isn't available in your registry, skip it for now — Module 2 includes a 15-line script that sends traces manually, and Module 1 works against any container's stdout logs.

### 0.3 Minimal service configs

`config/loki/config.yaml`
```yaml
auth_enabled: false
server:
  http_listen_port: 3100
common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory
schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
```

`config/tempo/config.yaml`
```yaml
server:
  http_listen_port: 3200
distributor:
  receivers:
    otlp:
      protocols:
        grpc:
        http:
storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
    wal:
      path: /var/tempo/wal
metrics_generator:
  registry:
    external_labels:
      source: tempo
  storage:
    path: /var/tempo/generator/wal
    remote_write:
      - url: http://prometheus:9090/api/v1/write
        send_exemplars: true
overrides:
  metrics_generator_processors: [service-graphs, span-metrics]
```

`config/prometheus/prometheus.yml`
```yaml
global:
  scrape_interval: 15s
storage:
  tsdb:
    out_of_order_time_window: 10m
remote_write_receiver: {}
scrape_configs:
  - job_name: tempo
    static_configs:
      - targets: ["tempo:3200"]
```

`config/alloy.alloy` (Grafana Alloy — discovers and tails all Docker container logs)
```river
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

loki.source.docker "default" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.docker.containers.targets
  forward_to = [loki.write.default.receiver]
}

loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

`config/grafana/provisioning/datasources/datasources.yaml` — pre-wires the correlation from Module 3 so it's ready when you get there:
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: false
    jsonData:
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    uid: loki
    jsonData:
      derivedFields:
        - datasourceUid: tempo
          matcherRegex: "trace_id=(\\w+)"
          name: TraceID
          url: "$${__value.raw}"

  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
    uid: tempo
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki
        spanStartTimeShift: "-5m"
        spanEndTimeShift: "5m"
        filterByTraceID: false
        filterBySpanID: false
      serviceMap:
        datasourceUid: prometheus
```

### 0.4 Start the stack

```bash
docker compose up -d
docker compose ps
```

Open Grafana at **http://localhost:3000**. You should see Prometheus, Loki, and Tempo already listed under **Connections → Data sources**.

**Checkpoint:** all containers show `Up` in `docker compose ps` before moving on.

---

## Module 1 — Log Aggregation with Loki

### Lab 1.1 — Loki architecture, by observation

**Goal:** see the write path and read path with your own eyes instead of just reading about them.

1. Generate some log volume:
   ```bash
   for i in $(seq 1 50); do
     docker run --rm busybox echo "hello from container run $i at $(date -u +%FT%TZ)"
   done
   ```
2. Watch Alloy pick these up:
   ```bash
   docker compose logs -f alloy | grep -i "loki"
   ```
3. Confirm Loki received them via its HTTP API directly (bypassing Grafana):
   ```bash
   curl -s "http://localhost:3100/loki/api/v1/labels" | jq
   curl -s "http://localhost:3100/loki/api/v1/label/container/values" | jq
   ```

**Questions to answer (write down your answers):**
- Which component did your `curl` request hit — distributor or querier? How do you know from the URL?
- Why does `/label/container/values` return quickly even with thousands of log lines — what is Loki indexing to answer that?

<details><summary>Answer key</summary>

Both endpoints are served by the **querier** (label queries are reads). The distributor is only in the write path — Alloy's push to `/loki/api/v1/push` hits it, not your `curl`. The label query is fast because Loki indexes only the **label set** (metadata), not log content — `container` is a label, so listing its values is an index lookup, not a log scan.
</details>

### Lab 1.2 — LogQL: label filter → line filter → parser

Use **Explore** (left nav → compass icon) with the Loki data source selected for all of these.

1. **Label filter only** — select every stream from the busybox runs:
   ```logql
   {container=~"busybox.*"}
   ```
2. **Add a line filter** — narrow to lines mentioning a specific run number:
   ```logql
   {container=~"busybox.*"} |= "run 12"
   ```
3. **Add a parser** — extract structured fields from JSON logs (switch targets to a JSON-logging container, or use the demo app if running):
   ```logql
   {container="demo-app"} | json | line_format "{{.level}} {{.msg}}"
   ```
4. **Metric query** — turn logs into a rate graph:
   ```logql
   sum(rate({container=~"busybox.*"}[1m]))
   ```
   Switch the Explore panel to **Graph** view to see it plotted.

**Exercise:** write a LogQL query that returns only lines from `busybox` containers containing an odd-numbered run (hint: you can't do modulo in LogQL — combine a line filter with a regex on the number instead). Try:
```logql
{container=~"busybox.*"} |~ "run (1|3|5|7|9)[0-9]?"
```

### Lab 1.3 — Ship a real application's logs

1. Run a container that logs JSON to stdout:
   ```bash
   docker run -d --name log-generator \
     alpine sh -c 'while true; do
       echo "{\"level\":\"info\",\"msg\":\"request handled\",\"duration_ms\":$((RANDOM % 500)),\"trace_id\":\"'$(openssl rand -hex 16)'\"}"
       sleep 2
     done'
   ```
2. Confirm Alloy discovered it without any config change:
   ```bash
   curl -s "http://localhost:3100/loki/api/v1/label/container/values" | jq
   ```
   You should see `log-generator` appear within ~15 seconds — this is Alloy's Docker service discovery working, the same mechanism Promtail's `docker_sd_configs` provides.
3. In Explore, query it and parse duration as a number:
   ```logql
   {container="log-generator"} | json | duration_ms > 300
   ```

**Checkpoint:** you should get back only the "slow" synthetic requests.

### Lab 1.4 — Build a log panel in a dashboard

1. **Dashboards → New → New Dashboard → Add visualization → Loki**.
2. Panel 1 — **Logs panel**: query `{container="log-generator"}`, panel type "Logs".
3. Panel 2 — **Time series panel**: query `sum(rate({container="log-generator"} | json | duration_ms > 300 [1m]))`, panel type "Time series" — this is your "slow request rate" panel.
4. Add a **dashboard variable** named `container` of type "Query", sourced from the Loki data source, query `label_values(container)`. Use `$container` in place of the hardcoded value in both panels.
5. Save the dashboard as **"Log Overview"**.

**Stretch goal:** add a **Table panel** using `{container="log-generator"} | json` and enable "Instant" mode with the `duration_ms` and `trace_id` fields as columns.

---

## Module 2 — Distributed Tracing with Tempo

### Lab 2.1 — Send your first trace via OTLP

Rather than instrumenting a full app, send one trace by hand so you can see exactly what OTLP ingestion looks like.

`send_trace.py` (requires `pip install opentelemetry-sdk opentelemetry-exporter-otlp`):
```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
import time

resource = Resource.create({"service.name": "checkout-lab"})
provider = TracerProvider(resource=resource)
exporter = OTLPSpanExporter(endpoint="http://localhost:4318/v1/traces")
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("lab.manual")

with tracer.start_as_current_span("gateway") as gw:
    time.sleep(0.05)
    with tracer.start_as_current_span("checkout") as co:
        co.set_attribute("http.status_code", 200)
        time.sleep(0.12)
        with tracer.start_as_current_span("payments") as pay:
            pay.set_attribute("http.status_code", 500)
            time.sleep(0.08)

provider.shutdown()
print("Trace sent.")
```

```bash
pip install opentelemetry-sdk opentelemetry-exporter-otlp
python3 send_trace.py
```

### Lab 2.2 — Find it in Tempo

1. In Grafana, **Explore → Tempo**, switch query type to **Search**.
2. Filter by `service.name = checkout-lab` and run the search.
3. Open the trace. You should see three nested spans: `gateway → checkout → payments`, with `payments` showing `http.status_code = 500`.

**Questions to answer:**
- Where in the timeline does `payments` sit relative to `checkout`'s total duration — fully nested, or overlapping?
- What identifies this whole set of spans as one trace, even though they were created as three separate `start_as_current_span` calls?

<details><summary>Answer key</summary>

`payments` is fully nested inside `checkout`'s span duration, because it was started as a child span within `checkout`'s context. All three spans share one **trace ID**, generated automatically when the outermost span (`gateway`) was started — every child span inherits it, which is what lets Tempo group them into a single trace on lookup.
</details>

### Lab 2.3 — TraceQL

Still in Explore → Tempo, switch to **TraceQL** query type.

1. Match a span attribute directly:
   ```traceql
   { span.http.status_code = 500 }
   ```
2. Combine conditions across the trace:
   ```traceql
   { span.http.status_code = 500 } && { name = "checkout-lab" }
   ```
   *(Note: adjust to match your resource attributes — try `{ resource.service.name = "checkout-lab" }` if the above returns nothing.)*
3. Structural query — find traces where `gateway` is a direct parent of `payments`:
   ```traceql
   { name = "gateway" } > { name = "payments" }
   ```
4. Aggregate filter — traces with more than 2 spans:
   ```traceql
   { } | count() > 2
   ```

**Exercise:** run `send_trace.py` five more times with the sleep values edited to vary, then write a TraceQL query that finds only traces where the `payments` span took longer than 100ms:
```traceql
{ name = "payments" && duration > 100ms }
```

### Lab 2.4 — Service graph

1. Run `send_trace.py` **10–15 times** in a loop so the metrics-generator has enough data:
   ```bash
   for i in $(seq 1 15); do python3 send_trace.py; sleep 1; done
   ```
2. In Grafana, go to **Explore → Tempo → Service Graph** tab (or add a **Service Graph** panel to a new dashboard).
3. You should see three nodes — `gateway`, `checkout`, `payments` — connected by edges showing request rate and error rate.
4. Click the edge between `checkout` and `payments`. It should filter straight into the traces containing that specific call.

**Checkpoint:** the `checkout → payments` edge should show a nonzero error rate, since every synthetic trace has `payments` returning 500.

---

## Module 3 — Unified Correlation in Grafana

### Lab 3.1 — Logs → Trace, using derived fields

1. Send a log line that embeds a real trace ID from Lab 2.1–2.3. First, capture a trace ID:
   ```bash
   python3 send_trace.py 2>&1
   ```
   Grab a trace ID from Tempo's Explore search results (click a trace, copy its ID from the URL or trace header).
2. Emit a log line containing it:
   ```bash
   docker run --rm alpine sh -c 'echo "{\"level\":\"error\",\"msg\":\"payment failed\",\"trace_id\":\"PASTE_TRACE_ID_HERE\"}"'
   ```
3. In Explore → Loki, query for it:
   ```logql
   {container=~"alpine.*"} |= "payment failed"
   ```
4. Expand the log line. Because of the `derivedFields` config in `datasources.yaml` (Section 0.3), you should see a **"TraceID"** button/link next to the parsed `trace_id` field — click it.

**Checkpoint:** clicking the link should switch Explore to Tempo and load that exact trace.

### Lab 3.2 — Trace → Logs

1. In Explore → Tempo, open any trace from Module 2 (e.g. from the service graph edge in Lab 2.4).
2. Click into the `payments` span, then use the **"Logs for this span"** action.
3. Grafana runs a Loki query scoped to the span's service and a time window around its start/end (configured via `tracesToLogsV2` in Section 0.3).

Since `send_trace.py` doesn't itself emit logs, this query will likely come back empty — that's expected and instructive. **Fix it:** modify `send_trace.py` to `print()` a JSON line containing `trace.get_current_span().get_span_context().trace_id` right before each span closes, pipe that output into a file Alloy can tail, and retry the trace-to-logs jump.

<details><summary>Minimal fix</summary>

```python
import json
span_ctx = pay.get_span_context()
print(json.dumps({
    "level": "error",
    "msg": "payment failed",
    "trace_id": format(span_ctx.trace_id, "032x")
}))
```
Add this inside the `with tracer.start_as_current_span("payments")` block, run the script with output redirected to a file under a path Alloy/Promtail is configured to tail, and re-run the trace-to-logs action.
</details>

### Lab 3.3 — Exemplars

1. Confirm Tempo's metrics-generator is writing exemplars to Prometheus:
   ```bash
   curl -s "http://localhost:9090/api/v1/query?query=traces_spanmetrics_latency_bucket" | jq '.data.result | length'
   ```
2. In Explore → Prometheus, run:
   ```promql
   histogram_quantile(0.95, sum(rate(traces_spanmetrics_latency_bucket[5m])) by (le, service))
   ```
3. Switch the panel to **Time series** and enable **Exemplars** in panel options if not already on. Small diamond markers should appear on the graph.
4. Click a diamond. It should jump straight into the exact trace behind that latency sample.

**Checkpoint:** if no exemplars appear, generate more traffic (`for i in $(seq 1 20); do python3 send_trace.py; done`) — exemplars need enough samples to land in the scraped window.

### Lab 3.4 — Build the single-pane dashboard

Create one new dashboard, **"Checkout Service — Unified View"**, with:

| Panel | Data source | Query |
|---|---|---|
| Request latency (p95) | Prometheus | `histogram_quantile(0.95, sum(rate(traces_spanmetrics_latency_bucket{service="$service"}[5m])) by (le))` — enable exemplars |
| Error rate | Prometheus | `sum(rate(traces_spanmetrics_calls_total{service="$service", status_code="STATUS_CODE_ERROR"}[5m]))` |
| Service graph | Tempo | Service Graph panel type, filtered to `$service` |
| Recent traces | Tempo | TraceQL: `{ resource.service.name = "$service" }` |
| Correlated logs | Loki | `{container=~"$service.*"}` |

1. Add a dashboard variable `service` (type: Query, data source: Prometheus, query: `label_values(traces_spanmetrics_calls_total, service)`).
2. Wire every panel to `$service` instead of a hardcoded value.
3. Set the dashboard time range to **Last 30 minutes** and confirm changing `$service` updates all five panels together.
4. From the latency panel, click an exemplar → confirm it opens the right trace → from that trace, jump to logs → confirm the log panel's data lines up with the same time window.

**Final checkpoint — the full loop:**
> Metric spike (exemplar) → trace → span → correlated logs, all without leaving Grafana or manually copying a trace ID.

---

## Wrap-up

You should now have:
- A running Loki + Alloy + Tempo + Prometheus + Grafana stack you can keep using
- A "Log Overview" dashboard (Module 1)
- Working knowledge of TraceQL and the service graph (Module 2)
- A "Checkout Service — Unified View" dashboard wiring all three signals together (Module 3)

**Cleanup:**
```bash
docker compose down -v
docker rm -f log-generator
```

**Next steps to explore on your own:**
- Swap the filesystem storage backends for S3/GCS/MinIO and re-run Module 1–2 against object storage
- Add `logql` recording rules and Loki alerting on the "slow request rate" query from Lab 1.4
- Instrument a real multi-service app (e.g. the [Grafana microservices demo](https://github.com/grafana/microservices-demo)) instead of the synthetic `send_trace.py` script, and repeat Module 3 end-to-end
