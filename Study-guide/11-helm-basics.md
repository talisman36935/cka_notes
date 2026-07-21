# ✅ Helm Basics

Helm is Kubernetes package management. A **chart** is a package; a **release** is an installed instance of a chart; **values** are user-supplied config that override chart defaults.

> [!note] Helm 3 does **not** use Tiller. Everything runs client-side.

---

## Repo management

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo list
helm repo update           # refresh cached index from all repos
helm repo remove bitnami
```

---

## Searching

```bash
helm search hub wordpress          # searches Artifact Hub (public registry)
helm search repo nginx             # searches locally added repos
helm search repo nginx --versions  # show all available versions
```

---

## Installing and managing releases

```bash
# Install a chart as a named release
helm install my-release bitnami/nginx

# Install into a specific namespace (creates it if it doesn't exist)
helm install my-release bitnami/nginx --namespace monitoring --create-namespace

# Install a specific chart version
helm install my-release bitnami/nginx --version 15.0.0

# Override values at install time
helm install my-release bitnami/nginx --set replicaCount=3
helm install my-release bitnami/nginx -f custom-values.yaml

# Upgrade an existing release
helm upgrade my-release bitnami/nginx

# Upgrade to a specific chart version
helm upgrade my-release bitnami/nginx --version 15.1.0

# Install or upgrade in one command
helm upgrade --install my-release bitnami/nginx

# Roll back to a previous revision
helm rollback my-release 1

# Uninstall a release
helm uninstall my-release
helm uninstall my-release --namespace monitoring
```

---

## Inspecting releases and charts

```bash
helm list                          # releases in default namespace
helm list -A                       # all namespaces
helm list --namespace monitoring

helm status my-release             # show release status and notes
helm get values my-release         # show user-supplied values
helm get manifest my-release       # show rendered Kubernetes manifests

helm show chart bitnami/nginx      # chart metadata
helm show values bitnami/nginx     # all default values for a chart
helm show readme bitnami/nginx
```

---

## Exam essentials

```bash
# Find what's installed across the cluster
helm list -A

# Check what values a chart accepts before installing
helm show values bitnami/nginx | grep -A5 replicaCount

# Dry run to preview what would be applied
helm install my-release bitnami/nginx --dry-run

# Generate the YAML without installing (template rendering)
helm template my-release bitnami/nginx
```

> [!tip] The CKA tests that you can install/upgrade a release, list releases, inspect values, and roll back. You won't need to write a chart from scratch.
