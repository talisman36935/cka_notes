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