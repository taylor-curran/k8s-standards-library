# Rule 01 – Enforce Resource Requests & Limits

*Avoid “noisy‑neighbor” outages and surprise node evictions.*

## Why we care

Kubernetes will happily schedule a pod with no limits; in a multi‑tenant cluster that means one runaway service can starve everything else. Financial‑services platforms must guarantee isolation and predictability.

## What good looks like

| Container field | Required? | Typical baseline |
| --- | --- | --- |
| `resources.requests.cpu` | ✅ | ≥ 50 m (0.05 vCPU) |
| `resources.requests.memory` | ✅ | ≥ 128 Mi |
| `resources.limits.cpu` | ✅ | ≤ 4 vCPU |
| `resources.limits.memory` | ✅ | ≤ 2 Gi |

*Rule of thumb:* **requests ≈ 60 % of limits** to give the HPA head‑room.

### Non‑compliant example

```yaml
# charts/petclinic/templates/deployment.yaml
resources: {}   # ‼️ nothing set
```

### Compliant example

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### Quick lint idea

Use `kube‑linter` checks `no‑pod‑limits`, `no‑pod‑requests` and flag severity **error**.

---