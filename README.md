# GitHub Actions-Based Continuous Deployment with Kubernetes and KEDA

## Overview
This project demonstrates a modular and automated approach for **Continuous Deployment (CD)** to a Kubernetes cluster using **GitHub Actions**, along with **KEDA** for event-driven autoscaling.

It includes:
- GitHub Actions-based CI/CD pipelines
- Helm charts for KEDA and demo applications
- Load testing and autoscaling via Prometheus metrics

---

## 🔧 Host Environment
- **Operating System**: Linux Ubuntu (for stability, compatibility, and ease of use in DevOps tools)
- **Cluster Platform**: Minikube (runs Kubernetes locally for development/testing)
- **Container Engine**: Docker (required to run Minikube in Docker driver mode)

---

## ⚙️ Cluster Setup

### Step 1: Install Prerequisites

- **Install Docker**  
  [Docker Installation Guide](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

- **Install Minikube**  
  [Minikube Installation Guide](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download)

- **Install kubectl**  
  [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

### Step 2: Start Minikube with Docker Driver
```bash
minikube start --driver=docker
```

---

## 🚀 GitHub Actions Setup

### Why GitHub Actions?
- Seamless GitHub integration for automated CD on push events.
- Customizable and scalable workflows using YAML.

### Why Self-Hosted Runner?
- Required to access Minikube on `localhost`.
- This config can work with any Kubernetes cluster via `kubeconfig`.

### Add a Self-Hosted Runner (Ubuntu)
- Reference: [GitHub Docs](https://docs.github.com/en/actions/hosting-your-own-runners)

### Add Kubeconfig as GitHub Secret
```bash
cat ~/.kube/config | base64 -w 0
```
Use the base64 output as the secret value in GitHub (e.g., `KUBECONFIG_DATA`).

---

## 📦 Deploy Prometheus (Metrics Collection)
- Prometheus deployed via Helm.
- Port-forwarded service to access dashboard:
```bash
kubectl port-forward svc/prometheus-server 9090:80 -n default
```

---

## 📈 Deploy KEDA (Autoscaler)
- **Helm Chart**: [KEDA Chart Repo](https://github.com/prabhun2023/Devops-Mini-Projects/tree/main/charts/keda)
- **Version**: 2.14.2

```bash
helm dependency update ./charts/keda
helm upgrade --install keda ./charts/keda --namespace keda --create-namespace
```

---

## 🌐 Deploy Demo Application (NGINX)
- **Helm Chart**: [NGINX Chart](https://github.com/prabhun2023/Devops-Mini-Projects/tree/main/charts/nginx)
- Namespace: `ns-apps`
- Accessible via port-forwarding:
```bash
kubectl port-forward svc/nginx-service 5500:80 -n ns-apps
```
Access NGINX: [http://localhost:5500](http://localhost:5500)

---

## 📊 ScaledObject Configuration
- Monitors CPU usage using Prometheus query.
- Defined in Helm values:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-scaledobject
  namespace: ns-apps
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    name: nginx
  minReplicaCount: 1
  maxReplicaCount: 10
  cooldownPeriod: 10
  pollingInterval: 20
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-server.default.svc.cluster.local:80
      metricName: nginx_cpu_usage_percentage
      query: |
        sum(rate(container_cpu_usage_seconds_total{namespace="ns-apps", pod=~"nginx-.*"}[3m])) / count(rate(container_cpu_usage_seconds_total{namespace="ns-apps", pod=~"nginx-.*"}[3m])) * 1000
      threshold: "20.0"
```

---

## 📥 GitHub Actions Workflows

### 1. **Automated CD Workflow**
- Triggered on push to `main` branch if `charts/*` are updated
- Deploys:
  - KEDA
  - All Helm charts in `charts/`
  - Verifies deployments
- [Workflow Run Link](https://github.com/prabhun2023/Devops-Mini-Projects/actions/runs/14560513835/job/40843178991)

### 2. **Manual Chart Deployment**
- Triggered manually by user
- Inputs: `chart_name`, `namespace`
- [Workflow YAML](https://github.com/prabhun2023/Devops-Mini-Projects/blob/main/.github/workflows/Manual_CD_Automation.yaml)
- [Workflow Run](https://github.com/prabhun2023/Devops-Mini-Projects/actions/runs/14557501554/job/40836376103)

---

## 📚 Workflow Usage Guide

### For Manual Deployment
1. Go to GitHub Actions
2. Select **Manual CD Automation** workflow
3. Provide input: `chart_name`, `namespace`
4. Run workflow
5. View deployment logs

---

## 🧪 Load Testing with Hey

### Why "hey" tool?
Lightweight HTTP load generator for performance testing web applications.

### Why run inside Minikube?
- External requests hit only port-forwarded pod (no load-balancing).
- Running inside ensures access via ClusterIP and balances across replicas.

### Installation & Execution
```bash
docker exec -it <minikube-container-id> /bin/bash
wget https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64
mv ./hey_linux_amd64 /usr/local/bin/hey
chmod 777 /usr/local/bin/hey

# Run test
hey -z 600s -c 500 http://<serviceClusterIP>:<ServicePort>
```

---

## 📉 CPU Usage & Autoscaling Logic
- Query:
```bash
sum(rate(container_cpu_usage_seconds_total{namespace="ns-apps", pod=~"nginx-.*"}[3m])) / count(rate(container_cpu_usage_seconds_total{namespace="ns-apps", pod=~"nginx-.*"}[3m])) * 100
```
- NGINX CPU limit = 25m = 2.5% of 1 core
- Threshold = 2% (20m) → 80% of limit
- KEDA triggers scale-out if average usage > 2%
- Automatically scales down when usage drops below threshold for cooldown period

---

## 📊 Prometheus Dashboard
[Open Dashboard](http://localhost:9090/query)

---

## ✅ Summary
This project showcases a powerful combination of:
- GitHub Actions for CI/CD
- Minikube + Docker for local K8s testing
- Prometheus for metrics
- KEDA for autoscaling
- Helm for templated, reusable deployments

All configured to enable scalable, modular, and reproducible deployments.

