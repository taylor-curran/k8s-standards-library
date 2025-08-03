# Rule 05 – Logging & Observability Hooks

*Every service must emit JSON logs and expose Prometheus metrics.*

## Logging

-   Output structured JSON to `stdout`; no file‑system logs.
    
-   Sidecar **fluent‑bit** (Helm library chart `common/fluent-bit`) ships to the central OpenShift Loki stack.
    

## Metrics

-   HTTP port **8080** → `/metrics` must return Prometheus format.
    
-   Add the annotations:
    

```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "8080"
```

### Quick win demo

1.  Deploy mis‑configured chart – Windsurf flags missing annotations.
    
2.  Accept fix – Windsurf injects the snippet; service auto‑appears in Grafana.
    

---