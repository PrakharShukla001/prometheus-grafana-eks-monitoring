# Prometheus & Grafana Monitoring on AWS EKS

Production-grade Kubernetes monitoring stack deployed using Helm.

## Tech Stack
- AWS EKS
- Prometheus
- Grafana
- Helm
- HPA (Horizontal Pod Autoscaler)
- EBS CSI Driver

## Setup Commands

### Create EKS Cluster
eksctl create cluster --name landing-transform-my-eks-cluster --region us-east-1

### Deploy Prometheus
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm install prometheus-deployment prometheus/prometheus --namespace prometheus --set alertmanager.persistentVolume.storageClass=gp3 --set server.persistentVolume.storageClass=gp3

### Deploy Grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm install grafana grafana/grafana --namespace grafana --set persistence.storageClassName=gp3 --set persistence.enabled=true --set adminPassword=secret --set service.type=LoadBalancer --values prometheus-datasource.yaml

### Setup HPA
kubectl apply -f hpa.yaml
kubectl autoscale deployment hpa-deploy --cpu-percent=50 --min=1 --max=10

## Grafana Dashboard

Yeh saare dashboard IDs the jo humne try kiye:

Dashboard 1 — 6417 (Original - Kubernetes Cluster Prometheus)
ID-6417
Dashboard 2 — 3119 (Kubernetes cluster monitoring)
ID-3119
Dashboard 3 — 13770 (Kubernetes all-in-one - Chinese)
ID-13770
Dashboard 4 — 15661 (EKS specific)
ID-15661
Dashboard 5 — 7249 (Kubernetes Cluster Summary ✅)
ID-7249
Dashboard 6 — 1860 (Node Exporter Full)
ID-1860
Dashboard 7 — 15757 (Kubernetes Cluster Monitoring)
ID-15757
Dashboard 8 — 3662 (Prometheus Stats)
ID-3662
