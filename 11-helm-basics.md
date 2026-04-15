# ✅ Helm Basics

Helm is Kubernetes package management.

Key concepts:
- **chart** → package template
- **release** → installed instance of a chart
- **values** → user-supplied config

Useful commands:

```bash
helm install myapp chart
helm upgrade myapp chart
helm list
helm uninstall myapp
helm show values chart
```

> [!note] Helm 3
> Helm 3 does **not** use Tiller.