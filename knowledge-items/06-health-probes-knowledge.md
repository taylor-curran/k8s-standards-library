# Knowledge Item: Rule 06 – Liveness & Readiness Probes

## Overview
Health probes are essential for Kubernetes to automatically detect and respond to application failures. This rule establishes requirements for liveness and readiness probes that enable automatic container restarts, traffic routing decisions, and overall application resilience in production environments.

## Purpose
- Enable automatic detection of crashed or unresponsive applications
- Prevent traffic routing to unhealthy application instances
- Support zero-downtime deployments and rolling updates
- Improve overall application availability and reliability
- Reduce mean time to recovery (MTTR) for application failures

## Requirements

### Probe Configuration Standards
All application containers must implement both liveness and readiness probes with the following specifications:

| Probe Type | Recommended Endpoint | Initial Delay | Failure Threshold | Purpose |
| --- | --- | --- | --- | --- |
| **Liveness** | `/actuator/health/liveness` | 30s | 3 | Detect crashed containers |
| **Readiness** | `/actuator/health/readiness` | 10s | 1 | Control traffic routing |

### Probe Timing Guidelines
- **Initial Delay**: Allow sufficient time for application startup
- **Period**: Default 10s interval between checks
- **Timeout**: Default 1s timeout per probe
- **Success Threshold**: 1 consecutive success (readiness only)
- **Failure Threshold**: Varies by probe type

### Health Check Endpoints
Applications should implement dedicated health check endpoints that:
- Return HTTP 200 for healthy state
- Return HTTP 4xx/5xx for unhealthy state
- Include dependency checks (database, external services)
- Respond quickly (< 1 second)
- Are lightweight and non-intrusive

## Examples

### Non-compliant Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unhealthy-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        ports:
        - containerPort: 8000
        livenessProbe: null   # ❌ No liveness probe
        readinessProbe: null  # ❌ No readiness probe
```

**Issues with this approach:**
- Kubernetes cannot detect crashed containers
- Traffic may be routed to unhealthy instances
- No automatic recovery from application failures
- Manual intervention required for stuck processes

### Compliant Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: healthy-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        ports:
        - containerPort: 8000
          name: http-app
        - containerPort: 8080
          name: http-health
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 1
          failureThreshold: 1
```

### Advanced Example with Multiple Probe Types
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: robust-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        ports:
        - containerPort: 8000
          name: http-app
        - containerPort: 8080
          name: http-health
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: http-health
            httpHeaders:
            - name: Custom-Header
              value: liveness-check
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          successThreshold: 1
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: http-health
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 1
          successThreshold: 1
        startupProbe:
          httpGet:
            path: /actuator/health/startup
            port: http-health
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 30
          successThreshold: 1
```

## Implementation Details

### 1. Health Check Endpoint Implementation

#### Spring Boot Actuator Example
```java
@RestController
@RequestMapping("/actuator/health")
public class HealthController {
    
    @Autowired
    private DatabaseHealthIndicator databaseHealth;
    
    @GetMapping("/liveness")
    public ResponseEntity<Map<String, String>> liveness() {
        Map<String, String> status = new HashMap<>();
        status.put("status", "UP");
        status.put("timestamp", Instant.now().toString());
        return ResponseEntity.ok(status);
    }
    
    @GetMapping("/readiness")
    public ResponseEntity<Map<String, Object>> readiness() {
        Map<String, Object> health = new HashMap<>();
        
        try {
            boolean dbHealthy = databaseHealth.isHealthy();
            boolean externalServicesHealthy = checkExternalServices();
            
            if (dbHealthy && externalServicesHealthy) {
                health.put("status", "UP");
                health.put("database", "UP");
                health.put("external_services", "UP");
                return ResponseEntity.ok(health);
            } else {
                health.put("status", "DOWN");
                health.put("database", dbHealthy ? "UP" : "DOWN");
                health.put("external_services", externalServicesHealthy ? "UP" : "DOWN");
                return ResponseEntity.status(503).body(health);
            }
        } catch (Exception e) {
            health.put("status", "DOWN");
            health.put("error", e.getMessage());
            return ResponseEntity.status(503).body(health);
        }
    }
}
```

#### Node.js Express Example
```javascript
const express = require('express');
const app = express();

app.get('/actuator/health/liveness', (req, res) => {
  res.status(200).json({
    status: 'UP',
    timestamp: new Date().toISOString()
  });
});

app.get('/actuator/health/readiness', async (req, res) => {
  try {
    const dbHealthy = await checkDatabase();
    const externalHealthy = await checkExternalServices();
    
    if (dbHealthy && externalHealthy) {
      res.status(200).json({
        status: 'UP',
        database: 'UP',
        external_services: 'UP'
      });
    } else {
      res.status(503).json({
        status: 'DOWN',
        database: dbHealthy ? 'UP' : 'DOWN',
        external_services: externalHealthy ? 'UP' : 'DOWN'
      });
    }
  } catch (error) {
    res.status(503).json({
      status: 'DOWN',
      error: error.message
    });
  }
});
```

## Enforcement Mechanisms

### 1. OPA Gatekeeper Policy
```rego
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Deployment"
  container := input.request.object.spec.template.spec.containers[_]
  not container.livenessProbe
  msg := sprintf("Container '%s' must have a liveness probe", [container.name])
}

deny[msg] {
  input.request.kind.kind == "Deployment"
  container := input.request.object.spec.template.spec.containers[_]
  not container.readinessProbe
  msg := sprintf("Container '%s' must have a readiness probe", [container.name])
}
```

### 2. Helm Chart Template
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 1
```

## Operational Benefits

### 1. Automatic Failure Recovery
```bash
kubectl exec deployment/petclinic -- pkill -9 java
kubectl get pods -w
```

### 2. Traffic Management
```bash
kubectl get pods -o wide
kubectl get endpoints
kubectl describe pod <pod-name>
```

## Troubleshooting

### Common Issues

#### Probe Failures
```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl port-forward pod/<pod-name> 8080:8080
curl http://localhost:8080/actuator/health/liveness
```

#### Timing Issues
```bash
kubectl get events --field-selector involvedObject.name=<pod-name>
kubectl patch deployment <deployment-name> -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","livenessProbe":{"initialDelaySeconds":60}}]}}}}'
```

### Best Practices

#### Probe Design
1. **Separate endpoints**: Use different endpoints for liveness and readiness
2. **Lightweight checks**: Keep probe logic simple and fast
3. **Dependency awareness**: Include critical dependencies in readiness checks
4. **Graceful degradation**: Handle partial failures appropriately

#### Timing Optimization
1. **Application-specific**: Adjust timing based on application characteristics
2. **Environment considerations**: Account for resource constraints
3. **Startup time**: Allow sufficient time for application initialization
4. **Failure tolerance**: Balance responsiveness with stability

## Demo Workflow

### Demo Idea Steps
1. **Deploy application without probes** – Show manual failure detection
2. **Add health probes** – Demonstrate automatic failure detection
3. **Simulate crash**: Run `kubectl exec -- pkill -9 java`
4. **Observe recovery**: Liveness probe restarts container within seconds
5. **Show traffic management**: Readiness probe controls service routing

> **Demo idea:** Run `kubectl exec -- pkill -9 java`; liveness restarts the container within seconds.

## Related Standards
- Kubernetes probe configuration best practices
- Application health check patterns
- Site reliability engineering (SRE) principles
- Microservices resilience patterns
