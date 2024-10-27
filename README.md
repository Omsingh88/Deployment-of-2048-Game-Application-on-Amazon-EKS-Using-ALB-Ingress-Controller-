# EKS Deployment of 2048 Game Application with ALB Ingress Controller

## Overview

This project demonstrates the deployment of the classic **2048 game** on **Amazon EKS** using **Fargate** and an **Application Load Balancer (ALB)** for external access. The aim is to create a scalable and robust architecture for running a containerized application on Kubernetes.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Project Setup](#project-setup)
- [Deployment Steps](#deployment-steps)
- [Screenshots](#screenshots)
- [Conclusion](#conclusion)

## Prerequisites

Before you begin, ensure you have the following:
- AWS Account
- AWS CLI installed and configured
- `kubectl` installed
- `eksctl` installed
- `Helm` installed

## Project Setup

1. **Create EKS Cluster with Fargate**  
The first step is to set up an EKS cluster with Fargate, which allows running Kubernetes pods without managing the underlying infrastructure.
```bash
   eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```
2. **Update Kubeconfig to Connect with the Cluster**
To interact with the EKS cluster, update the kubeconfig file with your cluster's credentials.

```bash
   aws eks update-kubeconfig --name demo-cluster --region us-east-1
```
3. **Create Fargate Profile**
   Define a Fargate profile to specify which pods run on Fargate. This configuration isolates the game application into its namespace, game-2048.
```bash
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region us-east-1 \
  --name fargate-profile \
  --namespace game-2048
```
4. **Deploy the 2048 Game Application**
   Download and apply the YAML configuration file that defines the Kubernetes resources needed for deploying the 2048 game. This configuration includes a Deployment, Service, and Ingress.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
Here’s the content of the configuration:
This YAML configuration defines:

**Namespace for isolating the game resources.**
**Deployment with 5 replicas of the 2048 game app.**
**Service of type NodePort for exposing the application internally.**
**Ingress resource for managing external access via an Application Load Balancer (ALB).**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: game-2048
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: game-2048
  name: deployment-2048
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048
  replicas: 5
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048
    spec:
      containers:
      - image: public.ecr.aws/l6m2t8p7/docker-2048:latest
        imagePullPolicy: Always
        name: app-2048
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  namespace: game-2048
  name: service-2048
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    app.kubernetes.io/name: app-2048
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: service-2048
              port:
                number: 80  
```
5. **Associate IAM OIDC Provider with the EKS Cluster**
To allow AWS services to access Kubernetes resources, create an IAM OIDC provider.

```bash
   eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```
6. **Download the IAM Policy for ALB Controller**
Download the IAM policy that grants the necessary permissions for the ALB Ingress Controller.

```bash
   curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

7. **Create the IAM Policy**
Create an IAM policy in AWS using the policy file downloaded above.

```bash
   aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
8. **Create IAM Role for the ALB Ingress Controller**
This role enables the ALB Ingress Controller to authenticate and interact with AWS resources.
```
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
9. **Deploy the AWS ALB Ingress Controller**
Add the EKS Helm repository, update it, and install the ALB Ingress Controller on your cluster.
- Add Helm Repository

```
helm repo add eks https://aws.github.io/eks-charts
```
- Update Helm Repository
```
helm repo update eks
```
- Install ALB Controller
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```
10. **Verify the Deployment**
Check if the ALB Ingress Controller has been successfully deployed.
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
11. **Access the 2048 Game Application**
Finally, check the ingress to get the address or domain to access the application in your browser:
```
kubectl get ingress -n game-2048
```
