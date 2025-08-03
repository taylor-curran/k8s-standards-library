# Knowledge Item: Rule 03 – Immutable, Trusted Images

## Overview
Container image provenance ensures that only trusted, immutable, and verified images are deployed in production environments. This rule prevents the use of mutable tags, restricts image sources to approved registries, and requires cryptographic signatures for production workloads.

## Purpose
- Prevent supply chain attacks through untrusted images
- Ensure reproducible deployments with immutable references
- Maintain audit trails for security compliance
- Reduce risk of malicious or compromised container images
- Enable automated security scanning and vulnerability management

## Requirements

### 1. Tag Pinning Requirements
Images **MUST NOT** use mutable tags such as `latest`, `stable`, or version ranges.

**Acceptable formats:**
- Semantic version tags: `registry.bank.internal/petclinic:1.4.2`
- Digest references: `registry.bank.internal/petclinic@sha256:abc123...`
- Combined tag and digest: `registry.bank.internal/petclinic:1.4.2@sha256:abc123...`

**Prohibited formats:**
- `latest` tag
- `stable`, `main`, `master` tags
- Version ranges or wildcards

### 2. Registry Allow-List
All container images must originate from approved registries:

**Approved registries:**
- `registry.bank.internal/*` (Internal enterprise registry)
- `quay.io/redhat-openshift-approved/*` (Red Hat certified images)

**Additional considerations:**
- Public registries (docker.io, gcr.io, etc.) are prohibited
- Registry URLs must be explicitly specified
- No implicit registry assumptions

### 3. Sigstore/Cosign Signature Verification
Production images **must** have valid Cosign signatures:
- Verification handled by OpenShift Image Policies
- Signatures must be from trusted signers
- Signature verification occurs at deployment time

## Examples

### Non-compliant Examples

#### Using Latest Tag
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: petclinic:latest   # ❌ Mutable, unsigned, no registry specified
```

#### Using Public Registry
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: docker.io/library/nginx:1.21   # ❌ Public registry not allowed
```

### Compliant Examples

#### Basic Compliant Image Reference
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: good-deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: registry.bank.internal/petclinic:1.4.2
```

#### Fully Compliant with Digest
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: best-deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: registry.bank.internal/petclinic:1.4.2@sha256:abc123def456...
```

#### Multi-container Pod Example
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-container-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: registry.bank.internal/petclinic:1.4.2@sha256:abc123...
      - name: sidecar
        image: quay.io/redhat-openshift-approved/fluent-bit:2.1.0@sha256:def456...
      initContainers:
      - name: init-db
        image: registry.bank.internal/db-migrator:2.0.1@sha256:789abc...
```

## Enforcement Mechanisms

### 1. OPA Gatekeeper Policy (Rego)
```rego
package kubernetes.admission

# Deny latest tag usage
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  endswith(container.image, ":latest")
  msg := "Use a pinned tag or digest; ':latest' is disallowed."
}

# Deny images without approved registry
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not starts_with(container.image, "registry.bank.internal/")
  not starts_with(container.image, "quay.io/redhat-openshift-approved/")
  msg := sprintf("Image '%s' is not from an approved registry", [container.image])
}

# Require digest for production
deny[msg] {
  input.request.kind.kind == "Pod"
  input.request.object.metadata.namespace == "production"
  container := input.request.object.spec.containers[_]
  not contains(container.image, "@sha256:")
  msg := "Production images must include digest reference"
}
```

### 2. OpenShift Image Policy Configuration
```yaml
apiVersion: config.openshift.io/v1
kind: Image
metadata:
  name: cluster
spec:
  registrySources:
    allowedRegistries:
    - registry.bank.internal
    - quay.io/redhat-openshift-approved
  additionalTrustedCA:
    name: additional-trust-bundle
```

### 3. Cosign Signature Verification
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cosign-policy
data:
  policy.json: |
    {
      "default": [
        {
          "type": "reject"
        }
      ],
      "transports": {
        "docker": {
          "registry.bank.internal": [
            {
              "type": "sigstoreSigned",
              "keyPath": "/etc/cosign/cosign.pub"
            }
          ]
        }
      }
    }
```

## Implementation Tips

### 1. CI/CD Pipeline Integration
```yaml
# Example GitLab CI stage
validate-images:
  stage: validate
  script:
    - |
      # Extract images from manifests
      images=$(grep -r "image:" k8s/ | grep -v "#" | awk '{print $2}')
      
      # Validate each image
      for image in $images; do
        # Check for latest tag
        if [[ $image == *":latest" ]]; then
          echo "ERROR: Latest tag found: $image"
          exit 1
        fi
        
        # Check registry allowlist
        if [[ ! $image == registry.bank.internal/* ]] && [[ ! $image == quay.io/redhat-openshift-approved/* ]]; then
          echo "ERROR: Unapproved registry: $image"
          exit 1
        fi
      done
```

### 2. Automated Digest Resolution
```bash
#!/bin/bash
# Script to resolve tags to digests
resolve_digest() {
  local image=$1
  local digest=$(skopeo inspect docker://$image | jq -r '.Digest')
  echo "${image}@${digest}"
}

# Usage
resolved_image=$(resolve_digest "registry.bank.internal/petclinic:1.4.2")
echo "Resolved: $resolved_image"
```

### 3. Image Scanning Integration
```yaml
# Example security scanning step
scan-images:
  stage: security
  script:
    - |
      for image in $(extract_images_from_manifests); do
        # Scan with Trivy or similar tool
        trivy image --exit-code 1 --severity HIGH,CRITICAL $image
        
        # Verify signature
        cosign verify --key cosign.pub $image
      done
```

## Troubleshooting

### Common Issues

#### Image Pull Errors
```bash
# Check registry connectivity
curl -I https://registry.bank.internal/v2/

# Verify credentials
kubectl get secret regcred -o yaml

# Test image pull manually
podman pull registry.bank.internal/petclinic:1.4.2
```

#### Signature Verification Failures
```bash
# Verify signature manually
cosign verify --key cosign.pub registry.bank.internal/petclinic:1.4.2

# Check policy configuration
oc get image.config.openshift.io/cluster -o yaml
```

### Best Practices

#### Image Lifecycle Management
1. **Tagging strategy**: Use semantic versioning for releases
2. **Digest tracking**: Maintain mapping of tags to digests
3. **Vulnerability management**: Regular scanning and updates
4. **Signature rotation**: Plan for key rotation and updates

#### Development Workflow
1. **Local development**: Use approved base images
2. **Testing**: Validate image compliance in CI
3. **Staging**: Test with production-like policies
4. **Production**: Full signature verification required

## Related Standards
- SLSA (Supply-chain Levels for Software Artifacts) framework
- NIST Cybersecurity Framework
- Container image security best practices
- Software bill of materials (SBOM) requirements
