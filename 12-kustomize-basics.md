---
tags:
  - kubernetes
  - cka
  - kustomize
---

# ✅ Kustomize Basics

Kustomize customizes YAML without template syntax.

Key concepts:
- base
- overlays
- patches
- transformers
- components

Useful command:

```bash
kubectl apply -k .
```

Comparison:
- Helm → templating engine
- Kustomize → patch/overlay engine