# Observability Stack for DevOps

Metrics, logs, and traces — one `docker compose up` away.

![Docker + cAdvisor Stack](assets/docker.png)
![NodeExporter Stack](assets/nodeexporter.png)

## What's in the box

```
Prometheus  ← scrapes ← Node Exporter, cAdvisor, OTEL Collector
Loki        ← pushes  ← Promtail (container logs)
OTEL        ← OTLP    → exports metrics to Prometheus
Grafana     → queries  → Prometheus + Loki (auto-provisioned)
Notes App   → demo workload
```

## Quick start

```bash
git clone https://github.com/yourusername/Observability-For-DevOps.git
cd Observability-For-DevOps
docker compose up -d
```

That's it. Open http://localhost:3000 (admin/admin).

## Endpoints

| Service | URL | What it does |
|---------|-----|--------------|
| Grafana | [:3000](http://localhost:3000) | Dashboards for metrics + logs |
| Prometheus | [:9090](http://localhost:9090) | Metrics store, check `/targets` |
| Loki | [:3100](http://localhost:3100) | Log aggregation backend |
| cAdvisor | [:8080](http://localhost:8080) | Container resource metrics |
| OTEL Collector | `:4317` gRPC / `:4318` HTTP | Send OTLP traces, metrics, logs |
| Notes App | [:8000](http://localhost:8000) | Sample Django app |

Node Exporter and Promtail run internally (no exposed ports needed).

## Sending traces to OTEL

The collector accepts OTLP on ports 4317 (gRPC) and 4318 (HTTP). Point your instrumented app at it:

```bash
# quick test
curl -X POST http://localhost:4318/v1/traces \
  -H 'Content-Type: application/json' \
  -d '{"resourceSpans":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"my-app"}}]},"scopeSpans":[{"spans":[{"traceId":"aaaabbbbccccddddaaaabbbbccccdddd","spanId":"aaaabbbbccccdddd","name":"test","kind":1,"startTimeUnixNano":"1000000000","endTimeUnixNano":"2000000000","status":{}}]}]}]}'
```

Metrics from OTLP get exported to Prometheus. Traces go to debug logs (swap in Jaeger/Tempo when ready).

## Project structure

```
.
├── docker-compose.yml
├── prometheus.yml                          # scrape config
├── otel-collector/otel-collector-config.yml
├── loki/loki-config.yml
├── promtail/promtail-config.yml
├── grafana/provisioning/
│   ├── datasources/datasources.yml         # Prometheus + Loki
│   └── dashboards/dashboards.yml           # drop JSON files in dashboards/json/
└── notes-app/                              # Django demo app
```

## Tear down

```bash
docker compose down          # stop everything
docker compose down -v       # stop + delete all data
```


# Personal Learning 

## 1. Problem / real-world requirement

Building a centralized observability stack for a containerized application. The goal was to monitor infrastructure and containers, aggregate application logs, and provide a foundation for distributed tracing.

## 2. Architecture

I used Docker Compose to run the stack. Prometheus collects metrics from Node Exporter, cAdvisor and the OTEL Collector. Promtail collects Docker logs and sends them to Loki, while Grafana provides a single visualization layer.

## 3. Problems I actually encountered — THIS is valuable in interviews

Docker volume mounting error: Prometheus/Loki expected a file → identified the file-vs-directory mismatch and corrected the bind mounts.
Loki failed to start: Loki 3 rejected my schema v12 configuration because of structured metadata → investigated the startup error and added allow_structured_metadata: false.
Promtail failed: at least one client config must be provided → traced the issue through the mounted configuration path and corrected the config filename/mount/command.
Prometheus target DOWN: configured cadvisor:9100 for Node Exporter → understood Docker service DNS and corrected it to node-exporter:9100.
Loki returned 404: initially thought Loki was broken → learned that :3100 is not a UI endpoint and validated it using /ready and API endpoints.


I built the stack incrementally, validated each component, investigated failures using container logs and service endpoints, and corrected configuration and networking issues.

## What real-world problem does your project solve?
In a real production environment, having metrics, logs and traces distributed across different services makes troubleshooting difficult. This project demonstrates how I can centralize those signals and correlate infrastructure health, container resource usage and application behavior through an observability stack.
