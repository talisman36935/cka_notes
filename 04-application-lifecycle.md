# ✅ Application Lifecycle Management

## Rolling updates and rollbacks

Deployments support rolling updates by default.

Useful commands:

```bash
kubectl rollout status deployment app
kubectl rollout history deployment app
kubectl rollout undo deployment app
```

Useful strategy snippet:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

---

## Commands and arguments

Docker mapping:
- `ENTRYPOINT` → Kubernetes `command`
- `CMD` → Kubernetes `args`

Example:

```yaml
command: ["sleep"]
args: ["3600"]
```

> [!tip] Exam use
> This is a common source of broken Pods when the image starts the wrong process.

---

## Secrets

Secrets store sensitive values like passwords and tokens.

Important facts:
- values are base64 encoded, not encrypted by default
- can be consumed as env vars or volumes
- live in `etcd`

Useful commands:

```bash
kubectl create secret generic mysecret --from-literal=password=123
kubectl get secret mysecret
kubectl get secret mysecret -o yaml
```

Decode:

```bash
echo <base64-value> | base64 -d
```

---

## Encrypting Secrets at rest

Without encryption at rest, Secrets remain readable in `etcd` to anyone with sufficient access.

Concepts to know:
- `EncryptionConfiguration`
- API server configuration
- etcd as the backing store

---

## Multi-container Pods

Common patterns:
- **sidecar** → helper container beside main app
- **ambassador** → proxy/helper for external communication
- **adapter** → transform output for another system

Useful commands:

```bash
kubectl logs pod -c container-name
kubectl exec -it pod -c container-name -- sh
```

---

## Autoscaling

### HPA

Horizontal Pod Autoscaler changes the number of replicas based on metrics.

Useful commands:

```bash
kubectl autoscale deployment app --cpu-percent=50 --min=1 --max=5
kubectl get hpa
kubectl describe hpa <name>
```

### VPA

Vertical Pod Autoscaler changes CPU/memory requests.

Usually conceptual for CKA:
- not installed by default in many clusters
- less likely than HPA to be a core hands-on task

### In-place Pod resize

Newer Kubernetes versions may support resource updates without full Pod recreation, depending on features and cluster version.

Conceptual knowledge is enough.