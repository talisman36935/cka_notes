# NetworkPolicy Exam Tips & Recipes

> [!tip] Exam goal
> Translate the sentence **“allow source A to reach destination B on port P”** into selectors without getting lost in the YAML.

Based on [Ahmet Balkan’s Kubernetes Network Policy Recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes) and the [official Kubernetes NetworkPolicy documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

---

## 1. The mental model that prevents most mistakes

```text
source Pod --egress--> destination Pod --ingress--> container port
```

For a connection to work:

1. The source Pod’s **egress** must allow it, if that Pod is egress-isolated.
2. The destination Pod’s **ingress** must allow it, if that Pod is ingress-isolated.
3. The Service must select a ready destination Pod, if connecting through a Service.
4. The NetworkPolicy-capable CNI must actually enforce policies.

> [!important] Policies are additive
> Multiple policies do not override one another. Kubernetes takes the **union of all allowed traffic** from every policy selecting the Pod. There is no policy priority and no “last rule wins”.

> [!warning] There is no explicit `deny` rule
> You isolate a Pod for a direction, then list what is allowed. Anything not allowed is denied.

---

## 2. Read every policy in this order

```yaml
metadata:
  namespace: shop       # 1. Where does this policy live?
spec:
  podSelector:          # 2. Which destination/source Pods does it control?
    matchLabels:
      app: api
  policyTypes:          # 3. Which direction becomes isolated?
  - Ingress
  ingress:              # 4. What is allowed in that direction?
  - from:
    - podSelector:      # 5. Which peers are allowed?
        matchLabels:
          app: web
    ports:              # 6. Which destination port is allowed?
    - protocol: TCP
      port: 8080
```

Say it aloud:

> “In namespace `shop`, select Pods labelled `app=api`, isolate their ingress, and allow Pods labelled `app=web` in the same namespace to connect to destination TCP port 8080.”

---

## 3. Selector rules worth memorising

| YAML location | Meaning |
|---|---|
| `spec.podSelector` | Pods the policy applies **to** |
| `ingress[].from[].podSelector` | Allowed source Pods |
| `egress[].to[].podSelector` | Allowed destination Pods |
| `namespaceSelector` | Namespaces selected by namespace labels |
| `ipBlock` | CIDR range; normally use for external addresses, not ephemeral Pod IPs |
| `ports` | Destination ports, not the client’s random source port |

An empty selector has meaning:

```yaml
podSelector: {}          # every Pod in this policy's namespace
namespaceSelector: {}    # every namespace
```

### Same namespace versus another namespace

`podSelector` alone in `from` or `to` selects Pods in the **policy’s namespace**:

```yaml
from:
- podSelector:
    matchLabels:
      app: web
```

To select Pods in another namespace, use both selectors in the **same list item**:

```yaml
from:
- namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: frontend
  podSelector:
    matchLabels:
      app: web
```

That means `frontend namespace AND app=web`.

This indentation means something different:

```yaml
from:
- namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: frontend
- podSelector:
    matchLabels:
      app: web
```

That means `any Pod in frontend namespace OR app=web in the policy’s namespace`.

> [!danger] High-value exam trap
> Same list item = **AND**. Separate `-` list items = **OR**.

---

## 4. Recipe: default deny everything

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: shop
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

No `ingress` or `egress` rules means no traffic is allowed in those directions.

> [!warning] DNS will stop
> A default-deny egress policy blocks DNS queries too. If later tests use Service names, allow DNS egress.

---

## 5. Recipe: allow DNS after default-deny egress

First discover the DNS labels rather than assuming them:

```bash
kubectl get svc -n kube-system --show-labels
kubectl get pod -n kube-system --show-labels | grep -E 'dns|coredns'
```

Common CoreDNS policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: shop
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

Use the labels present in the exam cluster. Some clusters label CoreDNS differently.

---

## 6. Recipe: web → api → db only

Assume all Pods are in namespace `shop` and have labels `app=web`, `app=api`, and `app=db`.

### Allow web egress to api

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-to-api
  namespace: shop
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes: [Egress]
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 80
```

### Allow api ingress from web and egress to db

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-chain
  namespace: shop
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - protocol: TCP
      port: 5432
```

### Allow db ingress from api

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-from-api
  namespace: shop
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
```

Apply these after the default-deny policy. Add DNS egress only if tests use names.

---

## 7. Recipe: allow a named namespace

Modern Kubernetes automatically labels namespaces with their name:

```bash
kubectl get ns --show-labels
```

Allow all Pods from namespace `frontend` to reach `app=api` on TCP 8080:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-from-frontend-namespace
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
    ports:
    - protocol: TCP
      port: 8080
```

If the task asks for only particular Pods in that namespace, add a `podSelector` beside the `namespaceSelector` in the same item.

---

## 8. Recipe: allow external CIDR except one subnet

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/8
      except:
      - 10.20.0.0/16
  ports:
  - protocol: TCP
    port: 443
```

Use `ipBlock` cautiously for Pod/Service traffic because address translation can vary by CNI and Service implementation.

---

## 9. Fast creation and inspection workflow

There is no useful imperative `kubectl create networkpolicy` command. Start from a memorised skeleton or copy a nearby policy:

```bash
kubectl get netpol -n shop
kubectl get netpol -n shop -o yaml
kubectl describe netpol <name> -n shop
kubectl get pod -n shop --show-labels -o wide
kubectl get svc,endpoints -n shop
```

Validate before applying:

```bash
kubectl apply --dry-run=server -f policy.yaml
kubectl apply -f policy.yaml
kubectl describe netpol -n shop
```

`kubectl describe` is especially useful for checking whether indentation created AND or OR selector behaviour.

---

## 10. Test matrix: prove allowed and denied paths

Create a disposable client if one is not supplied:

```bash
kubectl run test-client -n shop --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl exec -n shop test-client -- wget -qO- --timeout=2 http://api-svc
kubectl exec -n shop test-client -- nslookup api-svc
```

For a chain, test both positive and negative cases:

| From | To | Expected |
|---|---|---|
| web | api | allow |
| web | db | deny |
| api | db | allow |
| api | web | deny |
| db | api | deny |

Use short timeouts so denied traffic does not waste exam time:

```bash
kubectl exec -n shop <client> -- wget -qO- -T 2 http://<service>
kubectl exec -n shop <client> -- nc -vz -w 2 <service> <port>
```

Tool availability varies by image. Use `wget` with BusyBox, `curl` with curl images, or `nc` when installed.

---

## 11. Find whether the CNI enforces NetworkPolicy

```bash
kubectl get pod -n kube-system -o wide
kubectl get daemonset -n kube-system
kubectl get crd | grep -Ei 'cilium|calico|antrea'
```

Common clues:

| Pods/DaemonSets | Likely CNI |
|---|---|
| `calico-node` | Calico |
| `cilium` | Cilium |
| `antrea-agent` | Antrea |
| `kube-flannel-ds` | Flannel; enforcement depends on setup/extensions |

> [!important] Behaviour beats naming
> The quickest proof is a harmless default-deny test in the task namespace followed by a connectivity test. Creating a NetworkPolicy object does not prove it is enforced.

---

## 12. Ten exam traps

1. Forgetting that NetworkPolicies are namespaced.
2. Assuming `podSelector` in a peer selects Pods across namespaces.
3. Using separate list items when the task requires namespace **AND** Pod labels.
4. Forgetting the source egress side after applying default-deny egress.
5. Forgetting the destination ingress side.
6. Blocking DNS and mistaking the result for a Service or selector failure.
7. Using the Service port when the policy needs the destination Pod/container port.
8. Forgetting that policies add together; an older allow policy may keep traffic open.
9. Debugging policy before checking Service endpoints and Pod labels.
10. Assuming the CNI enforces NetworkPolicy without testing.

---

## 13. Thirty-second checklist

- [ ] Correct namespace on the NetworkPolicy
- [ ] `spec.podSelector` selects the intended controlled Pods
- [ ] Correct `policyTypes`
- [ ] Peer selector is in the correct namespace scope
- [ ] AND/OR indentation is intentional
- [ ] Destination port is correct
- [ ] DNS allowed if egress is isolated and names are used
- [ ] Source egress and destination ingress both considered
- [ ] Existing policies checked because allows are additive
- [ ] Positive and negative paths tested with a short timeout
