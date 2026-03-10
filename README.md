#!/bin/bash

# ================================================================
#   Prometheus & Grafana Monitoring on AWS EKS — Full Automation
#   Production-grade Kubernetes monitoring stack via Helm
# ================================================================

set -euo pipefail

# ──────────────────────────────────────────────
#  CONFIG  (change these if needed)
# ──────────────────────────────────────────────
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
  [7249]="Kubernetes Cluster Summary ✅"
  [1860]="Node Exporter Full"
  [15757]="Kubernetes Cluster Monitoring v2"
  [3662]="Prometheus Stats"
)

# ──────────────────────────────────────────────
#  COLORS & LOGGING
# ──────────────────────────────────────────────
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'
BLUE='\033[0;34m'; CYAN='\033[0;36m'; BOLD='\033[1m'; NC='\033[0m'

info()    { echo -e "${BLUE}[INFO]${NC}  $*"; }
ok()      { echo -e "${GREEN}[ OK ]${NC}  $*"; }
warn()    { echo -e "${YELLOW}[WARN]${NC}  $*"; }
error()   { echo -e "${RED}[ERR ]${NC}  $*"; exit 1; }
section() {
  echo ""
  echo -e "${BOLD}${CYAN}──────────────────────────────────────────────${NC}"
  echo -e "${BOLD}${CYAN}  $*${NC}"
  echo -e "${BOLD}${CYAN}──────────────────────────────────────────────${NC}"
  echo ""
}

# ──────────────────────────────────────────────
#  PREREQUISITES
# ──────────────────────────────────────────────
check_prerequisites() {
  section "Prerequisites Check"

  for tool in eksctl kubectl helm aws curl; do
    if command -v "$tool" &>/dev/null; then
      ok "$tool found"
    else
      error "$tool is not installed. Please install it and retry."
    fi
  done

  info "Verifying AWS credentials..."
  if aws sts get-caller-identity &>/dev/null; then
    ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
    ok "AWS credentials valid — Account: $ACCOUNT_ID"
  else
    error "AWS credentials not configured. Run: aws configure"
  fi
}

# ──────────────────────────────────────────────
#  STEP 1 — Create EKS Cluster
# ──────────────────────────────────────────────
step1_create_eks_cluster() {
  section "Step 1 — Create EKS Cluster"

  if eksctl get cluster --name "$CLUSTER_NAME" --region "$REGION" &>/dev/null; then
    warn "Cluster '$CLUSTER_NAME' already exists. Skipping creation."
  else
    info "Creating EKS cluster: $CLUSTER_NAME in $REGION ..."
    eksctl create cluster \
      --name "$CLUSTER_NAME" \
      --region "$REGION"
    ok "Cluster created!"
  fi

  info "Updating kubeconfig..."
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$REGION"
  ok "kubeconfig updated"

  info "Verifying nodes..."
  kubectl get nodes
}

# ──────────────────────────────────────────────
#  STEP 2 — Install EBS CSI Driver
# ──────────────────────────────────────────────
step2_install_ebs_csi_driver() {
  section "Step 2 — Install EBS CSI Driver (gp3 Storage)"

  info "Associating OIDC provider..."
  eksctl utils associate-iam-oidc-provider \
    --cluster "$CLUSTER_NAME" \
    --region "$REGION" \
    --approve && ok "OIDC provider associated" || warn "OIDC may already be associated. Continuing..."

  if kubectl get serviceaccount ebs-csi-controller-sa -n kube-system &>/dev/null; then
    warn "IAM service account already exists. Skipping."
  else
    info "Creating IAM service account for EBS CSI Driver..."
    eksctl create iamserviceaccount \
      --name ebs-csi-controller-sa \
      --namespace kube-system \
      --cluster "$CLUSTER_NAME" \
      --region "$REGION" \
      --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
      --approve \
      --role-only \
      --role-name AmazonEKS_EBS_CSI_DriverRole
    ok "IAM service account created"
  fi

  if aws eks describe-addon \
    --cluster-name "$CLUSTER_NAME" \
    --addon-name aws-ebs-csi-driver \
    --region "$REGION" &>/dev/null; then
    warn "EBS CSI addon already installed. Skipping."
  else
    info "Installing EBS CSI Driver addon..."
    aws eks create-addon \
      --cluster-name "$CLUSTER_NAME" \
      --addon-name aws-ebs-csi-driver \
      --service-account-role-arn "arn:aws:iam::${ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole" \
      --region "$REGION"
    ok "EBS CSI Driver addon installed"
  fi

  if kubectl get storageclass "$STORAGE_CLASS" &>/dev/null; then
    warn "StorageClass '$STORAGE_CLASS' already exists. Skipping."
  else
    info "Creating gp3 StorageClass..."
    kubectl apply -f - <<EOF
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
    ok "gp3 StorageClass created"
  fi
}

# ──────────────────────────────────────────────
#  STEP 3 — Deploy Prometheus
# ──────────────────────────────────────────────
step3_deploy_prometheus() {
  section "Step 3 — Deploy Prometheus"

  info "Adding Prometheus Helm repo..."
  helm repo add prometheus https://prometheus-community.github.io/helm-charts
  helm repo update
  ok "Helm repo ready"

  kubectl get namespace prometheus &>/dev/null || kubectl create namespace prometheus && ok "Namespace 'prometheus' ready"

  if helm status prometheus-deployment -n prometheus &>/dev/null; then
    warn "Prometheus already deployed. Skipping."
  else
    info "Installing Prometheus..."
    helm install prometheus-deployment prometheus/prometheus \
      --namespace prometheus \
      --set alertmanager.persistentVolume.storageClass="$STORAGE_CLASS" \
      --set server.persistentVolume.storageClass="$STORAGE_CLASS"
    ok "Prometheus deployed!"
  fi

  info "Waiting for Prometheus pods..."
  kubectl wait --for=condition=ready pod \
    -l app.kubernetes.io/name=prometheus \
    -n prometheus \
    --timeout=120s 2>/dev/null || warn "Pods not ready yet — check manually with: kubectl get pods -n prometheus"

  kubectl get pods -n prometheus
}

# ──────────────────────────────────────────────
#  STEP 4 — Create Grafana Datasource Config
# ──────────────────────────────────────────────
step4_create_datasource_config() {
  section "Step 4 — Create Grafana Datasource Config"

  PROM_SVC=$(kubectl get svc -n prometheus \
    -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/component=server" \
    -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || echo "prometheus-deployment-server")

  info "Prometheus Service detected: $PROM_SVC"

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

  ok "prometheus-datasource.yaml created at /tmp/prometheus-datasource.yaml"
}

# ──────────────────────────────────────────────
#  STEP 5 — Deploy Grafana
# ──────────────────────────────────────────────
step5_deploy_grafana() {
  section "Step 5 — Deploy Grafana"

  info "Adding Grafana Helm repo..."
  helm repo add grafana https://grafana.github.io/helm-charts
  helm repo update
  ok "Helm repo ready"

  kubectl get namespace grafana &>/dev/null || kubectl create namespace grafana && ok "Namespace 'grafana' ready"

  if helm status grafana -n grafana &>/dev/null; then
    warn "Grafana already deployed. Skipping."
  else
    info "Installing Grafana..."
    helm install grafana grafana/grafana \
      --namespace grafana \
      --set persistence.storageClassName="$STORAGE_CLASS" \
      --set persistence.enabled=true \
      --set adminPassword="$GRAFANA_PASSWORD" \
      --set service.type=LoadBalancer \
      --values /tmp/prometheus-datasource.yaml
    ok "Grafana deployed!"
  fi

  info "Waiting for Grafana pods..."
  kubectl wait --for=condition=ready pod \
    -l app.kubernetes.io/name=grafana \
    -n grafana \
    --timeout=120s 2>/dev/null || warn "Pods not ready yet — check manually with: kubectl get pods -n grafana"

  kubectl get pods -n grafana
  info "Get Grafana URL: kubectl get svc grafana -n grafana"
  info "Login: admin / $GRAFANA_PASSWORD"
}

# ──────────────────────────────────────────────
#  STEP 6 — Setup HPA
# ──────────────────────────────────────────────
step6_setup_hpa() {
  section "Step 6 — Setup HPA (Horizontal Pod Autoscaler)"

  if [ -f "hpa.yaml" ]; then
    info "Applying hpa.yaml..."
    kubectl apply -f hpa.yaml
    ok "hpa.yaml applied"
  else
    warn "hpa.yaml not found. Creating sample deployment..."
    kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-deploy
  namespace: default
  labels:
    app: hpa-deploy
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
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
EOF
    ok "Sample hpa-deploy Deployment created"
  fi

  if kubectl get hpa hpa-deploy -n default &>/dev/null; then
    warn "HPA for hpa-deploy already exists. Skipping."
  else
    info "Configuring HPA autoscale..."
    kubectl autoscale deployment hpa-deploy \
      --cpu-percent="$HPA_CPU_PERCENT" \
      --min="$HPA_MIN" \
      --max="$HPA_MAX"
    ok "HPA configured — CPU: ${HPA_CPU_PERCENT}%, Min: ${HPA_MIN}, Max: ${HPA_MAX}"
  fi

  kubectl get hpa
}

# ──────────────────────────────────────────────
#  STEP 7 — Import Grafana Dashboards
# ──────────────────────────────────────────────
step7_import_dashboards() {
  section "Step 7 — Import Grafana Dashboards"

  info "Waiting for Grafana LoadBalancer external IP/hostname..."
  local elapsed=0 grafana_url=""

  while [ $elapsed -lt 180 ]; do
    grafana_url=$(kubectl get svc grafana -n grafana \
      -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || true)
    [ -z "$grafana_url" ] && grafana_url=$(kubectl get svc grafana -n grafana \
      -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)

    [ -n "$grafana_url" ] && break
    info "Still waiting... ($elapsed/180s)"
    sleep 10; elapsed=$((elapsed + 10))
  done

  if [ -z "$grafana_url" ]; then
    warn "LoadBalancer IP pending. Using localhost via port-forward..."
    warn "Run this in another terminal: kubectl port-forward svc/grafana 3000:80 -n grafana"
    grafana_url="localhost:3000"
  else
    ok "Grafana URL: http://$grafana_url"
  fi

  GRAFANA_URL="http://$grafana_url"

  info "Importing ${#DASHBOARD_IDS[@]} dashboards..."
  echo ""
  for ID in "${DASHBOARD_IDS[@]}"; do
    NAME="${DASHBOARD_NAMES[$ID]}"
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
      -X POST "$GRAFANA_URL/api/dashboards/import" \
      -H "Content-Type: application/json" \
      -u "admin:$GRAFANA_PASSWORD" \
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
      }" 2>/dev/null || echo "000")

    if [ "$HTTP_CODE" = "200" ]; then
      ok "  ID $ID — $NAME"
    else
      warn "  ID $ID — $NAME (HTTP $HTTP_CODE — may need manual import)"
    fi
  done
}

# ──────────────────────────────────────────────
#  SUMMARY
# ──────────────────────────────────────────────
print_summary() {
  section "Setup Complete"

  GRAFANA_URL=$(kubectl get svc grafana -n grafana \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || \
    kubectl get svc grafana -n grafana \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || \
    echo "<pending>")

  echo -e "${GREEN}${BOLD}"
  echo "  ╔══════════════════════════════════════════════════════╗"
  echo "  ║      Monitoring Stack Deployed Successfully! 🚀      ║"
  echo "  ╚══════════════════════════════════════════════════════╝"
  echo -e "${NC}"

  echo -e "  ${BOLD}Cluster:${NC}         $CLUSTER_NAME  ($REGION)"
  echo -e "  ${BOLD}Grafana URL:${NC}     http://$GRAFANA_URL"
  echo -e "  ${BOLD}Grafana Login:${NC}   admin / $GRAFANA_PASSWORD"
  echo ""
  echo -e "  ${BOLD}Dashboards Imported:${NC}"
  for ID in "${DASHBOARD_IDS[@]}"; do
    echo -e "    ${GREEN}✔${NC}  ID $ID — ${DASHBOARD_NAMES[$ID]}"
  done
  echo ""
  echo -e "  ${BOLD}Useful Commands:${NC}"
  echo "    kubectl get pods -n prometheus"
  echo "    kubectl get pods -n grafana"
  echo "    kubectl get hpa"
  echo "    kubectl port-forward svc/grafana 3000:80 -n grafana"
  echo "    kubectl port-forward svc/prometheus-deployment-server 9090:80 -n prometheus"
  echo ""
  echo -e "  ${BOLD}Cleanup:${NC}"
  echo "    helm uninstall grafana -n grafana"
  echo "    helm uninstall prometheus-deployment -n prometheus"
  echo "    kubectl delete namespace grafana prometheus"
  echo "    eksctl delete cluster --name $CLUSTER_NAME --region $REGION"
  echo ""
  echo -e "  ${YELLOW}Note:${NC} Dashboard ID 7249 (Kubernetes Cluster Summary) is confirmed working."
  echo "        Start with that if others show empty panels."
  echo ""
}

# ──────────────────────────────────────────────
#  MAIN
# ──────────────────────────────────────────────
main() {
  echo -e "${BOLD}${CYAN}"
  echo "  ╔════════════════════════════════════════════════════╗"
  echo "  ║   Prometheus + Grafana on AWS EKS — Automation    ║"
  echo "  ╚════════════════════════════════════════════════════╝"
  echo -e "${NC}"

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
