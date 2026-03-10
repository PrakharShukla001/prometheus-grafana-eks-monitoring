# Prometheus & Grafana Monitoring on AWS EKS

> Production-grade Kubernetes monitoring stack deployed using Helm.

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| AWS EKS | Managed Kubernetes cluster |
| Prometheus | Metrics collection & alerting |
| Grafana | Visualization & dashboards |
| Helm | Kubernetes package manager |
| HPA | Horizontal Pod Autoscaler |
| EBS CSI Driver | Persistent storage (gp3) |

---

## Prerequisites

Make sure these tools are installed before starting:

```bash
eksctl version
kubectl version
helm version
aws --version
```

Also verify AWS credentials:

```bash
aws sts get-caller-identity
```

---

## Setup Commands

### Step 1 — Create EKS Cluster

```bash
# Create cluster
eksctl create cluster \
  --name landing-transform-my-eks-cluster \
  --region us-east-1

# Update kubeconfig
aws eks update-kubeconfig \
  --name landing-transform-my-eks-cluster \
  --region us-east-1

# Verify nodes
kubectl get nodes
```

---

### Step 2 — Install EBS CSI Driver (gp3 Storage)

```bash
# Associate OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster landing-transform-my-eks-cluster \
  --region us-east-1 \
  --approve

# Create IAM service account
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster landing-transform-my-eks-cluster \
  --region us-east-1 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole

# Install EBS CSI addon
aws eks create-addon \
  --cluster-name landing-transform-my-eks-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole \
  --region us-east-1
```

`storageclass-gp3.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f storageclass-gp3.yaml
```

---

### Step 3 — Deploy Prometheus

```bash
# Add Helm repo
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace + install
kubectl create namespace prometheus

helm install prometheus-deployment prometheus/prometheus \
  --namespace prometheus \
  --set alertmanager.persistentVolume.storageClass=gp3 \
  --set server.persistentVolume.storageClass=gp3

# Verify pods
kubectl get pods -n prometheus
```

---

### Step 4 — Create Grafana Datasource Config

`prometheus-datasource.yaml`:

```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-deployment-server.prometheus.svc.cluster.local
        isDefault: true
        access: proxy
        jsonData:
          timeInterval: "5s"
```

---

### Step 5 — Deploy Grafana

```bash
# Add Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create namespace + install
kubectl create namespace grafana

helm install grafana grafana/grafana \
  --namespace grafana \
  --set persistence.storageClassName=gp3 \
  --set persistence.enabled=true \
  --set adminPassword=secret \
  --set service.type=LoadBalancer \
  --values prometheus-datasource.yaml

# Get external URL
kubectl get svc grafana -n grafana
```

> **Login:** `admin` / `secret`

---

### Step 6 — Setup HPA (Horizontal Pod Autoscaler)

```bash
# Apply deployment
kubectl apply -f hpa.yaml

# Configure autoscaling
kubectl autoscale deployment hpa-deploy \
  --cpu-percent=50 \
  --min=1 \
  --max=10

# Verify
kubectl get hpa
```

`hpa.yaml` (sample):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-deploy
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-deploy
  template:
    metadata:
      labels:
        app: hpa-deploy
    spec:
      containers:
        - name: hpa-app
          image: nginx:latest
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

---

### Step 7 — Import Grafana Dashboards

| # | Dashboard ID | Name |
|---|-------------|------|
| 1 | `6417` | Kubernetes Cluster Prometheus |
| 2 | `3119` | Kubernetes Cluster Monitoring |
| 3 | `13770` | Kubernetes All-in-One |
| 4 | `15661` | EKS Specific |
| 5 | `7249` | Kubernetes Cluster Summary ✅ |
| 6 | `1860` | Node Exporter Full |
| 7 | `15757` | Kubernetes Cluster Monitoring v2 |
| 8 | `3662` | Prometheus Stats |

Import via Grafana API:

```bash
GRAFANA_URL="http://<GRAFANA_URL>"

for ID in 6417 3119 13770 15661 7249 1860 15757 3662; do
  curl -s -X POST "$GRAFANA_URL/api/dashboards/import" \
    -H "Content-Type: application/json" \
    -u "admin:secret" \
    -d "{
      \"inputs\": [{
        \"name\": \"DS_PROMETHEUS\",
        \"type\": \"datasource\",
        \"pluginId\": \"prometheus\",
        \"value\": \"Prometheus\"
      }],
      \"folderId\": 0,
      \"overwrite\": true,
      \"id\": $ID
    }"
  echo "Imported Dashboard ID: $ID"
done
```

> **Note:** Dashboard ID `7249` (Kubernetes Cluster Summary) is confirmed working. Start with that if others show empty panels.

---

## Useful Commands

```bash
# Check all pods
kubectl get pods -n prometheus
kubectl get pods -n grafana

# Check HPA status
kubectl get hpa

# Port-forward Grafana locally
kubectl port-forward svc/grafana 3000:80 -n grafana

# Port-forward Prometheus locally
kubectl port-forward svc/prometheus-deployment-server 9090:80 -n prometheus

# Check Grafana logs
kubectl logs -l app.kubernetes.io/name=grafana -n grafana

# Check Prometheus logs
kubectl logs -l app.kubernetes.io/name=prometheus -n prometheus
```

---

## Automation Script

Save as `setup.sh` and run:

```bash
chmod +x setup.sh && ./setup.sh
```

```bash
#!/bin/bash
# ================================================================
#   Prometheus & Grafana Monitoring on AWS EKS — Full Automation
# ================================================================

set -euo pipefail

CLUSTER_NAME="landing-transform-my-eks-cluster"
REGION="us-east-1"
STORAGE_CLASS="gp3"
GRAFANA_PASSWORD="secret"
HPA_CPU_PERCENT=50
HPA_MIN=1
HPA_MAX=10

DASHBOARD_IDS=(6417 3119 13770 15661 7249 1860 15757 3662)
declare -A DASHBOARD_NAMES=(
  [6417]="Kubernetes Cluster Prometheus"
  [3119]="Kubernetes Cluster Monitoring"
  [13770]="Kubernetes All-in-One"
  [15661]="EKS Specific"
  [7249]="Kubernetes Cluster Summary"
  [1860]="Node Exporter Full"
  [15757]="Kubernetes Cluster Monitoring v2"
  [3662]="Prometheus Stats"
)

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'
BLUE='\033[0;34m'; CYAN='\033[0;36m'; BOLD='\033[1m'; NC='\033[0m'

info()    { echo -e "${BLUE}[INFO]${NC}  $*"; }
ok()      { echo -e "${GREEN}[ OK ]${NC}  $*"; }
warn()    { echo -e "${YELLOW}[WARN]${NC}  $*"; }
error()   { echo -e "${RED}[ERR ]${NC}  $*"; exit 1; }
section() { echo -e "\n${BOLD}${CYAN}──────────────────────────────────────────────${NC}\n${BOLD}${CYAN}  $*${NC}\n${BOLD}${CYAN}──────────────────────────────────────────────${NC}\n"; }

check_prerequisites() {
  section "Prerequisites Check"
  for tool in eksctl kubectl helm aws curl; do
    command -v "$tool" &>/dev/null && ok "$tool found" || error "$tool not installed"
  done
  aws sts get-caller-identity &>/dev/null || error "AWS credentials not configured. Run: aws configure"
  ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
  ok "AWS credentials valid — Account: $ACCOUNT_ID"
}

step1_create_eks_cluster() {
  section "Step 1 — Create EKS Cluster"
  eksctl get cluster --name "$CLUSTER_NAME" --region "$REGION" &>/dev/null \
    && warn "Cluster already exists. Skipping." \
    || { eksctl create cluster --name "$CLUSTER_NAME" --region "$REGION" && ok "Cluster created!"; }
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$REGION"
  ok "kubeconfig updated"
  kubectl get nodes
}

step2_install_ebs_csi_driver() {
  section "Step 2 — Install EBS CSI Driver"
  eksctl utils associate-iam-oidc-provider \
    --cluster "$CLUSTER_NAME" --region "$REGION" --approve \
    && ok "OIDC provider associated" || warn "May already exist. Continuing..."

  kubectl get serviceaccount ebs-csi-controller-sa -n kube-system &>/dev/null \
    && warn "IAM service account already exists. Skipping." \
    || { eksctl create iamserviceaccount \
      --name ebs-csi-controller-sa --namespace kube-system \
      --cluster "$CLUSTER_NAME" --region "$REGION" \
      --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
      --approve --role-only --role-name AmazonEKS_EBS_CSI_DriverRole && ok "IAM service account created"; }

  aws eks describe-addon --cluster-name "$CLUSTER_NAME" \
    --addon-name aws-ebs-csi-driver --region "$REGION" &>/dev/null \
    && warn "EBS CSI addon already installed. Skipping." \
    || { aws eks create-addon \
      --cluster-name "$CLUSTER_NAME" --addon-name aws-ebs-csi-driver \
      --service-account-role-arn "arn:aws:iam::${ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole" \
      --region "$REGION" && ok "EBS CSI Driver installed"; }

  kubectl get storageclass "$STORAGE_CLASS" &>/dev/null \
    && warn "StorageClass '$STORAGE_CLASS' already exists. Skipping." \
    || { kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
    ok "gp3 StorageClass created"; }
}

step3_deploy_prometheus() {
  section "Step 3 — Deploy Prometheus"
  helm repo add prometheus https://prometheus-community.github.io/helm-charts && helm repo update
  kubectl get namespace prometheus &>/dev/null || kubectl create namespace prometheus
  helm status prometheus-deployment -n prometheus &>/dev/null \
    && warn "Prometheus already deployed. Skipping." \
    || { helm install prometheus-deployment prometheus/prometheus \
      --namespace prometheus \
      --set alertmanager.persistentVolume.storageClass="$STORAGE_CLASS" \
      --set server.persistentVolume.storageClass="$STORAGE_CLASS" && ok "Prometheus deployed!"; }
  kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=prometheus \
    -n prometheus --timeout=120s 2>/dev/null || warn "Check: kubectl get pods -n prometheus"
  kubectl get pods -n prometheus
}

step4_create_datasource_config() {
  section "Step 4 — Create Grafana Datasource Config"
  PROM_SVC=$(kubectl get svc -n prometheus \
    -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/component=server" \
    -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || echo "prometheus-deployment-server")
  cat > /tmp/prometheus-datasource.yaml <<EOF
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://${PROM_SVC}.prometheus.svc.cluster.local
        isDefault: true
        access: proxy
        jsonData:
          timeInterval: "5s"
EOF
  ok "prometheus-datasource.yaml created"
}

step5_deploy_grafana() {
  section "Step 5 — Deploy Grafana"
  helm repo add grafana https://grafana.github.io/helm-charts && helm repo update
  kubectl get namespace grafana &>/dev/null || kubectl create namespace grafana
  helm status grafana -n grafana &>/dev/null \
    && warn "Grafana already deployed. Skipping." \
    || { helm install grafana grafana/grafana \
      --namespace grafana \
      --set persistence.storageClassName="$STORAGE_CLASS" \
      --set persistence.enabled=true \
      --set adminPassword="$GRAFANA_PASSWORD" \
      --set service.type=LoadBalancer \
      --values /tmp/prometheus-datasource.yaml && ok "Grafana deployed!"; }
  kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=grafana \
    -n grafana --timeout=120s 2>/dev/null || warn "Check: kubectl get pods -n grafana"
  kubectl get pods -n grafana
}

step6_setup_hpa() {
  section "Step 6 — Setup HPA"
  if [ -f "hpa.yaml" ]; then
    kubectl apply -f hpa.yaml && ok "hpa.yaml applied"
  else
    warn "hpa.yaml not found. Creating sample deployment..."
    kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-deploy
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-deploy
  template:
    metadata:
      labels:
        app: hpa-deploy
    spec:
      containers:
        - name: hpa-app
          image: nginx:latest
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
EOF
    ok "Sample hpa-deploy created"
  fi
  kubectl get hpa hpa-deploy -n default &>/dev/null \
    && warn "HPA already exists. Skipping." \
    || { kubectl autoscale deployment hpa-deploy \
      --cpu-percent="$HPA_CPU_PERCENT" --min="$HPA_MIN" --max="$HPA_MAX" \
      && ok "HPA configured — CPU: ${HPA_CPU_PERCENT}%, Min: ${HPA_MIN}, Max: ${HPA_MAX}"; }
  kubectl get hpa
}

step7_import_dashboards() {
  section "Step 7 — Import Grafana Dashboards"
  local elapsed=0 grafana_url=""
  info "Waiting for Grafana LoadBalancer IP..."
  while [ $elapsed -lt 180 ]; do
    grafana_url=$(kubectl get svc grafana -n grafana \
      -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || true)
    [ -z "$grafana_url" ] && grafana_url=$(kubectl get svc grafana -n grafana \
      -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)
    [ -n "$grafana_url" ] && break
    info "Waiting... ($elapsed/180s)"; sleep 10; elapsed=$((elapsed + 10))
  done
  if [ -z "$grafana_url" ]; then
    warn "LoadBalancer IP pending. Using localhost."
    warn "Run in another terminal: kubectl port-forward svc/grafana 3000:80 -n grafana"
    grafana_url="localhost:3000"
  else
    ok "Grafana URL: http://$grafana_url"
  fi
  for ID in "${DASHBOARD_IDS[@]}"; do
    NAME="${DASHBOARD_NAMES[$ID]}"
    CODE=$(curl -s -o /dev/null -w "%{http_code}" \
      -X POST "http://$grafana_url/api/dashboards/import" \
      -H "Content-Type: application/json" -u "admin:$GRAFANA_PASSWORD" \
      -d "{\"inputs\":[{\"name\":\"DS_PROMETHEUS\",\"type\":\"datasource\",\"pluginId\":\"prometheus\",\"value\":\"Prometheus\"}],\"folderId\":0,\"overwrite\":true,\"id\":$ID}" \
      2>/dev/null || echo "000")
    [ "$CODE" = "200" ] && ok "  ID $ID — $NAME" || warn "  ID $ID — $NAME (HTTP $CODE)"
  done
}

print_summary() {
  section "Setup Complete"
  GRAFANA_URL=$(kubectl get svc grafana -n grafana \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo "<pending>")
  echo -e "${GREEN}${BOLD}"
  echo "  ╔══════════════════════════════════════════════════════╗"
  echo "  ║      Monitoring Stack Deployed Successfully! 🚀      ║"
  echo "  ╚══════════════════════════════════════════════════════╝"
  echo -e "${NC}"
  echo -e "  ${BOLD}Cluster:${NC}     $CLUSTER_NAME ($REGION)"
  echo -e "  ${BOLD}Grafana:${NC}     http://$GRAFANA_URL  (admin / $GRAFANA_PASSWORD)"
  echo ""
  echo -e "  ${BOLD}Dashboards:${NC}"
  for ID in "${DASHBOARD_IDS[@]}"; do echo -e "    ${GREEN}✔${NC}  ID $ID — ${DASHBOARD_NAMES[$ID]}"; done
  echo ""
  echo -e "  ${YELLOW}Note:${NC} Dashboard 7249 confirmed working. Start there if others show empty panels."
}

main() {
  check_prerequisites
  step1_create_eks_cluster
  step2_install_ebs_csi_driver
  step3_deploy_prometheus
  step4_create_datasource_config
  step5_deploy_grafana
  step6_setup_hpa
  step7_import_dashboards
  print_summary
}

main "$@"
```

---

## Cleanup

> ⚠️ **Destructive** — removes all resources

```bash
helm uninstall grafana -n grafana
helm uninstall prometheus-deployment -n prometheus
kubectl delete namespace grafana
kubectl delete namespace prometheus
eksctl delete cluster \
  --name landing-transform-my-eks-cluster \
  --region us-east-1
```
