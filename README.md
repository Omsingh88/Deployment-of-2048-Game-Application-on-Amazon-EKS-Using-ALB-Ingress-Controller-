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
   Use the following command to create an EKS cluster with Fargate:
   ```bash
   eksctl create cluster --name demo-cluster --region us-east-1 --fargate
Update Kubeconfig
Update your kubeconfig file to access the newly created EKS cluster:

```bash
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```
Create Fargate Profile
Create a Fargate profile for the application:
```bash
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region us-east-1 \
  --name fargate-profile \
  --namespace game-2048
```
Deploy the Game Application
Apply the YAML configuration for the 2048 game:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```
Here’s the content of the configuration:
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
Using IAM OIDC Provider
Integrate IAM with the OIDC provider:

```bash
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```
Download IAM Policy
Download the IAM policy for the ALB Ingress Controller:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

Create IAM Policy
Create the IAM policy:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```
Create IAM Role
Create an IAM role for the ALB Ingress Controller:
```
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
Deploy ALB Controller
Add the Helm repository and install the ALB controller:

```
helm repo add eks https://aws.github.io/eks-charts

helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region> \
  --set vpcId=<your-vpc-id>
```
Verify the Deployment
Check if the ALB Ingress Controller is running:
```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
Access the Application
Finally, check the ingress to get the address or domain to access the application in your browser:
```
kubectl get ingress -n game-2048
```
