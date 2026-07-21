> [!info] How to use this file
> Work through each question yourself on `ssh ckad9999`, then expand the solution and check your answer against the markscheme checklist at the bottom of each question.

---

## Tags

`#CKA` `#CKAD` `#markscheme` `#mock-exam` `#exam-prep`

---

# 📋 CK-X Mock Markscheme

> [!warning] Cluster prerequisites
> Some tasks depend on lab-provided components such as StorageClasses, metrics-server, a GatewayClass/controller, and a NetworkPolicy-capable CNI. Verify those prerequisites before treating a Pending or unaccepted resource as a manifest error.

---

## Q1 — Dynamic PVC and Pod

**Instance:** `ssh ckad9999`<br>
**Namespace:** `storage-task`<br>
**Concepts:** `storage`, `persistent-volumes`, `pods`

### Task

Create a dynamic PVC named `data-pvc` using StorageClass `standard`, access mode `ReadWriteOnce`, and a `2Gi` request. Create Pod `data-pod` with image `nginx`; mount the claim as volume `data` at `/usr/share/nginx/html`. Keep both resources in `storage-task`.

---

### Solution

> [!tip] Exam-speed route
> There is no useful imperative PVC generator in the adjacent notes. Write the short PVC YAML, but scaffold the Pod instead of typing its boilerplate:
>
> ```bash
> kubectl run data-pod -n storage-task --image=nginx --dry-run=client -o yaml > /tmp/data-pod.yaml
> # Add volumeMounts + volumes, then:
> kubectl apply -f /tmp/data-pod.yaml
> kubectl get pvc,pod -n storage-task
> ```
>
> The final `get` checks both objects at once.

> [!attention] Personal weakness drill
> **Volumes & VolumeMounts:** trace the chain `volumeMount → volume → claimName → PVC → StorageClass`. Practise spotting a mismatched volume name in under 20 seconds.

```text
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: storage-task
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: data-pod
  namespace: storage-task
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
```

---

### ✅ Markscheme

- [ ] PVC `data-pvc` exists in `storage-task`
- [ ] StorageClass is `standard`; access mode is `ReadWriteOnce`; request is `2Gi`
- [ ] Pod `data-pod` uses image `nginx`
- [ ] Volume is named `data`, references `data-pvc`, and mounts at `/usr/share/nginx/html`

---

## Q2 — Manual PersistentVolume

**Instance:** `ssh ckad9999`<br>
**Namespace:** `manual-storage`<br>
**Concepts:** `storage`, `persistent-volumes`, `node-affinity`

### Task

Create PV `manual-pv` with `1Gi`, `ReadWriteOnce`, hostPath `/mnt/data`, and required node affinity for `k3d-cluster-agent-0`. Create `manual-pvc` in `manual-storage` and bind it to that PV. Create `manual-pod` using `busybox`, mount the claim at `/data`, and run `sleep 3600`.

---

### Solution

> [!tip] Exam-speed route
> PV/PVC creation is a YAML task. Save time by copying your Q1 storage manifest and changing names, size, StorageClass behavior, `volumeName`, hostPath, and node affinity. Verify binding before debugging the Pod:
>
> ```bash
> kubectl get pv manual-pv
> kubectl get pvc manual-pvc -n manual-storage
> kubectl describe pvc manual-pvc -n manual-storage   # only if it is not Bound
> ```
>
> Do not put a namespace on the PV—it is cluster-scoped.

> [!attention] Personal weakness drill
> **PV/PVC binding:** remember that the PV is cluster-scoped, the PVC is namespaced, and `volumeName: manual-pv` makes the intended static binding explicit.

```text
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k3d-cluster-agent-0
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: manual-pvc
  namespace: manual-storage
spec:
  storageClassName: ""
  volumeName: manual-pv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: manual-pod
  namespace: manual-storage
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: manual-pvc
```

---

### ✅ Markscheme

- [ ] Cluster-scoped PV `manual-pv` has the requested capacity, access mode, and hostPath
- [ ] PV node affinity requires hostname `k3d-cluster-agent-0`
- [ ] PVC `manual-pvc` is explicitly bound to `manual-pv` and becomes `Bound`
- [ ] Pod `manual-pod` uses `busybox`, runs `sleep 3600`, and mounts the claim at `/data`

---

## Q3 — Default StorageClass

**Instance:** `ssh ckad9999`<br>
**Namespace:** `storage-class`<br>
**Concepts:** `storage`, `storage-classes`

### Task

Create StorageClass `fast-local` with provisioner `rancher.io/local-path` and `WaitForFirstConsumer`. Make it the default and remove the default annotation from every existing default StorageClass.

---

### Solution

> [!tip] Exam-speed route
> Write the new StorageClass YAML, then use patches for the default annotations; this is faster and less error-prone than editing several objects:
>
> ```bash
> kubectl get sc
> kubectl patch sc <old-default> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
> kubectl apply -f fast-local.yaml
> kubectl get sc
> ```
>
> Only one row should show `(default)` at the end.

> [!attention] Personal weakness drill
> **kubectl patch:** type the nested annotation patch yourself once. The outer path is `metadata → annotations`; quote the annotation value as a string.

```text
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-local
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
Commands to remove default from other StorageClass:

kubectl patch storageclass default-test -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'
kubectl patch storageclass local-path -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'
```

---

### ✅ Markscheme

- [ ] StorageClass `fast-local` exists
- [ ] Provisioner and binding mode match the task
- [ ] `fast-local` is annotated as default
- [ ] No other StorageClass remains marked as default

---

## Q4 — Deployment with HPA

**Instance:** `ssh ckad9999`<br>
**Namespace:** `scaling`<br>
**Concepts:** `autoscaling`, `deployments`, `resource-management`

### Task

Create Deployment `scaling-app` with image `nginx`, 2 replicas, requests of `200m` CPU and `256Mi` memory, and limits of `500m` CPU and `512Mi` memory. Create an `autoscaling/v1` HPA with min 2, max 5, and target CPU utilization 70%.

---

### Solution

> [!tip] Exam-speed route
> The Deployment, resources, and HPA can all be created quickly without hand-writing their boilerplate:
>
> ```bash
> kubectl create deploy scaling-app -n scaling --image=nginx --replicas=2
> kubectl set resources deploy/scaling-app -n scaling --requests=cpu=200m,memory=256Mi --limits=cpu=500m,memory=512Mi
> kubectl autoscale deploy scaling-app -n scaling --min=2 --max=5 --cpu-percent=70
> kubectl get deploy,hpa -n scaling
> ```
>
> Because the task explicitly requires `autoscaling/v1`, first test the generator with `--dry-run=client -o yaml`. If it emits `autoscaling/v2`, use the short supplied v1 HPA YAML instead of converting its metrics structure under exam pressure.

> [!attention] Personal weakness drill
> **HPA + set resources:** repeat both commands from memory. The HPA needs a CPU request to calculate utilization, so `kubectl set resources` is not optional groundwork.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scaling-app
  namespace: scaling
spec:
  replicas: 2
  selector:
    matchLabels:
      app: scaling-app
  template:
    metadata:
      labels:
        app: scaling-app
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: scaling-app
  namespace: scaling
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: scaling-app
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

---

### ✅ Markscheme

- [ ] Deployment exists in `scaling` with 2 replicas and image `nginx`
- [ ] Requests and limits exactly match
- [ ] HPA targets Deployment `scaling-app`
- [ ] HPA uses `autoscaling/v1`, min 2, max 5, target 70%

---

## Q5 — Required Node Affinity

**Instance:** `ssh ckad9999`<br>
**Namespace:** `scheduling`<br>
**Concepts:** `scheduling`, `node-affinity`, `deployments`

### Task

Label node `k3d-cluster-agent-1` with `disk=ssd`. Create Deployment `app-scheduling` with 3 `nginx` replicas that can run only on that node using `requiredDuringSchedulingIgnoredDuringExecution` node affinity, matched by hostname (not `nodeSelector`).

---

### Solution

> [!tip] Exam-speed route
> Do the easy imperative parts first, then generate a Deployment skeleton and add only the affinity block:
>
> ```bash
> kubectl label node k3d-cluster-agent-1 disk=ssd --overwrite
> kubectl create deploy app-scheduling -n scheduling --image=nginx --replicas=3 --dry-run=client -o yaml > /tmp/app-scheduling.yaml
> # Add required nodeAffinity matching kubernetes.io/hostname, then apply.
> kubectl get pod -n scheduling -l app=app-scheduling -o wide
> ```
>
> The `-o wide` check proves all replicas landed on the required node.

```text
# Label the node
kubectl label node k3d-cluster-agent-1 disk=ssd
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-scheduling
  namespace: scheduling
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-scheduling
  template:
    metadata:
      labels:
        app: app-scheduling
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - k3d-cluster-agent-1
      containers:
      - name: nginx
        image: nginx
```

---

### ✅ Markscheme

- [ ] Target node has label `disk=ssd`
- [ ] Deployment has 3 replicas using `nginx`
- [ ] Required node affinity matches `kubernetes.io/hostname=k3d-cluster-agent-1`
- [ ] No `nodeSelector` is used; all three Pods land on the target node

---

## Q6 — Restricted Pod Security

**Instance:** `ssh ckad9999`<br>
**Namespace:** `security`<br>
**Concepts:** `security`, `pod-security-admission`

### Task

Create namespace `security` with Pod Security Admission enforcing `restricted` at `latest`. Create Pod `secure-pod` using `nginx`; run as non-root UID 1000, prevent privilege escalation, drop all capabilities, use `RuntimeDefault` seccomp, and mount emptyDir `html` at `/usr/share/nginx/html`.

---

### Solution

> [!tip] Exam-speed route
> Create and label the namespace imperatively, then generate a Pod skeleton and add the security/volume fields:
>
> ```bash
> kubectl create ns security
> kubectl label ns security pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/enforce-version=latest --overwrite
> kubectl run secure-pod -n security --image=nginx --dry-run=client -o yaml > /tmp/secure-pod.yaml
> # Edit, apply, then inspect any PSA rejection:
> kubectl get events -n security --sort-by=.metadata.creationTimestamp | tail
> ```
>
> Remember: capabilities and `allowPrivilegeEscalation` belong at container level; seccomp may be set at Pod level.

> [!attention] Personal weakness drill
> **Security contexts:** say the level aloud while editing: seccomp/runAs fields may be Pod-level; capabilities and `allowPrivilegeEscalation` are container-level.

```text
kubectl create namespace security
kubectl label namespace security pod-security.kubernetes.io/enforce=restricted  pod-security.kubernetes.io/enforce-version=latest
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: security
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
          - ALL
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
  - name: html
    emptyDir: {}
```

---

### ✅ Markscheme

- [ ] Namespace labels enforce `restricted` at `latest`
- [ ] Pod-level context enables non-root UID 1000 and RuntimeDefault seccomp
- [ ] Container prevents privilege escalation, runs non-root as UID 1000, and drops `ALL` capabilities
- [ ] emptyDir `html` is mounted at `/usr/share/nginx/html`; Pod is admitted and running

---

## Q7 — Taints and Tolerations

**Instance:** `ssh ckad9999`<br>
**Namespace:** `scheduling`<br>
**Concepts:** `scheduling`, `taints`, `tolerations`

### Task

Taint `k3d-cluster-agent-1` with `special-workload=true:NoSchedule`. Create 2-replica `toleration-deploy` using `nginx` that tolerates the taint, plus 2-replica `normal-deploy` using `nginx` that does not tolerate it and therefore must not run there.

---

### Solution

> [!tip] Exam-speed route
> The taint is a one-liner. Generate one Deployment manifest, add the toleration, then copy it for the normal Deployment and remove the toleration:
>
> ```bash
> kubectl taint node k3d-cluster-agent-1 special-workload=true:NoSchedule --overwrite
> kubectl create deploy toleration-deploy -n scheduling --image=nginx --replicas=2 --dry-run=client -o yaml > /tmp/toleration.yaml
> kubectl get pod -n scheduling -o wide
> ```
>
> A toleration permits use of the node; it does not force the Pods onto it.

```text
kubectl taint node k3d-cluster-agent-1 special-workload=true:NoSchedule
apiVersion: apps/v1
kind: Deployment
metadata:
  name: toleration-deploy
  namespace: scheduling
spec:
  replicas: 2
  selector:
    matchLabels:
      app: toleration-deploy
  template:
    metadata:
      labels:
        app: toleration-deploy
    spec:
      tolerations:
      - key: "special-workload"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      containers:
      - name: nginx
        image: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: normal-deploy
  namespace: scheduling
spec:
  replicas: 2
  selector:
    matchLabels:
      app: normal-deploy
  template:
    metadata:
      labels:
        app: normal-deploy
    spec:
      containers:
      - name: nginx
        image: nginx
```

---

### ✅ Markscheme

- [ ] Node has the exact `NoSchedule` taint
- [ ] `toleration-deploy` has 2 replicas and the exact toleration
- [ ] `normal-deploy` has 2 replicas and no matching toleration
- [ ] No `normal-deploy` Pod is scheduled on the tainted node

---

## Q8 — StatefulSet and Headless Service

**Instance:** `ssh ckad9999`<br>
**Namespace:** `stateful`<br>
**Concepts:** `statefulsets`, `headless-services`, `storage`

### Task

Create StatefulSet `web` with 3 sequentially created `nginx` replicas. Create headless Service `web-svc`. Give every Pod a volume at `/usr/share/nginx/html` using a `1Gi` volume claim template and StorageClass `cold`, preserving stable network identity.

---

### Solution

> [!tip] Exam-speed route
> There is no dependable imperative StatefulSet generator. Type or copy a known StatefulSet skeleton. The quick win is verifying all three requirements in one view:
>
> ```bash
> kubectl get sts,pod,pvc,svc -n stateful
> kubectl get pod -n stateful -l app=web -o custom-columns=NAME:.metadata.name,PVC:.spec.volumes[*].persistentVolumeClaim.claimName
> ```
>
> Do not replace the StatefulSet with a Deployment—stable identity and claim templates are the point of the question.

```text
bash kubectl create namespace stateful

apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: stateful
spec:
  clusterIP: None
  selector:
    app: web
  ports:
  - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: stateful
spec:
  serviceName: web-svc
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: cold
      resources:
        requests:
          storage: 1Gi
```

---

### ✅ Markscheme

- [ ] `web-svc` is headless (`clusterIP: None`) and selects the StatefulSet Pods
- [ ] StatefulSet `web` has 3 `nginx` replicas and `serviceName: web-svc`
- [ ] Pod management is sequential (`OrderedReady`, explicitly or by default)
- [ ] Claim template requests `1Gi` from StorageClass `cold` and mounts at the required path
- [ ] Pods have stable names and service DNS identities

---

## Q9 — DNS Configuration and Debugging

**Instance:** `ssh ckad9999`<br>
**Namespace:** `dns-debug`<br>
**Concepts:** `dns`, `service-discovery`, `networking`

### Task

Create 3-replica Deployment `web-app`, ClusterIP Service `web-svc`, test Pod `dns-test` using `busybox`, and ConfigMap `dns-config`. The test Pod must fetch both the short service name and FQDN, verify service and Pod DNS, then sleep. Configure custom search domains.

---

### Solution

> [!tip] Exam-speed route
> Create the Deployment and Service imperatively; reserve YAML editing for the specialised test Pod and DNS settings:
>
> ```bash
> kubectl create deploy web-app -n dns-debug --image=nginx --replicas=3
> kubectl expose deploy web-app -n dns-debug --name=web-svc --port=80
> kubectl get endpoints web-svc -n dns-debug
> kubectl exec -n dns-debug dns-test -- cat /etc/resolv.conf
> ```
>
> Empty endpoints means a selector/readiness problem, not a DNS problem.

> [!attention] Personal weakness drill
> **DNS structure + `tr`:** learn `service.namespace.svc.cluster.local` and `dashed-pod-ip.namespace.pod.cluster.local`. This solution deliberately uses `tr . -` to reinforce the recorded shell weakness.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: dns-debug
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: dns-debug
spec:
  selector:
    app: web-app
  ports:
  - port: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: dns-test
  namespace: dns-debug
spec:
  containers:
  - name: busybox
    image: busybox
    command:
    - sh
    - -c
    - "wget -qO- http://web-svc && wget -qO- http://web-svc.dns-debug.svc.cluster.local && sleep 36000"
  dnsConfig:
    searches:
    - dns-debug.svc.cluster.local
    - svc.cluster.local
    - cluster.local
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: dns-config
  namespace: dns-debug
data:
  search-domains: |
    search dns-debug.svc.cluster.local svc.cluster.local cluster.local

# Verify a Pod DNS entry (derive the dashed name from an actual web-app Pod IP)
POD_IP=$(kubectl get pod -n dns-debug -l app=web-app -o jsonpath='{.items[0].status.podIP}')
POD_DNS=$(printf '%s' "$POD_IP" | tr . -).dns-debug.pod.cluster.local
kubectl exec -n dns-debug dns-test -- nslookup "$POD_DNS"
```

---

### ✅ Markscheme

- [ ] Deployment has 3 ready `nginx` replicas and Service has endpoints
- [ ] `dns-test` resolves/fetches both `web-svc` and its FQDN
- [ ] Pod DNS entries are tested against actual Pod IP-derived DNS names
- [ ] ConfigMap `dns-config` contains the custom search domains
- [ ] Test Pod remains available for inspection

---

## Q10 — Basic DNS Service Discovery

**Instance:** `ssh ckad9999`<br>
**Namespace:** `dns-config`<br>
**Concepts:** `dns`, `service-discovery`, `networking`

### Task

Create 2-replica `dns-app`, Service `dns-svc`, and Pod `dns-tester` using `infoblox/dnstools`. Test both `dns-svc` and `dns-svc.dns-config.svc.cluster.local`, writing both results to `/tmp/dns-test.txt`.

---

### Solution

> [!tip] Exam-speed route
> This question is mostly imperative. Generate the tester Pod so only its command needs editing:
>
> ```bash
> kubectl create deploy dns-app -n dns-config --image=nginx --replicas=2
> kubectl expose deploy dns-app -n dns-config --name=dns-svc --port=80
> kubectl run dns-tester -n dns-config --image=infoblox/dnstools --dry-run=client -o yaml > /tmp/dns-tester.yaml
> kubectl exec -n dns-config dns-tester -- cat /tmp/dns-test.txt
> ```
>
> Use `>` for the first lookup and `>>` for the second so both results survive.

> [!attention] Personal weakness drill
> **DNS verification:** test the short Service name and FQDN separately, then inspect `/etc/resolv.conf` if only the short form fails. Keep both outputs in the required file.

```text
# Create the deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dns-app
  namespace: dns-config
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dns-app
  template:
    metadata:
      labels:
        app: dns-app
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
# Create the service
apiVersion: v1
kind: Service
metadata:
  name: dns-svc
  namespace: dns-config
spec:
  selector:
    app: dns-app
  ports:
  - port: 80
    targetPort: 80
---
# Create the DNS tester pod
apiVersion: v1
kind: Pod
metadata:
  name: dns-tester
  namespace: dns-config
spec:
  containers:
  - name: dns-tester
    image: infoblox/dnstools
    command:
    - sh
    - -c
    - |
      nslookup dns-svc > /tmp/dns-test.txt
      nslookup dns-svc.dns-config.svc.cluster.local >> /tmp/dns-test.txt
      sleep 3600

# Verify the setup:

# Check if service is resolvable
kubectl exec -n dns-config dns-tester -- cat /tmp/dns-test.txt

# Verify deployment is running
kubectl get deployment -n dns-config dns-app

# Check service endpoints
kubectl get endpoints -n dns-config dns-svc
```

---

### ✅ Markscheme

- [ ] Deployment has 2 ready `nginx` replicas and Service has endpoints
- [ ] Tester uses `infoblox/dnstools`
- [ ] Both short-name and FQDN lookups are written to `/tmp/dns-test.txt`
- [ ] The results file can be read and shows successful resolution

---

## Q11 — Helm Chart Deployment

**Instance:** `ssh ckad9999`<br>
**Namespace:** `helm-test`<br>
**Concepts:** `helm`, `package-management`

### Task

Add the Bitnami repository, install `bitnami/nginx` as release `web-release` in `helm-test`, set Service type `NodePort`, set replica count 2, and verify the release and Pods.

---

### Solution

> [!tip] Exam-speed route
> The supplied Helm command is already the fastest route. Combine namespace creation and values into the install, then verify without waiting through verbose output:
>
> ```bash
> helm repo add bitnami https://charts.bitnami.com/bitnami
> helm repo update
> helm install web-release bitnami/nginx -n helm-test --create-namespace --set service.type=NodePort --set replicaCount=2 --wait
> helm status web-release -n helm-test
> ```
>
> `--wait` makes success/failure clear before you move on.

> [!attention] Personal weakness drill
> **Helm charts:** practise the sequence `repo add → repo update → install → status`. Use `helm show values bitnami/nginx | grep -n` when you cannot remember a value key.

```text
# Add bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install nginx chart
helm install web-release bitnami/nginx \
  --namespace helm-test \
  --create-namespace \
  --set service.type=NodePort \
  --set replicaCount=2

# Verify
helm status web-release -n helm-test
kubectl get pods,svc -n helm-test
```

---

### ✅ Markscheme

- [ ] Bitnami repository is configured and updated
- [ ] Release `web-release` exists in namespace `helm-test`
- [ ] Chart values set `service.type=NodePort` and `replicaCount=2`
- [ ] Helm status is deployed and 2 Pods are ready

---

## Q12 — Kustomize Production Overlay

**Instance:** `ssh ckad9999`<br>
**Namespace:** `kustomize`<br>
**Concepts:** `kustomize`, `configuration-management`

### Task

Under `/tmp/exam/kustomize/`, create an nginx base Deployment with 2 replicas and a production overlay that labels resources `environment=production`, scales to 3 replicas, generates ConfigMap `nginx-config` with `index.html=Welcome to Production`, and mounts it as volume `nginx-index`. Apply the overlay to namespace `kustomize`.

---

### Solution

> [!tip] Exam-speed route
> Use kubectl to generate the base Deployment, then let Kustomize own the overlay changes:
>
> ```bash
> kubectl create deploy nginx --image=nginx --replicas=2 --dry-run=client -o yaml > /tmp/exam/kustomize/base/deployment.yaml
> kubectl kustomize /tmp/exam/kustomize/overlays/production   # preview before changing the cluster
> kubectl apply -k /tmp/exam/kustomize/overlays/production
> kubectl get all,cm -n kustomize
> ```
>
> Previewing with `kubectl kustomize` catches wrong paths and patches faster than debugging a failed apply.

> [!attention] Personal weakness drill
> **Kustomize patches:** identify which part is base, which part is overlay, and why this JSON6902 operation uses `/spec/replicas`. Preview with `kubectl kustomize` before applying.

```text
First, create the directory structure:

mkdir -p /tmp/exam/kustomize/base
mkdir -p /tmp/exam/kustomize/overlays/production
Create base files:

# /tmp/exam/kustomize/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: nginx-index
          mountPath: /usr/share/nginx/html/
      volumes:
      - name: nginx-index
        configMap:
          name: nginx-config

# /tmp/exam/kustomize/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml

# /tmp/exam/kustomize/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: kustomize
resources:
- ../../base
patches:
- patch: |
    - op: replace
      path: /spec/replicas
      value: 3
  target:
    kind: Deployment
    name: nginx
commonLabels:
  environment: production
configMapGenerator:
- name: nginx-config
  literals:
  - index.html=Welcome to Production
Apply the configuration:

kubectl create namespace kustomize
kubectl apply -k /tmp/exam/kustomize/overlays/production/
Verify the deployment:

kubectl get deployments -n kustomize
kubectl get configmaps -n kustomize
kubectl get pods -n kustomize
```

---

### ✅ Markscheme

- [ ] Base and overlay contain valid `kustomization.yaml` files
- [ ] Rendered overlay has 3 replicas and label `environment=production`
- [ ] Generated ConfigMap contains the exact key/value
- [ ] Deployment mounts ConfigMap volume `nginx-index`
- [ ] Overlay is applied in `kustomize` and 3 Pods are ready

---

## Q13 — Gateway API Routing

**Instance:** `ssh ckad9999`<br>
**Namespace:** `gateway`<br>
**Concepts:** `gateway-api`, `networking`

### Task

Create Gateway `main-gateway` listening on HTTP port 80 and an HTTPRoute sending `/app1` to `app1-svc:8080` and `/app2` to `app2-svc:8080`. Create `app1` and `app2` Deployments and corresponding Services.

---

### Solution

> [!tip] Exam-speed route
> Gateway and HTTPRoute need YAML, but the test backends do not:
>
> ```bash
> kubectl create deploy app1 -n gateway --image=nginx
> kubectl expose deploy app1 -n gateway --name=app1-svc --port=8080 --target-port=80
> kubectl create deploy app2 -n gateway --image=nginx
> kubectl expose deploy app2 -n gateway --name=app2-svc --port=8080 --target-port=80
> kubectl get gateway,httproute -n gateway
> ```
>
> Check the installed versions first with `kubectl api-resources | grep -E 'gateway|httproute'`; use the served API version rather than guessing.

> [!attention] Personal weakness drill
> **Gateway API relationships:** recite `GatewayClass → Gateway listener → HTTPRoute parentRefs → Service backend`. The Services are internal ClusterIP backends; do not reach for NodePort unless the task asks for node-level exposure.

```text
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
  namespace: gateway
spec:
  gatewayClassName: standard
  listeners:
  - name: http
    port: 80
    protocol: HTTP
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: app-routes
  namespace: gateway
spec:
  parentRefs:
  - name: main-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /app1
    backendRefs:
    - name: app1-svc
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /app2
    backendRefs:
    - name: app2-svc
      port: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  namespace: gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: app1-svc
  namespace: gateway
spec:
  selector:
    app: app1
  ports:
  - port: 8080
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
  namespace: gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: app2-svc
  namespace: gateway
spec:
  selector:
    app: app2
  ports:
  - port: 8080
    targetPort: 80
```

---

### ✅ Markscheme

- [ ] Gateway is accepted by an installed GatewayClass and listens on HTTP 80
- [ ] HTTPRoute is attached to `main-gateway`
- [ ] Path-prefix matches route `/app1` and `/app2` to the correct backends
- [ ] Both Deployments are ready and both Services expose port 8080 to container port 80

---

## Q14 — LimitRange and ResourceQuota

**Instance:** `ssh ckad9999`<br>
**Namespace:** `limits`<br>
**Concepts:** `resource-management`, `limitrange`, `resourcequota`

### Task

Create a LimitRange with default request `100m`/`128Mi`, default limit `200m`/`256Mi`, and max `500m`/`512Mi`. Create a ResourceQuota limiting total CPU to 2, memory to 2Gi, and Pods to 5. Create 2-replica Deployment `test-limits` to verify defaults.

---

### Solution

> [!tip] Exam-speed route
> ResourceQuota and the test Deployment have fast imperative commands; only the LimitRange needs hand-written YAML:
>
> ```bash
> kubectl create quota compute-quota -n limits --hard=cpu=2,memory=2Gi,pods=5
> kubectl create deploy test-limits -n limits --image=nginx --replicas=2
> kubectl get pod -n limits -l app=test-limits -o jsonpath='{.items[0].spec.containers[0].resources}'
> ```
>
> Create the LimitRange before the Deployment or the defaults will not be injected into already-created Pods.

> [!attention] Personal weakness drill
> **Resource commands:** use the imperative ResourceQuota command, but keep LimitRange YAML. Verify injected defaults with JSONPath rather than visually scanning a full Pod manifest.

```text
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: limits
spec:
  limits:
  - type: Container
    default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: 500m
      memory: 512Mi
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: limits
spec:
  hard:
    cpu: "2"
    memory: 2Gi
    pods: "5"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-limits
  namespace: limits
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-limits
  template:
    metadata:
      labels:
        app: test-limits
    spec:
      containers:
      - name: nginx
        image: nginx
```

---

### ✅ Markscheme

- [ ] LimitRange has the exact defaults, default requests, and maxima
- [ ] ResourceQuota has exact CPU, memory, and Pod hard limits
- [ ] Deployment `test-limits` has 2 ready replicas
- [ ] Created containers receive the expected default requests and limits

---

## Q15 — Resource Consumer HPA

**Instance:** `ssh ckad9999`<br>
**Namespace:** `monitoring`<br>
**Concepts:** `monitoring`, `resource-management`, `autoscaling`

### Task

Create 3-replica Deployment `resource-consumer` using `gcr.io/kubernetes-e2e-test-images/resource-consumer:1.5`, requests `100m`/`128Mi`, and limits `200m`/`256Mi`. Create an HPA with min 3, max 6, and CPU target 50%.

---

### Solution

> [!tip] Exam-speed route
> This is another create → set resources → autoscale sequence:
>
> ```bash
> kubectl create deploy resource-consumer -n monitoring --image=gcr.io/kubernetes-e2e-test-images/resource-consumer:1.5 --replicas=3
> kubectl set resources deploy/resource-consumer -n monitoring --requests=cpu=100m,memory=128Mi --limits=cpu=200m,memory=256Mi
> kubectl autoscale deploy resource-consumer -n monitoring --min=3 --max=6 --cpu-percent=50
> kubectl get deploy,hpa -n monitoring
> ```
>
> If HPA shows `<unknown>`, check `kubectl top pod -n monitoring` before changing the manifest.

> [!attention] Personal weakness drill
> **HPA repetition:** this is a second deliberate drill for `kubectl set resources` followed by `kubectl autoscale`. Use `kubectl top` before blaming the HPA manifest.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-consumer
  namespace: monitoring
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resource-consumer
  template:
    metadata:
      labels:
        app: resource-consumer
    spec:
      containers:
      - name: resource-consumer
        image: gcr.io/kubernetes-e2e-test-images/resource-consumer:1.5
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: resource-consumer
  namespace: monitoring
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: resource-consumer
  minReplicas: 3
  maxReplicas: 6
  targetCPUUtilizationPercentage: 50
```

---

### ✅ Markscheme

- [ ] Deployment has 3 replicas and the exact image
- [ ] CPU and memory requests/limits match
- [ ] HPA targets the correct Deployment with min 3 and max 6
- [ ] Target CPU utilization is 50% and metrics are available

---

## Q16 — Namespace RBAC

**Instance:** `ssh ckad9999`<br>
**Namespace:** `cluster-admin`<br>
**Concepts:** `rbac`, `service-accounts`, `security`

### Task

Create ServiceAccount `app-admin`, a namespaced Role allowing get/list/watch on Pods and Deployments, create/delete on ConfigMaps, and update on Deployments, then bind it. Create `admin-pod` using `bitnami/kubectl:latest`, `sleep 3600`, and an explicitly mounted ServiceAccount token. Verify allowed actions succeed and creating Pods is denied.

---

### Solution

> [!tip] Exam-speed route
> Create the simple RBAC objects imperatively, but keep the Role YAML for its three different verb sets:
>
> ```bash
> kubectl create sa app-admin -n cluster-admin
> kubectl create rolebinding app-admin -n cluster-admin --role=app-admin --serviceaccount=cluster-admin:app-admin
> kubectl auth can-i --list -n cluster-admin --as=system:serviceaccount:cluster-admin:app-admin
> kubectl auth can-i create pods -n cluster-admin --as=system:serviceaccount:cluster-admin:app-admin
> ```
>
> Do not collapse the Role into one imperative rule: that would accidentally grant every verb on every listed resource.

> [!attention] Personal weakness drill
> **RBAC + projected volumes:** keep the three Role rules separate because their verbs differ. Confirm the projected ServiceAccount token and use `kubectl auth can-i` for proof.

```text
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-admin
  namespace: cluster-admin
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-admin
  namespace: cluster-admin
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["list", "get", "watch", "update"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-admin
  namespace: cluster-admin
subjects:
- kind: ServiceAccount
  name: app-admin
  namespace: cluster-admin
roleRef:
  kind: Role
  name: app-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: admin-pod
  namespace: cluster-admin
spec:
  serviceAccountName: app-admin
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          expirationSeconds: 3600
          audience: kubernetes.default.svc

# Verify authorization as the ServiceAccount
kubectl auth can-i list pods -n cluster-admin --as=system:serviceaccount:cluster-admin:app-admin
kubectl auth can-i update deployments.apps -n cluster-admin --as=system:serviceaccount:cluster-admin:app-admin
kubectl auth can-i create configmaps -n cluster-admin --as=system:serviceaccount:cluster-admin:app-admin
kubectl auth can-i create pods -n cluster-admin --as=system:serviceaccount:cluster-admin:app-admin  # expected: no
```

---

### ✅ Markscheme

- [ ] ServiceAccount, Role, and RoleBinding exist in `cluster-admin`
- [ ] Role verbs and API groups exactly cover the requested permissions
- [ ] `admin-pod` uses the ServiceAccount and has a projected token mount
- [ ] Authorization checks allow requested actions
- [ ] Authorization check for creating Pods returns no/denied

---

## Q17 — NetworkPolicy Traffic Chain

**Instance:** `ssh ckad9999`<br>
**Namespace:** `network`<br>
**Concepts:** `networking`, `network-policies`, `security`

### Task

Create `web` and `api` Deployments using `nginx`, and `db` using `postgres` with `POSTGRES_HOST_AUTH_METHOD=trust`; label them `app=web`, `app=api`, and `app=db`. Enforce web→api and api→db only, denying all other Pod traffic.

---

### Solution

> [!tip] Exam-speed route
> Generate the three Deployments quickly, add the database environment variable, then spend your time on the NetworkPolicy YAML:
>
> ```bash
> kubectl create deploy web -n network --image=nginx
> kubectl create deploy api -n network --image=nginx
> kubectl create deploy db -n network --image=postgres
> kubectl set env deploy/db -n network POSTGRES_HOST_AUTH_METHOD=trust
> kubectl get pod -n network --show-labels
> ```
>
> Start policies with default-deny ingress **and** egress, then add only web→api and api→db. NetworkPolicy testing is meaningful only if the cluster CNI enforces it.

> [!attention] Personal weakness drill
> **NetworkPolicy + selectors:** build this from scratch. `podSelector` selects the Pods the policy applies to; `from`/`to` selectors select permitted peers. Check Deployment labels before policy logic.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: network
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: network
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  namespace: network
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: postgres
        image: postgres
        env:
        - name: POSTGRES_HOST_AUTH_METHOD
          value: trust
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: network
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-policy
  namespace: network
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: api
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: network
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Egress
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: network
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
```

---

### ✅ Markscheme

- [ ] All three Deployments exist with correct images, labels, and Postgres environment variable
- [ ] Default-deny ingress and egress applies to all Pods
- [ ] web egress/API ingress allow web→api
- [ ] api egress/DB ingress allow api→db
- [ ] All other Pod-to-Pod paths are denied

---

## Q18 — Rolling Update and Rollback

**Instance:** `ssh ckad9999`<br>
**Namespace:** `upgrade`<br>
**Concepts:** `deployments`, `rolling-updates`, `rollback`

### Task

Create Deployment `app-v1` with 4 replicas using `nginx:1.19`. Set max unavailable 1 and max surge 1, update to `nginx:1.20` with a recorded change cause, save rollout history to `/tmp/exam/rollout-history.txt`, then roll back to the previous version.

---

### Solution

> [!tip] Exam-speed route
> This whole workflow is faster imperatively. Use a change-cause annotation because `--record` is deprecated:
>
> ```bash
> kubectl create deploy app-v1 -n upgrade --image=nginx:1.19 --replicas=4
> kubectl patch deploy app-v1 -n upgrade -p '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":1,"maxSurge":1}}}}'
> kubectl set image deploy/app-v1 -n upgrade nginx=nginx:1.20
> kubectl annotate deploy/app-v1 -n upgrade kubernetes.io/change-cause='nginx 1.19 to 1.20' --overwrite
> kubectl rollout status deploy/app-v1 -n upgrade
> kubectl rollout history deploy/app-v1 -n upgrade > /tmp/exam/rollout-history.txt
> kubectl rollout undo deploy/app-v1 -n upgrade
> ```
>
> Always wait for rollout status before saving history or undoing.

> [!attention] Personal weakness drill
> **Patch + rollout commands:** practise the nested strategy patch, then `set image → annotate change-cause → status → history → undo`. Do not use deprecated `--record`.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
  namespace: upgrade
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: app-v1
  template:
    metadata:
      labels:
        app: app-v1
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
# Perform update
kubectl set image deployment/app-v1 nginx=nginx:1.20 -n upgrade
kubectl annotate deployment app-v1 -n upgrade kubernetes.io/change-cause='Update nginx from 1.19 to 1.20' --overwrite

# Save rollout history
kubectl rollout history deployment app-v1 -n upgrade > /tmp/exam/rollout-history.txt

# Rollback
kubectl rollout undo deployment/app-v1 -n upgrade
```

---

### ✅ Markscheme

- [ ] Initial Deployment uses 4 replicas and `nginx:1.19`
- [ ] RollingUpdate strategy uses max unavailable 1 and max surge 1
- [ ] Revision history records the update to `nginx:1.20`
- [ ] History output exists at the exact path
- [ ] Rollback completes and image returns to `nginx:1.19`

---

## Q19 — Priority and Pod Anti-Affinity

**Instance:** `ssh ckad9999`<br>
**Namespace:** `scheduling`<br>
**Concepts:** `scheduling`, `priority`, `anti-affinity`

### Task

Create priority classes valued 1000 and 100 and Pods `high-priority` and `low-priority` using them. Configure required Pod anti-affinity so they cannot share a node. Generate enough controlled resource pressure with `polinux/stress` workloads to observe higher-priority scheduling/preemption behavior.

---

### Solution

> [!tip] Exam-speed route
> PriorityClasses have an imperative command; anti-affinity still needs YAML:
>
> ```bash
> kubectl create priorityclass high-priority --value=1000 --description='high priority'
> kubectl create priorityclass low-priority --value=100 --description='low priority'
> kubectl run high-priority -n scheduling --image=nginx --dry-run=client -o yaml > /tmp/high.yaml
> kubectl get pod -n scheduling -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priorityClassName,NODE:.spec.nodeName
> kubectl get events -n scheduling --sort-by=.metadata.creationTimestamp | tail -20
> ```
>
> Increase stress replicas gradually; do not blindly exhaust the control plane. Scheduler pressure/preemption is driven by resource **requests**, not merely runtime CPU usage.

> [!attention] Personal weakness drill
> **JSONPath/custom-columns:** use the compact priority/node view instead of reading full YAML. This is also a PriorityClass and patch-syntax drill from the lab notes.

```text
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
---
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
  namespace: scheduling
  labels:
    priority: high  # ✅ Add this
spec:
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: priority
            operator: In
            values:
            - high
            - low
        topologyKey: kubernetes.io/hostname
---
apiVersion: v1
kind: Pod
metadata:
  name: low-priority
  namespace: scheduling
  labels:
    priority: low  # ✅ Add this
spec:
  priorityClassName: low-priority
  containers:
  - name: nginx
    image: nginx
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: priority
            operator: In
            values:
            - high
            - low
        topologyKey: kubernetes.io/hostname

# Example pressure workload (adjust replica count/requests to this lab's node capacity)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pressure
  namespace: scheduling
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pressure
  template:
    metadata:
      labels:
        app: pressure
    spec:
      containers:
      - name: stress
        image: polinux/stress
        command: ["stress"]
        args: ["-c", "4", "-m", "2", "--vm-bytes", "1G"]
        resources:
          requests:
            cpu: "1"
            memory: 1Gi
          limits:
            cpu: "2"
            memory: 1500Mi

# Observe scheduling/preemption; remove pressure after the test
kubectl get pods -n scheduling -o wide
kubectl get events -n scheduling --sort-by=.lastTimestamp
```

---

### ✅ Markscheme

- [ ] PriorityClasses have values 1000 and 100
- [ ] Each Pod references the correct PriorityClass and has matching labels
- [ ] Required anti-affinity uses hostname topology and prevents co-location
- [ ] Stress workload requests enough resources to create scheduler pressure
- [ ] Observed events/status demonstrate higher-priority behavior without destabilizing control-plane services

---

## Q20 — Troubleshoot Failing Deployment

**Instance:** `ssh ckad9999`<br>
**Namespace:** `troubleshoot`<br>
**Concepts:** `troubleshooting`, `debugging`, `deployment-configuration`

### Task

Fix existing Deployment `failing-app` while keeping image `nginx:1.25` and 3 replicas: change container port 8080→80, memory limit 64Mi→256Mi, and liveness-probe port 8080→80. Ensure all Pods run successfully.

---

### Solution

> [!tip] Exam-speed route
> This is a repair task: inspect first, then make one edit and watch the rollout:
>
> ```bash
> kubectl get deploy,pod -n troubleshoot
> kubectl describe pod -n troubleshoot -l app=failing-app
> kubectl edit deploy failing-app -n troubleshoot
> # Change the three incorrect values, save, then:
> kubectl rollout status deploy/failing-app -n troubleshoot
> kubectl get pod -n troubleshoot -l app=failing-app
> ```
>
> Edit the Deployment template, not an individual Pod; the controller would recreate a Pod from the broken template.

> [!attention] Personal weakness drill
> **Smallest-fix troubleshooting:** inspect first, edit the Deployment once, then watch rollout status. If the new Pods still fail, use `describe` and `logs --previous` rather than guessing.

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: failing-app
  namespace: troubleshoot
spec:
  replicas: 3
  selector:
    matchLabels:
      app: failing-app
  template:
    metadata:
      labels:
        app: failing-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80  # Fixed from 8080
        resources:
          limits:
            memory: 256Mi    # Fixed from 64Mi
        livenessProbe:
          httpGet:
            path: /
            port: 80        # Fixed from 8080
          initialDelaySeconds: 15
          periodSeconds: 10
```

---

### ✅ Markscheme

- [ ] Deployment remains named `failing-app`, uses `nginx:1.25`, and has 3 replicas
- [ ] Container port is 80
- [ ] Memory limit is `256Mi`
- [ ] Liveness probe checks port 80
- [ ] Rollout completes with 3 ready Pods and no repeated probe failures

---
