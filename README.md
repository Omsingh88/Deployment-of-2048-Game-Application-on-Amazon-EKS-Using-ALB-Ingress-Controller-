# EKS Deployment of 2048 Game Application with ALB Ingress Controller

## Overview
This project demonstrates deploying the 2048 game on Amazon EKS with Fargate and ALB. By following this guide, you’ll have a fully functioning game on AWS with load balancing and scalability.


## Prerequisites
Before we begin, make sure you have the following tools installed on your system:

- AWS CLI: This command-line tool allows you to interact with AWS services.
- eksctl: A handy tool that simplifies the process of creating and managing EKS clusters.
- kubectl: This command-line tool enables you to communicate with your Kubernetes clusters.
- Helm: A package manager for Kubernetes that makes it easier to deploy, manage, and version applications within your cluster.


Once these are installed, use the following command to configure your AWS credentials.

```bash
aws configure
```
![aws](images/1.png)

