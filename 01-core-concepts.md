---
tags:
  - kubernetes
  - cka
  - core-concepts
---

# ✅ Core Concepts

## Cluster architecture

At a high level, a Kubernetes cluster looks like this:

User → `kubectl` → `kube-apiserver` → `etcd`  
                                        ↓  
                     scheduler / controllers  
                                        ↓  
                             kubelet → container runtime

### Mental model

- The **API server** is the central control point.
- The **scheduler** decides *where* Pods should run.
- **Controllers** make sure reality matches desired state.
- The **kubelet** runs workloads on each node.
- The **container runtime** actually starts containers.
- **etcd** stores cluster state.

> [!important] High-value exam fact
> For most practical purposes, the **API server is the component that reads/writes cluster state in `etcd`**. If you understand that flow, many control plane questions become easier.

### Quick checks

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
kubectl get --raw /healthz
kubectl get --raw /readyz?verbose
```

---

## Control plane components

### `kube-apiserver`

The API server:
- receives requests from `kubectl`, controllers, schedulers, and kubelets
- authenticates and authorizes requests
- validates manifests
- stores and retrieves state from `etcd`

### `etcd`

`etcd` is a distributed key-value store that holds:
- Pods
- Deployments
- Services
- ConfigMaps
- Secrets
- RBAC objects
- node information
- cluster metadata

> [!warning] CKA priority
> `etcd` backup and restore is one of the highest-value exam topics.

Useful commands:

```bash
ETCDCTL_API=3 etcdctl snapshot save snapshot.db
ETCDCTL_API=3 etcdctl snapshot status snapshot.db --write-out=table
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup
```

### `kube-scheduler`

The scheduler watches for Pods without a node assignment and decides where they should run.

It mainly works in two phases:
1. **Filtering** — remove nodes that cannot run the Pod
2. **Scoring / Ranking** — rank remaining nodes and choose the best one

It considers:
- CPU and memory availability
- taints and tolerations
- node selectors
- node affinity
- topology constraints
- priorities

Useful checks:

```bash
kubectl describe pod <pod-name>
kubectl get events --sort-by=.metadata.creationTimestamp
```

### `kube-controller-manager`

Controllers keep actual state aligned with desired state.

Examples:
- Node controller
- Deployment controller
- ReplicaSet controller
- Job controller
- Endpoints controller

Mental model:
- desired state says 3 replicas
- actual state says 2 replicas
- controller creates 1 more Pod

---

## Worker node components

### `kubelet`

The kubelet runs on every node and:
- registers the node
- watches for Pods assigned to that node
- talks to the container runtime
- reports Pod and node status back to the API server

Useful checks:

```bash
systemctl status kubelet
journalctl -u kubelet
ps aux | grep kubelet
```

### `kube-proxy`

`kube-proxy` runs on nodes and programs service forwarding rules.

Common proxy backends:
- `iptables`
- `ipvs`

Its job:
- watch Service and Endpoint/EndpointSlice objects
- send traffic for a Service to matching backend Pods

Useful checks:

```bash
kubectl get daemonset -n kube-system
kubectl get pods -n kube-system
kubectl describe svc <service-name>
kubectl get endpoints <service-name>
```

### Container runtime and CRI

## CRI = Container Runtime Interface

CRI is the standard interface Kubernetes uses to talk to a container runtime.

Common runtimes:
- `containerd`
- `CRI-O`

Docker used to be common, but Kubernetes now prefers CRI-compatible runtimes directly.

> [!tip] Runtime tool cheat sheet
> - `ctr` → low-level containerd tool, mostly debugging
> - `nerdctl` → Docker-like CLI for containerd
> - `crictl` → Kubernetes-focused runtime debugging tool for CRI runtimes

Useful commands:

```bash
crictl ps
crictl ps -a
crictl images
crictl logs <container-id>
crictl inspect <container-id>
```

### Docker vs containerd

Docker is a broader platform:
- CLI
- image build tools
- auth
- volumes
- container runtime pieces

containerd is a focused runtime layer and integrates directly with Kubernetes through CRI.

> [!note] Exam takeaway
> You do **not** need Docker on Kubernetes nodes. You **do** need a working CRI-compatible runtime.

---

## Pods

A Pod is the smallest deployable unit in Kubernetes.

A Pod typically runs one main container, but it can run multiple tightly coupled containers.

Containers in the same Pod share:
- network namespace
- IP address
- port space
- volumes
- lifecycle

Useful commands:

```bash
kubectl run nginx --image=nginx
kubectl get pods -o wide
kubectl describe pod nginx
kubectl exec -it nginx -- sh
```

> [!important] Scaling rule
> Kubernetes scales by adding **more Pods**, not by adding more copies of the same app container into one Pod.

---

## ReplicaSets and Deployments

### ReplicaSet

A ReplicaSet ensures a specified number of Pod replicas exist.

Important fields:
- `replicas`
- `selector`
- `template`

> [!warning] Must remember
> The ReplicaSet selector must match the Pod template labels.

Useful commands:

```bash
kubectl get rs
kubectl describe rs <name>
kubectl scale rs <name> --replicas=3
```

### Deployment

A Deployment manages ReplicaSets and adds:
- rolling updates
- rollout history
- rollback
- declarative updates

Useful commands:

```bash
kubectl create deployment web --image=nginx
kubectl scale deployment web --replicas=3
kubectl rollout status deployment web
kubectl rollout history deployment web
kubectl rollout undo deployment web
```

---

## Services

A Service gives stable networking to a changing set of Pods.

### Service types

- **ClusterIP** → internal access only
- **NodePort** → exposes a port on every node
- **LoadBalancer** → cloud load balancer in supported environments

### NodePort port mapping

Three ports matter:
- `targetPort` → the Pod's port
- `port` → the Service's internal port
- `nodePort` → the external node port

Useful commands:

```bash
kubectl expose deployment web --port=80
kubectl expose deployment web --type=NodePort --port=80
kubectl get svc
kubectl describe svc web
kubectl get endpoints web
```

> [!tip] Service debugging flow
> If a Service is broken:
> 1. `kubectl get svc`
> 2. `kubectl describe svc`
> 3. `kubectl get endpoints`
> 4. `kubectl get pods --show-labels`
> 5. verify Pods are `Ready`

> [!warning] Common failure
> **No endpoints** usually means:
> - selector mismatch
> - Pods are not Ready
> - wrong labels on Pods
> - wrong namespace

---

## Namespaces

Namespaces logically separate resources inside a cluster.

Common namespaces:
- `default`
- `kube-system`
- `kube-public`
- `kube-node-lease`

Useful commands:

```bash
kubectl get ns
kubectl create ns dev
kubectl get pods -n kube-system
kubectl config set-context --current --namespace=dev
```

---

## Imperative vs declarative

### Imperative

You tell Kubernetes exactly what to do.

Examples:

```bash
kubectl run nginx --image=nginx
kubectl create deployment web --image=nginx
```

Best for:
- speed
- exam tasks
- quick scaffolding

### Declarative

You write YAML and apply it.

Examples:

```bash
kubectl apply -f pod.yaml
kubectl apply -f deploy.yaml
```

Best for:
- repeatability
- version control
- real-world operations

### Fast YAML generation

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl expose deployment web --port=80 --dry-run=client -o yaml > svc.yaml
```

### `kubectl apply` mental model

`kubectl apply` compares:
- your manifest
- the live object
- the last-applied configuration

It is ideal for managing objects declaratively over time.