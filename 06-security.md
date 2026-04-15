---
tags:
  - kubernetes
  - cka
  - security
---

# ✅ Security

## Security primitives

Kubernetes security is built around:
- securing hosts
- authenticating access
- authorizing actions
- controlling network traffic
- securing component communication with TLS

---

## Authentication

Authentication answers: **who are you?**

Common methods:
- certificates
- tokens
- external identity providers
- service accounts for workloads

---

## Authorization

Authorization answers: **what can you do?**

Common models:
- RBAC
- Node authorizer
- webhook
- ABAC (older)

---

## TLS in Kubernetes

Kubernetes components communicate using TLS.

Common cert directory on kubeadm clusters:

```
/etc/kubernetes/pki/
```

Inspect a certificate:

```bash
openssl x509 -in cert.crt -text -noout
```

> [!tip] Useful troubleshooting angle
> Control plane failures are often caused by:
> - wrong cert path
> - expired certificate
> - SAN mismatch
> - wrong CA

---

## Certificates API

Kubernetes can manage certificate requests via `CertificateSigningRequest` objects.

Useful commands:

```bash
kubectl get csr
kubectl describe csr <name>
kubectl certificate approve <name>
```

---

## Kubeconfig

A kubeconfig contains:
- clusters
- users
- contexts

Useful commands:

```bash
kubectl config view
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context-name>
```

---

## API groups

Core group example:

```
/api/v1
```

Named groups examples:

```
/apis/apps/v1
/apis/networking.k8s.io/v1
/apis/rbac.authorization.k8s.io/v1
```

Useful commands:

```bash
kubectl api-resources
kubectl api-versions
```

---

## RBAC

### Role and RoleBinding

Roles are namespace-scoped.

Useful commands:

```bash
kubectl create role pod-reader --verb=get,list,watch --resource=pods
kubectl create rolebinding read-pods --role=pod-reader --user=user1
kubectl get roles
kubectl get rolebindings
kubectl describe role pod-reader
kubectl describe rolebinding read-pods
```

### `kubectl auth can-i`

This is one of the best RBAC debugging commands.

```bash
kubectl auth can-i get pods
kubectl auth can-i create deployments --as dev-user
kubectl auth can-i list pods -n dev --as dev-user
```

### ClusterRole and ClusterRoleBinding

Use these for:
- cluster-scoped resources like `nodes`, `persistentvolumes`
- cross-namespace access

Useful commands:

```bash
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

> [!important] Exam memory hook
> - `Role` / `RoleBinding` → one namespace
> - `ClusterRole` / `ClusterRoleBinding` → cluster-wide or cluster-scoped resources

---

## Service Accounts

Service accounts are identities for Pods and automation.

Useful commands:

```bash
kubectl create serviceaccount dashboard-sa
kubectl get sa
kubectl describe sa dashboard-sa
kubectl create token dashboard-sa
```

Assign to a Pod:

```yaml
serviceAccountName: dashboard-sa
```

Disable token mount:

```yaml
automountServiceAccountToken: false
```

> [!note] Modern Kubernetes behavior
> Recent Kubernetes versions no longer automatically create long-lived service account token Secrets the old way. Prefer `kubectl create token` and projected tokens.

---

## Image security

Private registries usually require image pull secrets.

Useful command:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<password>
```

Then reference it with:

```yaml
imagePullSecrets:
  - name: regcred
```

---

## Security contexts

Security contexts let you define runtime security settings at:
- Pod level
- container level

Examples:
- `runAsUser`
- `runAsNonRoot`
- capabilities
- filesystem permissions

Example:

```yaml
securityContext:
  runAsUser: 1000
  capabilities:
    add: ["MAC_ADMIN"]
```

> [!tip] Precedence rule
> Container-level security context overrides pod-level settings for that container.

---

## Network Policies

NetworkPolicies control allowed ingress and/or egress for selected Pods.

Critical rules:
- if **no** policy selects a Pod, traffic is allowed by default
- once a policy selects a Pod for ingress/egress, that traffic becomes isolated
- only explicitly allowed traffic passes

Example pattern:
- allow only API Pods to reach DB Pods on port 3306

Useful commands:

```bash
kubectl get networkpolicy
kubectl describe networkpolicy <name>
```

> [!warning] Huge gotcha
> A NetworkPolicy only works if your CNI plugin **enforces** it.  
> Some CNIs support policies; some do not.

---

## CRDs, custom controllers, and operators

### CRD
Adds a new resource type to the Kubernetes API.

### Custom controller
Watches resources and reconciles state.

### Operator
A more advanced controller pattern for managing applications.

For CKA:
- know what they are
- hands-on depth is usually lower than networking/storage/troubleshooting