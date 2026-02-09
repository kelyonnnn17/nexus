# Project Nexus: GitOps-Based Kubernetes Platform

**Project Status:** Phase 1 (Containerization) & Phase 2 (GitOps Automation) Complete.

## 1. Project Overview

Project Nexus is a demonstration of a modern, self-healing cloud-native infrastructure. It implements a **GitOps** workflow, where the state of the infrastructure is defined declaratively in version control (Git) and enforced automatically by an operator (ArgoCD) running within the Kubernetes cluster.

### Architecture

The system follows a pull-based GitOps architecture:

1. **Source Code:** A Python Flask microservice hosted on GitHub.
2. **Artifact Registry:** Docker Hub hosts the container images.
3. **Infrastructure:** A local Minikube cluster simulating a production environment.
4. **Continuous Deployment:** ArgoCD watches the GitHub repository for changes to the Kubernetes manifests and synchronizes the cluster state to match the repository.

## 2. Technical Stack

* **Application:** Python 3.9, Flask
* **Containerization:** Docker (Multi-architecture support for AMD64/ARM64)
* **Orchestration:** Kubernetes (Minikube)
* **Continuous Deployment:** ArgoCD (GitOps Operator)
* **Version Control:** GitHub

## 3. Implementation Details

### Phase 1: Containerization & Artifact Management

The application was packaged using Docker. A critical requirement was ensuring cross-platform compatibility between Apple Silicon (ARM64) development machines and standard Linux servers (AMD64).

**Key Technical Decision:**
We utilized `docker buildx` to generate a multi-architecture image. This ensures the image runs natively on both Minikube (running on Mac) and standard cloud instances without emulation errors (`exec format error`).

**Build Command:**

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t kelyonnnn17/nexus-app:v1 --push .

```

### Phase 2: Infrastructure as Code (Kubernetes)

The infrastructure is defined in the `k8s/` directory:

* **deployment.yaml:** Defines a `Deployment` object managing 2 replicas of the application. It utilizes environment variables (e.g., `BG_COLOR`) to inject configuration at runtime.
* **service.yaml:** Defines a `Service` of type `LoadBalancer`, exposing port 80 to the outside world.

### Phase 3: GitOps Automation (ArgoCD)

Manual deployment (`kubectl apply`) was replaced by ArgoCD. ArgoCD was installed into the `argocd` namespace and configured to monitor the repository.

**Installation Strategy:**
We utilized the **Server-Side Apply** method during installation to bypass the client-side validation limits often encountered with large manifests like ArgoCD Custom Resource Definitions (CRDs).

**Installation Command:**

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts

```

**Application Configuration (`application.yaml`):**
This manifest connects the cluster to the Git repository. It enables **Auto-Sync** and **Self-Healing**, ensuring that if a resource is manually deleted or modified in the cluster, ArgoCD immediately reverts it to the state defined in Git.

## 4. Operational Workflow

The system is designed for "Zero-Touch" deployment. Developers do not require direct access to the cluster to release updates.

### How to Deploy Changes

1. **Modify Configuration:**
Edit the `k8s/deployment.yaml` file locally (e.g., change `BG_COLOR` from `blue` to `green`).
2. **Push to Git:**
Commit and push the changes to the `main` branch.
```bash
git add .
git commit -m "update: change background color to green"
git push origin main

```


3. **Automatic Synchronization:**
ArgoCD detects the commit (polling interval is typically 3 minutes, or instant via webhook). It compares the desired state (Git) vs. the live state (Cluster) and updates the application pods.
4. **Verification:**
Refresh the application URL. The new configuration is applied with zero downtime (rolling update).

## 5. Accessing the Environment

To interact with the local Minikube environment, port forwarding is required.

**Application Access:**

```bash
minikube service nexus-service

```

**ArgoCD Dashboard Access:**

```bash
minikube service argocd-server -n argocd

```

* **Username:** `admin`
* **Password Retrieval:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

```



## 6. Troubleshooting Guide

| Issue | Potential Cause | Resolution |
| --- | --- | --- |
| **ImagePullBackOff** | Docker image name typo or private repo permissions. | Verify image name in `deployment.yaml`. Ensure repo is Public on Docker Hub. |
| **Exec Format Error** | Architecture mismatch (AMD64 vs ARM64). | Rebuild image using the `docker buildx` command with both platforms specified. |
| **ArgoCD "OutOfSync"** | Cluster state differs from Git state. | Click "Sync" in the ArgoCD UI or check if Auto-Sync is enabled in `application.yaml`. |
| **Connection Refused** | Minikube tunnel is not running. | Ensure `minikube service <service-name>` is running in a separate terminal. |

## 7. Future Roadmap

* **Phase 3 (Chaos Engineering):** Introduction of chaos agents to randomly terminate pods, validating the reliability of the self-healing mechanism.
* **Phase 4 (Observability):** Implementation of Prometheus and Grafana to visualize metrics such as request latency and error rates.