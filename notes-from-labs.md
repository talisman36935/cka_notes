# CKA Lab Notes

> [!abstract] Flagged Weaknesses — Prioritise These
> - **Volumes & VolumeMounts** — binding pattern, volume types, PVC
> - **Security Contexts** — pod vs container level, capabilities
> - **Selectors** — Services vs Deployments vs NetworkPolicy
> - **Network Policies** — YAML from scratch, AND vs OR selectors
> - **Certificate / PKI paths** — know the file locations
> - **Projected Volumes** — combining multiple sources
> - **Ingress vs Gateway API** — conceptual difference, TLS termination
> - **JSONPath / custom-columns** — exam staple
> - **Kustomize patches** — JSON6902 vs Strategic Merge
> - **kubectl exec** — syntax differences from Docker

---

## Workloads & Scheduling

### Static Pods

Static pods are managed directly by the kubelet on a node — not by the control plane. They live in a manifest directory on the node and always have the node name as a suffix (e.g. `etcd-controlplane`). You can't delete them with `kubectl` — you delete the manifest file on the node.

```bash
# Find the static pod manifest path on a node
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Create a static pod manifest
kubectl run static-busybox --restart=Never --image=busybox \
  --dry-run=client -o yaml --command -- sleep 1000 \
  > /etc/kubernetes/manifests/static-busybox.yaml

# To delete: SSH to the node and remove the file
kubectl get nodes -o wide   # find node IP
ssh node01
rm /etc/kubernetes/manifests/static-busybox.yaml
```

---

### Priority Classes

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "High priority workloads"
```

```bash
# Compare priority classes across pods
kubectl get pods -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priorityClassName"
```

---

### Admission Controllers

| Controller | Type |
|---|---|
| `NamespaceAutoProvision` | Mutating |
| `NamespaceExists` | Validating |

Mutating webhooks run first, then validating webhooks.

---

### HPA & VPA

```bash
# Create an HPA
kubectl autoscale deployment nginx-deployment --max=3 --cpu-percent=80

# Generate manifest
kubectl autoscale deployment nginx-deployment --max=3 --cpu-percent=80 \
  --dry-run=client -o yaml > hpa.yaml

# Watch HPA status
kubectl get hpa --watch

# Debug metric issues
kubectl events hpa nginx-deployment | grep -i "FailedGetResourceMetric"

# Check VPA updater logs
kubectl logs \
  $(kubectl get pods -n kube-system --no-headers \
    -o custom-columns=":metadata.name" | grep vpa-updater) \
  -n kube-system
```

---

### Init Containers & Sidecar Containers

An **init container** runs to completion before the main container starts. A **sidecar** (with `restartPolicy: Always`) runs alongside the main container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: elastic-stack
spec:
  containers:
  - name: app
    image: kodekloud/event-simulator
    volumeMounts:
    - mountPath: /log
      name: log-volume
  initContainers:
  - name: sidecar
    image: kodekloud/filebeat-configured
    restartPolicy: Always      # makes this a sidecar, not a one-shot init
    volumeMounts:
    - mountPath: /var/log/event-simulator/
      name: log-volume
  volumes:
  - name: log-volume
    hostPath:
      path: /var/log/webapp
      type: DirectoryOrCreate
```

---

## Configuration

### Args vs Commands

| Dockerfile | Kubernetes field | Effect |
|---|---|---|
| `ENTRYPOINT` | `command` | Overrides the container entrypoint |
| `CMD` | `args` | Arguments passed to the entrypoint |

```bash
# Pass args at the command line — everything after -- goes to the container
kubectl run webapp-green --image=kodekloud/webapp-color -- --color=green
```

```yaml
spec:
  containers:
  - name: app
    image: kodekloud/webapp-color
    args: ["--color", "green"]    # use args (not command) when only overriding CMD
```

---

### ConfigMaps

```bash
# Imperative — from literals
kubectl create configmap webapp-config-map \
  --from-literal=APP_COLOR=darkblue \
  --from-literal=APP_OTHER=disregard

# Imperative — from file
kubectl create configmap app-config --from-file=config.properties
```

**Single key as env var:**
```yaml
env:
- name: APP_COLOR
  valueFrom:
    configMapKeyRef:
      name: webapp-config-map
      key: APP_COLOR
```

**All keys as env vars:**
```yaml
envFrom:
- configMapRef:
    name: webapp-config-map
```

**Mounted as files (each key becomes a file):**
```yaml
volumes:
- name: config-vol
  configMap:
    name: webapp-config-map
volumeMounts:
- name: config-vol
  mountPath: /etc/config
```

---

### Secrets

```bash
# Imperative
kubectl create secret generic db-secret \
  --from-literal=DB_Host=sql01 \
  --from-literal=DB_User=root \
  --from-literal=DB_Password=password123

# Check available options
kubectl create secret --help
```

> [!warning] If writing a Secret in YAML manually, values must be base64-encoded.
> ```bash
> echo -n "password123" | base64    # -n avoids a trailing newline character
> ```

**Single key as env var:**
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: DB_Password
```

---

## Volumes & Storage

> [!warning] Flagged as #1 weakness — study this section carefully

### Mental Model

Volumes are declared in two separate places in a pod spec. The `name` field is the glue between them.

```
spec.volumes[]                    ← declares the storage SOURCE
  └── name: my-vol                ← the binding key
  └── <type>: ...                 ← the actual storage backend

spec.containers[].volumeMounts[]  ← mounts the declared volume INTO the container
  └── name: my-vol                ← must exactly match spec.volumes[].name
  └── mountPath: /app/data        ← path inside the container
  └── readOnly: true              ← optional
```

---

### Volume Types Quick Reference

| Type | Use Case | Notes |
|---|---|---|
| `hostPath` | Node-local directory | Ephemeral, not for prod |
| `emptyDir` | Scratch space shared between containers | Lives only for pod lifetime |
| `configMap` | Mount ConfigMap keys as files | Auto-updates when CM changes |
| `secret` | Mount Secret keys as files | Values are base64-decoded automatically |
| `persistentVolumeClaim` | Persistent storage across pod restarts | References a PVC by name |
| `projected` | Combine multiple sources into one mount | e.g. SA token + configMap + secret |

---

### hostPath

```yaml
volumes:
- name: log-volume
  hostPath:
    path: /var/log/webapp    # path on the node
    type: Directory          # must already exist
```

| `type` value | Behaviour |
|---|---|
| `Directory` | Must already exist |
| `DirectoryOrCreate` | Created automatically if missing |
| `File` | Must already exist |
| `FileOrCreate` | Created automatically if missing |
| `""` (empty) | No checks performed |

---

### emptyDir

```yaml
volumes:
- name: shared-data
  emptyDir: {}    # created when pod starts, deleted when pod stops
```

Useful for sharing files between containers in the same pod (e.g. app + sidecar).

---

### Projected Volumes

A projected volume combines multiple sources (SA token, ConfigMap, Secret) into a single mount point. Useful when `automountServiceAccountToken: false` is set on the ServiceAccount and you want manual control.

```yaml
volumes:
- name: token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3607
    - configMap:
        name: app-config
        items:
        - key: config.yaml
          path: config.yaml
    - secret:
        name: app-secret
```

---

### PersistentVolumes & PersistentVolumeClaims

> [!note] No imperative commands exist for PV or PVC — you must write YAML.

**PersistentVolume** (PV) — the actual storage, cluster-scoped:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-log
spec:
  persistentVolumeReclaimPolicy: Retain    # Retain | Delete | Recycle
  accessModes:
  - ReadWriteMany                           # must match the PVC
  capacity:
    storage: 100Mi
  hostPath:
    path: /pv/log
```

**PersistentVolumeClaim** (PVC) — a request for storage, namespace-scoped:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-log-1
spec:
  accessModes:
  - ReadWriteMany    # must match the PV
  resources:
    requests:
      storage: 50Mi
```

**Referencing a PVC in a Pod:**
```yaml
volumes:
- name: log-volume
  persistentVolumeClaim:
    claimName: claim-log-1
```

#### Access Modes

| Mode | Short | Description |
|---|---|---|
| `ReadWriteOnce` | RWO | Read/write by one node at a time |
| `ReadOnlyMany` | ROX | Read-only by many nodes |
| `ReadWriteMany` | RWX | Read/write by many nodes |

> [!warning] Common binding failures
> - `accessModes` mismatch between PV and PVC → PVC stays `Pending`
> - No StorageClass set and no matching static PV → `FailedBinding` event
> - PVC stuck on `Terminating` → a pod is still using it; delete the pod first

---

### StorageClasses

```bash
kubectl get sc    # look for provisioner = kubernetes.io/no-provisioner (no dynamic provisioning)
```

> [!note] `WaitForFirstConsumer` binding mode delays PV creation until a Pod using the PVC is scheduled. Common with the `local-path` StorageClass.

---

## Security

### Security Contexts

> [!warning] No notes here previously — fill this gap

Security contexts control UID/GID and Linux capabilities for pods and containers.

**Pod-level** (applies to all containers):
```yaml
spec:
  securityContext:
    runAsUser: 1000       # UID to run all containers as
    runAsGroup: 3000      # primary GID
    fsGroup: 2000         # GID applied to mounted volumes (ownership)
    runAsNonRoot: true    # fails if container tries to run as root
```

**Container-level** (overrides pod-level for that container):
```yaml
spec:
  containers:
  - name: app
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
        drop: ["ALL"]
```

> [!important] Key rules
> - `capabilities` exists at **container level only** — not pod level
> - `fsGroup` exists at **pod level only** — sets GID for volume ownership
> - Container-level settings override pod-level settings

---

### RBAC

**Imperative (fastest in exam):**
```bash
# Role + RoleBinding (namespace-scoped)
kubectl create role developer --namespace=default \
  --verb=list,create,delete --resource=pods

kubectl create rolebinding dev-user-binding --namespace=default \
  --role=developer --user=dev-user

# ClusterRole + ClusterRoleBinding (cluster-wide)
kubectl create clusterrole cluster-reader \
  --verb=get,list --resource=storageclasses

kubectl create clusterrolebinding cluster-reader-binding \
  --clusterrole=cluster-reader --user=michelle

# Test permissions
kubectl auth can-i list pods --as dev-user
kubectl auth can-i list storageclasses --as michelle
kubectl get pods --as dev-user
```

**YAML:**
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: developer
rules:
- apiGroups: [""]           # "" = core API group (pods, services, configmaps, etc.)
  resources: ["pods"]
  verbs: ["list", "create", "delete"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dev-user-binding
  namespace: default
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

> [!tip] ClusterRoles are cluster-wide and not namespaced. Use them for non-namespaced resources (nodes, PVs, storageclasses) or to grant access across all namespaces at once.

```bash
# Count all cluster roles
kubectl get clusterroles --no-headers | wc -l
kubectl get clusterroles --no-headers -o json | jq '.items | length'

# Inspect a role binding
kubectl describe rolebinding kube-proxy -n kube-system
```

---

### Service Accounts

```bash
# Set a service account on a deployment (live update, no yaml edit needed)
kubectl set serviceaccount deploy/web-dashboard dashboard-sa

# Disable automatic token mounting
kubectl patch sa dashboard-sa -p '{"automountServiceAccountToken": false}'
```

**Manual projected token (when automount is disabled):**
```yaml
spec:
  serviceAccountName: dashboard-sa
  automountServiceAccountToken: false
  containers:
  - name: web-dashboard
    image: gcr.io/kodekloud/customimage/my-kubernetes-dashboard
    ports:
    - containerPort: 8080
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount/
      name: token
      readOnly: true
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
```

---

### Certificates & PKI Paths

> [!warning] Flagged weakness — know these paths

```
/etc/kubernetes/pki/
  ca.crt / ca.key                      # Cluster CA
  apiserver.crt / apiserver.key        # API server serving cert
  apiserver-kubelet-client.crt/.key    # API server → kubelet auth
  apiserver-etcd-client.crt/.key       # API server → etcd auth

/etc/kubernetes/pki/etcd/
  ca.crt / ca.key                      # etcd CA (separate from cluster CA)
  server.crt / server.key              # etcd server cert
  peer.crt / peer.key                  # etcd peer-to-peer

/var/lib/kubelet/pki/
  kubelet.crt                          # kubelet serving cert
```

```bash
# Inspect a certificate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout

# Focused inspection
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout \
  | grep -A2 "Subject\|Issuer\|Not After"

# When kube-apiserver is down, use crictl instead of kubectl
crictl ps -a
crictl logs <container-id>
```

---

### imagePullSecrets

Used to pull images from a private container registry.

```bash
# Create the registry credential secret
kubectl create secret docker-registry private-reg-cred \
  --docker-server=<registry-url> \
  --docker-username=<user> \
  --docker-password=<password> \
  --docker-email=<email>
```

```yaml
# Reference it in a Deployment under spec.template.spec
spec:
  template:
    spec:
      imagePullSecrets:
      - name: private-reg-cred
      containers:
      - name: app
        image: private-registry.io/myapp:latest
```

---

## Networking

### Selectors

Three different selector contexts in Kubernetes — the syntax differs:

**Service** → selects Pods with a flat label map:
```yaml
spec:
  selector:
    app: myapp        # flat map, no matchLabels
```

**Deployment** → declares which pods it owns (must match `template.metadata.labels`):
```yaml
spec:
  selector:
    matchLabels:      # matchLabels / matchExpressions syntax
      app: myapp
  template:
    metadata:
      labels:
        app: myapp    # must match the selector above
```

**NetworkPolicy** → selects which pods the policy applies to:
```yaml
spec:
  podSelector:
    matchLabels:
      role: db
```

> [!warning] Services use a flat `selector:` map. Deployments/ReplicaSets use `selector.matchLabels:`. Mixing them up is a common mistake.

---

### Network Policies

> [!warning] Flagged weakness — practice building from scratch, no imperative command exists

**Full skeleton:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
  namespace: default
spec:
  podSelector:         # which pods this policy applies TO
    matchLabels:
      name: internal
  policyTypes:
  - Ingress            # only list the directions you want to restrict
  - Egress
  ingress:
  - from:              # whitelist of allowed sources
    - podSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 3306
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
    - protocol: TCP
      port: 3306
  - to:
    - podSelector:
        matchLabels:
          name: payroll
    ports:
    - protocol: TCP
      port: 8080
  - ports:             # always include DNS egress — name resolution will break without it
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

#### AND vs OR Selector Trap

This is the most common mistake in network policy YAML:

```yaml
# AND — pod must match BOTH conditions (same list item, no leading dash on second)
ingress:
- from:
  - podSelector:
      matchLabels: {role: frontend}
    namespaceSelector:             # no leading dash here → same item → AND
      matchLabels: {env: prod}

# OR — pod matches EITHER condition (separate list items, each with a leading dash)
ingress:
- from:
  - podSelector:
      matchLabels: {role: frontend}
  - namespaceSelector:             # leading dash here → new item → OR
      matchLabels: {env: prod}
```

#### Allow All Ingress

```yaml
ingress:
- {}    # empty rule = allow all inbound traffic
```

---

### CNI

```bash
# Find the active CNI plugin
ls /etc/cni/net.d/                         # filename reveals the plugin
cat /etc/cni/net.d/10-flannel.conflist     # check the "type" field

# CNI binary location
ls /opt/cni/bin

# Find Pod CIDR range (more reliable than reading canal logs)
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep cluster-cidr
```

> [!note] etcd ports
> - `2379` — client traffic (all control plane components connect here)
> - `2380` — peer-to-peer only (only relevant in multi-control-plane setups)

```bash
# Useful networking commands
ip route show default       # show default gateway
netstat -nplt               # listening ports and owning processes
netstat -anp                # all connections

# Find kube-proxy mode
kubectl logs -n kube-system kube-proxy-<id>
```

---

### Ingress

> [!note] No imperative command for Ingress — YAML only

**Basic Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: webapp
spec:
  ingressClassName: nginx
  rules:
  - host: app.kodekloud.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app
            port:
              number: 80
```

**With TLS termination** (add the `tls` section):
```yaml
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.kodekloud.local
    secretName: app-tls       # references a Secret of type kubernetes.io/tls
  rules:
  - host: app.kodekloud.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app
            port:
              number: 80
```

**Force HTTP → HTTPS redirect:**
```bash
kubectl annotate ingress web-app-ingress \
  nginx.ingress.kubernetes.io/ssl-redirect="true"
```

---

### Gateway API

> [!warning] Flagged weakness — understand the component relationships

| Concept | Ingress Equivalent | Role |
|---|---|---|
| `GatewayClass` | `IngressClass` | Blueprint/template — defines the controller |
| `Gateway` | Ingress Controller instance | Deployed listener that references a GatewayClass |
| `HTTPRoute` | `Ingress` resource | Routing rules, attaches to a Gateway |

- **GatewayClass** = the template (e.g. nginx, istio). Controller-specific behaviour.
- **Gateway** = the deployed instance using that template. Has `listeners` (ports/protocols).
- **HTTPRoute** = attaches to a Gateway via `parentRefs`, defines which traffic goes where.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway         # which Gateway to attach to
    namespace: nginx-gateway
    sectionName: http           # which listener on the Gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-svc
      port: 80
```

**TLS termination with Gateway API** — config lives on the Gateway, not the Route:
```yaml
# Gateway listener
listeners:
- name: https
  port: 443
  protocol: HTTPS
  tls:
    mode: Terminate
    certificateRefs:
    - name: app-tls    # Secret name

# HTTPRoute just references the gateway — no TLS config needed here
```

> [!note]
> - `allowedRoutes` on a Gateway controls which namespaces can attach routes to it
> - Supported protocols: HTTP, HTTPS, TCP, UDP, TLS — **ICMP is not supported**

---

## Cluster Management

### kubeconfig & Context

```bash
# Set active context (specific kubeconfig file)
kubectl config --kubeconfig=/root/my-kube-config use-context research

# View current context
kubectl config --kubeconfig=/root/my-kube-config current-context

# Set a custom kubeconfig as the default
echo 'export KUBECONFIG=$HOME/my-kube-config' >> ~/.bashrc
source ~/.bashrc
```

---

### Cluster Upgrade

> [!tip]
> - `apt-cache madison kubeadm` — lists all available package versions in the configured repo
> - `apt-mark hold` — pins packages so `apt upgrade` won't auto-update them (critical for k8s)
> - `apt-mark unhold` — unpin before upgrading, then re-hold afterward

**Step 1: Update the apt repo URL to the new minor version**
```bash
vim /etc/apt/sources.list.d/kubernetes.list
# Change v1.34 → v1.35 in the URL, save and exit
```

**Step 2: Drain the node**
```bash
kubectl drain controlplane --ignore-daemonsets
```

**Step 3: Upgrade kubeadm**
```bash
apt-get update
apt-cache madison kubeadm              # find exact version string, e.g. 1.35.0-1.1
apt-get install kubeadm=1.35.0-1.1
```

**Step 4: Apply the upgrade**
```bash
kubeadm upgrade plan v1.35.0
kubeadm upgrade apply v1.35.0
```

**Step 5: Upgrade kubelet, reload, uncordon**
```bash
apt-get install kubelet=1.35.0-1.1
systemctl daemon-reload
systemctl restart kubelet
kubectl uncordon controlplane
```

> [!tip] For worker nodes: drain from controlplane → SSH to worker → upgrade kubelet → reload → uncordon from controlplane.

---

### etcd Backup & Restore

**Backup:**
```bash
etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/snapshot-pre-boot.db
```

**Restore:**

```bash
# 1. Stop kube-apiserver (prevents writes during restore)
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 30

# 2. Restore snapshot to a new directory
etcdutl snapshot restore /opt/snapshot-pre-boot.db \
  --data-dir /var/lib/etcd-from-backup

# 3. Update etcd.yaml to point at the restored directory
vim /etc/kubernetes/manifests/etcd.yaml
# In the volumes section, change:
#   path: /var/lib/etcd
# to:
#   path: /var/lib/etcd-from-backup
# etcd pod restarts automatically after saving (static pod behaviour)

# 4. Bring kube-apiserver back
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 5. Optionally restart controller-manager and scheduler the same way
#    then restart kubelet
systemctl restart kubelet

# 6. Monitor until all pods are Running
watch crictl ps
```

> [!warning] Use `etcdutl` (not `etcdctl`) for the restore command in newer versions.

---

### Node Management

```bash
kubectl drain controlplane --ignore-daemonsets    # evict pods + mark unschedulable
kubectl cordon node01                             # mark unschedulable only (no eviction)
kubectl uncordon controlplane                     # mark schedulable again
```

---

### Cluster Installation (kubeadm)

**Run on both nodes — prerequisites:**
```bash
# Enable IPv4 packet forwarding
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward=1
EOF
sudo sysctl --system

# Install kubeadm, kubelet, kubectl
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-cache madison kubeadm    # verify available versions

sudo apt-get install -y kubelet=1.35.0-1.1 kubeadm=1.35.0-1.1 kubectl=1.35.0-1.1
sudo apt-mark hold kubelet kubeadm kubectl    # prevent accidental upgrades
```

**Control plane only — initialise the cluster:**
```bash
kubeadm init \
  --apiserver-advertise-address <eth0-IP> \
  --apiserver-cert-extra-sans controlplane \
  --pod-network-cidr 172.17.0.0/16 \
  --service-cidr 172.20.0.0/16

# Set up kubeconfig for the current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Deploy Flannel CNI (with custom PodCIDR):**
```bash
curl -LO https://raw.githubusercontent.com/flannel-io/flannel/v0.20.2/Documentation/kube-flannel.yml
```

Edit `kube-flannel.yml` before applying:
1. Find the `kube-flannel-cfg` ConfigMap → update `"Network"` in `net-conf.json` to match `--pod-network-cidr` (e.g. `172.17.0.0/16`)
2. Find the flannel container args → add `- --iface=eth0`

```bash
kubectl apply -f kube-flannel.yml
watch kubectl get pods -A    # wait for all Running
```

---

## Observability & Debugging

### kubectl exec

> [!tip] Coming from Docker: `docker exec -it <container> bash` → `kubectl exec -it <pod> -- bash`

```bash
# Run a single command
kubectl exec <pod> -- cat /log/app.log

# Interactive shell
kubectl exec -it <pod> -- /bin/bash

# Specify container in a multi-container pod
kubectl exec -it <pod> -c <container-name> -- /bin/sh

# Different namespace
kubectl exec -it <pod> -n <namespace> -- /bin/sh

# Real examples from labs
kubectl exec webapp -- cat /log/app.log
kubectl -n elastic-stack exec -it app -- cat /log/app.log
```

The `--` separator tells kubectl that everything after it is the container command. You can drop it if the command has no flags that could be confused with kubectl flags.

---

### Logs & crictl

```bash
# Pod logs
kubectl logs <pod>
kubectl logs <pod> -c <container>    # multi-container pod
kubectl logs <pod> --previous        # logs from the previous container instance

# When kube-apiserver is down, use crictl
crictl ps -a
crictl logs <container-id>

# Broad resource views
kubectl get all -A
kubectl get deployments,services --all-namespaces
```

---

### kubectl replace

```bash
# Force replace (deletes then recreates) — useful for immutable fields
kubectl replace -f webapp.yaml --force
```

---

## kubectl Advanced

### JSONPath Queries

> [!warning] Flagged weakness — JSONPath appears in almost every mock exam

```bash
# Single field across all items
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'

# Multiple fields with formatting
kubectl get pods \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# All user names from a kubeconfig file
kubectl config view --kubeconfig=my-kube-config \
  -o jsonpath="{.users[*].name}"

# Conditional — find context for a specific user
kubectl config view --kubeconfig=my-kube-config \
  -o jsonpath="{.contexts[?(@.context.user=='aws-user')].name}"
```

| JSONPath syntax | Meaning |
|---|---|
| `.items[*]` | All items in a list |
| `{range .items[*]}...{end}` | Loop over items |
| `?(@.field=='value')` | Filter condition |
| `{"\n"}` / `{"\t"}` | Newline / tab formatting |

---

### Custom Columns & sort-by

```bash
# Custom columns — NAME:.path.to.field
kubectl get pods -o custom-columns="NAME:.metadata.name,NODE:.spec.nodeName"
kubectl get pods -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priorityClassName"

# Sort by a field
kubectl get pv --sort-by=.spec.capacity.storage

# Combine sort + custom columns (pipe to file)
kubectl get pv \
  --sort-by=.spec.capacity.storage \
  -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage \
  > /opt/outputs/pv-and-capacity-sorted.txt

# Count resources
kubectl get clusterroles --no-headers | wc -l
kubectl get clusterroles --no-headers -o json | jq '.items | length'
```

---

## Package Management

### Helm

```bash
# Install Helm
curl -fsSL -o get_helm.sh \
  https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh && ./get_helm.sh

# Repo management
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo list
helm repo update

# Search
helm search hub wordpress        # search Artifact Hub
helm search repo nginx           # search locally added repos

# Install a chart
helm install amaze-surf bitnami/apache

# Upgrade (pin to a specific chart version)
helm upgrade dazzling-web bitnami/nginx --version 18.3.6

# List installed releases
helm list
helm list -A    # all namespaces
```

---

### Kustomize

> [!tip] Use `-k` flag with kubectl to apply kustomize: `kubectl apply -k <dir>`

**kustomization.yaml structure:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- db/db-config.yaml
- db/db-depl.yaml
- nginx/nginx-depl.yaml

patches:
- path: my-patch.yaml
```

```bash
# Apply
kubectl apply -k /path/to/dir

# Alternative (build first, then pipe)
kustomize build k8s/ | kubectl apply -f -
```

---

### Kustomize Patches: JSON6902 vs Strategic Merge

> [!warning] Flagged weakness

**JSON6902** — surgical operations using a JSON Patch path:
```yaml
patches:
- target:
    kind: Deployment
    name: api-deployment
  patch: |-
    - op: replace          # replace | add | remove
      path: /spec/template/spec/containers/0/image
      value: caddy
    - op: add
      path: /spec/template/spec/containers/0/env/-    # - suffix appends to array
      value:
        name: FOO
        value: bar
```

**Strategic Merge Patch** — partial YAML; Kubernetes merges it using `name` as the key for lists:
```yaml
# api-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  template:
    spec:
      containers:
      - name: api          # "name" is the merge key — targets this specific container
        env:
        - name: REDIS_CONNECTION
          value: redis-service
```

Reference it in `kustomization.yaml`:
```yaml
patches:
- path: api-patch.yaml
```

| | JSON6902 | Strategic Merge |
|---|---|---|
| Syntax | JSON Patch operations | Partial YAML |
| Target arrays by | Index `[0]` | Name (merge key) |
| Best for | Precise single-field replacements | Adding/updating named resources |

> [!warning] Components vs Overlays
> A `kind: Component` cannot be applied standalone — it must be included in an overlay.
> Don't add `../../base/` inside a Component already used in an overlay — it causes a **duplicate resource error** during apply.
