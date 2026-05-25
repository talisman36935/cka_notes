# ✅ Scheduling

## Labels, selectors, and annotations

### Labels

Labels are key-value metadata used to organize and select resources.

Examples:
- `app=web`
- `env=prod`
- `tier=frontend`

Useful commands:

```bash
kubectl label pod nginx env=prod
kubectl get pods -l env=prod
kubectl get pods --show-labels
```

### Selectors

Selectors are used by:
- Services
- ReplicaSets
- Deployments
- your own `kubectl` queries

### Annotations

Annotations store extra metadata that is not used for selection.

Examples:
- build version
- owner info
- tooling metadata

> [!tip] Exam shortcut
> If you are troubleshooting a Service or ReplicaSet, compare:
> - the resource selector
> - the target Pod labels

---

## Manual scheduling

If the scheduler is bypassed, a Pod can be pinned using `nodeName`.

Example idea:
- set `spec.nodeName: node01`

This is useful conceptually, but in normal clusters the scheduler is preferred.

---

## Taints and tolerations

Taints are applied to **nodes**.  
Tolerations are applied to **Pods**.

Mental model:
- taint = repel
- toleration = permission to ignore repelling rule

Common taint effects:
- `NoSchedule`
- `PreferNoSchedule`
- `NoExecute`

Useful commands:

```bash
kubectl taint nodes node01 app=blue:NoSchedule
kubectl taint nodes node01 app=blue:NoExecute
kubectl taint nodes node01 app-
```

Example toleration:

```yaml
tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
```

> [!warning] Very common confusion
> Taints/tolerations do **not guarantee** a Pod will land on a node.
> They only say whether a Pod is allowed there.

---

## Node selector

Simple scheduling based on node labels.

Example:

```yaml
nodeSelector:
  size: Large
```

Useful commands:

```bash
kubectl label node node01 size=Large
kubectl get nodes --show-labels
```

---

## Node affinity

Node affinity is more expressive than `nodeSelector`.

Useful operators:
- `In`
- `NotIn`
- `Exists`

Important types:
- `requiredDuringSchedulingIgnoredDuringExecution`
- `preferredDuringSchedulingIgnoredDuringExecution`

Example:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: size
              operator: In
              values:
                - Large
```

> [!tip] Comparison
> - use `nodeSelector` for simple exact matching
> - use `nodeAffinity` for richer scheduling logic

---

## Taints/tolerations vs node affinity

These solve different problems:

- **Taints/tolerations** → keep unwanted Pods away from a node
- **Node affinity** → attract a Pod toward specific nodes

You often combine them.

---

## DaemonSets

A DaemonSet ensures one Pod runs on every matching node.

Common uses:
- log collectors
- monitoring agents
- CNI agents
- kube-proxy

Useful commands:

```bash
kubectl get daemonsets -A
kubectl describe daemonset <name>
```

> [!note] Good example
> `kube-proxy` is commonly deployed as a DaemonSet.

---

## Static Pods

Static Pods are managed directly by the kubelet from files on disk.

Common path:

```
/etc/kubernetes/manifests/
```

Important behaviors:
- kubelet watches the directory
- if file appears, Pod is created
- if file changes, Pod is recreated
- if file is removed, Pod is deleted

If the node is part of a cluster:
- kubelet also creates a **mirror Pod** visible in the API server
- but you still edit the file on disk to change the real static Pod

Useful commands:

```bash
ls /etc/kubernetes/manifests/
crictl ps
journalctl -u kubelet
kubectl get pods -n kube-system
```

> [!important] Critical exam concept
> On kubeadm clusters, control plane components such as:
> - `kube-apiserver`
> - `kube-controller-manager`
> - `kube-scheduler`
> - `etcd`
> often run as **static Pods**.

### Static Pods vs DaemonSets

| Topic | Static Pod | DaemonSet |
| --- | --- | --- |
| Managed by | kubelet directly | DaemonSet controller |
| Needs API server | no | yes |
| Use case | control plane / special local pods | one Pod per node |
| Editable via API | no | yes |

---

## Priority classes

Priority classes let Kubernetes prefer more important Pods first.

High-priority Pods:
- schedule before lower-priority Pods
- may preempt lower-priority Pods if enabled

---

## Multiple schedulers and scheduler profiles

You can run multiple schedulers in a cluster and choose one with `schedulerName`.

This is more advanced, but know:
- default scheduler is usually enough
- custom schedulers and profiles alter scheduling behavior

---

## Admission controllers

Admission controllers run after authentication/authorization but before object persistence.

They can:
- validate requests
- mutate requests

Examples:
- `NamespaceLifecycle`
- `LimitRanger`
- `ServiceAccount`
- `DefaultStorageClass`
- `ResourceQuota`
- mutating and validating webhooks

> [!note] Simple exam mental model
> Authentication = who are you?  
> Authorization = what can you do?  
> Admission = should this request be allowed or modified?

---

## Resource requests, limits, LimitRange, ResourceQuota

### Requests and limits

Set on individual containers. The scheduler uses **requests** to decide placement; **limits** cap actual usage at runtime.

```yaml
resources:
  requests:
    cpu: "250m"       # 250 millicores = 0.25 of a CPU
    memory: "64Mi"
  limits:
    cpu: "500m"
    memory: "128Mi"
```

**CPU units:** `1` = one core, `500m` = half a core, `100m` = 0.1 core  
**Memory units:** `Mi` = mebibytes, `Gi` = gibibytes, `M`/`G` = megabytes/gigabytes (avoid `M`/`G` — use `Mi`/`Gi`)

**What happens when you exceed the limit:**
- **CPU:** throttled (pod keeps running, just slower)
- **Memory:** OOMKilled — container is terminated and restarted (shows `OOMKilled` in `kubectl describe pod`)

**What happens with no limits set:**
- Pod can consume all CPU/memory on the node — it will starve other pods

**If only limits are set (no requests):** Kubernetes sets request = limit automatically.

### LimitRange

Sets **namespace-level defaults** for containers that don't specify their own requests/limits. Applied by the `LimitRanger` admission controller to newly created pods.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-cpu-defaults
  namespace: dev
spec:
  limits:
  - type: Container
    default:          # applied as limit if none set
      cpu: "500m"
      memory: "128Mi"
    defaultRequest:   # applied as request if none set
      cpu: "250m"
      memory: "64Mi"
    max:              # pods requesting more than this are rejected
      cpu: "1"
      memory: "512Mi"
    min:              # pods requesting less than this are rejected
      cpu: "50m"
      memory: "16Mi"
```

> [!note] LimitRange only affects **newly created** pods. Existing pods are not updated.

### ResourceQuota

Sets **hard limits on total resource consumption** across all pods in a namespace. Applied by the `ResourceQuota` admission controller.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "8"
    limits.memory: "8Gi"
    count/persistentvolumeclaims: "5"
```

```bash
kubectl get resourcequota -n dev
kubectl describe resourcequota ns-quota -n dev    # shows Used vs Hard
```

> [!warning] If a namespace has a ResourceQuota for CPU/memory, every pod in that namespace **must** specify requests and limits — pods without them will be rejected.