# three-vpc-nginx-eks

A three-VPC AWS web application with NGINX running as a microservice on Amazon EKS.

## Overview

- The **Security VPC** hosts an internet-facing Application Load Balancer.
- The **Frontend VPC** contains two private subnets in separate Availability Zones for the EKS control plane, managed worker nodes, and NGINX pods.
- The **Backend VPC** contains the DynamoDB-backed application data layer.
- A Transit Gateway connects the three VPCs. The ALB forwards to EKS worker private IPs on NodePort `30080`.
- The EKS Kubernetes API is private. NGINX runs two replicas behind a Kubernetes NodePort Service.
- An in-VPC Lambda reads the homepage HTML from DynamoDB and synchronizes it to a Kubernetes ConfigMap every minute. NGINX mounts that ConfigMap as `index.html`.
- Private ECR, EKS, EC2, STS, CloudWatch Logs, S3, and DynamoDB endpoints provide AWS service access without a NAT gateway.

## Templates

| Template | Purpose |
| --- | --- |
| `01-network.yaml` | Three VPCs, two frontend AZ subnets, Transit Gateway, private service endpoints, and ECR pull-through cache |
| `02-dynamodb.yaml` | DynamoDB table and exported table values |
| `03-eks.yaml` | Private EKS cluster, managed node group, NGINX Deployment/Service, and homepage synchronization |
| `04-loadbalancer.yaml` | Security VPC ALB and scheduled EKS node IP target reconciliation |
| `nested-stack.yaml` | Parent stack that deploys the child templates |

## Deployment

Deploy the parent stack in two stages. The private worker nodes cannot make the first public NGINX image pull, so seed the ECR pull-through cache after the network stack creates it and before enabling EKS.

First package and deploy with the default `DeployApplication=false`. This creates the VPCs, service endpoints, cache rule, and DynamoDB table:

```powershell
cd aws-cloudformation/03-three-vpc-webapp-eks
aws cloudformation package `
    --template-file nested-stack.yaml `
    --s3-bucket YOUR_ARTIFACT_BUCKET `
    --output-template-file packaged-nested-stack.yaml

aws cloudformation deploy `
    --template-file packaged-nested-stack.yaml `
    --stack-name three-vpc-webapp-eks `
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

After the first deployment completes, seed the NGINX image into the private ECR cache from a workstation with internet access:

```powershell
aws ecr batch-import-upstream-image `
    --repository-name public-ecr/nginx/nginx `
    --image-tag 1.27-alpine `
    --region YOUR_AWS_REGION
```

Then enable EKS and the ALB by updating the parent stack:

```powershell
aws cloudformation deploy `
    --template-file packaged-nested-stack.yaml `
    --stack-name three-vpc-webapp-eks `
    --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM `
    --parameter-overrides DeployApplication=true
```

The interface VPC endpoints have hourly and data-processing charges. Restrict `SecurityVpcCidr` and other CIDR parameters to the intended network ranges before deploying outside a sandbox.

## Access

After deployment, use `ApplicationLoadBalancerUrl` from the parent stack outputs. The ALB target reconciler runs every minute to handle managed node replacements and scaling. Homepage changes in DynamoDB are copied to the NGINX pods on the same interval.

## Removal

Delete the parent stack. CloudFormation removes the nested stacks and their resources in reverse dependency order.