# Explanation of Key Concepts in EKS Deployment

This document provides an overview of the core components used in the deployment process of applications on Amazon EKS (Elastic Kubernetes Service). Understanding these components clarifies their configuration and role in supporting the EKS architecture.

## 1. EKS Cluster

**Amazon EKS** is a managed Kubernetes service that simplifies running Kubernetes clusters in the AWS environment. By using EKS, you offload cluster management tasks such as scaling and updating, allowing you to focus on deploying applications.

## 2. OpenID Connect (OIDC) Provider

**OIDC (OpenID Connect)** is an identity layer built on top of OAuth 2.0 that enables Kubernetes clusters to delegate authentication and authorization to AWS IAM roles. By associating an OIDC provider with your EKS cluster, service accounts can assume IAM roles, granting them permissions to securely interact with AWS services.

## 3. Application Load Balancer (ALB)

**Application Load Balancer (ALB)** is a fully managed load balancing service that automatically distributes incoming application traffic across multiple targets, such as EC2 instances, containers (running on Fargate), or IP addresses.

### Why ALB?
- **Layer 7 Load Balancing**: Operates at the application layer, allowing for advanced routing and path-based rules, ideal for HTTP/HTTPS traffic.
- **Ingress Integration**: Used as an ingress controller to expose applications running inside the EKS cluster to the internet or a private network.

## 4. Target Groups

**Target Groups** are utilized to route requests to a group of targets registered behind a load balancer. In the context of EKS, targets are typically Kubernetes pods or services.

### How It Works:
- **Pod Registration**: The ALB ingress controller automatically registers the running pods in the corresponding target group when an ingress resource is applied.

## 5. Ingress Controller

**Ingress Controller** manages access to Kubernetes services from external traffic. An ingress resource defines rules for routing traffic to services, while the ingress controller implements these rules using a load balancer like ALB.

### How It Works:
- **Ingress Resource**: Defines routing rules for the load balancer. For instance, you can route `https://example.com/` to one service and `https://example.com/api/` to another.
- **ALB Controller**: Works with the ingress resource to configure an ALB that manages external access to your services, automatically creating target groups based on the backend services defined in the ingress resource.

## 6. IAM Roles for Service Accounts (IRSA)

**IAM Roles for Service Accounts (IRSA)** allows Kubernetes pods to assume IAM roles via service accounts, providing the necessary permissions for pods to interact securely with AWS resources.

### How It Works:
- **Service Account with IAM Role**: A Kubernetes service account is annotated with the ARN of the IAM role it should assume. When a pod uses this service account, it gains access to AWS resources as defined by that role.

## 7. Fargate Profile

**Fargate** is a serverless compute engine for containers that allows you to run Kubernetes pods without managing the underlying infrastructure. A Fargate profile defines which pods should run on Fargate.

### Why Fargate?
- **No Infrastructure Management**: Eliminates the need to provision or manage EC2 instances.
- **Auto-Scaling**: Automatically scales up and down based on pod demand.
- **Cost Efficiency**: You only pay for the resources utilized by running pods.

### How It Works:
- **Namespace and Labels**: A Fargate profile maps specific namespaces and labels to Fargate, ensuring that all pods in those namespaces are scheduled on Fargate rather than EC2 instances.

## Summary of Configuration

1. **OIDC** is configured to delegate IAM roles to Kubernetes service accounts for secure AWS resource access.
2. **ALB** is deployed as an ingress controller to expose applications running in the cluster and route traffic based on rules defined in the ingress resource.
3. **Target Groups** automatically manage and route traffic to the backend services (pods) running inside the cluster.
4. **Ingress Controller** defines how incoming traffic is routed to services using the ALB.
5. **IRSA** assigns specific permissions to pods based on the service account they run under, enhancing security.
6. **Fargate** provides a serverless environment for running Kubernetes pods, eliminating the need for managing EC2 instances.

---
