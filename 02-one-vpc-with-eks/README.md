# one-vpc-nginx-eks

A single-VPC AWS NGINX microservice on Amazon EKS, defined with CloudFormation.

## Overview

This project creates one VPC with five subnets and an EKS-hosted NGINX service:

- Two public subnets in separate Availability Zones host the internet-facing Application Load Balancer.
- Two private frontend subnets in separate Availability Zones provide EKS control-plane networking and managed worker nodes.
- A separate private backend subnet is reserved for backend workloads.
- The NGINX Deployment runs two pods and is exposed through a Kubernetes NodePort Service behind the ALB.
- The private subnets have no default internet route. ECR, EKS, STS, CloudWatch Logs, and S3 are reached through VPC endpoints.
- The NGINX image is served from an ECR pull-through cache. Seed the cache from an internet-connected workstation before enabling EKS.
- DynamoDB remains a separate managed service; it is not deployed inside a subnet.

The NGINX Deployment is created by the EKS CloudFormation stack using the official NGINX container image.

## Templates

| Template | Purpose |
| --- | --- |
| `01-network.yaml` | VPC, five subnets, private service endpoints, S3/DynamoDB gateway endpoints, and the ECR image cache rule |
| `02-dynamodb.yaml` | DynamoDB table and exported table values |
| `03-eks.yaml` | EKS control plane, managed node group, IAM access, and NGINX Kubernetes resources |
| `04-loadbalancer.yaml` | Application Load Balancer and EKS node group registration for the NGINX NodePort |
| `nested-stack.yaml` | Parent stack that deploys all four templates |

## Deployment

The private subnets have no internet route. Deploy in two stages so the ECR cache can be seeded before the EKS nodes try to pull NGINX.

First package and deploy the parent with its default `DeployApplication=false`. This creates the network, service endpoints, cache rule, and DynamoDB table:

```text
01-network.yaml
02-dynamodb.yaml
03-eks.yaml
04-loadbalancer.yaml
```
```powershell
cd aws-cloudformation/02-one-vpc-with-eks
aws cloudformation package `
	--template-file nested-stack.yaml `
	--s3-bucket YOUR_ARTIFACT_BUCKET `
	--output-template-file packaged-nested-stack.yaml

aws cloudformation deploy `
	--template-file packaged-nested-stack.yaml `
	--stack-name one-vpc-nginx-eks `
	--capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

After the network stack finishes, seed the ECR Public pull-through cache from a workstation with internet access. This avoids the first-cache-pull NAT requirement for workloads in the private subnets:

```powershell
aws ecr batch-import-upstream-image `
	--repository-name public-ecr/nginx/nginx `
	--image-tag 1.27-alpine `
	--region YOUR_AWS_REGION
```

Then enable the EKS and ALB stacks by updating the parent stack:

```powershell
aws cloudformation deploy `
	--template-file packaged-nested-stack.yaml `
	--stack-name one-vpc-nginx-eks `
	--capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM `
	--parameter-overrides DeployApplication=true ClusterPublicAccessCidrs='YOUR_PUBLIC_IP/32'
```

The cache rule uses Amazon ECR Public as its upstream. The cache must be seeded after the network stack creates the rule and before the application stack is enabled.

## Access

After deployment, use the Application Load Balancer URL from the parent stack outputs to reach NGINX. Restrict `ClusterPublicAccessCidrs` to trusted IP ranges; the example deploy command allows only your public IP. The manifest-provider Lambda uses the cluster's private endpoint, so this restriction does not block stack provisioning. Kubernetes IAM authentication is still required.

To administer the cluster with `kubectl`:

```powershell
aws eks update-kubeconfig --name nginx-microservice --region YOUR_AWS_REGION
kubectl get deployments,services
```

Interface endpoints have hourly and data-processing charges. The private subnet route tables contain no `0.0.0.0/0` route; AWS service traffic is limited to the configured endpoints.

## Removal

Delete the stacks in reverse deployment order:

```text
04-loadbalancer
03-eks
02-dynamodb
01-network
```

Or if using nested-stack.yaml - delete the parent stack; CloudFormation deletes the nested stacks and their resources in the required reverse dependency order.