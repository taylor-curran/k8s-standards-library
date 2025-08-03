# Rule 04 – Naming & Label Conventions

*Make workloads discoverable and automate cost/allocation tracking.*

## Mandatory labels

| Key | Example | Purpose |
| --- | --- | --- |
| `app.kubernetes.io/name` | `petclinic` | Stable app identifier |
| `app.kubernetes.io/version` | `1.4.2` | Traceable release |
| `app.kubernetes.io/part-of` | `retail-banking` | Business grouping |
| `environment` | `dev` / `test` / `prod` | Promotion gates |
| `managed-by` | `helm` / `openshift` | Tool provenance |

### Release‑name prefix

`<team>-<app>-<env>` → e.g. **`pe-eng-petclinic-dev`**

### Anti‑pattern

```yaml
metadata:
  name: myapp        # ‼️ unclear, conflicts likely
```

### Compliant

```yaml
metadata:
  name: pe-eng-petclinic-dev
  labels:
    app.kubernetes.io/name: petclinic
    app.kubernetes.io/version: "1.4.2"
    app.kubernetes.io/part-of: retail-banking
    environment: dev
    managed-by: helm
```

---
