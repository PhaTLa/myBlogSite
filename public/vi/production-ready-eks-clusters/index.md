# Xây dựng các cụm Kubernetes sẵn sàng sản xuất trên AWS EKS


<p align="center">
    <img src="https://pub-012e0e3c1b2643639bffe9b7fd5624e5.r2.dev/homelab.jpg" alt="Phạm Tùng Lâm Homelab" title="DevOps Engineer & Solution Architect" style="width:100%; height:240px; object-fit:cover; box-shadow:0 2px 8px rgba(0,0,0,0.1);" />
</p>

# Xây dựng các cụm Kubernetes sẵn sàng sản xuất trên AWS EKS

Amazon Elastic Kubernetes Service (EKS) đã trở thành giải pháp hàng đầu để chạy các workload Kubernetes trong AWS. Tuy nhiên, việc thiết lập một cụm EKS thực sự sẵn sàng cho sản xuất đòi hỏi nhiều hơn việc chỉ nhấp "Create Cluster" trong console AWS. Trong bài viết này, tôi sẽ hướng dẫn bạn qua các thành phần thiết yếu và thực tiễn tốt nhất mà tôi đã học được từ việc triển khai và quản lý các cụm EKS trong môi trường sản xuất.

## 🎯 Điều gì khiến một cụm "Sẵn sàng sản xuất"?

Trước khi đi vào chi tiết kỹ thuật, hãy định nghĩa ý nghĩa của "sẵn sàng sản xuất":

- **Tính khả dụng cao**: Triển khai Multi-AZ với phân phối node hợp lý
- **Bảo mật**: Cô lập mạng, RBAC, tiêu chuẩn bảo mật pod
- **Khả năng quan sát**: Giám sát, logging và cảnh báo toàn diện
- **Khả năng mở rộng**: Khả năng tự động mở rộng cho cả node và pod
- **Khôi phục thảm họa**: Chiến lược sao lưu và quy trình khôi phục
- **Tối ưu hóa chi phí**: Định kích thước đúng và sử dụng tài nguyên hiệu quả

## 🏗️ Nền tảng hạ tầng

### Thiết kế VPC

Nền tảng của bất kỳ cụm EKS nào là một VPC được thiết kế tốt. Đây là kiến trúc mà tôi thường sử dụng:

```yaml
# terraform/vpc.tf
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "eks-production-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-west-2a", "us-west-2b", "us-west-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  enable_dns_hostnames = true
  enable_dns_support = true
  
  # Bắt buộc cho EKS
  public_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                    = "1"
  }
  
  private_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"           = "1"
  }
}
```

### Cấu hình cụm EKS

```yaml
# terraform/eks.tf
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  
  cluster_name    = var.cluster_name
  cluster_version = "1.28"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  # Cấu hình endpoint cụm
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access  = true
  cluster_endpoint_public_access_cidrs = ["0.0.0.0/0"]
  
  # Mã hóa cụm
  cluster_encryption_config = [{
    provider_key_arn = aws_kms_key.eks.arn
    resources        = ["secrets"]
  }]
  
  # Logging cụm
  cluster_enabled_log_types = [
    "api", "audit", "authenticator", "controllerManager", "scheduler"
  ]
  
  # Nhóm node
  eks_managed_node_groups = {
    system = {
      name = "system-nodes"
      
      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"
      
      min_size     = 2
      max_size     = 4
      desired_size = 2
      
      labels = {
        "node-type" = "system"
      }
      
      taints = [{
        key    = "node-type"
        value  = "system"
        effect = "NO_SCHEDULE"
      }]
    }
    
    application = {
      name = "app-nodes"
      
      instance_types = ["t3.large", "t3.xlarge"]
      capacity_type  = "SPOT"
      
      min_size     = 2
      max_size     = 10
      desired_size = 3
      
      labels = {
        "node-type" = "application"
      }
    }
  }
}
```

## 🔐 Thực tiễn tốt nhất về bảo mật

### Bảo mật mạng

Bảo mật mạng bắt đầu với việc cô lập subnet đúng cách và security group:

```yaml
# terraform/security-groups.tf
resource "aws_security_group" "additional" {
  name_prefix = "${var.cluster_name}-additional"
  vpc_id      = module.vpc.vpc_id
  
  ingress {
    from_port = 22
    to_port   = 22
    protocol  = "tcp"
    cidr_blocks = [module.vpc.vpc_cidr_block]
  }
  
  tags = {
    Name = "${var.cluster_name}-additional"
  }
}
```

### Cấu hình RBAC

Triển khai kiểm soát truy cập dựa trên vai trò phù hợp:

```yaml
# k8s/rbac/developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps", "secrets", "services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
- kind: User
  name: developer-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

## 📊 Stack khả năng quan sát

### Thiết lập Prometheus và Grafana

Triển khai stack giám sát toàn diện:

```yaml
# k8s/monitoring/prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    
    resources:
      requests:
        memory: 2Gi
        cpu: 1000m
      limits:
        memory: 4Gi
        cpu: 2000m

grafana:
  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi
  
  adminPassword: "your-secure-password"
  
  grafana.ini:
    server:
      root_url: "https://grafana.yourdomain.com"
    security:
      disable_gravatar: true
```

### Cluster Autoscaler

Triển khai tự động mở rộng:

```yaml
# k8s/autoscaling/cluster-autoscaler.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
      - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.28.2
        name: cluster-autoscaler
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 100m
            memory: 300Mi
        command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/${CLUSTER_NAME}
        env:
        - name: AWS_REGION
          value: us-west-2
        - name: CLUSTER_NAME
          value: production-eks
```

## 🚀 Tự động hóa triển khai

Đây là script triển khai hoàn chỉnh kết hợp mọi thứ lại với nhau:

```bash
#!/bin/bash
# deploy.sh

set -e

CLUSTER_NAME="production-eks"
AWS_REGION="us-west-2"

echo "🚀 Triển khai cụm EKS: $CLUSTER_NAME"

# Triển khai hạ tầng với Terraform
echo "📦 Triển khai hạ tầng..."
cd terraform
terraform init
terraform plan -var="cluster_name=$CLUSTER_NAME"
terraform apply -auto-approve -var="cluster_name=$CLUSTER_NAME"

# Cập nhật kubeconfig
echo "🔧 Cập nhật kubeconfig..."
aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME

# Cài đặt các add-on thiết yếu
echo "🔌 Cài đặt các add-on cụm..."

# AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Prometheus monitoring stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace \
  -f ../k8s/monitoring/prometheus-values.yaml

# Cluster Autoscaler
kubectl apply -f ../k8s/autoscaling/

echo "✅ Triển khai cụm EKS hoàn tất!"
echo "🌐 Grafana Dashboard: kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80"
echo "📊 Prometheus UI: kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090"
```

## 💡 Mẹo sản xuất & Bài học kinh nghiệm

### 1. Tối ưu hóa chi phí

- Sử dụng **Spot Instances** cho các workload không quan trọng (có thể tiết kiệm 60-90%)
- Triển khai **Horizontal Pod Autoscaler** để định kích thước ứng dụng phù hợp
- **Kiểm toán sử dụng tài nguyên** thường xuyên để xác định lãng phí

### 2. Chiến lược sao lưu

```yaml
# velero-values.yaml
configuration:
  provider: aws
  backupStorageLocation:
    bucket: my-velero-backup-bucket
    config:
      region: us-west-2
  volumeSnapshotLocation:
    config:
      region: us-west-2
schedules:
  daily-backup:
    schedule: "0 2 * * *"
    template:
      ttl: "720h"
      includedNamespaces:
      - production
      - staging
```

### 3. Quét bảo mật

Triển khai quét bảo mật tự động:

```yaml
# .github/workflows/security-scan.yml
name: Security Scan
on:
  push:
    branches: [main]
jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'
```

## 🎉 Kết luận

Xây dựng các cụm EKS sẵn sàng cho sản xuất đòi hỏi lập kế hoạch cẩn thận và chú ý đến chi tiết. Cấu hình mà tôi đã chia sẻ ở đây tạo nên một nền tảng vững chắc, nhưng hãy nhớ rằng mỗi môi trường sản xuất đều có những yêu cầu riêng biệt.

Những điểm chính cần nhớ:
- **Bắt đầu với nền tảng VPC vững chắc**
- **Triển khai bảo mật từ ngày đầu**
- **Lập kế hoạch cho khả năng quan sát và giám sát**
- **Tự động hóa mọi thứ có thể**
- **Kiểm tra các quy trình khôi phục thảm họa của bạn**

Trong bài viết tiếp theo, tôi sẽ đi sâu hơn vào **các mẫu mạng EKS nâng cao** và **tích hợp service mesh**. Hãy theo dõi!

---

*Có câu hỏi về EKS hoặc muốn chia sẻ trải nghiệm của riêng bạn? Hãy liên hệ với tôi trên [LinkedIn](https://www.linkedin.com/in/alexpham15010305) hoặc gửi [email](mailto:phamtunglam.workmail.public@gmail.com).*
