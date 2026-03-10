#!/bin/bash

# ============================================================
# Prometheus & Grafana Monitoring Stack - AWS EKS Automation
# ============================================================

set -e

# ---------- Color Codes ----------
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
NC='\033[0m'

# ---------- Config ----------
CLUSTER_NAME="landing-transform-my-eks-cluster"
REGION="us-east-1"
GRAFANA_ADMIN_PASSWORD="secret"
STORAGE_CLASS="gp3"

GRAFANA_DASHBOARDS=(
  "6417:Kubernetes Cluster Prometheus"
  "3119:Kubernetes Cluster Monitoring"
  "13770:Kubernetes All-in-One"
  "15661:EKS Specific"
  "7249:Kubernetes Cluster Summary"
  "1860:Node Exporter Full"
  "15757:Kubernetes Cluster Monitoring v2"
  "3662:Prometheus Stats"
)

# ---------- Logging ----------
log_info()    { echo -e "${BLUE}[INFO]${NC}  $1"; }
log_success() { echo -e "${GREEN}[OK]${NC}    $1"; }
log_warn()    { echo -e "${YELLOW}[WARN]${NC}  $1"; }
log_error()   { echo -e "${RED}[ERROR]${NC} $1"; }
log_section() { echo -e "\n${BOLD}${CYAN}========================================${NC}"; echo -e "${BOLD}${CYAN}  $1${NC}"; echo -e "${BOLD}${CYAN}========================================${NC}\n"; }

# ---------- Prerequisite Check ----------
check_prerequisites() {
  log_section "Checking Prerequisites"
  local tools=("eksctl" "kubectl" "helm" "aws")
  local missing=()

  for tool in "${tools[@]}"; do
    if command -v "$tool" &>/dev/null; then
      log_success "$tool is installed"
    else
      log_error "$tool is NOT installed"
      missing+=("$tool")
    fi
  done

  if [ ${#missing[@]} -ne 0 ]; then
    log_error "Missing tools: ${missing[*]}. Please install them and re-run."
    exit 1
  fi

  log_info "Checking AWS credentials..."
  if aws sts get-caller-identity &>/dev/null; then
    log_success "AWS credentials are valid"
  else
    log_error "AWS credentials not configured. Run: aws configure"
    exit 1
  fi
}

# ---------- Step 1: Create EKS Cluster ----------
create_eks_cluster() {
  log_section "Step 1: Creating EKS Cluster"

  if eksctl get cluster --name "$CLUSTER_NAME" --region "$REGION" &>/dev/null; then
    log_warn "Cluster '$CLUSTER_NAME' already exists. Skipping creation."
  else
    log_info "Creating EKS cluster: $CLUSTER_NAME in $REGION ..."
    eksctl create cluster \
      --name "$CLUSTER_NAME" \
      --region "$REGION"
    log_success "EKS cluster created successfully!"
  fi

  log_info "Updating kubeconfig..."
  aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$REGION"
  log_success "kubeconfig updated"

  log_info "Verifying cluster connectivity..."
  kubectl get nodes
}

# ---------- Step 2: Install EBS CSI Driver ----------
install_ebs_csi_driver() {
  log_section "Step 2: Installing EBS CSI Driver"

  local cluster_oidc
  cluster_oidc=$(aws eks describe-cluster \
    --name "$CLUSTER_NAME" \
    --region "$REGION" \
    --query "cluster.identity.oidc.issuer" \
    --output text)

  log_info "OIDC Issuer: $cluster_oidc"

  # Check if OIDC provider exists
  if eksctl utils associate-iam-oidc-provider \
    --cluster "$CLUSTER_NAME" \
    --region "$REGION" \
    --approve 2>/dev/null; then
    log_success "OIDC provider associated"
  else
    log_warn "OIDC provider may already be associated. Continuing..."
  fi

  # Create IAM service account for EBS CSI
  if kubectl get serviceaccount ebs-csi-controller-sa -n kube-system &>/dev/null; then
    log_warn "EBS CSI service account already exists. Skipping."
  else
    log_info "Creating IAM service account for EBS CSI Driver..."
    eksctl create iamserviceaccount \
      --name ebs-csi-controller-sa \
      --namespace kube-system \
      --cluster "$CLUSTER_NAME" \
      --region "$REGION" \
      --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
      --approve \
      --role-only \
      --role-name AmazonEKS_EBS_CSI_DriverRole
    log_success "IAM service account created"
  fi

  # Add EBS CSI addon
  if aws eks describe-addon \
    --cluster-name "$CLUSTER_NAME" \
    --addon-name aws-ebs-csi-driver \
    --region "$REGION" &>/dev/null; then
    log_warn "EBS CSI addon already installed. Skipping."
  else
    local account_id
    account_id=$(aws sts get-caller-identity --query Account --output text)

    log_info "Installing EBS CSI Driver addon..."
    aws eks create-addon \
      --cluster-name "$CLUSTER_NAME" \
      --addon-name aws-ebs-csi-driver \
      --service-account-role-arn "arn:aws:iam::${account_id}:role/AmazonEKS_EBS_CSI_DriverRole" \
      --region "$REGION"
    log_success "EBS CSI Driver addon installed"
  fi

  # Verify gp3 StorageClass
  if kubectl get storageclass gp3 &>/dev/null; then
    log_success "gp3 StorageClass already exists"
  else
    log_info "Creating gp3 StorageClass..."
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
    log_success "gp3 StorageClass created"
  fi
}

# ---------- Step 3: Deploy Prometheus ----------
deploy_prometheus() {
  log_section "Step 3: Deploying Prometheus"

  log_info "Adding Prometheus Helm repo..."
  helm repo add prometheus https://prometheus-community.github.io/helm-charts
  helm repo update
  log_success "Helm repo added"

  if kubectl get namespace prometheus &>/dev/null; then
    log_warn "Namespace 'prometheus' already exists. Skipping creation."
  else
    kubectl create namespace prometheus
    log_success "Namespace 'prometheus' created"
  fi

  if helm status prometheus-deployment -n prometheus &>/dev/null; then
    log_warn "Prometheus already deployed. Skipping installation."
  else
    log_info "Installing Prometheus..."
    helm install prometheus-deployment prometheus/prometheus \
      --namespace prometheus \
      --set alertmanager.persistentVolume.storageClass="$STORAGE_CLASS" \
      --set server.persistentVolume.storageClass="$STORAGE_CLASS"
    log_success "Prometheus deployed!"
  fi

  log_info "Waiting for Prometheus pods to be ready..."
  kubectl wait --for=condition=ready pod \
    -l app.kubernetes.io/name=prometheus \
    -n prometheus \
    --timeout=120s 2>/dev/null || log_warn "Timeout waiting for Prometheus pods. Check manually."

  kubectl get pods -n prometheus
}

# ---------- Step 4: Create Grafana Datasource Config ----------
create_prometheus_datasource() {
  log_section "Step 4: Creating Grafana Datasource Config"

  local prometheus_svc
  prometheus_svc=$(kubectl get svc -n prometheus \
    -l app.kubernetes.io/name=prometheus,app.kubernetes.io/component=server \
    -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || echo "prometheus-deployment-server")

  log_info "Prometheus Service: $prometheus_svc"

  cat > /tmp/prometheus-datasource.yaml <<EOF
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        url: http://${prometheus_svc}.prometheus.svc.cluster.local
        isDefault: true
        access: proxy
        jsonData:
          timeInterval: "5s"
EOF

  log_success "prometheus-datasource.yaml created at /tmp/prometheus-datasource.yaml"
}

# ---------- Step 5: Deploy Grafana ----------
deploy_grafana() {
  log_section "Step 5: Deploying Grafana"

  log_info "Adding Grafana Helm repo..."
  helm repo add grafana https://grafana.github.io/helm-charts
  helm repo update
  log_success "Helm repo added"

  if kubectl get namespace grafana &>/dev/null; then
    log_warn "Namespace 'grafana' already exists. Skipping creation."
  else
    kubectl create namespace grafana
    log_success "Namespace 'grafana' created"
  fi

  if helm status grafana -n grafana &>/dev/null; then
    log_warn "Grafana already deployed. Skipping installation."
  else
    log_info "Installing Grafana..."
    helm install grafana grafana/grafana \
      --namespace grafana \
      --set persistence.storageClassName="$STORAGE_CLASS" \
      --set persistence.enabled=true \
      --set adminPassword="$GRAFANA_ADMIN_PASSWORD" \
      --set service.type=LoadBalancer \
      --values /tmp/prometheus-datasource.yaml
    log_success "Grafana deployed!"
  fi

  log_info "Waiting for Grafana pods to be ready..."
  kubectl wait --for=condition=ready pod \
    -l app.kubernetes.io/name=grafana \
    -n grafana \
    --timeout=120s 2>/dev/null || log_warn "Timeout waiting for Grafana pods. Check manually."

  kubectl get pods -n grafana
}

# ---------- Step 6: Setup HPA ----------
setup_hpa() {
  log_section "Step 6: Setting Up HPA (Horizontal Pod Autoscaler)"

  # Create sample HPA deployment if hpa.yaml doesn't exist
  if [ ! -f "hpa.yaml" ]; then
    log_warn "hpa.yaml not found. Creating a sample HPA deployment..."
    cat > /tmp/hpa.yaml <<EOF
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
    kubectl apply -f /tmp/hpa.yaml
    log_success "Sample hpa-deploy Deployment applied"
  else
    log_info "Applying hpa.yaml..."
    kubectl apply -f hpa.yaml
    log_success "hpa.yaml applied"
  fi

  log_info "Setting up HPA autoscale..."
  if kubectl get hpa hpa-deploy -n default &>/dev/null; then
    log_warn "HPA for hpa-deploy already exists. Skipping."
  else
    kubectl autoscale deployment hpa-deploy \
      --cpu-percent=50 \
      --min=1 \
      --max=10
    log_success "HPA configured: CPU 50%, min=1, max=10"
  fi

  kubectl get hpa
}

# ---------- Step 7: Import Grafana Dashboards ----------
import_grafana_dashboards() {
  log_section "Step 7: Importing Grafana Dashboards"

  log_info "Waiting for Grafana LoadBalancer to get external IP..."
  local max_wait=180
  local elapsed=0
  local grafana_url=""

  while [ $elapsed -lt $max_wait ]; do
    grafana_url=$(kubectl get svc grafana -n grafana \
      -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo "")

    if [ -z "$grafana_url" ]; then
      grafana_url=$(kubectl get svc grafana -n grafana \
        -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || echo "")
    fi

    if [ -n "$grafana_url" ]; then
      log_success "Grafana URL: http://$grafana_url"
      break
    fi

    log_info "Waiting for external IP... ($elapsed/${max_wait}s)"
    sleep 10
    elapsed=$((elapsed + 10))
  done

  if [ -z "$grafana_url" ]; then
    log_warn "Could not get Grafana external URL. You can port-forward and import dashboards manually."
    log_info "Run: kubectl port-forward svc/grafana 3000:80 -n grafana"
    grafana_url="localhost:3000"
  fi

  log_info "Importing dashboards into Grafana..."
  for entry in "${GRAFANA_DASHBOARDS[@]}"; do
    local dashboard_id="${entry%%:*}"
    local dashboard_name="${entry##*:}"

    log_info "  Importing Dashboard ID: $dashboard_id - $dashboard_name"
    local response
    response=$(curl -s -o /dev/null -w "%{http_code}" \
      -X POST "http://$grafana_url/api/dashboards/import" \
      -H "Content-Type: application/json" \
      -u "admin:$GRAFANA_ADMIN_PASSWORD" \
      -d "{
        \"inputs\": [{
          \"name\": \"DS_PROMETHEUS\",
          \"type\": \"datasource\",
          \"pluginId\": \"prometheus\",
          \"value\": \"Prometheus\"
        }],
        \"folderId\": 0,
        \"overwrite\": true,
        \"id\": $dashboard_id
      }" 2>/dev/null || echo "000")

    if [ "$response" = "200" ]; then
      log_success "  Dashboard $dashboard_id ($dashboard_name) imported"
    else
      log_warn "  Dashboard $dashboard_id ($dashboard_name) - HTTP $response (may need manual import)"
    fi
  done
}

# ---------- Step 8: Print Summary ----------
print_summary() {
  log_section "Setup Complete - Summary"

  local grafana_url
  grafana_url=$(kubectl get svc grafana -n grafana \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || \
    kubectl get svc grafana -n grafana \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || \
    echo "<pending>")

  echo -e "${GREEN}${BOLD}"
  echo "  ╔══════════════════════════════════════════════════════╗"
  echo "  ║         Monitoring Stack Deployed Successfully!      ║"
  echo "  ╚══════════════════════════════════════════════════════╝"
  echo -e "${NC}"

  echo -e "${BOLD}Cluster:${NC}        $CLUSTER_NAME ($REGION)"
  echo -e "${BOLD}Grafana URL:${NC}    http://$grafana_url"
  echo -e "${BOLD}Grafana Login:${NC}  admin / $GRAFANA_ADMIN_PASSWORD"
  echo ""
  echo -e "${BOLD}Imported Dashboards:${NC}"
  for entry in "${GRAFANA_DASHBOARDS[@]}"; do
    local id="${entry%%:*}"
    local name="${entry##*:}"
    echo -e "  ${GREEN}✔${NC}  ID $id — $name"
  done

  echo ""
  echo -e "${BOLD}Useful Commands:${NC}"
  echo "  kubectl get pods -n prometheus"
  echo "  kubectl get pods -n grafana"
  echo "  kubectl get hpa"
  echo "  kubectl port-forward svc/grafana 3000:80 -n grafana"
  echo "  kubectl port-forward svc/prometheus-deployment-server 9090:80 -n prometheus"
  echo ""
  log_success "All done! Happy monitoring 🚀"
}

# ---------- Main ----------
main() {
  echo -e "${BOLD}${CYAN}"
  echo "  ██████╗ ██████╗  ██████╗ ███╗   ███╗    "
  echo "  ██╔══██╗██╔══██╗██╔═══██╗████╗ ████║    "
  echo "  ██████╔╝██████╔╝██║   ██║██╔████╔██║    "
  echo "  ██╔═══╝ ██╔══██╗██║   ██║██║╚██╔╝██║    "
  echo "  ██║     ██║  ██║╚██████╔╝██║ ╚═╝ ██║    "
  echo "  ╚═╝     ╚═╝  ╚═╝ ╚═════╝ ╚═╝     ╚═╝    "
  echo ""
  echo "  + Grafana | AWS EKS Monitoring Automation"
  echo -e "${NC}"

  check_prerequisites
  create_eks_cluster
  install_ebs_csi_driver
  deploy_prometheus
  create_prometheus_datasource
  deploy_grafana
  setup_hpa
  import_grafana_dashboards
  print_summary
}

main "$@"
