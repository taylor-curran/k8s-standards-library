# Knowledge Item: Rule 02 – Pod Security Baseline

## Overview
Pod security contexts define privilege and access control settings for containers and pods. This rule establishes a security baseline that runs containers as non-root users, applies security profiles, locks down the filesystem, and drops dangerous capabilities to minimize attack surface.

## Purpose
- Prevent privilege escalation attacks
- Reduce container breakout risks
- Implement defense-in-depth security practices
- Comply with security frameworks and regulations
- Minimize the impact of potential container compromises

## Requirements

### Must-Have Security Context Settings
All containers must implement the following security context configurations:

| Field | Value / Pattern | Purpose |
| --- | --- | --- |
| `securityContext.runAsNonRoot` | `true` | Prevents running as root user (UID 0) |
| `securityContext.seccompProfile.type` | `RuntimeDefault` | Applies default seccomp filtering |
| `securityContext.readOnlyRootFilesystem` | `true` | Makes root filesystem immutable |
| `securityContext.capabilities.drop` | `["ALL"]` | Removes all Linux capabilities |

### Optional Enhancements
- `securityContext.runAsUser`: Specify explicit non-root UID
- `securityContext.runAsGroup`: Set specific group ID
- `securityContext.capabilities.add`: Add only required capabilities
- `securityContext.allowPrivilegeEscalation`: Set to `false` explicitly

## Examples

### Non-compliant Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: insecure-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        securityContext:
          runAsUser: 0        # ❌ Running as root
          # Missing other security settings
```

**Security risks:**
- Container runs with root privileges
- No seccomp filtering applied
- Filesystem is writable (potential for malware)
- All Linux capabilities available

### Compliant Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0.0
        securityContext:
          runAsNonRoot: true
          seccompProfile:
            type: RuntimeDefault
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
```

**Security benefits:**
- Container cannot run as root
- System calls are filtered by seccomp
- Root filesystem is immutable
- No Linux capabilities available

### Advanced Compliant Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app-advanced
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
      containers:
      - name: app
        image: myapp:1.0.0
        securityContext:
          allowPrivilegeEscalation: false
          seccompProfile:
            type: RuntimeDefault
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
            add: ["NET_BIND_SERVICE"]  # Only if needed for port 80/443
```

## Enforcement Mechanisms

### Pod Security Standards
Implement Kubernetes Pod Security Standards at the namespace level:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### OPA Gatekeeper Policy Example
```rego
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Pod"
  input.request.object.spec.securityContext.runAsNonRoot != true
  msg := "Containers must run as non-root user"
}

deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.securityContext.readOnlyRootFilesystem
  msg := "Containers must have read-only root filesystem"
}
```

### Admission Controller Validation
Use ValidatingAdmissionWebhooks or built-in Pod Security Standards to enforce these requirements automatically.

## Implementation Tips

### Handling Read-Only Filesystem
When using `readOnlyRootFilesystem: true`, applications may need writable directories:

```yaml
spec:
  containers:
  - name: app
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: tmp-volume
      mountPath: /tmp
    - name: var-log
      mountPath: /var/log
  volumes:
  - name: tmp-volume
    emptyDir: {}
  - name: var-log
    emptyDir: {}
```

### Container Image Considerations
- Build images with non-root users
- Use `USER` directive in Dockerfile
- Test images with security contexts applied
- Avoid setuid/setgid binaries

### Debugging Security Issues
When containers fail to start with security contexts:
1. Check container logs: `kubectl logs <pod-name>`
2. Describe pod for events: `kubectl describe pod <pod-name>`
3. Verify image supports non-root execution
4. Check file permissions and ownership

### Demo Workflow
> **Tip for demo**: Deploy the non-compliant manifest and run `kubectl describe pod` to see security violations. Then apply the compliant version and observe successful deployment.

## Related Standards
- Kubernetes Pod Security Standards
- CIS Kubernetes Benchmark
- NIST container security guidelines
- OpenShift Security Context Constraints (SCCs)

## Common Troubleshooting

### Application Won't Start
- Verify the application doesn't require root privileges
- Check if the application tries to write to read-only locations
- Ensure proper file ownership in the container image

### Permission Denied Errors
- Add necessary capabilities sparingly using `capabilities.add`
- Use volume mounts for writable directories
- Verify the application user has proper permissions
