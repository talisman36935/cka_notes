# ✅ Logging & Monitoring

## Managing application logs

Kubernetes does not centrally manage logs by default.

Most app logs go to:
- `stdout`
- `stderr`

Useful commands:

```bash
kubectl logs pod-name
kubectl logs pod-name -c container-name
kubectl logs -f pod-name
kubectl logs pod-name --previous
```

> [!tip] Use `--previous`
> If a container crashed and restarted, `--previous` often shows the real failure.

---

## Node-level and control plane logs

Container logs often appear under:

```
/var/log/containers/
```

Kubelet logs:

```bash
journalctl -u kubelet
```

Control plane / runtime logs:

```bash
crictl ps
crictl logs <container-id>
```

---

## Metrics Server

Metrics Server provides basic resource metrics used by:

- `kubectl top`
- HPA

Useful commands:

```bash
kubectl top node
kubectl top pod
kubectl get apiservices | grep metrics
```

Install example:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

> [!warning] Common lab issue
> HPA won't work if Metrics Server is missing or broken.