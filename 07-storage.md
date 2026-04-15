---
tags:
  - kubernetes
  - cka
  - storage
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

```bash
kubectl get pv
kubectl describe pv <pv-name>
```

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

```bash
kubectl get pvc
kubectl describe pvc <claim-name>
```

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

```bash
kubectl get storageclass
kubectl describe storageclass <name>
```

> [!important] Mental model
> PVC + StorageClass → provisioner creates PV dynamically