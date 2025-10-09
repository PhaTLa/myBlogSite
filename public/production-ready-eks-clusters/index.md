# Building Production-Ready Kubernetes Clusters on AWS EKS


<p align="center">
    <img src="https://pub-012e0e3c1b2643639bffe9b7fd5624e5.r2.dev/homelab.jpg" alt="Phạm Tùng Lâm Homelab" title="DevOps Engineer & Solution Architect" style="width:100%; height:240px; object-fit:cover; box-shadow:0 2px 8px rgba(0,0,0,0.1);" />
</p>

# Building Production-Ready Kubernetes Clusters on AWS EKS

Amazon Elastic Kubernetes Service (EKS) has become the go-to solution for running Kubernetes workloads in AWS. However, setting up a truly production-ready EKS cluster involves much more than just clicking "Create Cluster" in the AWS console. In this post, I'll walk you through the essential components and best practices I've learned from deploying and managing EKS clusters in production environments.

## 🎯 What Makes a Cluster "Production-Ready"?

Before diving into the technical details, let's define what we mean by "production-ready":

- **High Availability**: Multi-AZ deployment with proper node distribution
- **Security**: Network isolation, RBAC, pod security standards
- **Observability**: Comprehensive monitoring, logging, and alerting
- **Scalability**: Auto-scaling capabilities for both nodes and pods
- **Disaster Recovery**: Backup strategies and recovery procedures
- **Cost Optimization**: Right-sizing and efficient resource utilization

## 🏗️ Infrastructure Foundation

### VPC Design

The foundation of any EKS cluster is a well-designed VPC. Here's the architecture I typically use:

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
  
  # Required for EKS
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

### EKS Cluster Configuration

```yaml
# terraform/eks.tf
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  
  cluster_name    = var.cluster_name
  cluster_version = "1.28"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  # Cluster endpoint configuration
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access  = true
  cluster_endpoint_public_access_cidrs = ["0.0.0.0/0"]
  
  # Cluster encryption
  cluster_encryption_config = [{
    provider_key_arn = aws_kms_key.eks.arn
    resources        = ["secrets"]
  }]
  
  # Cluster logging
  cluster_enabled_log_types = [
    "api", "audit", "authenticator", "controllerManager", "scheduler"
  ]
  
  # Node groups
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

## 🔐 Security Best Practices

### Network Security

Network security starts with proper subnet isolation and security groups:

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

### RBAC Configuration

Implement proper Role-Based Access Control:

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

## 📊 Observability Stack

### Prometheus and Grafana Setup

Deploy a comprehensive monitoring stack:

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

Implement automatic scaling:

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

## 🚀 Deployment Automation

Here's a complete deployment script that ties everything together:

```bash
#!/bin/bash
# deploy.sh

set -e

CLUSTER_NAME="production-eks"
AWS_REGION="us-west-2"

echo "🚀 Deploying EKS cluster: $CLUSTER_NAME"

# Deploy infrastructure with Terraform
echo "📦 Deploying infrastructure..."
cd terraform
terraform init
terraform plan -var="cluster_name=$CLUSTER_NAME"
terraform apply -auto-approve -var="cluster_name=$CLUSTER_NAME"

# Update kubeconfig
echo "🔧 Updating kubeconfig..."
aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME

# Install essential add-ons
echo "🔌 Installing cluster add-ons..."

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

echo "✅ EKS cluster deployment completed!"
echo "🌐 Grafana Dashboard: kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80"
echo "📊 Prometheus UI: kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090"
```

## 💡 Production Tips & Lessons Learned

### 1. Cost Optimization

- Use **Spot Instances** for non-critical workloads (can save 60-90%)
- Implement **Horizontal Pod Autoscaler** to right-size your applications
- Regular **resource usage audits** to identify waste

### 2. Backup Strategy

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

### 3. Security Scanning

Implement automated security scanning:

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

## 🎉 Conclusion

Building production-ready EKS clusters requires careful planning and attention to detail. The configuration I've shared here forms a solid foundation, but remember that every production environment has unique requirements.

Key takeaways:
- **Start with a solid VPC foundation**
- **Implement security from day one**
- **Plan for observability and monitoring**
- **Automate everything possible**
- **Test your disaster recovery procedures**

In my next post, I'll dive deeper into **advanced EKS networking patterns** and **service mesh integration**. Stay tuned!

---

*Have questions about EKS or want to share your own experiences? Feel free to reach out on [LinkedIn](https://www.linkedin.com/in/alexpham15010305) or drop me an [email](mailto:phamtunglam.workmail.public@gmail.com).*
