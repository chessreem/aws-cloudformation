# three-vpc-webapp-dynamodb

A three-VPC AWS web application infrastructure project defined with CloudFormation.

## Overview

This project creates a separated AWS network for a small web application:

- **Security VPC** contains an internet-facing Application Load Balancer.
- **Frontend VPC** contains an EC2 instance running NGINX.
- **Backend VPC** provides the DynamoDB-backed application data layer.
- An AWS Transit Gateway connects the VPCs and their route tables.
- A DynamoDB VPC endpoint allows the frontend instance to access DynamoDB privately.

The EC2 UserData script installs NGINX and uses the instance IAM role to retrieve application content from DynamoDB. If the expected item does not exist, the bootstrap process creates a default item.

## Templates

| Template | Purpose |
| --- | --- |
| `01-network.yaml` | VPCs, subnets, route tables, Transit Gateway, security groups, routes, and the DynamoDB endpoint |
| `02-dynamodb.yaml` | DynamoDB table and exported table values |
| `03-compute.yaml` | EC2 frontend instance, IAM role, and NGINX bootstrap configuration |
| `04-loadbalancer.yaml` | Application Load Balancer, target group, and listener |

## Deployment

Deploy the stacks in this order:

```text
01-network.yaml
02-dynamodb.yaml
03-compute.yaml
04-loadbalancer.yaml
```

The templates use CloudFormation exports and imports, so deploy them in the listed order and in the same AWS Region.

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