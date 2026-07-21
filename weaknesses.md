# CKA Weaknesses — Expanded Exam Workbook

> [!info] How to use this file
> These are the original weak areas, expanded into compact explanations, commands, traps, and drills. Practise the command blocks until the first useful command is automatic.

---

## 1. NetworkPolicy

Deep-dive companion: [[NetworkPolicy Exam Tips]]

Core rules:

- Policies are namespaced and select Pods using labels.
- `spec.podSelector` selects the Pods controlled by the policy.
- `ingress.from` selects allowed sources; `egress.to` selects allowed destinations.
- There are no explicit deny rules. Isolate a direction, then allow exceptions.
- Policies are additive; one policy cannot cancel another policy’s allow.
- When both sides are isolated, source egress **and** destination ingress must allow the flow.
- Default-deny egress also blocks DNS.

```bash
kubectl get netpol -A
kubectl describe netpol <name> -n <namespace>
kubectl get pod -n <namespace> --show-labels
kubectl get svc,endpoints -n <namespace>
```

> [!tip] Drill
> Build default-deny, web→api, and api→db policies from scratch. Test every allowed and denied path with a two-second timeout.

Reference: [ahmetb/kubernetes-network-policy-recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes)

---

## 2. `command` versus `args`

Kubernetes maps these to the image configuration:

| Kubernetes | Container image field | Meaning |
|---|---|---|
| `command` | `ENTRYPOINT` | Executable or wrapper to run |
| `args` | `CMD` | Arguments passed to the command |

```yaml
containers:
- name: worker
  image: busybox
  command: ["sh", "-c"]
  args: ["while true; do date; sleep 5; done"]
```

Rules:

- Setting only `args` keeps the image entrypoint and replaces its default arguments.
- Setting `command` replaces the image entrypoint.
- Shell features such as pipes, redirects, `&&`, and variables require a shell such as `sh -c`.
- With `sh -c`, place the whole shell program in one string.

```bash
kubectl run cmd-test --image=busybox --restart=Never --command -- sh -c 'echo hello; sleep 3600'
kubectl get pod cmd-test -o jsonpath='{.spec.containers[0].command}{"\n"}{.spec.containers[0].args}{"\n"}'
```

---

## 3. Services: ClusterIP, NodePort, `expose`, and `create service`

| Requirement | Fast command |
|---|---|
| Internal Service for a Deployment | `kubectl expose deployment` |
| Internal Service without an existing workload | `kubectl create service clusterip` |
| Node-level exposure | `kubectl expose ... --type=NodePort` or `kubectl create service nodeport` |
| Exact `nodePort` value | Generate YAML and add `nodePort:` |

```bash
kubectl expose deployment my-deployment \
  --type=NodePort \
  --port=80 \
  --target-port=8080 \
  --name=my-service

kubectl create service clusterip internal-svc --tcp=80:8080
kubectl create service nodeport external-svc --tcp=80:8080
```

Remember:

- `port` is the Service port.
- `targetPort` is the destination Pod/container port.
- `nodePort` is the port opened on every node and is normally assigned automatically.
- A Service routes using labels, not a direct reference to a Deployment.

```bash
kubectl get svc,endpoints
kubectl describe svc my-service
kubectl get pod --show-labels
```

If endpoints are empty, check selector labels, namespace, and Pod readiness before checking networking.

---

## 4. JSONPath

Use JSONPath when you need one value, a list, or custom formatted output for a file.

```bash
# One item from a list
kubectl get pods -o=jsonpath='{.items[0].metadata.name}{"\n"}'

# A field on a named object: no .items[]
kubectl get pod nginx -o=jsonpath='{.spec.nodeName}{"\n"}'

# All items
kubectl get pods -o=jsonpath='{.items[*].metadata.name}{"\n"}'

# One row per item
kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Labels from the first item
kubectl get pods -o=jsonpath='{.items[0].metadata.labels}{"\n"}'

# Escape a dot in a label key
kubectl get nodes -o=jsonpath='{.items[0].metadata.labels.kubernetes\.io/hostname}{"\n"}'
```

Mental model:

```text
kubectl get pods      -> List object -> .items[0] needed
kubectl get pod NAME  -> Single Pod  -> start at .metadata/.spec/.status
```

> [!warning] `[0]` trap
> `[0]` means the first entry in an array. It is not needed when retrieving one named object.

Debug a path incrementally:

```bash
kubectl get pod nginx -o json | jq '.spec'
kubectl get pod nginx -o json | jq '.spec.containers'
kubectl get pod nginx -o json | jq -r '.spec.containers[0].image'
```

Official reference: [kubectl JSONPath support](https://kubernetes.io/docs/reference/kubectl/jsonpath/)

---

## 5. Custom columns

Use custom columns when you want a readable table with stable headings.

```bash
kubectl get service \
  -o=custom-columns='NAME:.metadata.name,IP:.spec.clusterIP'

kubectl get pod -A \
  -o=custom-columns='NS:.metadata.namespace,NAME:.metadata.name,NODE:.spec.nodeName,PHASE:.status.phase'

kubectl get pv \
  -o=custom-columns='NAME:.metadata.name,CAPACITY:.spec.capacity.storage,SC:.spec.storageClassName'
```

Use `--no-headers` when the output must contain only values:

```bash
kubectl get pod --no-headers \
  -o=custom-columns='NAME:.metadata.name,NODE:.spec.nodeName'
```

### JSONPath versus custom columns

| Need | Choose |
|---|---|
| One exact value | JSONPath |
| Loop with tabs/newlines | JSONPath `range` |
| Human-readable table | custom columns |
| Complex filtering/transformation | `jq` |

---

## 6. `jq`

`jq` reads JSON; use `-r` for raw text without quotation marks.

```bash
kubectl get pods -o json | jq -r '.items[].metadata.name'

kubectl get pods -o json | jq -r \
  '.items[] | [.metadata.name, .spec.nodeName, .status.phase] | @tsv'

kubectl get pods -o json | jq -r \
  '.items[] | select(.status.phase != "Running") | .metadata.name'

kubectl get deploy -o json | jq -r \
  '.items[] | .metadata.name as $n | .spec.template.spec.containers[] | [$n, .name, .image] | @tsv'
```

> [!tip] Fast rule
> Use JSONPath for familiar short paths. Switch to `jq` when filtering with `select`, restructuring arrays, or handling optional values becomes awkward.

---

## 7. `tr`

`tr` translates or deletes characters from a stream.

```bash
# Remove newlines from base64 output
cat /root/CKA/john.csr | base64 | tr -d '\n'

# Convert a Pod IP into the dashed Pod-DNS form
printf '%s' '10.42.1.7' | tr '.' '-'

# Lowercase to uppercase
printf '%s\n' 'ready' | tr '[:lower:]' '[:upper:]'
```

Prefer the base64 tool’s no-wrap option when it is available, but `tr -d '\n'` is portable and easy to remember.

---

## 8. CertificateSigningRequests

Typical workflow:

```bash
# 1. Create private key and CSR
openssl genrsa -out john.key 2048
openssl req -new -key john.key -out john.csr -subj '/CN=john/O=developers'

# 2. Encode CSR on one line
CSR=$(base64 < john.csr | tr -d '\n')
```

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john
spec:
  request: REPLACE_WITH_BASE64_CSR
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
```

```bash
# 3. Create, inspect, approve
kubectl apply -f john-csr.yaml
kubectl get csr
kubectl describe csr john
kubectl certificate approve john

# 4. Extract issued certificate
kubectl get csr john -o jsonpath='{.status.certificate}' | base64 -d > john.crt
openssl x509 -in john.crt -noout -subject -issuer -dates
```

Traps:

- `spec.request` contains the base64-encoded CSR, not the private key.
- `signerName` and `usages` are required in `certificates.k8s.io/v1`.
- Approval does not always guarantee signing; check whether `.status.certificate` is populated.

---

## 9. Helm charts

Fast investigation sequence:

```bash
helm list -A
helm repo list
helm repo update
helm search repo <repo>/<chart> --versions | head
helm show values <repo>/<chart> | less
```

Install and verify:

```bash
helm install web-release bitnami/nginx \
  -n helm-test --create-namespace \
  --set replicaCount=2 \
  --set service.type=NodePort \
  --wait

helm status web-release -n helm-test
helm get values web-release -n helm-test
```

Upgrade and rollback:

```bash
helm upgrade web-release bitnami/nginx -n helm-test --reuse-values --set replicaCount=3
helm history web-release -n helm-test
helm rollback web-release 1 -n helm-test
```

Local chart checks:

```bash
helm lint ./chart
helm template test ./chart > /tmp/rendered.yaml
```

> [!warning] Exam trap
> Release name, chart name, repository name, namespace, and chart version are different things. Read the task twice before running `helm upgrade`.

---

## 10. HorizontalPodAutoscaler

```bash
kubectl autoscale deployment <deployment> \
  --cpu-percent=50 --min=1 --max=10

kubectl get hpa
kubectl describe hpa <name>
kubectl top pod
```

The target Deployment needs CPU requests for utilization-based scaling:

```bash
kubectl set resources deployment wordpress \
  --limits=cpu=200m,memory=512Mi \
  --requests=cpu=100m,memory=256Mi
```

If the HPA target is `<unknown>`:

1. Check `kubectl top pod`.
2. Confirm metrics-server/APIService health.
3. Confirm CPU requests exist.
4. Wait briefly for metrics collection.
5. Check `kubectl describe hpa` events.

> [!warning] API-version requirement
> `kubectl autoscale` may generate a newer HPA API version. If the task explicitly requires `autoscaling/v1`, use the requested YAML shape.

---

## 11. `kubectl set resources`

```bash
kubectl set resources deployment wordpress \
  --limits=cpu=200m,memory=512Mi \
  --requests=cpu=100m,memory=256Mi

kubectl get deploy wordpress \
  -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
```

For a multi-container workload, specify the container:

```bash
kubectl set resources deployment app -c sidecar --requests=cpu=50m,memory=64Mi
```

---

## 12. `kubectl patch`

Patch is fastest when only a small nested field changes.

### Merge patch

```bash
kubectl patch deployment web -p \
  '{"spec":{"replicas":3}}'

kubectl patch storageclass old-default -p \
  '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

Read the nesting from left to right:

```text
spec → template → spec → containers
metadata → annotations → annotation-name
```

### JSON patch

```bash
kubectl patch deployment web --type=json -p='[
  {"op":"replace","path":"/spec/replicas","value":3}
]'
```

Safe workflow:

```bash
kubectl get deploy web -o yaml > /tmp/web-before.yaml
kubectl patch deploy web --dry-run=server -o yaml -p '{"spec":{"replicas":3}}'
kubectl patch deploy web -p '{"spec":{"replicas":3}}'
kubectl get deploy web -o yaml
```

> [!warning] Lists are the hard part
> Strategic-merge behaviour can merge lists by fields such as container `name`; JSON patch uses numeric array paths. When a list patch becomes confusing, use `kubectl edit` or generate the intended YAML instead.

---

## 13. Rollout restart, status, history, and undo

```bash
kubectl rollout restart deployment/<name> -n <namespace>
kubectl rollout status deployment/<name> -n <namespace>
kubectl rollout history deployment/<name> -n <namespace>
kubectl rollout undo deployment/<name> -n <namespace>
```

ConfigMap/Secret changes do not automatically restart existing Pods. Restart the Deployment when the task requires Pods to consume updated mounted/env configuration.

For recorded change causes, use an annotation rather than deprecated `--record`:

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl annotate deployment/web \
  kubernetes.io/change-cause='nginx 1.26 to 1.27' --overwrite
kubectl rollout status deployment/web
```

---

## 14. DNS structure

Service DNS:

```text
<service>.<namespace>.svc.<cluster-domain>
web.default.svc.cluster.local
```

Inside the same namespace, `web` normally works because the Pod search list includes its namespace.

Pod DNS commonly uses the dashed Pod IP:

```text
<dashed-pod-ip>.<namespace>.pod.cluster.local
10-42-1-7.default.pod.cluster.local
```

StatefulSet/headless Service identity:

```text
<pod-name>.<headless-service>.<namespace>.svc.cluster.local
web-0.web-svc.stateful.svc.cluster.local
```

Debug in this order:

```bash
kubectl exec <pod> -- cat /etc/resolv.conf
kubectl exec <pod> -- nslookup kubernetes.default.svc.cluster.local
kubectl exec <pod> -- nslookup <service>
kubectl get svc,endpoints -n <namespace>
kubectl get pod -n kube-system -l k8s-app=kube-dns
```

If the FQDN works but the short name fails, inspect search domains. If neither works, check CoreDNS and NetworkPolicy DNS egress.

---

## 15. Find the active CNI

```bash
kubectl get pod -n kube-system -o wide
kubectl get daemonset -n kube-system
kubectl get crd | grep -Ei 'calico|cilium|antrea'
```

On a node, when access is allowed:

```bash
ls -1 /etc/cni/net.d/
grep -R '"type"' /etc/cni/net.d/
```

Common identifiers:

| Identifier | Likely CNI |
|---|---|
| `calico-node` | Calico |
| `cilium` | Cilium |
| `antrea-agent` | Antrea |
| `kube-flannel-ds` | Flannel |

Do not assume NetworkPolicy support from the presence of NetworkPolicy API objects. Test enforcement.

---

## 16. `crictl` essentials

`crictl` talks directly to the container runtime through CRI. It is useful when Kubernetes components or Pods are unhealthy and `kubectl logs` is insufficient.

```bash
crictl ps                 # running containers
crictl ps -a              # include exited containers
crictl pods               # pod sandboxes
crictl images
crictl inspect <container-id>
crictl inspectp <pod-id>
crictl logs <container-id>
crictl logs --tail=50 <container-id>
crictl stats
```

Fast static/control-plane debugging:

```bash
crictl ps -a --name kube-apiserver
crictl logs $(crictl ps -a --name kube-apiserver -q | head -1)
```

If endpoint warnings waste time, inspect `/etc/crictl.yaml` or pass the runtime endpoint specified by the lab.

---

## 17. Daily weakness drills

### Five-minute output drill

- [ ] Print the first Pod name with JSONPath.
- [ ] Print every Pod as `name`, `node`, `phase` using `range`.
- [ ] Produce the same output with custom columns.
- [ ] Filter non-Running Pods with `jq`.
- [ ] Convert an IP to dashed DNS form with `tr`.

### Five-minute networking drill

- [ ] Create a Deployment and expose it as ClusterIP.
- [ ] Expose another Deployment as NodePort.
- [ ] Diagnose an empty Endpoints object.
- [ ] Create default-deny ingress and egress.
- [ ] Add DNS egress and one allowed application path.

### Five-minute operations drill

- [ ] Set requests and limits imperatively.
- [ ] Create an HPA and inspect its events.
- [ ] Patch a nested field after a server-side dry run.
- [ ] Restart a Deployment and watch rollout status.
- [ ] Inspect a failing runtime container with `crictl`.
