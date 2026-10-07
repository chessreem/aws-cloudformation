# Single-VPC NGINX on EKS

This stack creates one VPC, a public subnet for a single `t3.medium` managed worker node, and a second subnet in another Availability Zone for EKS control-plane placement. The EKS API endpoint has both public and private access enabled.

The `nginx.yaml` manifest creates a ConfigMap-backed NGINX homepage, a Deployment with two replicas, and a NodePort Service on TCP port `30080`. The CloudFormation template adds an inbound rule for port `30080` to the EKS cluster security group.

## Deploy

Create the infrastructure from this directory:

```powershell
aws cloudformation deploy `
    --template-file eks-cluster.yaml `
    --stack-name simple-eks-cluster `
    --capabilities CAPABILITY_IAM `
    --region YOUR_AWS_REGION
```

Wait for the stack deployment to complete, then configure `kubectl` for the cluster:

```powershell
aws eks update-kubeconfig `
    --name simple-eks-cluster `
    --region YOUR_AWS_REGION
```

Apply the NGINX manifest and check that its pods and service are ready:

```powershell
kubectl apply -f nginx.yaml
kubectl get pods,service nginx
```

## Access NGINX

Find the worker node's public IPv4 address in the EC2 console, then open:

```text
http://WORKER_PUBLIC_IP:30080
```

The page displays the homepage content from the `nginx-homepage` ConfigMap.

## Security

The port `30080` ingress currently allows traffic from `0.0.0.0/0`, so the NGINX NodePort is publicly reachable through the worker node's public IP. Restrict `CidrIp` on `NodePortIngress` in `eks-cluster.yaml` to your trusted client IP range before using this outside a temporary test. The worker node is in a public subnet and receives a public IP at launch.

## Remove

Remove the Kubernetes resources and then the CloudFormation stack:

```powershell
kubectl delete -f nginx.yaml
aws cloudformation delete-stack `
    --stack-name simple-eks-cluster `
    --region YOUR_AWS_REGION
```