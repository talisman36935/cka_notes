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

```
/opt/cni/bin
/etc/cni/net.d/
```

Useful commands:

```bash
ls /opt/cni/bin
ls /etc/cni/net.d
ip addr
ip route
```

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

```bash
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>
kubectl get pods -o wide
```

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

```bash
kubectl get pods -n kube-system
kubectl get svc -n kube-system
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes
kubectl exec -it <pod> -- nslookup myservice.mynamespace.svc.cluster.local
```

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

```bash
kubectl get ingress
kubectl describe ingress <name>
```

---

## Gateway API

Gateway API is a newer, more expressive networking API than classic Ingress.

For CKA:
- understand the concept
- Ingress is still more foundational