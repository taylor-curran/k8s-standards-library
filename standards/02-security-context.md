# Rule 02 – Pod Security Baseline

*Run as non‑root, drop dangerous capabilities, lock the file‑system.*

## Must‑have settings

| Field | Value / Pattern |
| --- | --- |
| `securityContext.runAsNonRoot` | `true` |
| `securityContext.seccompProfile.type` | `RuntimeDefault` |
| `securityContext.readOnlyRootFilesystem` | `true` |
| `securityContext.capabilities.drop` | `["ALL"]` (allow extra caps via `--set extraCaps=`) |

### Non‑compliant excerpt

```yaml
securityContext:
  runAsUser: 0        # ‼️ root
```

### Compliant excerpt

```yaml
securityContext:
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
```

> **Tip for demo** – spin up the bad manifest and run `kubectl describe pod`; Windsurf fix removes root user and rerun succeeds.

---