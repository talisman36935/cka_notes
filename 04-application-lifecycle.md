# ✅ Application Lifecycle Management

## Rolling updates and rollbacks

Deployments support rolling updates by default. Two strategies:
- `RollingUpdate` (default) — gradually replaces old pods with new ones
- `Recreate` — terminates all old pods before creating new ones (causes downtime)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1    # max pods that can be unavailable during update
    maxSurge: 1          # max pods that can be created above desired count
```

```bash
# Trigger a rollout (image update — fastest exam method)
kubectl set image deployment/app nginx=nginx:1.9.1

# Watch progress
kubectl rollout status deployment/app

# View revision history
kubectl rollout history deployment/app

# Roll back to previous revision
kubectl rollout undo deployment/app

# Roll back to a specific revision
kubectl rollout undo deployment/app --to-revision=2

# Pause a rollout midway (to inspect before continuing)
kubectl rollout pause deployment/app

# Resume a paused rollout
kubectl rollout resume deployment/app
```

> [!warning] `kubectl set image` updates the live deployment but does NOT modify your YAML file on disk. If you need the file in sync, use `kubectl edit` or update the YAML and `kubectl apply`.

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

Without encryption at rest, Secrets are stored as base64 plaintext in etcd — readable by anyone with etcd access.

**Step 1 — Generate an encryption key:**
```bash
head -c 32 /dev/urandom | base64
# e.g. output: 8kKuTYj3IM6Z7FpXqLmNvRsUwBdCeGhA==
```

**Step 2 — Create `/etc/kubernetes/encryption-config.yaml`:**
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - secretbox:               # strong, recommended
      keys:
      - name: key1
        secret: <base64-key-from-step-1>
  - identity: {}             # fallback: reads existing unencrypted data
```

> [!note] Provider order matters. The **first** provider encrypts new writes. All providers attempt decryption on reads (in order). Keep `identity: {}` at the end while migrating so existing unencrypted secrets can still be read.

**Step 3 — Add the flag to the kube-apiserver static pod manifest:**
```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
    volumeMounts:
    - name: enc-config
      mountPath: /etc/kubernetes/encryption-config.yaml
      readOnly: true
  volumes:
  - name: enc-config
    hostPath:
      path: /etc/kubernetes/encryption-config.yaml
      type: File
```
Saving the file restarts the API server automatically (static pod behaviour).

**Step 4 — Force-encrypt all existing Secrets (they were written before encryption was enabled):**
```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

**Step 5 — Verify:**
```bash
# Read directly from etcd — should show binary ciphertext, not plaintext
kubectl exec -n kube-system etcd-controlplane -- \
  etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret
# output should begin with k8s:enc:secretbox:... not readable text
```

---

## Probes

Probes let the kubelet know whether a container is alive, ready for traffic, and finished starting up.

| Probe | Failure action | Purpose |
|---|---|---|
| `livenessProbe` | Kill + restart container | Detect deadlocked/broken containers |
| `readinessProbe` | Remove pod from Service endpoints | Prevent traffic to unready containers |
| `startupProbe` | Kill + restart (like liveness) | Block liveness/readiness until app finishes starting |

> [!tip] Use `startupProbe` for slow-starting apps. It buys time (`failureThreshold × periodSeconds`) before liveness kicks in, preventing premature restarts.

### Probe mechanisms

**httpGet** — HTTP GET to a path/port; success = 2xx/3xx response:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
```

**tcpSocket** — TCP connection attempt; success = connection established:
```yaml
readinessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 5
  periodSeconds: 10
```

**exec** — Runs a command inside the container; success = exit code 0:
```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Key timing fields

| Field | Meaning | Default |
|---|---|---|
| `initialDelaySeconds` | Wait before first probe | 0 |
| `periodSeconds` | How often to probe | 10 |
| `timeoutSeconds` | Probe timeout | 1 |
| `failureThreshold` | Consecutive failures before action | 3 |
| `successThreshold` | Consecutive successes to mark healthy (readiness) | 1 |

### All three probes together

```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    startupProbe:
      httpGet:
        path: /ready
        port: 8080
      failureThreshold: 30    # 30 × 10s = 5 min max startup window
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 2
```

> [!warning] A failing `readinessProbe` does **not** restart the container. It only removes the pod from Service endpoints. The container keeps running. Only `livenessProbe` failure triggers a restart.

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