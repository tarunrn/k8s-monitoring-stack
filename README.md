# Kubernetes Monitoring Stack — Minikube + Prometheus + Grafana + ArgoCD

A production-grade Kubernetes observability and GitOps platform running locally at zero cost. Deploys a multi-service application on a local Kubernetes cluster with full monitoring, alerting, and automated GitOps-based continuous delivery.

---

## What This Project Does

This project sets up a complete Kubernetes platform that mirrors what engineering teams run in production:

- **Minikube** runs a real single-node Kubernetes cluster locally inside Docker
- **Prometheus** continuously scrapes metrics from all running pods and nodes
- **Grafana** visualizes those metrics in real-time dashboards (CPU, memory, disk, network)
- **ArgoCD** watches this GitHub repository and automatically deploys any changes to the cluster — no manual `kubectl apply` needed

---

## Architecture

```
GitHub Repo (source of truth)
        │
        ▼
   ArgoCD (GitOps)
        │  watches & syncs
        ▼
Kubernetes Cluster (Minikube)
        │
        ├── default namespace
        │     └── sample-app (2 nginx pods)
        │
        ├── monitoring namespace
        │     ├── Prometheus (metrics collection)
        │     ├── Grafana (dashboards)
        │     └── AlertManager (alerting)
        │
        └── argocd namespace
              └── ArgoCD (GitOps controller)
```

---

## Tech Stack

| Tool | Purpose | Cost |
|---|---|---|
| Minikube | Local Kubernetes cluster | Free |
| kubectl | Kubernetes CLI | Free |
| Helm | Package manager for K8s | Free |
| Prometheus | Metrics collection & storage | Free / Open Source |
| Grafana | Metrics visualization & dashboards | Free / Open Source |
| ArgoCD | GitOps continuous delivery | Free / Open Source |

**Total cost: $0**

---

## Prerequisites

- Docker Desktop (running)
- Homebrew (Mac)
- Git

---

## How to Run Locally

### 1. Install dependencies
```bash
brew install minikube kubectl helm
```

### 2. Start the Kubernetes cluster
```bash
minikube start --driver=docker
```

### 3. Deploy the sample app
```bash
kubectl apply -f app/deployment.yaml
kubectl get pods  # wait for 2 pods Running
```

### 4. Install Prometheus + Grafana
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### 5. Open Grafana
```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```
Open http://localhost:3000

Get the password:
```bash
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d
```

### 6. Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 7. Open ArgoCD
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Open https://localhost:8080

Get the password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### 8. Connect ArgoCD to this repo
In the ArgoCD UI create a new app pointing at this repo with path `app/` — ArgoCD will automatically sync and deploy any changes you push to GitHub.

---

## GitOps Demo

To see ArgoCD in action, change the replica count in `app/deployment.yaml`:

```yaml
spec:
  replicas: 3  # change from 2 to 3
```

Then push:
```bash
git add .
git commit -m "scale: increase replicas to 3"
git push
```

Within 30 seconds ArgoCD detects the change and automatically scales the deployment — no manual intervention needed.

---

## Grafana Dashboards

Once running, navigate to these dashboards in Grafana:

| Dashboard | What it shows |
|---|---|
| Node Exporter / Nodes | Live CPU, memory, disk, network for the cluster node |
| Kubernetes / Compute Resources / Namespace (Pods) | Per-pod CPU and memory usage |
| Kubernetes / Compute Resources / Cluster | Cluster-wide resource utilization |

---

## Key Technical Decisions

**Why Minikube over a cloud cluster?**
Minikube runs a real Kubernetes cluster using the same components as production (kubeadm, kubelet, etcd) — completely free. Every `kubectl` command works identically to AWS EKS or GCP GKE.

**Why Helm for Prometheus + Grafana?**
The `kube-prometheus-stack` Helm chart installs Prometheus, Grafana, AlertManager, and all required RBAC rules in a single command. Manually writing 2000+ lines of YAML would take days and be error-prone.

**Why ArgoCD for deployments?**
ArgoCD implements the GitOps pattern — the Git repository is the single source of truth for cluster state. This means deployments are auditable, reproducible, and rollbacks are as simple as reverting a git commit.

---

## Project Structure

```
k8s-monitoring-stack/
├── app/
│   └── deployment.yaml      # Kubernetes Deployment + Service for sample app
└── README.md
```

---

## What I Learned

- How to run and manage a real Kubernetes cluster locally
- How Prometheus scrapes metrics using ServiceMonitors and node exporters
- How to build Grafana dashboards for real-time infrastructure observability
- How GitOps works — using Git as the source of truth for Kubernetes state
- How ArgoCD detects drift between desired state (Git) and actual state (cluster)
