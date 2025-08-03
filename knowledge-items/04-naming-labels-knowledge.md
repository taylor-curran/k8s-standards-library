# Knowledge Item: Rule 04 – Naming & Label Conventions

## Overview
Consistent naming and labeling conventions make Kubernetes workloads discoverable, enable automated cost allocation tracking, and facilitate operational management. This rule establishes mandatory labels and naming patterns that support business grouping, environment management, and tool provenance tracking.

## Purpose
- Enable workload discovery and inventory management
- Automate cost allocation and chargeback processes
- Support environment-based promotion gates and policies
- Facilitate troubleshooting and operational procedures
- Ensure consistent metadata across all deployments

## Requirements

### Mandatory Labels
All Kubernetes resources must include the following labels:

| Key | Example Value | Purpose | Notes |
| --- | --- | --- | --- |
| `app.kubernetes.io/name` | `petclinic` | Stable application identifier | Immutable across versions |
| `app.kubernetes.io/version` | `1.4.2` | Traceable release version | Semantic versioning preferred |
| `app.kubernetes.io/part-of` | `retail-banking` | Business domain grouping | Enables cost allocation |
| `environment` | `dev` / `test` / `prod` | Deployment environment | Controls promotion gates |
| `managed-by` | `helm` / `openshift` | Tool provenance | Tracks deployment method |

### Release Name Convention
Follow the structured naming pattern for Helm releases and resource names:

**Format**: `<team>-<app>-<env>`

**Example**: `pe-eng-petclinic-dev`

**Components**:
- `team`: Team or organization identifier (e.g., `pe-eng`, `data-sci`, `platform`)
- `app`: Application name (matches `app.kubernetes.io/name`)
- `env`: Environment identifier (`dev`, `test`, `staging`, `prod`)

### Optional Recommended Labels
Additional labels that provide operational value:

| Key | Example | Purpose |
| --- | --- | --- |
| `app.kubernetes.io/component` | `frontend`, `backend`, `database` | Component identification |
| `app.kubernetes.io/instance` | `petclinic-prod-east` | Deployment instance |
| `version` | `v1.4.2` | Alternative version format |
| `tier` | `web`, `app`, `data` | Application tier |
| `owner` | `platform-engineering` | Ownership tracking |
| `cost-center` | `retail-banking-ops` | Financial allocation |

## Examples

### Non-compliant Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp        # ❌ Unclear naming, conflicts likely
  labels:
    app: myapp       # ❌ Non-standard label keys
    version: latest  # ❌ Mutable version reference
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
```

**Issues with this approach:**
- Generic name increases collision risk
- Non-standard label keys prevent automation
- Missing environment and ownership information
- No business domain association

### Compliant Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pe-eng-petclinic-dev
  labels:
    app.kubernetes.io/name: petclinic
    app.kubernetes.io/version: "1.4.2"
    app.kubernetes.io/part-of: retail-banking
    environment: dev
    managed-by: helm
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: petclinic
      app.kubernetes.io/instance: pe-eng-petclinic-dev
  template:
    metadata:
      labels:
        app.kubernetes.io/name: petclinic
        app.kubernetes.io/version: "1.4.2"
        app.kubernetes.io/part-of: retail-banking
        app.kubernetes.io/instance: pe-eng-petclinic-dev
        environment: dev
        managed-by: helm
```

### Advanced Example with Full Metadata
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pe-eng-petclinic-prod
  labels:
    app.kubernetes.io/name: petclinic
    app.kubernetes.io/version: "1.4.2"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: retail-banking
    app.kubernetes.io/instance: pe-eng-petclinic-prod
    environment: prod
    managed-by: helm
    tier: web
    owner: platform-engineering
    cost-center: retail-banking-ops
  annotations:
    deployment.kubernetes.io/revision: "3"
    kubectl.kubernetes.io/last-applied-configuration: |
      {...}
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: petclinic
      app.kubernetes.io/instance: pe-eng-petclinic-prod
  template:
    metadata:
      labels:
        app.kubernetes.io/name: petclinic
        app.kubernetes.io/version: "1.4.2"
        app.kubernetes.io/component: frontend
        app.kubernetes.io/part-of: retail-banking
        app.kubernetes.io/instance: pe-eng-petclinic-prod
        environment: prod
        managed-by: helm
        tier: web
```

## Enforcement Mechanisms

### 1. OPA Gatekeeper Policies
```rego
package kubernetes.admission

required_labels := [
  "app.kubernetes.io/name",
  "app.kubernetes.io/version", 
  "app.kubernetes.io/part-of",
  "environment",
  "managed-by"
]

deny[msg] {
  input.request.kind.kind == "Deployment"
  required := required_labels[_]
  not input.request.object.metadata.labels[required]
  msg := sprintf("Missing required label: %s", [required])
}

deny[msg] {
  input.request.kind.kind == "Deployment"
  name := input.request.object.metadata.name
  not regex.match("^[a-z0-9-]+-[a-z0-9-]+-[a-z0-9-]+$", name)
  msg := sprintf("Name '%s' does not follow team-app-env pattern", [name])
}
```

### 2. Helm Chart Template Validation
```yaml
# templates/_helpers.tpl
{{- define "petclinic.labels" -}}
app.kubernetes.io/name: {{ include "petclinic.name" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/part-of: {{ .Values.global.partOf | required "global.partOf is required" }}
environment: {{ .Values.global.environment | required "global.environment is required" }}
managed-by: helm
{{- end }}

# templates/deployment.yaml
metadata:
  name: {{ include "petclinic.fullname" . }}
  labels:
    {{- include "petclinic.labels" . | nindent 4 }}
```

### 3. Kustomize Label Transformer
```yaml
# kustomization.yaml
commonLabels:
  app.kubernetes.io/part-of: retail-banking
  environment: prod
  managed-by: kustomize

labelTransformer:
- target:
    kind: Deployment
  labels:
    app.kubernetes.io/name: petclinic
    app.kubernetes.io/version: "1.4.2"
```

## Implementation Tips

### 1. Automated Label Injection
```bash
#!/bin/bash
# Script to add missing labels to existing resources

NAMESPACE="default"
PART_OF="retail-banking"
ENVIRONMENT="dev"

kubectl get deployments -n $NAMESPACE -o name | while read deployment; do
  kubectl label $deployment -n $NAMESPACE \
    app.kubernetes.io/part-of=$PART_OF \
    environment=$ENVIRONMENT \
    --overwrite
done
```

### 2. Label Validation in CI/CD
```yaml
# GitLab CI validation stage
validate-labels:
  stage: validate
  script:
    - |
      # Check all YAML files for required labels
      for file in k8s/*.yaml; do
        if ! grep -q "app.kubernetes.io/name" $file; then
          echo "ERROR: Missing app.kubernetes.io/name in $file"
          exit 1
        fi
        if ! grep -q "environment:" $file; then
          echo "ERROR: Missing environment label in $file"
          exit 1
        fi
      done
```

### 3. Cost Allocation Queries
```bash
# Query resources by business domain
kubectl get pods -l app.kubernetes.io/part-of=retail-banking --all-namespaces

# Query by environment
kubectl get deployments -l environment=prod --all-namespaces

# Cost allocation report
kubectl get pods -l app.kubernetes.io/part-of=retail-banking \
  -o custom-columns=NAME:.metadata.name,NAMESPACE:.metadata.namespace,CPU:.spec.containers[*].resources.requests.cpu,MEMORY:.spec.containers[*].resources.requests.memory
```

## Operational Benefits

### 1. Service Discovery
```bash
# Find all components of an application
kubectl get all -l app.kubernetes.io/name=petclinic

# Find all services in a business domain
kubectl get services -l app.kubernetes.io/part-of=retail-banking
```

### 2. Environment Management
```bash
# Promote configuration from dev to test
kubectl get configmap -l environment=dev -o yaml | \
  sed 's/environment: dev/environment: test/' | \
  kubectl apply -f -
```

### 3. Monitoring and Alerting
```promql
# Prometheus queries using labels
sum(rate(http_requests_total[5m])) by (app_kubernetes_io_name, environment)

# Alert on missing labels
absent(up{app_kubernetes_io_name!=""})
```

## Troubleshooting

### Common Issues

#### Label Selector Mismatches
```bash
# Check if deployment selector matches pod labels
kubectl get deployment petclinic -o yaml | grep -A 10 selector
kubectl get pods -l app.kubernetes.io/name=petclinic --show-labels
```

#### Missing Labels on Existing Resources
```bash
# Find resources missing required labels
kubectl get deployments --all-namespaces -o json | \
  jq '.items[] | select(.metadata.labels["app.kubernetes.io/name"] == null) | .metadata.name'
```

### Best Practices

#### Label Management
1. **Consistency**: Use the same labels across all resource types
2. **Automation**: Inject labels via CI/CD or admission controllers
3. **Documentation**: Maintain a label registry for your organization
4. **Validation**: Implement automated checks for required labels

#### Naming Conventions
1. **Uniqueness**: Ensure names are unique within their scope
2. **Readability**: Use descriptive, human-readable names
3. **Length limits**: Stay within Kubernetes name length constraints
4. **Character restrictions**: Use only allowed characters (lowercase, numbers, hyphens)

## Related Standards
- Kubernetes recommended labels specification
- Helm chart best practices
- Prometheus monitoring label conventions
- Cost allocation and chargeback frameworks
