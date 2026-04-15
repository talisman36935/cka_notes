# ✅ Troubleshooting

> [!success] Default exam approach
> Never guess first.  
> Always inspect first.

## Pod troubleshooting flow

Use these commands in order:

```bash
kubectl get pods
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl exec -it <pod> -- sh
```

Common problems:
- `CrashLoopBackOff`
- `ImagePullBackOff`
- `ErrImagePull`
- bad command / args
- readiness probe failures
- missing env vars
- bad Secret / ConfigMap reference
- scheduling failure

---

## Application failure flow

If the app is unreachable:
1. check Pod status
2. check Service
3. check endpoints
4. test DNS
5. test from inside another Pod
6. inspect container logs

Useful commands:

```bash
kubectl describe svc web-service
kubectl get endpoints web-service
kubectl logs web-pod
kubectl exec -it client-pod -- curl http://web-service
```

---

## Service failure flow

Use this every time:

```bash
kubectl get svc
kubectl describe svc <name>
kubectl get endpoints <name>
kubectl get pods --show-labels
kubectl exec -it <pod> -- nslookup <service>
```

If endpoints are empty:
- selector mismatch
- Pods not Ready
- wrong namespace
- no matching Pods

---

## Node troubleshooting flow

Useful commands:

```bash
kubectl get nodes
kubectl describe node node01
systemctl status kubelet
journalctl -u kubelet
```

Look for conditions such as:
- `Ready`
- `MemoryPressure`
- `DiskPressure`
- `PIDPressure`
- `NetworkUnavailable`

> [!tip] Heartbeat clue
> In a node description, the last heartbeat / status update can tell you if the node stopped talking to the control plane.

---

## Control plane troubleshooting flow

On kubeadm clusters:

```bash
ls /etc/kubernetes/manifests/
kubectl get pods -n kube-system
crictl ps
crictl logs <container-id>
journalctl -u kubelet
```

On non-static-pod setups, also check:

```bash
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
```

Useful checks:
- is the API server up?
- are manifest files valid?
- are certs correct?
- is etcd healthy?
- is kubelet running?

---

## Worker node troubleshooting flow

Useful commands:

```bash
kubectl get nodes
kubectl describe node worker-1
systemctl status kubelet
journalctl -u kubelet
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -text -noout
```
