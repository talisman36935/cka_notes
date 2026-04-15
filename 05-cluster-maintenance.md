# ✅ Cluster Maintenance

## OS upgrades and node maintenance

Useful commands:

```bash
kubectl cordon node01
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon node01
```

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

```bash
kubeadm upgrade plan
kubeadm upgrade apply v1.29.0
kubectl get nodes
```

### Worker node upgrade flow

1. drain node
2. upgrade `kubeadm`
3. run `kubeadm upgrade node`
4. upgrade `kubelet`
5. restart kubelet
6. uncordon node

Commands:

```bash
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
kubeadm upgrade node
systemctl restart kubelet
kubectl uncordon node01
```

> [!important] One-minor-version rule
> In practice, upgrade one minor version at a time.

---

## Backup and restore methods

### What to back up

- YAML files in Git
- imperative cluster objects
- `etcd` snapshots

### API object backup

```bash
kubectl get all --all-namespaces -o yaml > all-resources.yaml
```

### `etcd` backup

Authenticated example:

```bash
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/etcd-server.crt \
  --key=/etc/etcd/etcd-server.key
```

### `etcd` restore

```bash
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
  --data-dir /var/lib/etcd-from-backup
```

After restore:
- point `etcd` to restored data directory
- update the static Pod manifest if needed
- restart kubelet / control plane components as required

> [!warning] Exam trap
> Snapshot restore alone is not enough if the running `etcd` manifest still points to the old data directory.