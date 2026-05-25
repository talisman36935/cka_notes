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

## StatefulSets

StatefulSets manage pods that need **stable, persistent identity** — each pod gets a fixed name, hostname, and its own PVC that survives rescheduling.

Use when: databases, message queues, any app that needs to know which replica it is.  
Use Deployment when: stateless apps where all replicas are interchangeable.

| | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random suffix (`web-6d4bf`) | Ordinal index (`web-0`, `web-1`) |
| Scaling order | Unordered | Sequential (0→1→2 up, 2→1→0 down) |
| Storage | Shared or none | Unique PVC per pod (persists on delete) |
| DNS | Single Service hostname | Per-pod DNS: `<pod>.<service>.<ns>.svc.cluster.local` |

### Requirements

StatefulSets **require** a headless Service (`clusterIP: None`) to provide per-pod DNS:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None        # headless — no load balancing, just DNS
  selector:
    app: mysql
  ports:
  - port: 3306
```

### StatefulSet YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"     # must match the headless Service name
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:    # creates a unique PVC for each pod
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 10Gi
```

**Resulting pod DNS names:** `mysql-0.mysql.default.svc.cluster.local`, `mysql-1.mysql...` etc.  
**Resulting PVC names:** `data-mysql-0`, `data-mysql-1`, `data-mysql-2`

> [!warning] Deleting a StatefulSet does **not** delete its PVCs — data is preserved intentionally. Delete PVCs manually if you want to clean up storage.

```bash
kubectl get statefulsets
kubectl describe statefulset mysql
kubectl scale statefulset mysql --replicas=5
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

## Jobs and CronJobs

### Job

A Job runs a pod to **completion** rather than keeping it running. The controller retries on failure until the desired number of successful completions is reached.

Key fields:

| Field | Purpose | Default |
|---|---|---|
| `completions` | Total successful completions needed | 1 |
| `parallelism` | Pods running simultaneously | 1 |
| `backoffLimit` | Max retries before the Job is marked failed | 6 |
| `restartPolicy` | Must be `Never` or `OnFailure` (not `Always`) | — |

`Never` → failed pod is left for inspection, a new pod is created for each retry.  
`OnFailure` → failed pod restarts in-place.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  completions: 3
  parallelism: 2
  backoffLimit: 4
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

```bash
# Imperative
kubectl create job pi --image=perl:5.34 -- perl -Mbignum=bpi -wle "print bpi(2000)"

kubectl get jobs
kubectl describe job pi
kubectl get pods --selector=batch.kubernetes.io/job-name=pi
kubectl logs jobs/pi          # logs from the job's pod
```

### CronJob

A CronJob creates a Job on a schedule. The `schedule` field uses standard cron syntax.

```
┌─ minute (0-59)
│ ┌─ hour (0-23)
│ │ ┌─ day of month (1-31)
│ │ │ ┌─ month (1-12)
│ │ │ │ ┌─ day of week (0-6, Sunday=0)
* * * * *
```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/5 * * * *"    # every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["echo", "hello"]
          restartPolicy: OnFailure
```

```bash
kubectl create cronjob hello --image=busybox --schedule="*/5 * * * *" -- echo hello

kubectl get cronjobs
kubectl get jobs          # jobs created by the cronjob appear here
```

> [!warning] Jobs require `restartPolicy: Never` or `OnFailure`. The default `Always` is invalid for Jobs and CronJobs and will cause a validation error.

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