# 🧠 Appendix C — Default Debug Flows

## Pod broken

```bash
kubectl get pods
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
```

## Service broken

```bash
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>
kubectl get pods --show-labels
```

## DNS broken

```bash
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes
kubectl get pods -n kube-system
kubectl get svc -n kube-system
```

## Node broken

```bash
kubectl get nodes
kubectl describe node <node>
systemctl status kubelet
journalctl -u kubelet
```

## Control plane broken

```bash
ls /etc/kubernetes/manifests/
crictl ps
crictl logs <id>
journalctl -u kubelet
```

## Storage broken

```bash
kubectl get pvc
kubectl describe pvc <claim>
kubectl get pv
kubectl get storageclass
```
