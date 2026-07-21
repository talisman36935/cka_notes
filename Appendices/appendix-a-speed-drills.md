# ⚡ Appendix A — Speed Drills

> [!tip] Goal
> Type these without thinking.

## Pods

```bash
kubectl run nginx --image=nginx
kubectl run nginx --image=nginx --dry-run=client -o yaml
kubectl exec -it nginx -- sh
kubectl delete pod nginx
```

## Deployments

```bash
kubectl create deployment web --image=nginx
kubectl scale deployment web --replicas=3
kubectl expose deployment web --port=80 --type=NodePort
kubectl rollout status deployment web
```

## YAML generation

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl expose deployment web --port=80 --dry-run=client -o yaml > svc.yaml
```

## Namespaces

```bash
kubectl create ns dev
kubectl run nginx --image=nginx -n dev
kubectl config set-context --current --namespace=dev
```

## Labels

```bash
kubectl label pod nginx env=prod
kubectl get pods -l env=prod
kubectl get pods --show-labels
```

## Logs

```bash
kubectl logs pod
kubectl logs -f pod
kubectl logs pod --previous
kubectl logs pod -c container
```

## Debugging

```bash
kubectl describe pod pod
kubectl get events --sort-by=.metadata.creationTimestamp
```

## Nodes

```bash
kubectl cordon node01
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon node01
```

## RBAC

```bash
kubectl create role reader --verb=get,list --resource=pods
kubectl create rolebinding read-pods --role=reader --user=user1
kubectl auth can-i get pods --as=user1
```

## Secrets and ConfigMaps

```bash
kubectl create secret generic mysecret --from-literal=password=123
kubectl get secret mysecret -o yaml
kubectl create configmap myconfig --from-literal=key=value
```

## Service debugging

```bash
kubectl get svc
kubectl describe svc myservice
kubectl get endpoints myservice
```

## Networking tests

```bash
kubectl exec -it pod -- nslookup kubernetes
kubectl exec -it pod -- curl http://myservice
```

## Storage

```bash
kubectl get pv
kubectl get pvc
kubectl describe pvc myclaim
kubectl get storageclass
```

## `etcd`

```bash
ETCDCTL_API=3 etcdctl snapshot save backup.db
ETCDCTL_API=3 etcdctl snapshot restore backup.db --data-dir /var/lib/etcd-from-backup
```

## `crictl`

```bash
crictl ps
crictl ps -a
crictl logs <id>
```

## kubectl shortcuts

```bash
alias k=kubectl
```

---

## 30-second drills checklist

- [ ] create Pod YAML
- [ ] create Deployment YAML
- [ ] scale a Deployment
- [ ] read logs from a crashing Pod
- [ ] check why a Pod is Pending
- [ ] fix a Service with empty endpoints
- [ ] drain and uncordon a node
- [ ] create a Role and RoleBinding
- [ ] back up `etcd`
- [ ] find control plane logs with `crictl`
