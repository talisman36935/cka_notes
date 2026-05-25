# Appendix D — JSONPath & Holistic Mental Models

---

## Part 1: JSONPath

### The core idea

`kubectl -o json` returns a JSON document. JSONPath is a query language that extracts fields from it. You wrap expressions in `{}` and pass the whole thing as a string to `-o jsonpath=`.

```bash
# Explore the structure first — don't guess field names
kubectl get pod nginx -o json | jq .
kubectl get nodes -o json | jq '.items[0].status.addresses'
```

---

### Single object vs list — the critical distinction

When kubectl returns **one resource** (e.g. `kubectl get pod nginx`), the root is the object itself:

```bash
kubectl get pod nginx -o jsonpath='{.metadata.name}'
kubectl get pod nginx -o jsonpath='{.spec.containers[0].image}'
```

When kubectl returns a **list** (e.g. `kubectl get pods`), the root is a List with `.items[]`:

```bash
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{.items[0].spec.containers[0].image}'
```

> [!warning] This is the most common JSONPath mistake. If your output is empty, check whether you need `.items[*]` or not.

---

### Syntax reference

| Syntax | Meaning | Example |
|--------|---------|---------|
| `.field` | Access a field by name | `.metadata.name` |
| `['field']` | Same, bracket notation | `['metadata']['name']` |
| `.items[*]` | All elements of an array | `.items[*].metadata.name` |
| `.items[0]` | First element | `.items[0].metadata.name` |
| `.items[-1]` | Last element | `.items[-1].metadata.name` |
| `{range .items[*]}...{end}` | Loop over array | see below |
| `?(@.field=="value")` | Filter — keep elements where condition is true | `.items[?(@.status.phase=="Running")]` |
| `{"\n"}` / `{"\t"}` | Literal newline / tab in output | see below |

---

### Pattern 1 — All values of one field across a list

```bash
# All node names, space-separated
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'

# All pod images (all containers across all pods)
kubectl get pods -A -o jsonpath='{.items[*].spec.containers[*].image}'
```

---

### Pattern 2 — Loop with range for formatted output

Use `{range ...}{end}` to produce one line per item.

```bash
# Node name + role, one per line
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[-1].type}{"\n"}{end}'

# Pod name + phase
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Pod name + node it's on
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'

# All container names + images in a deployment
kubectl get pods -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}{end}'
```

---

### Pattern 3 — Filter with `?(@.field==)`

The `@` refers to the current element in the array. Returns only matching elements.

```bash
# Names of Running pods only
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'

# Node InternalIP addresses only
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Containers using a specific image
kubectl get pods -o jsonpath='{.items[*].spec.containers[?(@.image=="nginx")].name}'
```

> [!note] Filter expressions return the **matching element**, not a boolean. Chain another field access after `]` to drill into the matched item.

---

### Pattern 4 — Nested arrays (range inside range)

```bash
# Every container name + image, across all pods
kubectl get pods -A \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}  {.name}: {.image}{"\n"}{end}{end}'
```

---

### Pattern 5 — kubeconfig queries

kubeconfig is a single object (not a list resource), so no `.items`.

```bash
# All context names
kubectl config view -o jsonpath='{.contexts[*].name}'

# All user names
kubectl config view --kubeconfig=/root/my-kube-config -o jsonpath='{.users[*].name}'

# Find the context name that uses a given user
kubectl config view --kubeconfig=/root/my-kube-config \
  -o jsonpath='{.contexts[?(@.context.user=="aws-user")].name}'

# Current context
kubectl config view -o jsonpath='{.current-context}'
```

---

### Pattern 6 — sort-by

`--sort-by` takes a single JSONPath expression. It applies to each item individually (no `.items[*]`).

```bash
# PVs sorted by capacity
kubectl get pv --sort-by=.spec.capacity.storage

# Pods sorted by creation time
kubectl get pods --sort-by=.metadata.creationTimestamp

# Events sorted by time (useful for debugging)
kubectl get events --sort-by=.lastTimestamp

# Combined: sort PVs and show custom columns
kubectl get pv \
  --sort-by=.spec.capacity.storage \
  -o custom-columns='NAME:.metadata.name,CAPACITY:.spec.capacity.storage,ACCESS:.spec.accessModes[0],CLAIM:.spec.claimRef.name'
```

---

### Custom columns

Custom columns are a cleaner alternative to JSONPath for tabular output. Format: `HEADER:.jsonpath.expression`.

```bash
# Pod name + node
kubectl get pods -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName'

# Pod name + priority class
kubectl get pods -o custom-columns='NAME:.metadata.name,PRIORITY:.spec.priorityClassName'

# Nodes with OS image
kubectl get nodes -o custom-columns='NAME:.metadata.name,OS:.status.nodeInfo.osImage,VERSION:.status.nodeInfo.kubeletVersion'

# PVs with reclaim policy and access mode
kubectl get pv -o custom-columns='NAME:.metadata.name,CAPACITY:.spec.capacity.storage,POLICY:.spec.persistentVolumeReclaimPolicy,ACCESS:.spec.accessModes[0]'
```

> [!tip] Unlike `-o jsonpath`, custom columns automatically prints headers and aligns columns. Use them when you want readable tabular output; use `-o jsonpath` when you need to pipe the values to a file or further processing.

---

### 10 exam-critical JSONPath patterns

```bash
# 1. All node names
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'

# 2. Node names + InternalIP, one per line
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}'

# 3. All pod images in the cluster
kubectl get pods -A -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u

# 4. Pod name + status, all namespaces
kubectl get pods -A \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# 5. Which context uses a given user
kubectl config view -o jsonpath='{.contexts[?(@.context.user=="dev-user")].name}'

# 6. PVs sorted by storage, showing key fields
kubectl get pv --sort-by=.spec.capacity.storage \
  -o custom-columns='NAME:.metadata.name,CAPACITY:.spec.capacity.storage'

# 7. Count resources without jq
kubectl get pods --no-headers | wc -l
kubectl get clusterroles --no-headers | wc -l

# 8. Service selectors for debugging
kubectl get svc myservice -o jsonpath='{.spec.selector}'

# 9. Find pods NOT on a given node (pipe + grep -v)
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}' \
  | grep -v node01

# 10. Image used by a specific container in a pod
kubectl get pod mypod -o jsonpath='{.spec.containers[?(@.name=="mycontainer")].image}'
```

---

## Part 2: Holistic Mental Models

> [!abstract] These are the conceptual threads that tie the whole exam together. If you can think in these patterns, you can reason about any scenario you haven't seen before.

---

### 1. Labels are the universal glue

Almost every relationship in Kubernetes is built on labels and selectors. Nothing links by name except where explicitly stated.

| Who uses selector | Selector field | Finds what |
|---|---|---|
| Service | `spec.selector` (flat map) | Pods to route to |
| Deployment / RS | `spec.selector.matchLabels` | Pods it owns |
| NetworkPolicy | `spec.podSelector` | Pods the policy applies to |
| NetworkPolicy ingress/egress | `from[]/to[].podSelector` | Pods allowed through |
| HPA | `scaleTargetRef.name` | Deployment by name (exception) |
| `kubectl get pods -l` | `-l key=value` | Your own queries |

The **pod template labels** (`spec.template.metadata.labels`) are what all these selectors match against. A Deployment's `spec.selector` must exactly match its `spec.template.metadata.labels` — mismatch is a validation error.

Services use a **flat map** (`selector: {app: web}`). Deployments use **matchLabels / matchExpressions**. Mixing them up is the most common exam mistake.

---

### 2. The volume binding pattern

Two separate stanzas in the pod spec, joined by `name`:

```
spec:
  volumes:
  - name: my-vol          ← declares the source (hostPath, PVC, configMap, etc.)
    hostPath:
      path: /data

  containers:
  - name: app
    volumeMounts:
    - name: my-vol        ← must exactly match spec.volumes[].name
      mountPath: /app/data
```

Get the `name` wrong and the pod fails to start with a `MountVolume` error. The `name` is only meaningful inside the pod spec — it's not a reference to anything external.

---

### 3. PV → PVC → Pod reference chain

```
PersistentVolume  (cluster-scoped, admin creates)
       │
       │  Kubernetes binds when:
       │  • accessModes match (RWO/ROX/RWX must overlap)
       │  • PV capacity ≥ PVC request
       │  • storageClassName matches (or both unset)
       ▼
PersistentVolumeClaim  (namespace-scoped, user creates)
       │
       │  Pod references by name:
       │  spec.volumes[].persistentVolumeClaim.claimName: my-pvc
       ▼
Pod
```

**Common breaks at each link:**
- PVC stays `Pending` → no matching PV (accessMode mismatch is #1 cause)
- PVC stays `Terminating` → a pod is still mounting it; delete the pod first
- Pod stays `Pending` → PVC is still Pending

**Reclaim policy** controls what happens to the PV when the PVC is deleted:
- `Retain` → PV stays, status becomes `Released`, not available for new PVCs until manually cleaned
- `Delete` → PV and the underlying storage are deleted (default for dynamic provisioning)

---

### 4. Scheduler decision tree

For every pending pod, the scheduler evaluates each node in this order:

```
1. nodeName set?              → skip scheduler entirely, bind directly
2. Node taints                → pod must have matching toleration for each taint, or node is filtered out
3. nodeSelector               → node must have every label listed
4. nodeAffinity required      → requiredDuringScheduling rules must all match
5. Resource requests          → node must have free CPU + memory ≥ request
6. nodeAffinity preferred     → score nodes, pick highest
7. Pod affinity / anti-affinity → adjust score by proximity to other pods
```

**Key mental distinction:**
- Taints/tolerations → nodes *repel* pods (opt-in to a node)
- Node affinity → pods *attract* to nodes (opt-in from the pod)
- They compose: taint keeps everyone else off, affinity pulls the right pod on

If a pod is stuck `Pending`:
1. `kubectl describe pod <name>` → look at `Events:` for the reason
2. "0/3 nodes are available" → read the reason (taints, resources, affinity)
3. Check `kubectl describe node` for taints, capacity, conditions

---

### 5. Network traffic path

```
Client Pod
    │
    │  (DNS resolves service name → ClusterIP)
    ▼
Service (ClusterIP — virtual, iptables/IPVS rules via kube-proxy)
    │
    │  (kube-proxy programs rules pointing to Endpoints)
    ▼
Endpoints object  (auto-populated: pod IPs where selector matches AND pod is Ready)
    │
    ▼
Target Pod
```

**If traffic isn't reaching the target, check in order:**
1. `kubectl get endpoints myservice` — if empty, the selector is broken or no pods are Ready
2. `kubectl get pods --show-labels` — do the pod labels match the service selector?
3. Are pods passing readiness probes? (`kubectl describe pod`)
4. Is there a NetworkPolicy blocking egress from the client or ingress to the target?
5. Is kube-proxy healthy? (`kubectl get pods -n kube-system -l k8s-app=kube-proxy`)

DNS within the cluster:
- `myservice` → resolves if caller is in same namespace
- `myservice.mynamespace` → cross-namespace short form
- `myservice.mynamespace.svc.cluster.local` → full FQDN, always works
- Pod FQDN: `10-244-2-5.default.pod.cluster.local` (dashes, not dots)

---

### 6. Authentication → Authorization → Admission

Every API request passes through three gates:

```
Request
  │
  ├─ Authentication (authn): who are you?
  │   certificates, service account tokens, OIDC tokens, basic auth
  │
  ├─ Authorization (authz): are you allowed?
  │   RBAC: Role/ClusterRole + Binding defines allowed verbs on resources
  │
  └─ Admission: should this be accepted/modified?
      Mutating webhooks → run first, can modify the object
      Validating webhooks → run second, accept or reject
      Built-in: LimitRanger, ResourceQuota, NamespaceLifecycle, DefaultStorageClass
```

**RBAC scope:**
- `Role` + `RoleBinding` → namespaced access (can only grant permissions within one namespace)
- `ClusterRole` + `ClusterRoleBinding` → cluster-wide access
- `ClusterRole` + `RoleBinding` → reusable role template, applied within a namespace
- Non-namespaced resources (Nodes, PVs, Namespaces, ClusterRoles) **require** a ClusterRole

```bash
# Fast permission check
kubectl auth can-i list pods --as=dev-user -n default
kubectl auth can-i get nodes --as=system:serviceaccount:default:my-sa
```

---

### 7. Namespaced vs cluster-scoped resources

**Namespaced** (most things):
Pods, Deployments, ReplicaSets, StatefulSets, DaemonSets, Jobs, CronJobs, Services, Endpoints, ConfigMaps, Secrets, PersistentVolumeClaims, ServiceAccounts, Roles, RoleBindings, NetworkPolicies, Ingress, HPA

**Cluster-scoped** (no namespace):
Nodes, PersistentVolumes, StorageClasses, ClusterRoles, ClusterRoleBindings, Namespaces, CRDs, PriorityClasses, IngressClasses

> [!tip] Quick test: does `kubectl get <resource> -n kube-system` vs `kubectl get <resource>` give different results? If `-n` has no effect, it's cluster-scoped.

---

### 8. Certificate hierarchy (kubeadm clusters)

```
/etc/kubernetes/pki/
├── ca.crt / ca.key                    Cluster CA — root of trust for everything
├── apiserver.crt                      API server TLS serving cert (signed by cluster CA)
├── apiserver-kubelet-client.crt       API server authenticates to kubelet with this
├── apiserver-etcd-client.crt          API server authenticates to etcd with this
│
└── etcd/
    ├── ca.crt / ca.key                etcd CA — separate CA, only for etcd
    ├── server.crt                     etcd server TLS cert
    └── peer.crt                       etcd peer-to-peer cert

/var/lib/kubelet/pki/
└── kubelet.crt                        kubelet serving cert (signed by cluster CA)
```

The etcd CA is **separate** from the cluster CA. The API server uses two different client certificates: one for talking to kubelets (`apiserver-kubelet-client.crt`, signed by cluster CA) and one for talking to etcd (`apiserver-etcd-client.crt`, signed by the etcd CA).

```bash
# Inspect any cert
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout \
  | grep -A2 "Subject\|Issuer\|Not After\|DNS:"

# If the API server is down: crictl, not kubectl
crictl ps -a
crictl logs <container-id>
journalctl -u kubelet | tail -50
```

---

### 9. Static pods and the control plane exception

kubelet watches `/etc/kubernetes/manifests/` and manages pods directly from files on disk — without the API server. This is how control plane components bootstrap themselves.

```
/etc/kubernetes/manifests/
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
├── kube-scheduler.yaml
└── etcd.yaml
```

**Consequences:**
- These pods appear in `kubectl get pods -n kube-system` as mirror pods, but are **read-only** via the API
- To change them: edit the manifest file → kubelet recreates the pod automatically
- To delete them: remove the manifest file (not `kubectl delete`)
- Name format: `<name>-<nodename>` (e.g. `etcd-controlplane`)
- When the API server is broken, use `crictl` and `journalctl -u kubelet` to diagnose

```bash
# Find the static pod path (in case it's non-standard)
cat /var/lib/kubelet/config.yaml | grep staticPodPath
```

---

### 10. The control loop — why Kubernetes behaves the way it does

Every controller runs a reconcile loop:

```
observe actual state → compare to desired state → act to close the gap → repeat
```

This explains many exam scenarios:
- Deleting a pod managed by a Deployment → controller immediately recreates it. To remove pods, scale the Deployment to 0 or delete the Deployment.
- Changing a Deployment image → controller creates a new ReplicaSet, scales it up, scales old one down (rolling update).
- Node goes NotReady → Node controller marks pods as `Unknown`, eventually evicts them and reschedules on other nodes (after `pod-eviction-timeout`, default 5 min).
- Job pod fails → Job controller creates a new pod (up to `backoffLimit`).

`kubectl apply` is reconcile-friendly: it patches the stored object, and the relevant controller picks up the diff.

`kubectl replace --force` forces a delete+recreate — useful when immutable fields (like `spec.selector`) need changing.

---

### 11. NetworkPolicy mental model — AND vs OR

The most misread YAML on the exam:

```yaml
# AND — pod must satisfy BOTH conditions (same list item)
ingress:
- from:
  - podSelector:
      matchLabels: {role: frontend}
    namespaceSelector:         # ← no leading dash → same item → AND
      matchLabels: {env: prod}

# OR — pod satisfies EITHER condition (separate list items)
ingress:
- from:
  - podSelector:
      matchLabels: {role: frontend}
  - namespaceSelector:         # ← leading dash → new item → OR
      matchLabels: {env: prod}
```

The rest of the mental model:
- If a pod has **no** NetworkPolicy selecting it → all traffic flows freely
- Once **any** policy selects a pod for ingress → all ingress is blocked except what's listed
- Same for egress independently
- Multiple policies are **additive** (union, not intersection)
- Always include a DNS egress rule (`port 53 UDP+TCP`) or name resolution breaks

> [!warning] CNI enforcement: NetworkPolicies are API objects but they do nothing without a CNI that enforces them. **Flannel does not enforce NetworkPolicies.** Calico, Cilium, Weave Net, and Kube-router do. If you apply a policy and traffic still flows freely, check your CNI.
