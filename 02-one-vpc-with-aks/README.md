# one-vpc-webapp-dynamodb

A single-VPC AWS web application infrastructure project defined with CloudFormation.

## Overview

This project creates a small web application network with one VPC and four subnets:

- Two public subnets in separate Availability Zones host the internet-facing Application Load Balancer.
- A private frontend subnet hosts the EC2 instance running NGINX.
- A separate private backend subnet is reserved for backend workloads. DynamoDB is an AWS-managed regional service, not a resource deployed inside a subnet; its VPC endpoint is associated with both private route tables.

The EC2 UserData script installs NGINX and uses the instance IAM role to retrieve application content from DynamoDB. If the expected item does not exist, the bootstrap process creates a default item.

## Templates

| Template | Purpose |
| --- | --- |
| `01-network.yaml` | VPC, four subnets, route tables, security groups, and S3/DynamoDB endpoints |
| `02-dynamodb.yaml` | DynamoDB table and exported table values |
| `03-compute.yaml` | EC2 frontend instance, IAM role, and NGINX bootstrap configuration |
| `04-loadbalancer.yaml` | Application Load Balancer, target group, and listener |
| `nested-stack.yaml` | Parent stack that deploys all four templates |

## Deployment

Deploy the stacks in this order:

```text
01-network.yaml
02-dynamodb.yaml
03-compute.yaml
04-loadbalancer.yaml
```
The templates use CloudFormation exports and imports, so deploy them in the listed order and in the same AWS Region.

The parent template can be packaged and deployed as one stack. CloudFormation uploads the four child templates to S3 during packaging.
So using the nested-stack.yaml:
```powershell
cd aws-cloudformation/02-one-vpc-with-aks
aws cloudformation package `
	--template-file nested-stack.yaml `
	--s3-bucket YOUR_ARTIFACT_BUCKET `
	--output-template-file packaged-nested-stack.yaml

aws cloudformation deploy `
	--template-file packaged-nested-stack.yaml `
	--stack-name one-vpc-webapp-dynamodb `
	--capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM `
	--parameter-overrides InstancePassword='REPLACE_WITH_A_STRONG_PASSWORD'
```

## Access

After the load balancer stack completes, use the Application Load Balancer DNS name from its outputs to access the application.

## Removal

Delete the stacks in reverse deployment order:

```text
04-loadbalancer
03-compute
02-dynamodb
01-network
```

Or if using nested-stack.yaml - delete the parent stack; CloudFormation deletes the nested stacks and their resources in the required reverse dependency order.