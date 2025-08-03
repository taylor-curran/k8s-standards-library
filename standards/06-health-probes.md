# Rule 06 – Liveness & Readiness Probes

*Catch crashed JVMs before users do.*

| Probe | Recommended Endpoint | Initial Delay | Failure Threshold |
| --- | --- | --- | --- |
| Liveness | `/actuator/health/liveness` | 30 s | 3 |
| Readiness | `/actuator/health/readiness` | 10 s | 1 |

### Common mistake

```yaml
livenessProbe: null   # ‼️ omitted
```

### Correct

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  failureThreshold: 1
```

> **Demo idea:** Run `kubectl exec -- pkill -9 java`; liveness restarts the container within seconds.

---