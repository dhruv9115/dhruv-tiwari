---
# layout: post
title: "Multi-Region EKS Setup"
date: 2024-09-17
categories: [Blog]
tags: [EKS, Multi-Region, Kubernetes]
---

## Overview
The objective of this article is to create an architecture with the following objectives:
- High availability across global regions with multi-region AWS EKS architecture.
- Automated deployment using Terraform for consistent, scalable infrastructure.
- CI/CD pipeline for automated infrastructure deployment across AWS regions.

## Architecture

<p align="center">
  <img src="{{ '/assets/images/architecture_overview.jpeg' | relative_url }}" alt="Architecture Diagram">
</p>

## IAC - Terraform with CloudPosse Atmos

### How It Works

1. **Create an EKS Component**: This component will contain the Terraform module for EKS. For this example, we’ll use `terraform-aws-modules/eks/aws`.

2. **Create `main.tf` with Parameters**:
   - Create the file in the directory `components/terraform/eks`.
   - Define the EKS module:

   {% highlight hcl %}
   module "eks_cluster" {
     source  = "terraform-aws-modules/eks/aws"
     version = "~> 19.0"
     
     # Cluster configuration
     cluster_name    = var.cluster_name
     cluster_version = var.cluster_version
     # ……
   }
   {% endhighlight %}

3. **Create `bastion.tf` to create bastion hosts to access the cluster**
   {% highlight hcl %}
   module "ec2_bastion" {
    for_each = local.public_subnet_ids
    source = "cloudposse/ec2-bastion-server/aws"

    enabled = module.this.enabled

    instance_type               = var.instance_type
    security_groups             = compact(concat([module.vpc.outputs.vpc_default_security_group_id], var.security_groups))
    subnets                     = each.value
    key_name                    = var.key_name
    user_data                   = var.user_data
    vpc_id                      = local.vpc_id
    associate_public_ip_address = var.associate_public_ip_address

    context = module.this.context
   }
   {% endhighlight %}

4. **Create `remote-state.tf` to import the VPC deployed through the VPC component**
   {% highlight hcl %}
   module "vpc" {
       source = "cloudposse/stack-config/yaml//modules/remote-state"
       version = "1.5.0"
       component = var.vpc_component_name
       atmos_base_path = "../../../"
       context = module.this.context
   }
   {% endhighlight %}

5. **Create `variables.tf`, `outputs.tf` according to the Terraform modules**

6. **Create a directory `stacks/catalog/eks` (catalog calls all the AWS service components)**

7. **Create a `defaults.yaml` inside the above directory, and call default values for the EKS service**
   {% highlight yaml %}
   components:
     terraform:
       eks:
       metadata:
           type: abstract
           component: eks
       vars:
           cluster_version: "1.30"
           eks_managed_node_groups:
             on_demand:
               instance_type: "t3.medium"
               desired_capacity: 3
               max_capacity: 6
               min_capacity: 3
               key_name: "eks-cluster-key"
               node_group_name_prefix: "on-demand"
               additional_tags:
                 on-demand: "true"
               taints:
               - key: "workload-type"
                 value: "frontend"
                 effect: "NoSchedule"

             spot:
               instance_type: "t3.medium"
               desired_capacity: 3
               max_capacity: 6
               min_capacity: 3
               key_name: "eks-cluster-key"
               node_group_name_prefix: "spot"
               spot_price: "0.05"
               additional_tags:
                 spot: "true"
               taints:
               - key: "workload-type"
                   value: "backend"
                   effect: "NoSchedule"

           fargate_profiles:
             backend:
               selectors:
               - namespace: "backend-services"
           key_name: "eks-cluster-key"
           associate_public_ip_address: true
   {% endhighlight %}

8. **Create `<env>.yaml`, for example, `qa.yaml`, which creates the EKS cluster, with on-demand, spot, and Fargate nodes, with proper taints:**
   {% highlight yaml %}
   import:
     - catalog/eks/defaults
   components:
     terraform:
       eks_cluster_qa:
       metadata:
           component: eks
       vars:
           environment: "qa"
           vpc_component_name: "vpc-eks"
           cluster_name: "qa-cluster"
   {% endhighlight %}

9. **Create `stacks/mixins/region/us-east-1.yaml` and `stacks/mixins/region/us-west-1.yaml` and declare default values for the region, for example:**
   {% highlight yaml %}
   vars:
     region: us-east-1
     environment: ue1
   {% endhighlight %}

10. **Create `stacks/orgs/<your_org_name>/<bu>/qa/us-east-1.yaml` and `stacks/orgs/<your_org_name>/<bu>/qa/us-west-1.yaml**
    {% highlight yaml %}
    import:
    - catalog/vpc/qa
    - catalog/eks/qa
    {% endhighlight %}

11. **Now it should be visible from `atmos` CLI**
    <p align="center">
      <img src="{{ '/assets/images/atmos-cli.png' | relative_url }}" alt="Example">
    </p>

12. **Run the plan for each region, for this example `us-east-1` and `us-west-1`**

13. **For apply, I would recommend using GitHub Actions**
    <p align="center">
      <img src="{{ '/assets/images/githubaction-run.png' | relative_url }}" alt="GitHub Actions run">
    </p>

    A sample workflow:
    {% highlight yaml %}
      - name: Plan Atmos Component
        uses: cloudposse/github-action-atmos-terraform-plan@v2
        with:
          component: ${{ github.event.inputs.component }}
          stack: ${{ github.event.inputs.stack }}
          debug: true
          atmos-config-path: /home/runner/work/_temp/atmos-config
          atmos-version: ${{ env.ATMOS_VERSION }}
        .....
      - name: Terraform Apply
        uses: cloudposse/github-action-atmos-terraform-apply@v2
        with:
          component: ${{ github.event.inputs.component }}
          stack: ${{ github.event.inputs.stack }}
          atmos-config-path: /home/runner/work/_temp/atmos-config
          atmos-version: ${{ env.ATMOS_VERSION }}
    {% endhighlight %}

## EKS Cluster Architecture

<p align="center">
  <img src="{{ '/assets/images/eks_arch.jpeg' | relative_url }}" alt="EKS Cluster Architecture">
</p>

### 1. Overall Architecture
- This architecture diagram represents a highly available and fault-tolerant AWS environment for running **Kubernetes nodes using Amazon EKS (Elastic Kubernetes Service)**.
- The architecture spans **three Availability Zones (AZs)** to ensure resilience and redundancy in case of failure.

### 2. VPC Configuration
- The infrastructure operates inside a **Virtual Private Cloud (VPC)** with a CIDR block of **10.88.0.0/16**.
- The VPC is divided into **public subnets** and **private subnets** across three availability zones (AZ A, AZ B, and AZ C).

### 3. Public Subnets
- Each availability zone contains a **Public Subnet**:
  - AZ A: **10.88.1.0/24**
  - AZ B: **10.88.2.0/24**
  - AZ C: **10.88.3.0/24**
- Resources hosted in these public subnets include:
  - **NAT Gateways**: Used to allow instances in private subnets to securely access the internet without exposing them directly.
  - **Bastion Hosts**: Provide secure SSH access to instances in private subnets, accessible only via trusted IP addresses.
  - **External ALB**: Routes traffic from the internet to services hosted in private subnets.

### 4. Private Subnets
- Each availability zone contains **Private Subnets**:
  - AZ A: **10.88.11.0/24**
  - AZ B: **10.88.12.0/24**
  - AZ C: **10.88.13.0/24**
- These private subnets host resources such as:
  - **Amazon EKS Worker Nodes**: Containers running Kubernetes workloads.
  - **Internal ALB**: Facilitates routing between internal services.
  - **Auto Scaling Group**: Automatically manages the scaling of EKS worker nodes.

### 5. Elastic Load Balancer (ALB)
- Two types of load balancers are used in this architecture:
  - **External ALB**: Exposed to the internet via the **Internet Gateway (IGW)**, it distributes incoming traffic across EKS worker nodes in private subnets.
  - **Internal ALB**: Resides in the **private subnets** and routes traffic between internal services and EKS worker nodes. It is not exposed to the internet.

### 6. Internet Gateway (IGW)
- The **Internet Gateway (IGW)** allows public subnets to communicate with the internet, providing inbound and outbound internet access for the external ALB, NAT gateways, and bastion hosts.

### 7. NAT Gateway
- **NAT Gateways** are deployed in each **public subnet** to provide **outbound internet access** for instances in private subnets.
- EKS nodes in private subnets can connect to the internet to pull Docker images or access other external resources without being directly exposed.

### 8. VPC Endpoints for S3 and DynamoDB (Optional)
- **VPC Gateway Endpoints** allow instances in private subnets to securely access **Amazon S3** and **Amazon DynamoDB** without going through the internet or NAT Gateway, improving security and reducing costs.
  - **S3 Endpoint**: Enables internal access to S3 buckets for object storage.
  - **DynamoDB Endpoint**: Enables direct access to DynamoDB tables from private subnets.

### 9. Stateful Resources (Amazon S3 and DynamoDB)
- **Amazon S3**:
  - Provides scalable, durable, and highly available **object storage**. S3 buckets can be accessed from the private subnets via **VPC Gateway Endpoints**.
  - **Cross-Region Replication (CRR)** is enabled for disaster recovery, replicating data to a different AWS region.
  
- **Amazon DynamoDB**:
  - A fully managed, highly available **NoSQL database**. DynamoDB tables can be accessed from private subnets through **DynamoDB VPC Gateway Endpoints**.
  - **Global Tables** are enabled to provide multi-region replication, ensuring low-latency access and high availability across different regions.

### 10. Security Considerations
- **Bastion Hosts**: Only accessible through sessions manager
- **Security Groups**: Applied to EKS nodes, ALBs, and NAT gateways to control both inbound and outbound traffic based on the principle of least privilege.
- **VPC Endpoints**: Improve security by enabling private access to S3 and DynamoDB without using the public internet.

### 11. Traffic Flow:
- **External Traffic** (Public):
  - **Internet -> IGW -> External ALB -> EKS Nodes (Private Subnets)**.

- **Internal Traffic** (Private):
  - **Internal Service -> Internal ALB -> EKS Nodes (Private Subnets)**.

- **Outbound Internet Traffic** (from Private Subnets):
  - **EKS Nodes (Private Subnets) -> NAT Gateway -> Internet**.

- **S3 and DynamoDB Access** (via VPC Endpoints):
  - **EKS Nodes (Private Subnets) -> VPC Gateway Endpoints -> S3/DynamoDB**.

### 12. Route Tables
- **Public Subnets**: Route to **Internet Gateway (IGW)** for public-facing traffic.
- **Private Subnets**: Route through **NAT Gateway** for outbound internet access.
- **VPC Endpoints**: Optional entries for direct access to **S3** and **DynamoDB** without NAT Gateway.

### 13. High Availability and Fault Tolerance
- The architecture is spread across **three Availability Zones (AZs)** to ensure high availability.
- Critical services like **NAT Gateways**, **ALBs**, and **EKS worker nodes** are distributed across all AZs to prevent single points of failure.
- **Auto Scaling** ensures that the number of worker nodes scales based on demand.



## Blue-Green Deployment with ArgoCD

Assuming that argocd has already been deployed to eks, either through manifests or through terraform

**Steps**
1. **Create Two Environments:** Deploy two versions of the application (blue and green) using separate namespaces or ArgoCD Applications.
2. **Deploy the Green Environment:** Deploy the new version (green) without affecting the blue version.
3. **Switch Traffic to Green:** Use an Ingress Controller or update the Service object to switch traffic to the green environment. ArgoCD can handle this update.
4. **Test the Green Version:** Ensure that the new (green) version is working correctly before fully switching traffic.
5. **Rollback to Blue if Needed:** If the green environment has issues, rollback by switching traffic back to the blue environment.


## Monitoring and Observability

### Monitoring Tools:
- **Amazon CloudWatch:** Use CloudWatch to monitor EKS cluster metrics, such as CPU/Memory usage, and set up alerts based on thresholds.
   {% highlight hcl %}
   module "eks_cluster" {
     source  = "terraform-aws-modules/eks/aws"
     version = "~> 19.0"
     
     # Cluster log configuration
     cluster_enabled_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]
     # ……
   }
   {% endhighlight %}

- **Prometheus and Grafana:** Install Prometheus in the EKS cluster to collect metrics from Kubernetes resources and applications.
   {% highlight hcl %}
    module "prometheus" {
      source  = "terraform-aws-modules/managed-service-prometheus/aws"
      version = "v3.0.0"

      workspace_alisas = "qa"
    }
   {% endhighlight %}
      {% highlight hcl %}
    module "grafana" {
      source  = "terraform-aws-modules/managed-service-grafana/aws"
      version = "v2.2.0"

      name   = var.name
      region = var.region

      permission_type  = "SERVICE_MANAGED"
      notification_destinations = ["SNS"]

      data_sources = ["CLOUDWATCH", "PROMETHEUS"]
    }

   {% endhighlight %}
## Cost and Cost Optimization

### Costs Involved
1. **EKS Clusters:** Charged for the control plane and worker nodes (EC2 instances).
2. **Load Balancers:** Costs for handling incoming traffic.
3. **Route 53:** Charged for DNS queries and health checks.
4. **S3, DynamoDB:** Storage costs for state files (if using Terraform backend).
5. **NAT Gateways:** Charged by data processing and usage.

### Cost Optimization Strategies
1. **Spot Instances:** Use spot instances for worker nodes to save up to 90% of EC2 costs.
2. **Auto Scaling:** Ensure efficient scaling of EKS worker nodes based on traffic.
3. **Rightsize Resources:** Optimize instance types and sizes for both nodes and bastion hosts.
4. **Reserved Instances/Savings Plans:** For long-running workloads, commit to reserved instances or savings plans for significant cost savings.
5. **Use Fargate for Small Workloads:** Instead of managing EC2 instances, use AWS Fargate for small workloads with lower traffic to avoid overhead costs.
