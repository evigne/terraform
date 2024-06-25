### File: `eks.tf`

This file defines the configuration for creating an EKS cluster using the `terraform-aws-modules/eks/aws` module. Here are the key components and their purposes:

#### Cluster Configuration

- **module "eks"**:
  - `source`: Specifies the Terraform module source.
  - `version`: Defines the module version.
  - `cluster_name` & `cluster_version`: Sets the EKS cluster name and version.
  - `enable_cluster_creator_admin_permissions`: Grants admin permissions to the Terraform identity.
  - `cluster_endpoint_public_access`: Allows public access to the cluster endpoint.
  - `cluster_addons`: Configures various EKS addons, such as CoreDNS, EKS Pod Identity, Kube-Proxy, and VPC-CNI.
  - `vpc_id` & `subnet_ids`: Specifies the VPC ID and subnets for the cluster.
  - `eks_managed_node_groups`: Defines managed node groups, including settings for instance types, sizes, labels, and taints.
  - `tags`: Merges additional tags, including a discovery tag for Karpenter.

- **output "configure_kubectl"**: Provides the command to configure `kubectl` for accessing the EKS cluster.

#### Karpenter Configuration

- **module "karpenter"**:
  - `source`: Specifies the Terraform module source for Karpenter.
  - `version`: Defines the module version.
  - `cluster_name`: Passes the cluster name.
  - `node_iam_role_use_name_prefix` & `node_iam_role_name`: Sets the IAM role for Karpenter nodes.
  - `create_pod_identity_association`: Enables Pod Identity Association.
  - `tags`: Adds tags for Karpenter resources.

#### Helm Chart for Karpenter

- **resource "helm_release" "karpenter"**:
  - `namespace`: Specifies the namespace for Karpenter.
  - `name`, `repository`, `chart`, `version`: Defines the Helm release name, repository, chart, and version.
  - `repository_username` & `repository_password`: Authenticates with the repository.
  - `values`: Sets custom values for the Helm chart.
  - `lifecycle { ignore_changes }`: Ignores changes to the repository password.

### File: `vpc.tf`

This file sets up the VPC using the `terraform-aws-modules/vpc/aws` module. Key components include:

- **module "vpc"**:
  - `source`: Specifies the module source.
  - `version`: Defines the module version.
  - `name`, `cidr`: Sets the VPC name and CIDR block.
  - `azs`: Lists the availability zones.
  - `private_subnets` & `public_subnets`: Defines the private and public subnets.
  - `enable_nat_gateway` & `single_nat_gateway`: Configures a single NAT gateway.
  - `public_subnet_tags` & `private_subnet_tags`: Tags subnets for Kubernetes and Karpenter auto-discovery.
  - `tags`: Adds tags to the VPC.

### File: `main.tf`

This file contains common Terraform configurations and provider setups.

- **terraform { required_version }**: Specifies the required Terraform version.
- **required_providers**: Defines required providers and their versions.
- **provider "aws"**: Configures the AWS provider.
- **provider "helm"**: Configures the Helm provider for managing Helm charts.
- **data "aws_ecrpublic_authorization_token" "token"**: Retrieves a public ECR authorization token.
- **data "aws_availability_zones" "available"**: Gets the available availability zones.
- **locals**: Defines local variables for the project name, region, VPC CIDR, availability zones, and tags.

### File: `karpenter.yaml`

This file configures Karpenter resources in Kubernetes.

- **apiVersion: karpenter.k8s.aws/v1beta1**:
  - **kind: EC2NodeClass**: Defines an EC2NodeClass resource.
    - `metadata`: Specifies the name.
    - `spec`: Configures the AMI family, IAM role, subnet and security group selectors, and tags.

- **apiVersion: karpenter.sh/v1beta1**:
  - **kind: NodePool**: Defines a NodePool resource.
    - `metadata`: Specifies the name.
    - `spec`: Configures the NodePool template, including node class reference, requirements, limits, and disruption policies.

### Steps to Set Up Karpenter

1. **Prepare Terraform Configuration**:
   - Ensure you have the `eks.tf`, `vpc.tf`, `main.tf`, and `karpenter.yaml` files in your Terraform project directory.

2. **Initialize Terraform**:
   ```sh
   terraform init
   ```

3. **Apply the VPC Configuration**:
   ```sh
   terraform apply -target=module.vpc
   ```

4. **Apply the EKS Cluster Configuration**:
   ```sh
   terraform apply -target=module.eks
   ```

5. **Deploy Karpenter Components**:
   ```sh
   terraform apply -target=module.karpenter
   ```

6. **Deploy Karpenter Helm Chart**:
   ```sh
   terraform apply -target=helm_release.karpenter
   ```

7. **Configure `kubectl`**:
   - Follow the output command from the `configure_kubectl` output to configure `kubectl`:
     ```sh
     aws eks --region us-west-2 update-kubeconfig --name <cluster_name>
     ```

8. **Deploy Karpenter Resources**:
   ```sh
   kubectl apply -f karpenter.yaml
   ```

By following these steps, you will set up an EKS cluster with Karpenter for efficient resource management and scaling.


