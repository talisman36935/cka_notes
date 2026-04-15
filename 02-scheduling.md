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