# Nexus: Self-Healing GitOps Platform

Nexus is a hands-on GitOps reference project for deploying a Flask microservice on Kubernetes with automatic reconciliation, self-healing, chaos testing, and monitoring.

## What this project demonstrates

- GitOps deployment with ArgoCD (auto-sync + self-heal)
- Kubernetes app deployment with Kustomize
- Observability stack with Prometheus + Grafana
- Chaos engineering with Chaos Mesh
- End-to-end workflow from Git commit to live cluster update

## Architecture

- **App**: Flask service (`app/`)
- **Container**: Docker image (`kelyonnnn17/nexus-app:v1` by default)
- **Cluster**: Minikube
- **GitOps**: ArgoCD Applications
  - `nexus-app` -> `k8s/`
  - `nexus-monitoring` -> `monitoring/`
  - `nexus-chaos` -> `chaos/`

## Repository structure

```text
app/                    Flask app + Dockerfile
k8s/                    Base app deployment/service manifests
monitoring/             Grafana datasource + PrometheusRule + dashboards
chaos/                  PodChaos + chaos schedule manifests
argocd/                 ArgoCD Application manifests for monitoring + chaos
application.yaml        ArgoCD Application manifest for core app
```

## Prerequisites

Install these locally:

- Docker Desktop
- Minikube
- kubectl
- Helm v3+

## Quick start

### 1) Start local Kubernetes

```powershell
minikube start --driver=docker
kubectl get nodes
```

If cluster access breaks after restart:

```powershell
minikube update-context
```

### 2) Install ArgoCD

```powershell
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts
kubectl get pods -n argocd
```

### 3) Install monitoring dependencies (Prometheus Operator CRDs)

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install kube-prom-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl get crd | findstr monitoring.coreos.com
```

### 4) Install chaos dependencies (Chaos Mesh CRDs)

```powershell
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update
helm upgrade --install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --create-namespace --set chaosDaemon.runtime=containerd --set chaosDaemon.socketPath=/run/containerd/containerd.sock
kubectl get crd | findstr chaos-mesh.org
```

### 5) Point ArgoCD manifests to your fork/branch

Update these files before applying:

- `application.yaml`
- `argocd/application-monitoring.yaml`
- `argocd/application-chaos.yaml`

Set:

- `spec.source.repoURL` -> your GitHub repo URL
- `spec.source.targetRevision` -> your branch (for example `main`)

### 6) Apply GitOps applications

```powershell
kubectl apply -f application.yaml
kubectl apply -f argocd/application-monitoring.yaml
kubectl apply -f argocd/application-chaos.yaml
kubectl get application -n argocd -o wide
```

Expected: all apps eventually show **Synced** and **Healthy/Progressing**.

## Accessing services

### App URL

```powershell
kubectl -n nexus port-forward svc/nexus-service 18080:80
```

Open: `http://127.0.0.1:18080`

### ArgoCD UI

```powershell
minikube service argocd-server -n argocd
```

Login:

- Username: `admin`
- Password (initial):

```powershell
$b64 = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($b64))
```

### Grafana UI (optional)

```powershell
kubectl -n monitoring port-forward svc/kube-prom-stack-grafana 3000:80
```

Open: `http://127.0.0.1:3000`

## Validate GitOps + self-healing

### GitOps change propagation

1. Edit a value in `k8s/deployment.yaml` (for example `BG_COLOR`).
2. Commit and push.
3. ArgoCD auto-syncs and rolls pods.

### Self-healing check

```powershell
kubectl delete pod -n nexus -l app=nexus-app
kubectl get pods -n nexus -w
```

Pods are recreated automatically.

### Chaos schedule check

```powershell
kubectl get schedule,podchaos -n chaos-mesh
kubectl get pods -n nexus -w
```

## Troubleshooting

- **`kubernetes cluster unreachable`**
  - Run: `minikube update-context`
  - Then: `minikube start --driver=docker`
- **Argo app OutOfSync due to missing CRDs**
  - Ensure Helm dependency stacks above are installed first
- **`minikube service` on Windows not reachable**
  - Use `kubectl port-forward` instead
- **ArgoCD password command fails with `base64 not recognized` on Windows**
  - Use the PowerShell decoding snippet shown above

## Contributing

Contributions are welcome. Please open an issue or PR with a clear description of:

- what changed
- why it is needed
- how it was tested

---

If you use this repository for learning or workshops, feel free to fork and adapt it.