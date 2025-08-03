# Rule 03 – Immutable, Trusted Images

*No `:latest`, only signed images from the internal registry.*

1.  **Tag pinning** – images **MUST NOT** use the `latest` tag.  
    *Acceptable*: `registry.bank.internal/petclinic:1.4.2` or digest `@sha256:…`.
    
2.  **Registry allow‑list** – all images originate from:
    
    -   `registry.bank.internal/*`
        
    -   `quay.io/redhat‑openshift‑approved/*`
        
3.  **Sigstore / Cosign signature** – production images **must** have a valid Cosign signature; verification handled by OpenShift Image Policies.
    

### Bad

```yaml
image: petclinic:latest   # ‼️ mutable, unsigned
```

### Good

```yaml
image: registry.bank.internal/petclinic:1.4.2@sha256:abc123…
```

### Remediation snippet (Rego idea)

```rego
deny[msg] {
  some i
  input.spec.template.spec.containers[i].image == "latest"
  msg := "Use a pinned tag or digest; ':latest' is disallowed."
}
```

---
