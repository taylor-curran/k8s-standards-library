# Knowledge Item: Rule 01 – Enforce Resource Requests & Limits

## Overview
Kubernetes resource requests and limits are essential for preventing "noisy-neighbor" outages and surprise node evictions in multi-tenant clusters. This rule ensures predictable resource allocation and isolation, which is critical for financial-services platforms that must guarantee stability and performance.

## Purpose
- Prevent runaway services from starving other workloads in shared clusters
- Ensure predictable resource allocation and scheduling behavior
- Enable proper capacity planning and cost allocation
- Support Horizontal Pod Autoscaler (HPA) functionality with appropriate headroom

## Requirements

### Mandatory Resource Fields
All containers must specify the following resource fields:

| Container field | Required? | Typical baseline | Notes |
| --- | --- | --- | --- |
| `resources.requests.cpu` | ✅ | ≥ 50m (0.05 vCPU) | Minimum guaranteed CPU |
| `resources.requests.memory` | ✅ | ≥ 128Mi | Minimum guaranteed memory |
| `resources.limits.cpu` | ✅ | ≤ 4 vCPU | Maximum CPU usage |
| `resources.limits.memory` | ✅ | ≤ 2Gi | Maximum memory usage |

### Resource Ratio Guidelines
- **Rule of thumb**: requests ≈ 60% of limits
- This ratio provides HPA with sufficient headroom for scaling decisions
- Ensures efficient cluster resource utilization

## Examples

### Non-compliant Example
```yaml
# charts/petclinic/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: petclinic
spec:
  template:
    spec:
      containers:
      - name: app
        image: petclinic:1.0.0
        resources: {}   # ❌ No resource specifications
```

**Issues with this approach:**
- No resource guarantees for the container
- Risk of resource starvation for other workloads
- Unpredictable scheduling behavior
- HPA cannot function properly

### Compliant Example
```yaml
# charts/petclinic/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: petclinic
spec:
  template:
    spec:
      containers:
      - name: app
        image: petclinic:1.0.0
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

**Benefits of this approach:**
- Guaranteed minimum resources (requests)
- Bounded maximum resource usage (limits)
- 60% request-to-limit ratio enables HPA scaling
- Predictable scheduling and resource allocation

## Enforcement Mechanisms

### Automated Linting
Use `kube-linter` with the following checks configured as **error** severity:
- `no-pod-limits`: Ensures all containers have resource limits
- `no-pod-requests`: Ensures all containers have resource requests

### Example kube-linter Configuration
```yaml
checks:
  no-pod-limits:
    severity: error
  no-pod-requests:
    severity: error
```

### Policy Enforcement
Consider implementing admission controllers or OPA Gatekeeper policies to enforce these requirements at the cluster level.

## Implementation Tips

### Setting Appropriate Values
1. **Start with monitoring**: Deploy with initial estimates and monitor actual usage
2. **Use metrics**: Base requests on P95 usage patterns over time
3. **Set limits defensively**: Allow for traffic spikes and memory leaks
4. **Test scaling**: Verify HPA behavior with your request/limit ratios

### Common Pitfalls
- Setting requests too high (wastes cluster resources)
- Setting limits too low (causes unnecessary throttling/OOMKills)
- Ignoring the request-to-limit ratio (breaks HPA scaling)
- Not monitoring actual resource usage patterns

### Integration with CI/CD
- Include resource validation in your deployment pipeline
- Use tools like `kube-score` or `polaris` for additional validation
- Implement resource budgets at the namespace level

## Related Standards
- Resource quotas and limit ranges at namespace level
- Horizontal Pod Autoscaler configuration
- Vertical Pod Autoscaler considerations for dynamic sizing
