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
Import Dashboard ID: 7249
