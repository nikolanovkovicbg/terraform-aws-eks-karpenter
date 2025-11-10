# Terraform AWS EKS with Karpenter

This project provisions an AWS EKS cluster with Karpenter for automatic node provisioning and scaling. It includes a VPC with public and private subnets, an EKS cluster, and Karpenter configured to automatically provision nodes based on pod requirements.

## Overview

The infrastructure includes:
- **VPC**: Custom VPC with public and private subnets across 3 availability zones
- **EKS Cluster**: Kubernetes cluster with a managed node group for Karpenter controller
- **Karpenter**: Node autoscaler that automatically provisions EC2 instances based on pod requirements
- **Example Pods**: Sample manifests for testing AMD64 and ARM64 workloads

## Prerequisites

Before you begin, ensure you have the following installed and configured:

1. **Terraform** >= 1.5.7
   ```bash
   terraform version
   ```

2. **AWS CLI** configured with appropriate credentials
   ```bash
   aws --version
   aws configure list-profiles
   ```

3. **kubectl** for interacting with the Kubernetes cluster
   ```bash
   kubectl version --client
   ```

4. **AWS Account** with permissions to create:
   - VPCs, subnets, NAT gateways
   - EKS clusters
   - EC2 instances
   - IAM roles and policies
   - S3 buckets (for Terraform state)

## Configuration

### 1. Configure AWS Profile

The project uses an AWS profile named `nikolanovkovicbgshowcase`. Update the profile name in `terraform/providers.tf` to match your AWS profile:

```hcl
provider "aws" {
  region  = var.aws_region
  profile = "your-aws-profile-name"  # Update this
}
```

Also update the profile in the Helm provider configuration (line 37) and the S3 backend (line 17).

### 2. Configure S3 Backend

Update the S3 backend configuration in `terraform/providers.tf` to use your own S3 bucket for Terraform state:

```hcl
backend "s3" {
  bucket       = "your-terraform-state-bucket"  # Update this
  key          = "terraform.tfstate"
  profile      = "your-aws-profile-name"       # Update this
  region       = "eu-central-1"
}
```

**Note**: If you don't have an S3 bucket for state storage, you can:
- Create one manually, or
- Remove the backend block temporarily to use local state (not recommended for production)

### 3. Configure Variables

Edit `terraform/terraform.tfvars` to customize your deployment:

```hcl
aws_region      = "eu-central-1"  # Your preferred AWS region
cluster_name    = "showcase"      # Your EKS cluster name
cluster_version = "1.34"          # Kubernetes version
```

### 4. Update Karpenter Manifests

If you changed the `cluster_name` in `terraform.tfvars`, update the cluster name in `k8s/manifests/karpenter.yaml`:

```yaml
subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: your-cluster-name  # Update this
securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: your-cluster-name  # Update this
```

## Deployment

### 1. Initialize Terraform

Navigate to the terraform directory and initialize Terraform:

```bash
cd terraform
terraform init
```

### 2. Review the Plan

Review what Terraform will create:

```bash
terraform plan
```

### 3. Apply the Infrastructure

Deploy the infrastructure:

```bash
terraform apply
```

This will create:
- VPC and networking resources
- EKS cluster
- Karpenter IAM roles and policies
- Karpenter Helm release

The deployment typically takes 15-20 minutes.

### 4. Configure kubectl

After the cluster is created, configure kubectl to connect to your cluster:

```bash
aws eks update-kubeconfig --name <cluster-name> --region <region> --profile <your-profile>
```

For example:
```bash
aws eks update-kubeconfig --name showcase --region eu-central-1 --profile nikolanovkovicbgshowcase
```

### 5. Deploy Karpenter Configuration

Apply the Karpenter NodePool and EC2NodeClass:

```bash
kubectl apply -f ../k8s/manifests/karpenter.yaml
```

Verify Karpenter is running:

```bash
kubectl get pods -n karpenter
kubectl get nodepools
kubectl get ec2nodeclasses
```

## Testing

### Test with Example Pods

The project includes example pod manifests to test Karpenter's node provisioning:

**Deploy an AMD64 pod:**
```bash
kubectl apply -f ../k8s/manifests/pod-amd.yaml
```

**Deploy an ARM64 pod:**
```bash
kubectl apply -f ../k8s/manifests/pod-arm.yaml
```

Watch Karpenter provision nodes:
```bash
kubectl get nodes -w
kubectl get pods -w
```

Karpenter will automatically:
1. Detect the pending pods
2. Provision appropriate EC2 instances based on the node selectors
3. Schedule the pods on the new nodes

### Verify Node Provisioning

Check the nodes:
```bash
kubectl get nodes --show-labels
```

You should see nodes with labels matching your pod requirements (e.g., `kubernetes.io/arch=amd64` or `arm64`).

## Cleanup

To destroy all resources:

```bash
cd terraform
terraform destroy
```

**Note**: Make sure to delete any pods or workloads first, as Karpenter-managed nodes will be automatically cleaned up when pods are removed.

## Architecture

- **VPC**: 10.0.0.0/16 CIDR with public and private subnets
- **EKS**: Managed Kubernetes cluster with API endpoint access
- **Karpenter Node Group**: Small managed node group (t2.medium) running Karpenter controller
- **Karpenter**: Automatically provisions nodes based on pod requirements, supports both spot and on-demand instances

## Cost Considerations

This setup includes:
- Single NAT gateway (cost optimization)
- CloudWatch logging disabled (cost savings for POC)
- Karpenter configured to use spot instances when possible
- Consolidation policy to reduce unused capacity

## Additional Resources

- [Karpenter Documentation](https://karpenter.sh/)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Terraform AWS EKS Module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/)

