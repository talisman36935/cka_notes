
> [!tip] How to use these notes Each section covers an exam scenario type. For each one you'll find: the **scenario pattern**, **key assumptions to make**, **step-by-step approach**, and the **critical commands/YAML**.

---

## Tags

`#CKA` `#kubernetes` `#exam-prep`

---

## Table of Contents

- [[#Pods & Multi-Container Pods]]
- [[#Deployments]]
- [[#Services & Networking]]
- [[#Storage - PV, PVC, StorageClass]]
- [[#Autoscaling - HPA & VPA]]
- [[#RBAC - Users, Roles & ServiceAccounts]]
- [[#NetworkPolicy]]
- [[#Node Management - Taints, Draining & Upgrades]]
- [[#Static Pods]]
- [[#DNS & Service Discovery]]
- [[#ConfigMaps & Secrets]]
- [[#Ingress]]
- [[#Gateway API]]
- [[#Helm]]
- [[#etcd Backup]]
- [[#Cluster Troubleshooting]]
- [[#Kubeconfig Troubleshooting]]
- [[#System-Level Prep (kubeadm)]]
- [[#CRI Runtime Installation]]
- [[#CRDs & Custom Resources]]
- [[#Priority Classes]]

---

## Pods & Multi-Container Pods

### Scenario Pattern

> Create a pod with multiple containers sharing a volume, where containers have specific commands or environment variables.

### Key Assumptions

- "Non-persistent volume" = `emptyDir: {}` — this is the default shared, ephemeral volume
- "Log to stdout" = use `tail -f` on the log file
- "Node name env var" = use the Downward API (`fieldRef: fieldPath: spec.nodeName`)
- Sidecar containers that continuously run alongside main container = regular containers (not initContainers), unless told otherwise
- "Sidecar that tails logs" = use `tail -f /path/to/file` as the command

### Step-by-Step Approach

1. Identify how many containers and their roles
2. Determine which containers need the shared volume (not always all of them)
3. Identify any special env vars (Downward API, literal values)
4. Write the YAML with `volumes` at spec level, `volumeMounts` in each container that needs it
5. [ ] Apply with `kubectl apply -f`

### Critical YAML Template

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mc-pod
  namespace: mc-namespace
spec:
  containers:
    - name: mc-pod-1
      image: nginx:1-alpine
      env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName   # Downward API for node name

    - name: mc-pod-2
      image: busybox:1
      volumeMounts:
        - name: shared-volume
          mountPath: /var/log/shared
      command:
        - "sh"
        - "-c"
        - "while true; do date >> /var/log/shared/date.log; sleep 1; done"

    - name: mc-pod-3
      image: busybox:1
      command:
        - "sh"
        - "-c"
        - "tail -f /var/log/shared/date.log"
      volumeMounts:
        - name: shared-volume
          mountPath: /var/log/shared

  volumes:
    - name: shared-volume
      emptyDir: {}    # Non-persistent, lost on pod restart
```

### Sidecar Logger Pattern (Deployment)

> [!note] Sidecar as initContainer with `restartPolicy: Always` In newer Kubernetes (1.29+), sidecars can be defined as `initContainers` with `restartPolicy: Always`. This keeps them running alongside the main container.

```yaml
initContainers:
  - name: log-agent
    image: busybox
    command: ["sh", "-c", "touch /var/log/app/app.log; tail -f /var/log/app/app.log"]
    volumeMounts:
      - name: log-volume
        mountPath: /var/log/app
    restartPolicy: Always    # Makes it a "native sidecar"
```

### Verify

```bash
kubectl logs <pod-name> -c <container-name>
kubectl describe pod <pod-name>
```

---

## Deployments

### Scenario Patterns

- Create deployment with specific image + replicas
- Rolling update to new image version
- Annotate to record the change

### Key Assumptions

- "Use rolling update" = default strategy, just use `kubectl set image`
- `--record` flag is **deprecated** — use `kubectl annotate` instead
- "1 replica" unless told otherwise

### Step-by-Step

```bash
# Create deployment
kubectl create deployment nginx-deploy --image=nginx:1.16 --replicas=1

# Update image (rolling update is default)
kubectl set image deployment/nginx-deploy nginx=nginx:1.17

# Record the change (replaces deprecated --record)
kubectl annotate deployment nginx-deploy kubernetes.io/change-cause="Updated nginx image to 1.17"

# Check rollout history
kubectl rollout history deployment nginx-deploy

# Verify
kubectl get deployment nginx-deploy
```

### Scaling

```bash
kubectl scale deployment nginx-deploy --replicas=3
```

### Troubleshooting a Broken Deployment

```bash
kubectl describe deployment <name>
kubectl describe pod <pod-name>   # look at Events section
kubectl get events --sort-by=.lastTimestamp
```

> [!warning] initContainer typo scenario If a pod is stuck due to a bad initContainer command (e.g. `sleeeep` instead of `sleep`):
> 
> 1. `kubectl edit po <name>` → fix the typo → save
> 2. Because you can't edit a running pod spec this way, kubectl saves to `/tmp/kubectl-edit-xxxx.yaml`
> 3. Run: `kubectl replace -f /tmp/kubectl-edit-xxxx.yaml --force`

---

## Services & Networking

### Scenario Patterns

- Expose a pod as ClusterIP
- Expose a deployment as NodePort on a specific port

### Key Commands

```bash
# Expose a pod (ClusterIP)
kubectl expose pod messaging --port=6379 --name messaging-service

# Expose a deployment as NodePort (dry-run first, then edit nodePort)
kubectl expose deployment hr-web-app \
  --type=NodePort \
  --port=8080 \
  --name=hr-web-app-service \
  --dry-run=client -o yaml > hr-web-app-service.yaml
```

Then edit the YAML to add `nodePort`:

```yaml
ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30082    # Add this manually
```

```bash
kubectl apply -f hr-web-app-service.yaml
```

### Key Assumptions

- "Expose on port X on nodes" = NodePort type
- "Within the cluster" = ClusterIP type (default)
- `--port` = the port the service listens on
- `targetPort` defaults to same as `--port` unless specified

---

## Storage - PV, PVC, StorageClass

### PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-analytics
spec:
  capacity:
    storage: 100Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /pv/data-analytics
```

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # Makes it default
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### Troubleshooting PVC Not Binding

> [!warning] Most common reason: access mode mismatch between PV and PVC

```bash
kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name> -n <namespace>
```

**Fix**: Export PVC, delete it, fix `accessModes` to match PV, re-create

```bash
kubectl get pvc app-pvc -n storage-ns -o yaml > pvc.yaml
kubectl delete pvc -n storage-ns app-pvc
# edit pvc.yaml: fix accessModes
kubectl apply -f pvc.yaml
```

> [!important] Never modify the PV — only fix the PVC

### Access Modes Cheatsheet

|Mode|Short|Meaning|
|---|---|---|
|ReadWriteOnce|RWO|One node, read-write|
|ReadOnlyMany|ROX|Many nodes, read-only|
|ReadWriteMany|RWX|Many nodes, read-write|

---

## Autoscaling - HPA & VPA

### HPA - CPU Utilization

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kkapp-deploy
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # Prevents rapid scale-down
```

### HPA - Memory Utilization

```yaml
metrics:
- type: Resource
  resource:
    name: memory    # Change cpu -> memory
    target:
      type: Utilization
      averageUtilization: 65
```

### HPA - Custom Metric

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"
```

### VPA

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: analytics-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: analytics-deployment
  updatePolicy:
    updateMode: "Recreate"    # Options: Off, Initial, Recreate, Auto
```

### Key Assumptions for HPA/VPA

- `autoscaling/v2` is the current API version (not v1)
- Stabilization window question → `behavior.scaleDown.stabilizationWindowSeconds`
- Custom metrics use `type: Pods` with `AverageValue`
- VPA updateMode `Recreate` = evicts and recreates pods with new requests

---

## RBAC - Users, Roles & ServiceAccounts

### Create User Access (CSR flow)

```bash
# 1. Create CSR object (base64-encode the .csr file content)
cat /root/CKA/john.csr | base64 | tr -d '\n'

# 2. Apply CSR YAML (paste base64 output into request field)
# 3. Approve it
kubectl certificate approve john-developer

# 4. Create Role
kubectl create role developer \
  --resource=pods \
  --verb=create,list,get,update,delete \
  --namespace=development

# 5. Create RoleBinding
kubectl create rolebinding developer-role-binding \
  --role=developer \
  --user=john \
  --namespace=development

# 6. Verify
kubectl auth can-i update pods --as=john --namespace=development
```

### ServiceAccount + ClusterRole

```bash
# Create SA
kubectl create serviceaccount pvviewer

# Create ClusterRole
kubectl create clusterrole pvviewer-role \
  --resource=persistentvolumes \
  --verb=list

# Create ClusterRoleBinding
kubectl create clusterrolebinding pvviewer-role-binding \
  --clusterrole=pvviewer-role \
  --serviceaccount=default:pvviewer
```

Assign SA to pod:

```yaml
spec:
  serviceAccountName: pvviewer
  containers:
  - image: redis
    name: pvviewer
```

### Key Assumptions

- CSR `signerName`: use `kubernetes.io/kube-apiserver-client` for user auth
- `usages`: must include `digital signature`, `key encipherment`, `client auth`
- ClusterRole = cluster-wide; Role = namespace-scoped
- ClusterRoleBinding binds ClusterRole to a user/SA across the cluster

---

## NetworkPolicy

### Allow All Ingress to a Pod on Specific Port

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ingress-to-nptest
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: np-test-1
  policyTypes:
  - Ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
    # No 'from:' field = allow from ALL sources
```

> [!important] Empty `from:` = allow all sources. Omitting `ingress:` entirely = deny all.

### Cross-Namespace Policy (allow frontend, deny databases)

```yaml
spec:
  podSelector: {}    # Applies to all pods in this namespace
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
    # Deliberately NOT including databases namespace
```

### Exam Strategy for Network Policy Questions

1. Read all provided YAML files carefully
2. Look for which one is most restrictive but still meets the requirement
3. Check: does it accidentally allow traffic it shouldn't? Does it deny what it should?
4. Apply only the correct one — **don't delete existing policies**

---

## Node Management - Taints, Draining & Upgrades

### Taints & Tolerations

```bash
# Add taint to node
kubectl taint node node01 env_type=production:NoSchedule

# Verify
kubectl describe node node01 | grep -i taint

# Remove taint (add - at end)
kubectl taint node node01 node-role.kubernetes.io/control-plane:NoSchedule-
```

Pod with toleration:

```yaml
spec:
  tolerations:
  - key: env_type
    operator: Equal
    value: production
    effect: NoSchedule
```

### Drain & Uncordon

```bash
kubectl drain <node> --ignore-daemonsets
kubectl uncordon <node>
```

> [!tip] After draining controlplane during upgrade, remove the control-plane taint before scheduling workloads there: `kubectl taint node controlplane node-role.kubernetes.io/control-plane:NoSchedule-`

### Cluster Upgrade (kubeadm)

**Upgrade order**: controlplane first, then worker nodes. Drain before upgrading each.

```bash
# 1. Update apt source to new version
vim /etc/apt/sources.list.d/kubernetes.list
# Change: v1.34 → v1.35

# 2. Drain the node
kubectl drain controlplane --ignore-daemonsets

# 3. Install new kubeadm
apt update
apt-get install kubeadm=1.35.0-1.1

# 4. Plan and apply upgrade
kubeadm upgrade plan v1.35.0
kubeadm upgrade apply v1.35.0

# 5. Upgrade kubelet
apt-get install kubelet=1.35.0-1.1
systemctl daemon-reload
systemctl restart kubelet

# 6. Uncordon
kubectl uncordon controlplane

# --- On worker node (SSH in first) ---
# Same steps but use: kubeadm upgrade node (not apply)
```

---

## Static Pods

### Scenario: Create a static pod on a specific node

```bash
# 1. Generate YAML on controlplane
kubectl run nginx-critical --image=nginx --dry-run=client -o yaml > static.yaml

# 2. Copy to the node
scp static.yaml node01:/root/

# 3. SSH to node
ssh node01

# 4. Ensure manifests directory exists
mkdir -p /etc/kubernetes/manifests

# 5. Check kubelet config has the right staticPodPath
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# 6. Copy the manifest
cp /root/static.yaml /etc/kubernetes/manifests/

# 7. Exit back to controlplane and verify
exit
kubectl get pods   # Look for nginx-critical-node01
```

> [!note] Static pod names are suffixed with the node name, e.g. `nginx-critical-node01`

---

## DNS & Service Discovery

### nslookup from within cluster

```bash
# Create a temporary busybox pod to test DNS
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never \
  -- nslookup nginx-resolver-service

# Save output to file
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never \
  -- nslookup nginx-resolver-service > /root/CKA/nginx.svc

# Pod IP DNS lookup (replace dots with hyphens in IP)
# If pod IP is 10.244.1.5, use: 10-244-1-5.default.pod
kubectl run test-nslookup --image=busybox:1.28 --rm -it --restart=Never \
  -- nslookup 10-244-1-5.default.pod > /root/CKA/nginx.pod
```

### Key Assumptions

- Always use `busybox:1.28` for DNS lookups (newer versions have issues with nslookup)
- Pod DNS format: `<IP-with-dashes>.<namespace>.pod`
- Service DNS format: `<service-name>.<namespace>.svc.cluster.local`

---

## ConfigMaps & Secrets

### Create ConfigMap and inject into Deployment

```bash
# Create ConfigMap
kubectl create configmap app-config -n cm-namespace \
  --from-literal=ENV=production \
  --from-literal=LOG_LEVEL=info

# Inject all keys from ConfigMap as env vars into deployment
kubectl set env deployment/cm-webapp -n cm-namespace \
  --from=configmap/app-config

# Verify
kubectl describe deployment cm-webapp -n cm-namespace | grep -A 10 Environment
```

### Secret Volume Mount (read-only)

```yaml
spec:
  volumes:
  - name: secret-volume
    secret:
      secretName: dotfile-secret
  containers:
  - name: secret-admin
    image: busybox
    command: ["sleep", "4800"]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret-volume
      readOnly: true
```

---

## Ingress

### Create Ingress resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: ingress-ns
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: kodekloud-ingress.app
    http:
      paths:
      - path: /
        pathType: Prefix    # Always check if Prefix or Exact is required
        backend:
          service:
            name: webapp-svc
            port:
              number: 80
```

### Key Assumptions

- Almost always need `ingressClassName: nginx` for nginx ingress controller
- `pathType: Prefix` routes everything under the path; `Exact` is strict
- Test with: `curl http://<hostname>/`

---

## Gateway API

### Create Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

### HTTPS with TLS

```yaml
spec:
  gatewayClassName: kodekloud
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: kodekloud.com
      tls:
        certificateRefs:
          - name: kodekloud-tls    # Name of the TLS secret
```

### HTTPRoute with Traffic Splitting

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: default
spec:
  parentRefs:
    - name: web-gateway
      namespace: default
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: web-service
          port: 80
          weight: 80    # 80% of traffic
        - name: web-service-v2
          port: 80
          weight: 20    # 20% of traffic
```

---

## Helm

### Key Commands Cheatsheet

```bash
# List releases (all namespaces)
helm ls -A

# List repos
helm repo ls

# Update a specific repo
helm repo update <repo-name>

# Search for chart versions
helm search repo <repo-name>/<chart-name> -l | head -n 30

# Upgrade to specific version
helm upgrade <release-name> <repo-name>/<chart-name> -n <namespace> --version=6.11.2

# Validate a chart
helm lint ./chart-directory

# Install new release
helm install <release-name> ./chart-directory

# Uninstall
helm uninstall <release-name> -n <namespace>

# Find image used by a release
kubectl get deploy -n <ns> <deploy-name> -o json | jq -r '.spec.template.spec.containers[].image'
```

### Exam Strategy for Helm Questions

1. `helm ls -A` to see all releases and their namespaces
2. Identify the target release/namespace
3. Follow the specific task (update repo → upgrade version, or lint → install → uninstall old)

---

## etcd Backup

```bash
export ETCDCTL_API=3
etcdctl snapshot save \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --endpoints=127.0.0.1:2379 \
  /opt/etcd-backup.db
```

> [!tip] These cert paths are standard for kubeadm clusters. Always verify with: `kubectl describe pod etcd-controlplane -n kube-system | grep -A 20 Command`

---

## Cluster Troubleshooting

### Controller Manager Down / Replicas Not Scaling

```bash
# Check control plane pods
kubectl get pods -n kube-system

# If controller-manager is not running, check its manifest for typos
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep command

# Fix binary name typo (common exam scenario: kube-contro1ler-manager)
sed -i 's/kube-contro1ler-manager/kube-controller-manager/g' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml

# kubelet auto-detects the change and restarts within ~20 seconds
kubectl get pods -n kube-system | grep controller-manager
```

### PVC Stuck / Alpha MySQL Not Starting

```bash
kubectl describe pod <pod-name> -n <namespace>   # Look at Events
# Common issue: PVC not found or wrong storageClass

# Fix: create the missing PVC with matching specs
kubectl describe pv alpha-pv   # Check storageClassName, accessModes, capacity
```

### Pod CIDR

```bash
# Cluster-wide CIDR (from kubeadm config - bigger range e.g. /16)
kubectl -n kube-system get configmap kubeadm-config -o yaml | grep podSubnet

# Per-node CIDR (smaller slice e.g. /24) - don't use this for cluster-wide questions
kubectl get node <name> -o jsonpath='{.spec.podCIDR}'

# Save to file
kubectl -n kube-system get configmap kubeadm-config -o yaml \
  | awk '/podSubnet:/{print $2}' > /root/pod-cidr.txt
```

---

## Kubeconfig Troubleshooting

### Common Issues

- Wrong port (e.g. 9999 or 4380 instead of **6443**)
- Wrong server hostname/IP

```bash
# Test a specific kubeconfig
kubectl cluster-info --kubeconfig=/root/CKA/super.kubeconfig

# Fix by editing the file
vim /root/CKA/super.kubeconfig
# Change: https://controlplane:9999 → https://controlplane:6443
```

---

## System-Level Prep (kubeadm)

### Persist sysctl settings across reboots

```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-iptables = 1' >> /etc/sysctl.conf
sysctl -p   # Apply immediately

# Verify
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

> [!important] `/etc/sysctl.d/` files also persist reboots and are the preferred modern location, but `/etc/sysctl.conf` works and is simpler to remember for the exam.

---

## CRI Runtime Installation

### Install cri-docker from .deb package

```bash
# SSH to node
ssh bob@node01   # password: caleston123 (in KodeKloud labs)
sudo -i

# Install
dpkg -i /root/cri-docker_0.3.16.3-0.debian.deb

# Start and enable
systemctl start cri-docker
systemctl enable cri-docker

# Verify
systemctl is-active cri-docker    # should return: active
systemctl is-enabled cri-docker   # should return: enabled
```

---

## CRDs & Custom Resources

### Find CRDs by name pattern

```bash
# List all CRDs, filter by keyword
kubectl get crd -o custom-columns=NAME:.metadata.name | grep verticalpodautoscaler \
  > /root/vpa-crds.txt

# Verify
cat /root/vpa-crds.txt
```

---

## Priority Classes

### Create PriorityClass and assign to Pod

```yaml
# priority-class.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 50000
globalDefault: false
description: "Low priority class"
```

```bash
kubectl apply -f pc.yaml

# Get existing pod YAML to recreate it
kubectl get pod lp-pod -n low-priority -o yaml > lp-pod.yaml

# Edit: add priorityClassName under spec
# Delete and recreate
kubectl delete pod lp-pod -n low-priority
kubectl apply -f lp-pod.yaml
```

```yaml
spec:
  priorityClassName: low-priority   # Add this field
  containers:
  - name: nginx
    image: nginx
```

---

## Custom Columns Output (Deployment Listing)

```bash
kubectl -n admin2406 get deployment \
  -o custom-columns=\
DEPLOYMENT:.metadata.name,\
CONTAINER_IMAGE:.spec.template.spec.containers[].image,\
READY_REPLICAS:.status.readyReplicas,\
NAMESPACE:.metadata.namespace \
  --sort-by=.metadata.name \
  > /opt/admin2406_data
```

---

## Quick Reference - Common Mistakes

> [!danger] Watch out for these on exam day

|Mistake|Fix|
|---|---|
|Using `--record` flag|Deprecated — use `kubectl annotate` instead|
|Wrong API version for HPA|Use `autoscaling/v2` not `v1`|
|Forgetting `emptyDir: {}` for shared volumes|Non-persistent = emptyDir|
|PVC not binding|Check accessModes match between PV and PVC|
|Wrong port in kubeconfig|Should be 6443|
|Static pod not appearing|Check `staticPodPath` in `/var/lib/kubelet/config.yaml`|
|busybox DNS issues|Always use `busybox:1.28` for nslookup|
|Forgetting `-n <namespace>`|Always check which namespace the resource lives in|
|`kubectl replace` vs `kubectl apply`|Use `replace --force` when you can't edit a running pod|
|Cluster CIDR vs Node CIDR|Cluster = kubeadm-config ConfigMap; Node = `spec.podCIDR`|

---

## Imperative Command Cheatsheet

```bash
# Pod
kubectl run <name> --image=<image> --namespace=<ns>
kubectl run <name> --image=<image> --dry-run=client -o yaml > pod.yaml

# Deployment
kubectl create deployment <name> --image=<image> --replicas=<n>

# Service (expose)
kubectl expose pod <name> --port=<port> --name=<svc-name>
kubectl expose deployment <name> --type=NodePort --port=<port> --name=<svc-name>

# ConfigMap
kubectl create configmap <name> --from-literal=KEY=value

# ServiceAccount
kubectl create serviceaccount <name>

# Role / ClusterRole
kubectl create role <name> --resource=pods --verb=get,list,create
kubectl create clusterrole <name> --resource=persistentvolumes --verb=list

# RoleBinding / ClusterRoleBinding
kubectl create rolebinding <name> --role=<role> --user=<user> -n <ns>
kubectl create clusterrolebinding <name> --clusterrole=<cr> --serviceaccount=<ns>:<sa>

# Drain / Uncordon
kubectl drain <node> --ignore-daemonsets [--delete-emptydir-data]
kubectl uncordon <node>

# Taint
kubectl taint node <node> key=value:Effect
kubectl taint node <node> key=value:Effect-   # Remove taint

# Auth check
kubectl auth can-i <verb> <resource> --as=<user> -n <ns>
```

---

_Last updated for CKA exam prep — killer.sh and official CKA_
