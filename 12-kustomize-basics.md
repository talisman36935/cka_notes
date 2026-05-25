---
tags:
  - kubernetes
  - cka
  - kustomize
---

# ✅ Kustomize Basics

Kustomize customises Kubernetes YAML without template syntax — it patches and overlays existing manifests. Built into `kubectl` via the `-k` flag (no separate install needed).

**Helm vs Kustomize:**
- Helm → templating engine; charts ship with Go template variables
- Kustomize → patch engine; works on plain YAML, no templating language

---

## kustomization.yaml

Every Kustomize directory needs a `kustomization.yaml` at its root:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Resources to include
resources:
- deployment.yaml
- service.yaml
- ../base/            # reference a base directory

# Common labels added to all resources
commonLabels:
  env: production

# Image overrides
images:
- name: nginx
  newTag: "1.25"

# Patches
patches:
- path: replica-patch.yaml
```

---

## Applying Kustomize

```bash
# Apply directly (kubectl handles kustomize internally)
kubectl apply -k ./overlays/production/

# Preview what would be applied without applying
kubectl diff -k ./overlays/production/

# Build YAML to stdout (inspect before applying)
kustomize build ./overlays/production/
kustomize build ./overlays/production/ | kubectl apply -f -
```

---

## Patches: JSON6902 vs Strategic Merge

### Strategic Merge Patch (partial YAML — most readable)

Write a partial YAML that mirrors the target resource structure. Kubernetes merges it using `name` as the key for lists.

```yaml
# replica-patch.yaml — targets the Deployment by name
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: api          # 'name' is the merge key for containers list
        env:
        - name: LOG_LEVEL
          value: debug
```

Reference in `kustomization.yaml`:
```yaml
patches:
- path: replica-patch.yaml
```

### JSON6902 Patch (surgical operations)

Uses RFC 6902 JSON Patch operations. Targets by resource kind/name and uses a path expression.

```yaml
patches:
- target:
    kind: Deployment
    name: api-deployment
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
    - op: replace
      path: /spec/template/spec/containers/0/image
      value: nginx:1.25
    - op: add
      path: /spec/template/spec/containers/0/env/-    # '-' appends to array
      value:
        name: LOG_LEVEL
        value: debug
    - op: remove
      path: /spec/template/spec/containers/0/resources/limits
```

| | Strategic Merge | JSON6902 |
|---|---|---|
| Syntax | Partial YAML | JSON Patch operations |
| Targets array items by | Name (merge key) | Index `[0]` or append `-` |
| Best for | Adding/updating named resources | Precise single-field replacements |

---

## Bases and overlays

A common pattern: shared base + environment-specific overlays.

```
k8s/
├── base/
│   ├── kustomization.yaml   (lists deployment.yaml, service.yaml)
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml   (references ../../base, patches replicas=1)
    └── prod/
        └── kustomization.yaml   (references ../../base, patches replicas=5, image tag)
```

---

## Components

A `kind: Component` is a reusable bundle of resources and patches that can be included in multiple overlays.

> [!warning] A Component cannot be applied standalone — it must be included in an overlay via `components:`. Don't add `../../base/` inside a Component that's already used by an overlay — it causes a **duplicate resource error**.

```yaml
# In an overlay's kustomization.yaml
components:
- ../../components/monitoring
```

---

## Exam essentials

```bash
# Most common exam command
kubectl apply -k .

# Check what will be applied
kustomize build . | less

# Validate without applying
kubectl diff -k .
```

> [!tip] The CKA tests that you can apply a kustomization directory, understand patch types, and know the base/overlay pattern. You won't need to write a complex overlay from scratch — but reading and modifying one is fair game.
