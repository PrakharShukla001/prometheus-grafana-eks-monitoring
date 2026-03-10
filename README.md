<div align="center">

# 🔥 Prometheus & Grafana on AWS EKS

**Production-grade Kubernetes monitoring stack deployed using Helm**

![AWS EKS](https://img.shields.io/badge/AWS-EKS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)

</div>

---

## 🧱 Tech Stack

| Tool | Purpose |
|------|---------|
| ![EKS](https://img.shields.io/badge/AWS_EKS-FF9900?style=flat&logo=amazon-aws&logoColor=white) | Managed Kubernetes cluster on AWS |
| ![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white) | Metrics collection & alerting |
| ![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white) | Visualization & dashboards |
| ![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white) | Kubernetes package manager |
| ⚖️ **HPA** | Horizontal Pod Autoscaler |
| 💾 **EBS CSI Driver** | Persistent storage on AWS |

---

## 🚀 Setup Commands

### 1️⃣ Create EKS Cluster

```bash
eksctl create cluster \
  --name landing-transform-my-eks-cluster \
  --region us-east-1
```

---

### 2️⃣ Deploy Prometheus

```bash
# Add Prometheus Helm repo
helm repo add prometheus https://prometheus-community.github.io/helm-charts

# Install Prometheus
helm install prometheus-deployment prometheus/prometheus \
  --namespace prometheus \
  --set alertmanager.persistentVolume.storageClass=gp3 \
  --set server.persistentVolume.storageClass=gp3
```

---

### 3️⃣ Deploy Grafana

```bash
# Add Grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts

# Install Grafana
helm install grafana grafana/grafana \
  --namespace grafana \
  --set persistence.storageClassName=gp3 \
  --set persistence.enabled=true \
  --set adminPassword=secret \
  --set service.type=LoadBalancer \
  --values prometheus-datasource.yaml
```

---

### 4️⃣ Setup HPA (Horizontal Pod Autoscaler)

```bash
# Apply HPA manifest
kubectl apply -f hpa.yaml

# Autoscale deployment
kubectl autoscale deployment hpa-deploy \
  --cpu-percent=50 \
  --min=1 \
  --max=10
```

---

## 📊 Grafana Dashboards

> Yeh saare dashboard IDs the jo humne try kiye:

| # | Dashboard ID | Name | Status |
|---|-------------|------|--------|
| 1 | `6417` | Kubernetes Cluster Prometheus (Original) | — |
| 2 | `3119` | Kubernetes Cluster Monitoring | — |
| 3 | `13770` | Kubernetes All-in-One (Chinese) | — |
| 4 | `15661` | EKS Specific | — |
| 5 | `7249` | Kubernetes Cluster Summary | ✅ |
| 6 | `1860` | Node Exporter Full | — |
| 7 | `15757` | Kubernetes Cluster Monitoring | — |
| 8 | `3662` | Prometheus Stats | — |

---

<div align="center">

**Built with ❤️ using Prometheus · Grafana · AWS EKS · Helm**

</div>
