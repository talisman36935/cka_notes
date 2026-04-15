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

```bash
kubeadm init
kubeadm token create --print-join-command
kubeadm join <...>
```

---

## `etcd` in HA

### Quorum

Quorum formula:

```
(total nodes / 2) + 1
```

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