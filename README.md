# ⚙️ codemask-gitops

> **Kubernetes manifests for CodeMask — managed via GitOps with ArgoCD.**

This repository contains the Kubernetes manifests for deploying [CodeMask](https://github.com/shubham-singhS2/CodeMask) on a K3s cluster. It follows the GitOps model — ArgoCD watches this repo and automatically syncs any changes to the cluster.

**You should never run `kubectl apply` manually after initial setup. All deployments happen by pushing to this repo.**

---

## How It Works

```
CodeMask CI/CD pipeline (GitHub Actions)
        │
        │  updates image tag in deployment.yaml
        ▼
codemask-gitops repo (this repo)
        │
        │  ArgoCD polls every ~3 minutes (or via webhook)
        ▼
K3s Cluster
        │
        └── Rolling update → new pod → app live
```

The only thing that changes in this repo during normal operation is the image tag in `deployment.yaml`. Every deploy is a git commit — giving you a full audit trail of every deployment.

---

## Repository Structure

```
codemask-gitops/
├── namespace.yaml      # codemask namespace
├── deployment.yaml     # Deployment — image tag updated by CI/CD
├── service.yaml        # NodePort service (port 30080)
└── argocd-app.yaml     # ArgoCD Application definition
```

---

## Cluster Setup

### Prerequisites
- Two VMs with K3s installed (VM1 = server, VM2 = agent)
- ArgoCD installed in the `argocd` namespace
- `kubectl` access to the cluster

### K3s Setup

**VM1 (control plane):**
```bash
curl -sfL https://get.k3s.io | sh -
cat /var/lib/rancher/k3s/server/node-token   # copy this for VM2
```

**VM2 (worker):**
```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<VM1_IP>:6443 \
  K3S_TOKEN=<token-from-VM1> sh -
```

**Verify:**
```bash
kubectl get nodes
# NAME   STATUS   ROLES                  AGE
# vm1    Ready    control-plane,master   1m
# vm2    Ready    <none>                 30s
```

### ArgoCD Setup

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl get pods -n argocd -w

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Access UI via NodePort
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort"}}'

kubectl get svc -n argocd argocd-server
# Note the NodePort — access at https://VM1_IP:<nodeport>
```

---

## Initial Deployment

Apply manifests once manually to bootstrap:

```bash
git clone https://github.com/shubham-singhS2/codemask-gitops.git
cd codemask-gitops

kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f argocd-app.yaml   # connects ArgoCD to this repo
```

After this, ArgoCD takes over — never run `kubectl apply` again for normal deployments.

---

## Verify Deployment

```bash
# Check pods are running
kubectl get pods -n codemask

# Check service
kubectl get svc -n codemask

# Access the app
http://VM1_IP:30080
```

---

## ArgoCD Application

The `argocd-app.yaml` configures ArgoCD to:
- Watch **this repo** (`codemask-gitops`) on the `main` branch
- Sync to the `codemask` namespace on the cluster
- **Auto-sync** — deploys automatically when this repo changes
- **Self-heal** — reverts any manual `kubectl` changes back to what's in this repo
- **Prune** — removes resources that are deleted from this repo

---

## How Deployments Work

1. Developer pushes code to [CodeMask](https://github.com/shubham-singhS2/CodeMask)
2. GitHub Actions builds a new Docker image tagged with the commit SHA
3. GitHub Actions updates `deployment.yaml` in **this repo** with the new tag:
   ```yaml
   image: shubhamsinghs2/codemask:a3f4c1d  # ← updated automatically
   ```
4. ArgoCD detects the change and syncs to the cluster
5. K3s performs a rolling update — zero downtime

---

## Manual Sync (if needed)

ArgoCD auto-syncs every ~3 minutes. To force an immediate sync:

```bash
# Via CLI
argocd app sync codemask

# Or click "Sync" in the ArgoCD UI
```

---

## Related

- **App repo:** [CodeMask](https://github.com/shubham-singhS2/CodeMask)
- **Docker Hub:** [shubhamsinghs2/codemask](https://hub.docker.com/r/shubhamsinghs2/codemask)

---

Built by [Shubham Singh](https://shubham-singh.in) · DevSecOps Engineer
