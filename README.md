# EKS Application Deployment with ALB

This guide walks through the steps to deploy an application on an AWS EKS cluster, configure ingress with ALB, set up OIDC authentication, add necessary security groups, and ensure ALB access via ingress by configuring IAM permissions.

## Prerequisites

Ensure the following tools are installed and configured:

- **AWS CLI**: Download and configure the AWS CLI from [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
  
  After installation, configure it with your AWS credentials:

  ```bash
  aws configure
  ```

- **eksctl**: Install eksctl using the following command:

  ```bash
  curl -LO https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz
  tar -xzf eksctl_Linux_amd64.tar.gz
  sudo mv eksctl /usr/local/bin
  ```

- **kubectl**: Install kubectl using the following command:

  ```bash
  curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
  chmod +x ./kubectl
  sudo mv ./kubectl /usr/local/bin/kubectl
  ```

## Steps

### 1. Set Up an EKS Cluster

Create an EKS cluster using `eksctl`. In this example, we are creating a Fargate-enabled cluster:

```bash
eksctl create cluster \
    --name <cluster-name> \
    --region <region> \
    --fargate
```

After the cluster is created, update your kubeconfig to interact with it:

```bash
aws eks update-kubeconfig --name <cluster-name> --region <region>
```

### 2. Create a Fargate Profile

To run pods on AWS Fargate, create a Fargate profile for your application. Here, we use the namespace `game-2048`:

```bash
eksctl create fargateprofile \
    --cluster <cluster-name> \
    --region <region> \
    --name alb-sample-app \
    --namespace game-2048
```

### 3. Deploy the Application

Deploy your application on the EKS cluster. The example below uses a Kubernetes YAML file to deploy the 2048 game:

```bash
kubectl apply -f 2048_app.yml
```

Ensure the `2048_app.yml` file contains the correct `Deployment`, `Service`, and `Namespace` configurations.

### 4. Configure OIDC Connector

To enable Kubernetes service accounts to access AWS resources, configure an OIDC provider for your EKS cluster:

1. Check if your cluster has an OIDC provider associated with it:

   ```bash
   aws eks describe-cluster --name <cluster-name> --region <region> --query "cluster.identity.oidc.issuer"
   ```

2. If no OIDC provider exists, create one using the following command:

   ```bash
   eksctl utils associate-iam-oidc-provider \
       --region <region> \
       --cluster <cluster-name> \
       --approve
   ```

### 5. Configure ALB Ingress Controller

The ALB Ingress Controller enables you to manage Application Load Balancers for your ingress resources.

#### Step-by-Step ALB Configuration:

1. **Create a Security Group for ALB**

   You need to create a security group that allows inbound access on ports 80 and 443 for HTTP/HTTPS traffic to your ALB:

   ```bash
   aws ec2 create-security-group --group-name ALB-SecurityGroup --description "Security group for ALB" --vpc-id <vpc-id>
   ```

   Authorize inbound access on port 80 (HTTP):

   ```bash
   aws ec2 authorize-security-group-ingress --group-id <security-group-id> --protocol tcp --port 80 --cidr 0.0.0.0/0
   ```

   Authorize inbound access on port 443 (HTTPS):

   ```bash
   aws ec2 authorize-security-group-ingress --group-id <security-group-id> --protocol tcp --port 443 --cidr 0.0.0.0/0
   ```

2. **Create an IAM Policy for the ALB Ingress Controller**

   Download the policy document from AWS:

   ```bash
   curl -o alb-ingress-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json
   ```

   Create the IAM policy:

   ```bash
   aws iam create-policy \
       --policy-name ALBIngressControllerIAMPolicy \
       --policy-document file://alb-ingress-iam-policy.json
   ```

3. **Create a Kubernetes Service Account with IAM Role**

   Create a Kubernetes service account and associate it with the IAM policy:

   ```bash
    eksctl create iamserviceaccount \
    --cluster=<cluster-name> \
    --namespace=kube-system \
    --name=alb-ingress-controller \
    --attach-policy-arn=arn:aws:iam::<account-id>:policy/ALBIngressControllerIAMPolicy \
    --role-name=alb-ingress-controller-role \
    --approve
   ```

4. **Deploy ALB Ingress Controller**

   Add the Helm repository for the ALB Ingress Controller:

   ```bash
   helm repo add eks https://aws.github.io/eks-charts
   helm repo update
   ```

   Install the ALB Ingress Controller:

   ```bash
   helm install alb-ingress-controller eks/aws-load-balancer-controller \
       --namespace kube-system \
       --set clusterName=<cluster-name> \
       --set serviceAccount.create=false \
       --set serviceAccount.name=alb-ingress-controller
   ```

### 5. Access the Application via ALB

After deploying the ALB Ingress Controller and configuring ingress, verify the ALB:

1. Retrieve the public DNS name of the ALB:

   ```bash
   kubectl get ingress -n game-2048
   ```

   This will return the external ALB URL. You can access your application by navigating to this URL in a browser.

2. **(Optional)**: Set up a custom domain by configuring DNS settings in Route53 or your DNS provider. Update the Ingress resource to use your custom domain under the `host` field.

### 6. Delete the Cluster

To clean up resources, delete the EKS cluster:

```bash
eksctl delete cluster --name <cluster-name> --region <region>
```

---