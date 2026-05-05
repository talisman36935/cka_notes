# 🚀 CKA Cheat Sheet — Pass (1-page)

Use this for **speed**: find the failing component → verify with the right command → apply the smallest fix.

https://learn.kodekloud.com/user/courses/udemy-labs-certified-kubernetes-administrator-with-practice-tests/module/8e4261a6-bac4-4dfe-82c5-0a1bb8c527da/lesson/82f88613-8cc6-458d-8e29-e55154890fc7

---

## 0) Immediate cluster checks (always start here)
```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods -A  #for all namespaces
kubectl get --raw /healthz
kubectl get --raw /readyz?verbose
kubectl get events -A --sort-by=.metadata.creationTimestamp
# example: if lab says namespace is dev -> kubectl get events -n dev --sort-by=.metadata.creationTimestamp
```

---

## 1) Imperative create + YAML generation + edit (exam speed)
```bash
# Pods
kubectl run nginx --image=nginx
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Deployments
kubectl create deployment web --image=nginx
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl scale deployment web --replicas=3

# Services
kubectl expose deployment web --port=80 --type=ClusterIP
kubectl expose deployment web --port=80 --type=NodePort
kubectl expose deployment web --port=80 --dry-run=client -o yaml > svc.yaml

# Edit + apply
kubectl edit deploy web
kubectl apply -f pod.yaml
kubectl apply -f deploy.yaml
```

---

## 2) The default debug flows (don’t guess)

### Pods broken
```bash
kubectl get pods
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl exec -it <pod> -- sh
```

### Services broken (always check endpoints)
```bash
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>
kubectl get pods --show-labels
kubectl get pods -o wide
```
If endpoints are empty → selector mismatch, wrong namespace, or Pods not Ready.

### App unreachable / connect failed
```bash
kubectl describe svc <svc>
kubectl get endpoints <svc>
kubectl logs <pod>
# test from inside another Pod
kubectl exec -it <client-pod> -- curl http://<service>
```

### DNS broken
```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes
kubectl exec -it <pod> -- nslookup <svc>.<ns>.svc.cluster.local
kubectl get pods -n kube-system
kubectl get svc -n kube-system
```

### Node broken
```bash
kubectl get nodes
kubectl describe node <node>
systemctl status kubelet
journalctl -u kubelet
```

### Control plane broken (kubeadm/static pods)
```bash
ls /etc/kubernetes/manifests/
kubectl get pods -n kube-system
crictl ps
crictl logs <container-id>
journalctl -u kubelet
```

### Storage broken
```bash
kubectl get pvc
kubectl describe pvc <claim>
kubectl get pv
kubectl get storageclass
```

---

## 3) Rollouts + scaling + rollbacks
```bash
kubectl scale deployment <name> --replicas=<n>
kubectl rollout status deployment <name>
kubectl rollout history deployment <name>
kubectl rollout undo deployment <name>
```

---

## 4) Namespaces + context shortcuts
```bash
kubectl get ns
kubectl create ns dev
kubectl get pods -n kube-system
kubectl config set-context --current --namespace=dev

# common selector shortcuts
-n <ns>
-A
```

---

## 5) Labels / selectors (high-frequency)
```bash
kubectl label pod nginx env=prod
kubectl get pods -l env=prod
kubectl get pods --show-labels
```

---

## 6) Placement: scheduling controls
### Node labels / selectors
```bash
kubectl label node node01 size=Large
kubectl get nodes --show-labels
```

### Taints + tolerations
```bash
kubectl taint nodes node01 app=blue:NoSchedule
kubectl taint nodes node01 app=blue:NoExecute
kubectl taint nodes node01 app-
```
Reminder: taints/tolerations control *permission to be scheduled*, not a guarantee.

### DaemonSets + static Pods
```bash
kubectl get daemonsets -A
kubectl describe daemonset <name>

ls /etc/kubernetes/manifests/
crictl ps
journalctl -u kubelet
```

---

## 7) Logs + runtime details (the fastest truth)
```bash
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs -f <pod>
kubectl logs <pod> --previous

# node-level / runtime
journalctl -u kubelet
crictl ps
crictl logs <container-id>
```

---

## 8) Metrics Server + HPA
```bash
kubectl top node
kubectl top pod
kubectl get apiservices | grep metrics

kubectl autoscale deployment <app> --cpu-percent=50 --min=1 --max=5
kubectl get hpa
kubectl describe hpa <name>
```
Install Metrics Server (if missing):
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## 9) JSONPath output (high-yield for speed)
```bash
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

---

## 10) RBAC + auth debugging (permissions screwdriver)
```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods
kubectl create rolebinding read-pods --role=pod-reader --user=user1
kubectl get roles
kubectl get rolebindings
kubectl describe role <role>
kubectl describe rolebinding <binding>

kubectl auth can-i get pods
kubectl auth can-i create deployments --as <user>
kubectl auth can-i list pods -n <ns> --as <user>

# service accounts
kubectl create serviceaccount <sa>
kubectl get sa
kubectl describe sa <sa>
kubectl create token <sa>
```

---

## 11) TLS / certificates (CKA-grade fundamentals)
```bash
kubectl get csr
kubectl describe csr <name>
kubectl certificate approve <name>

# kubeadm debugging angle
openssl x509 -in <cert-path> -text -noout
```

---

## 12) NetworkPolicy (common gotcha)
```bash
kubectl get networkpolicy
kubectl describe networkpolicy <name>
```
Gotcha: NetworkPolicy only works if your CNI **enforces** it.

---

## 13) Storage: PV / PVC / StorageClass (binding failures checklist)
```bash
kubectl get pv
kubectl describe pv <pv>
kubectl get pvc
kubectl describe pvc <claim>
kubectl get storageclass
kubectl describe storageclass <sc>
```
Common PVC Pending causes: no matching PV, wrong storageClassName, access mode mismatch, size too large, or dynamic provisioner missing.

---

## 14) Cluster maintenance: cordon / drain / etcd backup
### Drain safely
```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```
Draining can fail due to DaemonSets, local data, or PodDisruptionBudgets.

### etcd backup/restore (highest-value topic)
```bash
ETCDCTL_API=3 etcdctl snapshot save snapshot.db
ETCDCTL_API=3 etcdctl snapshot status snapshot.db --write-out=table

ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup
```
After restore: ensure manifests/static pods point to the restored data dir.

---

## 15) Rolling/declaring with tooling (know the command)
- Kustomize:
```bash
kubectl apply -k .
```
- Helm:
```bash
helm install <release> <chart>
helm upgrade <release> <chart>
helm list
helm uninstall <release>
helm show values <chart>
```
