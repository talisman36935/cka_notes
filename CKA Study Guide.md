---
tags:
  - kubernetes
  - cka
  - study-guide
aliases:
  - CKA Guide
  - Certified Kubernetes Administrator Guide
---

# 🚀 CKA Study Guide

> [!info] What this guide is for
> This guide is built for the **Certified Kubernetes Administrator (CKA)** exam:
> - practical understanding over memorization
> - fast troubleshooting over perfect theory
> - hands-on commands you can type under pressure

> [!success] Winning mindset
> The exam is mostly:
> - identify the failing component quickly
> - verify your assumption with the right command
> - make the smallest correct fix
> - move on

## How to use this guide

1. Read each section until the flow makes sense.
2. Practice every command in a lab.
3. Use the **Speed Drills** appendix daily.
4. Use the **1-Week Crash Plan** appendix if your exam is close.

## Table of contents

- [✅ Core Concepts](#-core-concepts)
  - [Cluster architecture](#cluster-architecture)
  - [Control plane components](#control-plane-components)
  - [Worker node components](#worker-node-components)
  - [CRI = Container Runtime Interface](#cri--container-runtime-interface)
  - [Pods](#pods)
  - [ReplicaSets and Deployments](#replicasets-and-deployments)
  - [Services](#services)
  - [Namespaces](#namespaces)
  - [Imperative vs declarative](#imperative-vs-declarative)
- [✅ Scheduling](#-scheduling)
  - [Labels, selectors, and annotations](#labels-selectors-and-annotations)
  - [Manual scheduling](#manual-scheduling)
  - [Taints and tolerations](#taints-and-tolerations)
  - [Node selector](#node-selector)
  - [Node affinity](#node-affinity)
  - [Taints/tolerations vs node affinity](#taintstolerations-vs-node-affinity)
  - [DaemonSets](#daemonsets)
  - [Static Pods](#static-pods)
  - [Priority classes](#priority-classes)
  - [Multiple schedulers and scheduler profiles](#multiple-schedulers-and-scheduler-profiles)
  - [Admission controllers](#admission-controllers)
- [✅ Logging & Monitoring](#-logging--monitoring)
  - [Managing application logs](#managing-application-logs)
  - [Node-level and control plane logs](#node-level-and-control-plane-logs)
  - [Metrics Server](#metrics-server)
- [✅ Application Lifecycle Management](#-application-lifecycle-management)
  - [Rolling updates and rollbacks](#rolling-updates-and-rollbacks)
  - [Commands and arguments](#commands-and-arguments)
  - [Secrets](#secrets)
  - [Encrypting Secrets at rest](#encrypting-secrets-at-rest)
  - [Multi-container Pods](#multi-container-pods)
  - [Autoscaling](#autoscaling)
- [✅ Cluster Maintenance](#-cluster-maintenance)
  - [OS upgrades and node maintenance](#os-upgrades-and-node-maintenance)
  - [Cluster upgrade with kubeadm](#cluster-upgrade-with-kubeadm)
  - [Backup and restore methods](#backup-and-restore-methods)
- [✅ Security](#-security)
  - [Security primitives](#security-primitives)
  - [Authentication](#authentication)
  - [Authorization](#authorization)
  - [TLS in Kubernetes](#tls-in-kubernetes)
  - [Certificates API](#certificates-api)
  - [Kubeconfig](#kubeconfig)
  - [API groups](#api-groups)
  - [RBAC](#rbac)
  - [Service Accounts](#service-accounts)
  - [Image security](#image-security)
  - [Security contexts](#security-contexts)
  - [Network Policies](#network-policies)
  - [CRDs, custom controllers, and operators](#crds-custom-controllers-and-operators)
- [✅ Storage](#-storage)
  - [Storage basics](#storage-basics)
  - [Volumes](#volumes)
  - [CSI = Container Storage Interface](#csi--container-storage-interface)
  - [Persistent Volumes (PV)](#persistent-volumes-pv)
  - [Persistent Volume Claims (PVC)](#persistent-volume-claims-pvc)
  - [Reclaim policy](#reclaim-policy)
  - [StorageClass](#storageclass)
- [✅ Networking](#-networking)
  - [Network namespaces](#network-namespaces)
  - [CNI = Container Network Interface](#cni--container-network-interface)
  - [Cluster networking basics](#cluster-networking-basics)
  - [Service networking](#service-networking)
  - [DNS in Kubernetes](#dns-in-kubernetes)
  - [Ingress](#ingress)
  - [Gateway API](#gateway-api)
- [✅ Troubleshooting](#-troubleshooting)
  - [Pod troubleshooting flow](#pod-troubleshooting-flow)
  - [Application failure flow](#application-failure-flow)
  - [Service failure flow](#service-failure-flow)
  - [Node troubleshooting flow](#node-troubleshooting-flow)
  - [Control plane troubleshooting flow](#control-plane-troubleshooting-flow)
  - [Worker node troubleshooting flow](#worker-node-troubleshooting-flow)
- [✅ Design and Install a Kubernetes Cluster](#-design-and-install-a-kubernetes-cluster)
  - [Choosing infrastructure](#choosing-infrastructure)
  - [kubeadm essentials](#kubeadm-essentials)
  - [`etcd` in HA](#etcd-in-ha)
- [✅ Helm Basics](#-helm-basics)
- [✅ Kustomize Basics](#-kustomize-basics)
- [✅ Exam Priority Guide](#-exam-priority-guide)
- [✅ Final Rule](#-final-rule)
- [⚡ Appendix A — Speed Drills](#-appendix-a--speed-drills)
- [📅 Appendix B — 1-Week Crash Plan](#-appendix-b--1-week-crash-plan)
- [🧠 Appendix C — Default Debug Flows](#-appendix-c--default-debug-flows)
- [🏁 Final Reminder](#-final-reminder)

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

    kubectl cluster-info
    kubectl get nodes
    kubectl get pods -A
    kubectl get --raw /healthz
    kubectl get --raw /readyz?verbose

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

    ETCDCTL_API=3 etcdctl snapshot save snapshot.db
    ETCDCTL_API=3 etcdctl snapshot status snapshot.db --write-out=table
    ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup

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

    kubectl describe pod <pod-name>
    kubectl get events --sort-by=.metadata.creationTimestamp

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

    systemctl status kubelet
    journalctl -u kubelet
    ps aux | grep kubelet

### `kube-proxy`

`kube-proxy` runs on nodes and programs service forwarding rules.

Common proxy backends:
- `iptables`
- `ipvs`

Its job:
- watch Service and Endpoint/EndpointSlice objects
- send traffic for a Service to matching backend Pods

Useful checks:

    kubectl get daemonset -n kube-system
    kubectl get pods -n kube-system
    kubectl describe svc <service-name>
    kubectl get endpoints <service-name>

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

    crictl ps
    crictl ps -a
    crictl images
    crictl logs <container-id>
    crictl inspect <container-id>

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

    kubectl run nginx --image=nginx
    kubectl get pods -o wide
    kubectl describe pod nginx
    kubectl exec -it nginx -- sh

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

    kubectl get rs
    kubectl describe rs <name>
    kubectl scale rs <name> --replicas=3

### Deployment

A Deployment manages ReplicaSets and adds:
- rolling updates
- rollout history
- rollback
- declarative updates

Useful commands:

    kubectl create deployment web --image=nginx
    kubectl scale deployment web --replicas=3
    kubectl rollout status deployment web
    kubectl rollout history deployment web
    kubectl rollout undo deployment web

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

    kubectl expose deployment web --port=80
    kubectl expose deployment web --type=NodePort --port=80
    kubectl get svc
    kubectl describe svc web
    kubectl get endpoints web

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

    kubectl get ns
    kubectl create ns dev
    kubectl get pods -n kube-system
    kubectl config set-context --current --namespace=dev

---

## Imperative vs declarative

### Imperative

You tell Kubernetes exactly what to do.

Examples:

    kubectl run nginx --image=nginx
    kubectl create deployment web --image=nginx

Best for:
- speed
- exam tasks
- quick scaffolding

### Declarative

You write YAML and apply it.

Examples:

    kubectl apply -f pod.yaml
    kubectl apply -f deploy.yaml

Best for:
- repeatability
- version control
- real-world operations

### Fast YAML generation

    kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
    kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
    kubectl expose deployment web --port=80 --dry-run=client -o yaml > svc.yaml

### `kubectl apply` mental model

`kubectl apply` compares:
- your manifest
- the live object
- the last-applied configuration

It is ideal for managing objects declaratively over time.

---

# ✅ Scheduling

## Labels, selectors, and annotations

### Labels

Labels are key-value metadata used to organize and select resources.

Examples:
- `app=web`
- `env=prod`
- `tier=frontend`

Useful commands:

    kubectl label pod nginx env=prod
    kubectl get pods -l env=prod
    kubectl get pods --show-labels

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

    kubectl taint nodes node01 app=blue:NoSchedule
    kubectl taint nodes node01 app=blue:NoExecute
    kubectl taint nodes node01 app-

Example toleration:

    tolerations:
      - key: "app"
        operator: "Equal"
        value: "blue"
        effect: "NoSchedule"

> [!warning] Very common confusion
> Taints/tolerations do **not guarantee** a Pod will land on a node.
> They only say whether a Pod is allowed there.

---

## Node selector

Simple scheduling based on node labels.

Example:

    nodeSelector:
      size: Large

Useful commands:

    kubectl label node node01 size=Large
    kubectl get nodes --show-labels

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

    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: size
                  operator: In
                  values:
                    - Large

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

    kubectl get daemonsets -A
    kubectl describe daemonset <name>

> [!note] Good example
> `kube-proxy` is commonly deployed as a DaemonSet.

---

## Static Pods

Static Pods are managed directly by the kubelet from files on disk.

Common path:

    /etc/kubernetes/manifests/

Important behaviors:
- kubelet watches the directory
- if file appears, Pod is created
- if file changes, Pod is recreated
- if file is removed, Pod is deleted

If the node is part of a cluster:
- kubelet also creates a **mirror Pod** visible in the API server
- but you still edit the file on disk to change the real static Pod

Useful commands:

    ls /etc/kubernetes/manifests/
    crictl ps
    journalctl -u kubelet
    kubectl get pods -n kube-system

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

# ✅ Logging & Monitoring

## Managing application logs

Kubernetes does not centrally manage logs by default.

Most app logs go to:
- `stdout`
- `stderr`

Useful commands:

    kubectl logs pod-name
    kubectl logs pod-name -c container-name
    kubectl logs -f pod-name
    kubectl logs pod-name --previous

> [!tip] Use `--previous`
> If a container crashed and restarted, `--previous` often shows the real failure.

---

## Node-level and control plane logs

Container logs often appear under:

    /var/log/containers/

Kubelet logs:

    journalctl -u kubelet

Control plane / runtime logs:

    crictl ps
    crictl logs <container-id>

---

## Metrics Server

Metrics Server provides basic resource metrics used by:

- `kubectl top`
- HPA

Useful commands:

    kubectl top node
    kubectl top pod
    kubectl get apiservices | grep metrics

Install example:

    kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

> [!warning] Common lab issue
> HPA won't work if Metrics Server is missing or broken.

---

# ✅ Application Lifecycle Management

## Rolling updates and rollbacks

Deployments support rolling updates by default.

Useful commands:

    kubectl rollout status deployment app
    kubectl rollout history deployment app
    kubectl rollout undo deployment app

Useful strategy snippet:

    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 1
        maxSurge: 1

---

## Commands and arguments

Docker mapping:
- `ENTRYPOINT` → Kubernetes `command`
- `CMD` → Kubernetes `args`

Example:

    command: ["sleep"]
    args: ["3600"]

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

    kubectl create secret generic mysecret --from-literal=password=123
    kubectl get secret mysecret
    kubectl get secret mysecret -o yaml

Decode:

    echo <base64-value> | base64 -d

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

    kubectl logs pod -c container-name
    kubectl exec -it pod -c container-name -- sh

---

## Autoscaling

### HPA

Horizontal Pod Autoscaler changes the number of replicas based on metrics.

Useful commands:

    kubectl autoscale deployment app --cpu-percent=50 --min=1 --max=5
    kubectl get hpa
    kubectl describe hpa <name>

### VPA

Vertical Pod Autoscaler changes CPU/memory requests.

Usually conceptual for CKA:
- not installed by default in many clusters
- less likely than HPA to be a core hands-on task

### In-place Pod resize

Newer Kubernetes versions may support resource updates without full Pod recreation, depending on features and cluster version.

Conceptual knowledge is enough.

---

# ✅ Cluster Maintenance

## OS upgrades and node maintenance

Useful commands:

    kubectl cordon node01
    kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
    kubectl uncordon node01

Mental model:
- `cordon` → stop new scheduling
- `drain` → evict safe workloads
- `uncordon` → allow scheduling again

> [!warning] Common exam gotcha
> Draining can fail because of:
> - DaemonSets
> - local data
> - PodDisruptionBudgets

---

## Cluster upgrade with kubeadm

Version skew rules matter.

General principles:
- no component should be newer than the API server
- controller manager and scheduler can be slightly older
- kubelet and kube-proxy can be older within supported skew

### Control plane upgrade flow

1. upgrade `kubeadm`
2. run `kubeadm upgrade plan`
3. run `kubeadm upgrade apply <version>`
4. upgrade `kubelet` and `kubectl`
5. restart kubelet if needed

Commands:

    kubeadm upgrade plan
    kubeadm upgrade apply v1.29.0
    kubectl get nodes

### Worker node upgrade flow

1. drain node
2. upgrade `kubeadm`
3. run `kubeadm upgrade node`
4. upgrade `kubelet`
5. restart kubelet
6. uncordon node

Commands:

    kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
    kubeadm upgrade node
    systemctl restart kubelet
    kubectl uncordon node01

> [!important] One-minor-version rule
> In practice, upgrade one minor version at a time.

---

## Backup and restore methods

### What to back up

- YAML files in Git
- imperative cluster objects
- `etcd` snapshots

### API object backup

    kubectl get all --all-namespaces -o yaml > all-resources.yaml

### `etcd` backup

Authenticated example:

    ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/etcd/ca.crt \
      --cert=/etc/etcd/etcd-server.crt \
      --key=/etc/etcd/etcd-server.key

### `etcd` restore

    ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
      --data-dir /var/lib/etcd-from-backup

After restore:
- point `etcd` to restored data directory
- update the static Pod manifest if needed
- restart kubelet / control plane components as required

> [!warning] Exam trap
> Snapshot restore alone is not enough if the running `etcd` manifest still points to the old data directory.

---

# ✅ Security

## Security primitives

Kubernetes security is built around:
- securing hosts
- authenticating access
- authorizing actions
- controlling network traffic
- securing component communication with TLS

---

## Authentication

Authentication answers: **who are you?**

Common methods:
- certificates
- tokens
- external identity providers
- service accounts for workloads

---

## Authorization

Authorization answers: **what can you do?**

Common models:
- RBAC
- Node authorizer
- webhook
- ABAC (older)

---

## TLS in Kubernetes

Kubernetes components communicate using TLS.

Common cert directory on kubeadm clusters:

    /etc/kubernetes/pki/

Inspect a certificate:

    openssl x509 -in cert.crt -text -noout

> [!tip] Useful troubleshooting angle
> Control plane failures are often caused by:
> - wrong cert path
> - expired certificate
> - SAN mismatch
> - wrong CA

---

## Certificates API

Kubernetes can manage certificate requests via `CertificateSigningRequest` objects.

Useful commands:

    kubectl get csr
    kubectl describe csr <name>
    kubectl certificate approve <name>

---

## Kubeconfig

A kubeconfig contains:
- clusters
- users
- contexts

Useful commands:

    kubectl config view
    kubectl config get-contexts
    kubectl config current-context
    kubectl config use-context <context-name>

---

## API groups

Core group example:

    /api/v1

Named groups examples:

    /apis/apps/v1
    /apis/networking.k8s.io/v1
    /apis/rbac.authorization.k8s.io/v1

Useful commands:

    kubectl api-resources
    kubectl api-versions

---

## RBAC

### Role and RoleBinding

Roles are namespace-scoped.

Useful commands:

    kubectl create role pod-reader --verb=get,list,watch --resource=pods
    kubectl create rolebinding read-pods --role=pod-reader --user=user1
    kubectl get roles
    kubectl get rolebindings
    kubectl describe role pod-reader
    kubectl describe rolebinding read-pods

### `kubectl auth can-i`

This is one of the best RBAC debugging commands.

    kubectl auth can-i get pods
    kubectl auth can-i create deployments --as dev-user
    kubectl auth can-i list pods -n dev --as dev-user

### ClusterRole and ClusterRoleBinding

Use these for:
- cluster-scoped resources like `nodes`, `persistentvolumes`
- cross-namespace access

Useful commands:

    kubectl api-resources --namespaced=true
    kubectl api-resources --namespaced=false

> [!important] Exam memory hook
> - `Role` / `RoleBinding` → one namespace
> - `ClusterRole` / `ClusterRoleBinding` → cluster-wide or cluster-scoped resources

---

## Service Accounts

Service accounts are identities for Pods and automation.

Useful commands:

    kubectl create serviceaccount dashboard-sa
    kubectl get sa
    kubectl describe sa dashboard-sa
    kubectl create token dashboard-sa

Assign to a Pod:

    serviceAccountName: dashboard-sa

Disable token mount:

    automountServiceAccountToken: false

> [!note] Modern Kubernetes behavior
> Recent Kubernetes versions no longer automatically create long-lived service account token Secrets the old way. Prefer `kubectl create token` and projected tokens.

---

## Image security

Private registries usually require image pull secrets.

Useful command:

    kubectl create secret docker-registry regcred \
      --docker-server=<registry> \
      --docker-username=<user> \
      --docker-password=<password>

Then reference it with:

    imagePullSecrets:
      - name: regcred

---

## Security contexts

Security contexts let you define runtime security settings at:
- Pod level
- container level

Examples:
- `runAsUser`
- `runAsNonRoot`
- capabilities
- filesystem permissions

Example:

    securityContext:
      runAsUser: 1000
      capabilities:
        add: ["MAC_ADMIN"]

> [!tip] Precedence rule
> Container-level security context overrides pod-level settings for that container.

---

## Network Policies

NetworkPolicies control allowed ingress and/or egress for selected Pods.

Critical rules:
- if **no** policy selects a Pod, traffic is allowed by default
- once a policy selects a Pod for ingress/egress, that traffic becomes isolated
- only explicitly allowed traffic passes

Example pattern:
- allow only API Pods to reach DB Pods on port 3306

Useful commands:

    kubectl get networkpolicy
    kubectl describe networkpolicy <name>

> [!warning] Huge gotcha
> A NetworkPolicy only works if your CNI plugin **enforces** it.  
> Some CNIs support policies; some do not.

---

## CRDs, custom controllers, and operators

### CRD
Adds a new resource type to the Kubernetes API.

### Custom controller
Watches resources and reconciles state.

### Operator
A more advanced controller pattern for managing applications.

For CKA:
- know what they are
- hands-on depth is usually lower than networking/storage/troubleshooting

---

# ✅ Storage

## Storage basics

Container writable layers are ephemeral.

If the container is recreated, that writable layer is lost.

So persistent storage is needed for stateful apps.

---

## Volumes

Common volume types:
- `emptyDir`
- `hostPath`
- `configMap`
- `secret`

`hostPath` is easy for labs but not a good production default.

---

## CSI = Container Storage Interface

CSI standardizes how orchestrators like Kubernetes talk to storage systems.

Mental model:
- Kubernetes asks a CSI driver to create, attach, mount, resize, and delete storage
- the CSI driver talks to the storage backend

Examples:
- AWS EBS
- Azure Disk
- NetApp
- Portworx

> [!tip] Compare interfaces
> - CRI → runtime
> - CNI → networking
> - CSI → storage

---

## Persistent Volumes (PV)

A PV is cluster storage provided by an admin or dynamically provisioned.

Important fields:
- capacity
- access modes
- storage backend
- reclaim policy

Useful commands:

    kubectl get pv
    kubectl describe pv <pv-name>

Common access modes:
- `ReadWriteOnce`
- `ReadOnlyMany`
- `ReadWriteMany`

---

## Persistent Volume Claims (PVC)

A PVC is a storage request from a user/workload.

Binding depends on:
- requested size
- access modes
- storage class
- selectors / compatibility

Useful commands:

    kubectl get pvc
    kubectl describe pvc <claim-name>

Common states:
- `Pending`
- `Bound`

> [!warning] Common PVC failure causes
> - no matching PV
> - wrong `storageClassName`
> - access mode mismatch
> - requested size too large
> - dynamic provisioner missing

---

## Reclaim policy

Common PV reclaim policies:
- `Retain`
- `Delete`
- `Recycle` (deprecated)

Meaning:
- `Retain` → volume remains after claim deletion
- `Delete` → backing storage removed automatically

---

## StorageClass

StorageClass enables dynamic provisioning.

Important fields:
- `provisioner`
- `parameters`
- `reclaimPolicy`
- `volumeBindingMode`

Useful commands:

    kubectl get storageclass
    kubectl describe storageclass <name>

> [!important] Mental model
> PVC + StorageClass → provisioner creates PV dynamically

---

# ✅ Networking

## Network namespaces

Each Pod gets its own network namespace.

Containers within the same Pod share:
- IP
- loopback
- ports

That is why containers inside a Pod can talk over `localhost`.

---

## CNI = Container Network Interface

CNI is the plugin system used for Pod networking.

It is responsible for:
- assigning Pod IP addresses
- connecting Pods to the network
- wiring routes and interfaces

Common plugin locations:

    /opt/cni/bin
    /etc/cni/net.d/

Useful commands:

    ls /opt/cni/bin
    ls /etc/cni/net.d
    ip addr
    ip route

> [!important] Runtime relationship
> The container runtime invokes the CNI plugin when creating Pod networking.

### Common CNI implementations
- Calico
- Flannel
- Cilium
- Weave

> [!warning] Important distinction
> A CNI may provide Pod connectivity, but not every CNI enforces NetworkPolicies.

---

## Cluster networking basics

Pod CIDR and Service CIDR should not overlap.

Examples:
- Pod network: `10.244.0.0/16`
- Service range: `10.96.0.0/12`

Useful check ideas:
- Pod IPs from Pod CIDR
- Service IPs from Service CIDR
- routes between nodes for Pod traffic

---

## Service networking

Service networking is virtual; Services do not get their own real network interface the way Pods do.

Traffic flow:

Pod → Service ClusterIP → kube-proxy rules → backend Pod

Useful commands:

    kubectl get svc
    kubectl describe svc <name>
    kubectl get endpoints <name>
    kubectl get pods -o wide

If using iptables mode, kube-proxy creates NAT rules.

> [!tip] High-value debugging shortcut
> If a Service exists but does not work, check endpoints immediately.

---

## DNS in Kubernetes

CoreDNS usually runs in `kube-system` and provides service discovery.

Common service name forms:
- `web`
- `web.default`
- `web.default.svc`
- `web.default.svc.cluster.local`

Typical Pod `resolv.conf` search domains include:
- `<namespace>.svc.cluster.local`
- `svc.cluster.local`
- `cluster.local`

Useful commands:

    kubectl get pods -n kube-system
    kubectl get svc -n kube-system
    kubectl exec -it <pod> -- cat /etc/resolv.conf
    kubectl exec -it <pod> -- nslookup kubernetes
    kubectl exec -it <pod> -- nslookup myservice.mynamespace.svc.cluster.local

> [!note] CoreDNS service name
> The CoreDNS service is commonly still named `kube-dns` for compatibility, even when the implementation is CoreDNS.

---

## Ingress

Ingress provides HTTP/HTTPS routing to Services based on:
- host
- path
- TLS

Important point:
- an Ingress resource alone does nothing
- you need an **Ingress Controller**

Useful commands:

    kubectl get ingress
    kubectl describe ingress <name>

---

## Gateway API

Gateway API is a newer, more expressive networking API than classic Ingress.

For CKA:
- understand the concept
- Ingress is still more foundational

---

# ✅ Troubleshooting

> [!success] Default exam approach
> Never guess first.  
> Always inspect first.

## Pod troubleshooting flow

Use these commands in order:

    kubectl get pods
    kubectl describe pod <pod>
    kubectl logs <pod>
    kubectl logs <pod> --previous
    kubectl exec -it <pod> -- sh

Common problems:
- `CrashLoopBackOff`
- `ImagePullBackOff`
- `ErrImagePull`
- bad command / args
- readiness probe failures
- missing env vars
- bad Secret / ConfigMap reference
- scheduling failure

---

## Application failure flow

If the app is unreachable:
1. check Pod status
2. check Service
3. check endpoints
4. test DNS
5. test from inside another Pod
6. inspect container logs

Useful commands:

    kubectl describe svc web-service
    kubectl get endpoints web-service
    kubectl logs web-pod
    kubectl exec -it client-pod -- curl http://web-service

---

## Service failure flow

Use this every time:

    kubectl get svc
    kubectl describe svc <name>
    kubectl get endpoints <name>
    kubectl get pods --show-labels
    kubectl exec -it <pod> -- nslookup <service>

If endpoints are empty:
- selector mismatch
- Pods not Ready
- wrong namespace
- no matching Pods

---

## Node troubleshooting flow

Useful commands:

    kubectl get nodes
    kubectl describe node node01
    systemctl status kubelet
    journalctl -u kubelet

Look for conditions such as:
- `Ready`
- `MemoryPressure`
- `DiskPressure`
- `PIDPressure`
- `NetworkUnavailable`

> [!tip] Heartbeat clue
> In a node description, the last heartbeat / status update can tell you if the node stopped talking to the control plane.

---

## Control plane troubleshooting flow

On kubeadm clusters:

    ls /etc/kubernetes/manifests/
    kubectl get pods -n kube-system
    crictl ps
    crictl logs <container-id>
    journalctl -u kubelet

On non-static-pod setups, also check:

    systemctl status kube-apiserver
    systemctl status kube-controller-manager
    systemctl status kube-scheduler

Useful checks:
- is the API server up?
- are manifest files valid?
- are certs correct?
- is etcd healthy?
- is kubelet running?

---

## Worker node troubleshooting flow

Useful commands:

    kubectl get nodes
    kubectl describe node worker-1
    systemctl status kubelet
    journalctl -u kubelet
    openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -text -noout

---

# ✅ Design and Install a Kubernetes Cluster

## Choosing infrastructure

Common deployment options:
- bare metal
- self-managed VMs
- cloud IaaS
- managed Kubernetes

CKA focuses heavily on **kubeadm-style operational understanding**.

---

## kubeadm essentials

Useful kubeadm-related paths:
- `/etc/kubernetes/manifests/`
- `/etc/kubernetes/pki/`
- `/etc/kubernetes/admin.conf`

Useful commands:

    kubeadm init
    kubeadm token create --print-join-command
    kubeadm join <...>

---

## `etcd` in HA

### Quorum

Quorum formula:

(total nodes / 2) + 1

Examples:
- 3 nodes → quorum 2
- 5 nodes → quorum 3
- 7 nodes → quorum 4

### Why odd numbers?

Odd-sized clusters maximize fault tolerance per node count.

Examples:
- 3 nodes tolerate 1 failure
- 5 nodes tolerate 2 failures

> [!warning] Important
> Two-node `etcd` is not truly HA.  
> In practice, use at least 3 members.

### Raft

`etcd` uses the Raft consensus protocol:
- one leader handles writes
- followers replicate state
- a majority must acknowledge writes

---

# ✅ Helm Basics

Helm is Kubernetes package management.

Key concepts:
- **chart** → package template
- **release** → installed instance of a chart
- **values** → user-supplied config

Useful commands:

    helm install myapp chart
    helm upgrade myapp chart
    helm list
    helm uninstall myapp
    helm show values chart

> [!note] Helm 3
> Helm 3 does **not** use Tiller.

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

    kubectl apply -k .

Comparison:
- Helm → templating engine
- Kustomize → patch/overlay engine

---

# ✅ Exam Priority Guide

## Highest priority
- troubleshooting
- networking
- storage
- RBAC
- `etcd` backup/restore
- upgrades
- Services and endpoints
- static Pods / control plane inspection

## Medium priority
- HPA
- Ingress
- kubeadm operations
- scheduling controls
- logs and monitoring
- service accounts
- security contexts

## Lower priority but still know the concept
- CRDs
- operators
- Gateway API
- VPA
- advanced scheduler customization

---

# ✅ Final Rule

> [!success] If these are automatic, you are in strong shape
> - trace request flow through the cluster
> - debug Pods fast
> - diagnose Service / endpoint issues quickly
> - understand PVC binding failures
> - create and test RBAC quickly
> - drain / uncordon nodes safely
> - restore `etcd` confidently

---

# ⚡ Appendix A — Speed Drills

> [!tip] Goal
> Type these without thinking.

## Pods

    kubectl run nginx --image=nginx
    kubectl run nginx --image=nginx --dry-run=client -o yaml
    kubectl exec -it nginx -- sh
    kubectl delete pod nginx

## Deployments

    kubectl create deployment web --image=nginx
    kubectl scale deployment web --replicas=3
    kubectl expose deployment web --port=80 --type=NodePort
    kubectl rollout status deployment web

## YAML generation

    kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
    kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
    kubectl expose deployment web --port=80 --dry-run=client -o yaml > svc.yaml

## Namespaces

    kubectl create ns dev
    kubectl run nginx --image=nginx -n dev
    kubectl config set-context --current --namespace=dev

## Labels

    kubectl label pod nginx env=prod
    kubectl get pods -l env=prod
    kubectl get pods --show-labels

## Logs

    kubectl logs pod
    kubectl logs -f pod
    kubectl logs pod --previous
    kubectl logs pod -c container

## Debugging

    kubectl describe pod pod
    kubectl get events --sort-by=.metadata.creationTimestamp

## Nodes

    kubectl cordon node01
    kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
    kubectl uncordon node01

## RBAC

    kubectl create role reader --verb=get,list --resource=pods
    kubectl create rolebinding read-pods --role=reader --user=user1
    kubectl auth can-i get pods --as=user1

## Secrets and ConfigMaps

    kubectl create secret generic mysecret --from-literal=password=123
    kubectl get secret mysecret -o yaml
    kubectl create configmap myconfig --from-literal=key=value

## Service debugging

    kubectl get svc
    kubectl describe svc myservice
    kubectl get endpoints myservice

## Networking tests

    kubectl exec -it pod -- nslookup kubernetes
    kubectl exec -it pod -- curl http://myservice

## Storage

    kubectl get pv
    kubectl get pvc
    kubectl describe pvc myclaim
    kubectl get storageclass

## `etcd`

    ETCDCTL_API=3 etcdctl snapshot save backup.db
    ETCDCTL_API=3 etcdctl snapshot restore backup.db --data-dir /var/lib/etcd-from-backup

## `crictl`

    crictl ps
    crictl ps -a
    crictl logs <id>

## kubectl shortcuts

    alias k=kubectl

---

## 30-second drills checklist

- [ ] create Pod YAML
- [ ] create Deployment YAML
- [ ] scale a Deployment
- [ ] read logs from a crashing Pod
- [ ] check why a Pod is Pending
- [ ] fix a Service with empty endpoints
- [ ] drain and uncordon a node
- [ ] create a Role and RoleBinding
- [ ] back up `etcd`
- [ ] find control plane logs with `crictl`

---

# 📅 Appendix B — 1-Week Crash Plan

> [!info] Core rule
> Spend **70–80%** of your remaining time in labs, not notes.

## Day 1 — Core + kubectl speed

### Goal
Get fast with:
- Pods
- Deployments
- Services
- namespaces
- dry-run YAML

### Practice
- create resources imperatively
- convert them to YAML
- edit and apply manifests
- use `describe`, `logs`, `exec`

### End-of-day target
- create Pod YAML in under 10 seconds
- scale a Deployment without pausing
- know your basic debug flow by memory

---

## Day 2 — Scheduling

### Goal
Understand why Pods do or do not schedule.

### Practice
- labels/selectors
- taints/tolerations
- node selectors
- node affinity
- DaemonSets
- static Pods

### Lab ideas
- force a Pod onto a node
- fix a `Pending` Pod
- inspect static Pod manifests

---

## Day 3 — Networking

### Goal
Get fast at Service and DNS troubleshooting.

### Practice
- Services
- endpoints
- kube-proxy flow
- CoreDNS
- CNI basics

### Lab ideas
- break/fix Service selectors
- test connectivity between Pods
- run DNS lookups from inside Pods

> [!important] Must know
> If networking is broken, check `endpoints` early.

---

## Day 4 — Storage

### Goal
Understand the PV/PVC/StorageClass lifecycle.

### Practice
- volumes
- PVs
- PVCs
- StorageClasses
- CSI concept

### Lab ideas
- create a PVC
- bind a PV
- diagnose a `Pending` PVC

---

## Day 5 — Security + RBAC

### Goal
Fix access issues quickly.

### Practice
- Roles / RoleBindings
- ClusterRoles / ClusterRoleBindings
- service accounts
- Secrets
- security contexts
- NetworkPolicies

### Lab ideas
- create a user permission set
- test it with `kubectl auth can-i`
- assign a service account to a Pod

---

## Day 6 — Troubleshooting + `etcd`

### Goal
Be strong on the highest-value admin tasks.

### Practice
- `etcd` backup/restore
- control plane inspection
- node failures
- kubelet logs
- `crictl`

### Lab ideas
- snapshot and restore `etcd`
- inspect static Pods
- debug a NotReady node

---

## Day 7 — Mock exam

### Goal
Simulate real pressure.

### Do
- one full mock exam
- strict timing
- skip hard questions early, come back later

### Review
- where did you hesitate?
- which commands were not automatic?
- which troubleshooting flow broke down?

---

## Suggested daily rhythm

1. 20–30 min notes review
2. 2–4 hours labs
3. 20–30 min speed drills
4. short review of mistakes

---

# 🧠 Appendix C — Default Debug Flows

## Pod broken

    kubectl get pods
    kubectl describe pod <pod>
    kubectl logs <pod>
    kubectl logs <pod> --previous

## Service broken

    kubectl get svc
    kubectl describe svc <name>
    kubectl get endpoints <name>
    kubectl get pods --show-labels

## DNS broken

    kubectl exec -it <pod> -- cat /etc/resolv.conf
    kubectl exec -it <pod> -- nslookup kubernetes
    kubectl get pods -n kube-system
    kubectl get svc -n kube-system

## Node broken

    kubectl get nodes
    kubectl describe node <node>
    systemctl status kubelet
    journalctl -u kubelet

## Control plane broken

    ls /etc/kubernetes/manifests/
    crictl ps
    crictl logs <id>
    journalctl -u kubelet

## Storage broken

    kubectl get pvc
    kubectl describe pvc <claim>
    kubectl get pv
    kubectl get storageclass

---

# 🏁 Final Reminder

> [!success] You do not need infinite theory.
> You need:
> - enough theory to recognize the failure pattern
> - enough command fluency to prove it
> - enough repetition to fix it quickly

If the concepts in this guide make sense, the next step is not more note-taking.

It is:
- labs
- mocks
- speed drills
- more labs
