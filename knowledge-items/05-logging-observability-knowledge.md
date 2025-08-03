# Knowledge Item: Rule 05 – Logging & Observability Hooks

## Overview
Comprehensive logging and observability are essential for monitoring, debugging, and maintaining Kubernetes applications. This rule establishes requirements for structured logging output and Prometheus metrics exposition, enabling centralized log aggregation and automated monitoring across all services.

## Purpose
- Enable centralized log aggregation and analysis
- Provide standardized metrics for monitoring and alerting
- Support debugging and troubleshooting workflows
- Ensure consistent observability across all applications
- Enable automated performance monitoring and capacity planning

## Requirements

### Logging Standards

#### 1. Structured JSON Output
- All application logs **must** be output as structured JSON to `stdout`
- No file-system based logging (logs written to files)
- Consistent JSON schema across all applications
- Include standard fields: timestamp, level, message, service metadata

#### 2. Log Aggregation via Sidecar
- Deploy **fluent-bit** sidecar container alongside application containers
- Use Helm library chart `common/fluent-bit` for consistent configuration
- Sidecar ships logs to central OpenShift Loki stack
- No direct log shipping from application containers

### Metrics Standards

#### 1. Prometheus Metrics Endpoint
- HTTP port **8080** must serve `/metrics` endpoint
- Endpoint must return Prometheus-compatible format
- Include application-specific and standard infrastructure metrics
- Ensure metrics are properly labeled for aggregation

#### 2. Required Annotations
All pods must include the following annotations for automatic metrics discovery:

```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "8080"
```

## Examples

### Non-compliant Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-logging-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        # ❌ No logging configuration
        # ❌ No metrics annotations
        # ❌ No fluent-bit sidecar
```

**Issues with this approach:**
- No structured logging output
- Logs not collected centrally
- No metrics exposition
- Missing observability annotations

### Compliant Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: observable-app
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        ports:
        - containerPort: 8080
          name: http-metrics
        env:
        - name: LOG_FORMAT
          value: "json"
        - name: LOG_LEVEL
          value: "info"
      - name: fluent-bit
        image: fluent/fluent-bit:2.1.0
        volumeMounts:
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
      - name: varlog
        emptyDir: {}
```

### Advanced Example with Helm Chart Integration
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 8080
          name: http-metrics
          protocol: TCP
        - containerPort: 8000
          name: http-app
          protocol: TCP
        env:
        - name: LOG_FORMAT
          value: "json"
        - name: LOG_LEVEL
          value: {{ .Values.logging.level | quote }}
        livenessProbe:
          httpGet:
            path: /health
            port: http-app
        readinessProbe:
          httpGet:
            path: /ready
            port: http-app
      {{- if .Values.logging.fluentBit.enabled }}
      - name: fluent-bit
        image: {{ .Values.logging.fluentBit.image }}
        resources:
          limits:
            memory: 128Mi
          requests:
            cpu: 100m
            memory: 128Mi
        volumeMounts:
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc
      {{- end }}
      volumes:
      {{- if .Values.logging.fluentBit.enabled }}
      - name: fluent-bit-config
        configMap:
          name: {{ include "myapp.fullname" . }}-fluent-bit
      {{- end }}
```

## Implementation Details

### 1. JSON Logging Format
```json
{
  "timestamp": "2024-08-03T23:30:00.000Z",
  "level": "info",
  "message": "User authentication successful",
  "service": "petclinic-auth",
  "version": "1.4.2",
  "environment": "prod",
  "trace_id": "abc123def456",
  "user_id": "user_12345",
  "request_id": "req_789xyz",
  "duration_ms": 45
}
```

### 2. Fluent-Bit Configuration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token

    [OUTPUT]
        Name  loki
        Match *
        Host  loki-gateway.openshift-logging.svc.cluster.local
        Port  3100
        Labels job=fluent-bit
```

### 3. Prometheus Metrics Examples
```go
// Go application metrics example
package main

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
)

var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
    
    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Help: "HTTP request duration in seconds",
        },
        []string{"method", "endpoint"},
    )
)

func init() {
    prometheus.MustRegister(httpRequestsTotal)
    prometheus.MustRegister(httpRequestDuration)
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

## Enforcement Mechanisms

### 1. OPA Gatekeeper Policy
```rego
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Pod"
  not input.request.object.metadata.annotations["prometheus.io/scrape"]
  msg := "Pod must have prometheus.io/scrape annotation"
}

deny[msg] {
  input.request.kind.kind == "Pod"
  not input.request.object.metadata.annotations["prometheus.io/port"]
  msg := "Pod must have prometheus.io/port annotation"
}

deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  container.name != "fluent-bit"
  count(input.request.object.spec.containers) == 1
  msg := "Pod must include fluent-bit sidecar for log collection"
}
```

### 2. Helm Chart Validation
```yaml
# values.yaml
logging:
  level: info
  format: json
  fluentBit:
    enabled: true
    image: fluent/fluent-bit:2.1.0

metrics:
  enabled: true
  port: 8080
  path: /metrics

# Chart.yaml validation
dependencies:
- name: fluent-bit
  version: "0.20.0"
  repository: "https://fluent.github.io/helm-charts"
  condition: logging.fluentBit.enabled
```

### 3. ServiceMonitor for Prometheus
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: petclinic-metrics
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: petclinic
  endpoints:
  - port: http-metrics
    path: /metrics
    interval: 30s
```

## Operational Benefits

### 1. Centralized Logging Queries
```bash
# Query logs in Loki
curl -G -s "http://loki:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={app="petclinic"} |= "error"' \
  --data-urlencode 'start=2024-08-03T20:00:00Z' \
  --data-urlencode 'end=2024-08-03T23:00:00Z'
```

### 2. Prometheus Monitoring
```promql
# Application error rate
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

# Response time percentiles
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Application availability
up{job="petclinic"}
```

### 3. Grafana Dashboard Integration
```json
{
  "dashboard": {
    "title": "Application Observability",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ]
      },
      {
        "title": "Error Logs",
        "targets": [
          {
            "expr": "{app=\"petclinic\"} |= \"error\"",
            "refId": "A"
          }
        ]
      }
    ]
  }
}
```

## Troubleshooting

### Common Issues

#### Logs Not Appearing in Loki
```bash
# Check fluent-bit sidecar status
kubectl logs pod-name -c fluent-bit

# Verify fluent-bit configuration
kubectl get configmap fluent-bit-config -o yaml

# Test Loki connectivity
kubectl exec -it pod-name -c fluent-bit -- curl loki-gateway:3100/ready
```

#### Metrics Not Scraped by Prometheus
```bash
# Check metrics endpoint
kubectl port-forward pod-name 8080:8080
curl http://localhost:8080/metrics

# Verify ServiceMonitor
kubectl get servicemonitor -o yaml

# Check Prometheus targets
curl http://prometheus:9090/api/v1/targets
```

### Best Practices

#### Logging
1. **Structured format**: Always use JSON for machine readability
2. **Log levels**: Use appropriate levels (debug, info, warn, error)
3. **Correlation IDs**: Include trace/request IDs for distributed tracing
4. **Sensitive data**: Never log passwords, tokens, or PII

#### Metrics
1. **Naming conventions**: Follow Prometheus naming standards
2. **Label cardinality**: Avoid high-cardinality labels
3. **Business metrics**: Include domain-specific metrics
4. **Resource metrics**: Monitor CPU, memory, and network usage

## Demo Workflow

### Quick Win Demo Steps
1. **Deploy misconfigured chart** – Missing prometheus annotations
2. **Windsurf flags missing annotations** – Automated detection
3. **Accept fix** – Windsurf injects the required snippet
4. **Service auto-appears in Grafana** – Immediate observability

## Related Standards
- OpenTelemetry specification
- Prometheus metrics best practices
- Structured logging standards (ECS, Common Log Format)
- CNCF observability landscape recommendations
