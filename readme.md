---
tags:
  - kubernetes
  - cka
  - study-guide
aliases:
  - CKA Guide
  - Certified Kubernetes Administrator Guide
---

> [!info] Session notes — 25 May 2026
>
> **Corrections to `Notes from labs.md`**
> - Cluster upgrade: fixed drain order (drain must come *after* `kubeadm upgrade apply`, not before); added `apt-mark unhold/hold` around every install; added `kubeadm upgrade node` for worker nodes and additional control planes
> - PV access modes: added `ReadWriteOncePod` (RWOP)
> - Reclaim policy: flagged `Recycle` as deprecated
> - HPA events: fixed `kubectl events` syntax to use `--for=hpa/<name>`
> - Ingress: removed false "no imperative command" claim; added `kubectl create ingress`
> - etcd backup: added `export ETCDCTL_API=3`; cert paths were already correct (`/etc/kubernetes/pki/etcd/`)
>
> **Corrections to `05-cluster-maintenance.md`**
> - etcd backup/restore: changed inline `ETCDCTL_API=3` to `export ETCDCTL_API=3`; fixed cert paths from `/etc/etcd/` to `/etc/kubernetes/pki/etcd/`; switched restore command from `etcdctl` to `etcdutl`
>
> **New file: [[Appendices/appendix-d-jsonpath-and-mental-models]]**
> - JSONPath deep reference: single-object vs list distinction, syntax table, 6 named patterns, 10 exam-critical copy-paste examples, custom columns and sort-by
> - 11 holistic mental models: label glue, volume binding name-as-key, PV→PVC→Pod chain, scheduler decision tree, network traffic path, auth pipeline, namespaced vs cluster-scoped, certificate hierarchy, static pod pattern, the control loop, NetworkPolicy AND/OR YAML trap
>
> **Gaps filled across existing files**
> - `01-core-concepts.md`: Jobs & CronJobs (completions, parallelism, backoffLimit, restartPolicy gotcha); StatefulSets (headless Service requirement, volumeClaimTemplates, ordinal naming, PVC persistence)
> - `02-scheduling.md`: Resource requests/limits (CPU units, OOM vs throttle); LimitRange; ResourceQuota (the "must declare requests when quota exists" trap)
> - `04-application-lifecycle.md`: Probes — all three types (liveness/readiness/startup), all mechanisms (httpGet/tcpSocket/exec), timing fields, readiness-doesn't-restart gotcha; rolling updates expanded with `kubectl set image`, `rollout pause/resume`, `--to-revision`; EncryptionConfiguration — full workflow (key generation, secretbox YAML, API server flag, force-reencrypt existing secrets, etcd verification)
> - `06-security.md`: CSR full workflow (openssl key gen → CSR object creation → approval → cert extraction → kubeconfig); Flannel explicitly named as non-enforcing CNI for NetworkPolicies
> - `11-helm-basics.md`: full rewrite — repo management, search, install/upgrade/rollback, release inspection
> - `12-kustomize-basics.md`: full rewrite — kustomization.yaml structure, Strategic Merge vs JSON6902 patches, base/overlay pattern, Components gotcha
>
> **New sections in `Notes from labs.md`**
> - Rolling updates: `kubectl set image`, `kubernetes.io/change-cause` annotation (replaces deprecated `--record`), `rollout status/history/undo`, JSONPath image check
> - Secrets: mounted as read-only volume — full pod YAML pattern, `readOnly: true`, busybox sleep command, `ls -la` dotfile gotcha
>
> **Addendum — sourced from github.com/vj2201/CKA-PREP-2025-v2**
> - Taints & Tolerations: new section — `kubectl taint`, toleration YAML, NoSchedule/NoExecute effects, pending-pod trap
> - Rolling updates expanded: `kubectl rollout restart` (required after ConfigMap edits); `kubectl scale --replicas=0` pattern for safe deployment editing
> - HPA: added `autoscaling/v2` YAML with `stabilizationWindowSeconds` for scale-down behaviour
> - StorageClass: `kubectl patch sc` to set/unset default StorageClass via annotation
> - Services: new NodePort section — explicit `nodePort:` in YAML, `kubectl patch deployment` to add containerPort
> - Gateway API: replaced partial TLS snippet with full Gateway + HTTPRoute YAML; added describe commands for verification
> - Helm: `helm template` to render manifests without installing
> - kubectl Advanced: `kubectl explain` with dotted path; JSONPath extracted into shell variable + `/etc/hosts` pattern
> - Observability: `journalctl -u kube-apiserver | tail` for control plane log access

# 🚀 CKA Study Guide

> [!info] What this guide is for
> This guide is built for the **Certified Kubernetes Administrator (CKA)** exam:
> - practical understanding over memorization
> - fast troubleshooting over perfect theory
> - hands-on commands you can type under pressure

> [!success] Winning mindset
> The exam is mostly:
> - identify the failing component quickly
> - verify your assumption with the right command
> - make the smallest correct fix
> - move on

## How to use this guide

1. Read each section until the flow makes sense.
2. Practice every command in a lab.
3. Use the **Speed Drills** appendix daily.
4. Use the **1-Week Crash Plan** appendix if your exam is close.

## Table of contents

### Study guide

- [[Study-guide/01-core-concepts|✅ Core Concepts]]
- [[Study-guide/02-scheduling|✅ Scheduling]]
- [[Study-guide/03-logging-monitoring|✅ Logging & Monitoring]]
- [[Study-guide/04-application-lifecycle|✅ Application Lifecycle Management]]
- [[Study-guide/05-cluster-maintenance|✅ Cluster Maintenance]]
- [[Study-guide/06-security|✅ Security]]
- [[Study-guide/07-storage|✅ Storage]]
- [[Study-guide/08-networking|✅ Networking]]
- [[Study-guide/09-troubleshooting|✅ Troubleshooting]]
- [[Study-guide/10-design-install|✅ Design and Install a Kubernetes Cluster]]
- [[Study-guide/11-helm-basics|✅ Helm Basics]]
- [[Study-guide/12-kustomize-basics|✅ Kustomize Basics]]
- [[Study-guide/13-exam-priority|✅ Exam Priority Guide]]
- [[Study-guide/14-final-rule|✅ Final Rule]]

### Appendices

- [[Appendices/appendix-a-speed-drills|⚡ Appendix A — Speed Drills]]
- [[Appendices/appendix-b-crash-plan|📅 Appendix B — 1-Week Crash Plan]]
- [[Appendices/appendix-c-debug-flows|🧠 Appendix C — Default Debug Flows]]
- [[Appendices/appendix-d-jsonpath-and-mental-models|🧭 Appendix D — JSONPath & Mental Models]]

### Practice and reference

- [[Practice-notes/notes-from-labs|🧪 Notes from Labs]]
- [[Practice-notes/preparation-notes|📝 Preparation Notes]]
- [[Reference/CKA resource linkage|🔗 CKA Resource Linkage]]

### Quick reference

- [[Quick-reference/CKA Cheat Sheet - Pass|📌 CKA Cheat Sheet — Pass]]
- [[Quick-reference/final-reminder|🏁 Final Reminder]]
