# 🚀 Deploying a hello world FastAPI App on AWS EKS (Manual Setup via AWS Console)

This project demonstrates how to deploy a simple **FastAPI "Hello World"** application on **AWS EKS (Kubernetes)** using:

- Docker  
- Amazon ECR (Container Registry)  
- Amazon VPC (Networking)  
- Amazon EKS (Kubernetes Cluster)  
- AWS Load Balancer (Public access)

The final result is a **public URL** that returns:

```json
{"message": "hello, world!"}
```
## Architecture
```
Internet
   |
AWS Load Balancer (created by Kubernetes Service)
   |
EKS Cluster
   |
Worker Nodes (EC2)
   |
FastAPI Pod (Docker image from ECR)
```
## Prerequisites
- AWS account with AdministratorAccess
- Region: ap-south-1 (Mumbai)
- Local tools:
  - AWS CLI
  - kubectl
  - Docker
- Configure AWS CLI: `aws configure`

## PHASE 1 – Set AWS Region
1. Go to AWS Console
2. Top-right region selector → choose Asia Pacific (Mumbai) – ap-south-1
From now on, do everything in this region only.

## PHASE 2 – Create ECR (Docker Image Registry)
**Create Repository**
1. Go to ECR
2. Click Create repository
3. Choose:
  Visibility: Private
  Repository name: hello-fastapi
  Click Create repository

**Push Your Docker Image to ECR**
1. Click your repo → View push commands
2. AWS shows 4 commands. Run them in your terminal exactly. Example:
```
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
docker build -t hello-fastapi .
docker tag hello-fastapi:latest <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/hello-fastapi:latest
docker push <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/hello-fastapi:latest
```
After this:
- Go back to ECR Console
- Open repo → Images
- You should see 1.0

## PHASE 3 – Create VPC (Network)
1. Go to VPC
2. Click Create VPC
3. Choose VPC and more
4. Fill:
```
| Setting         | Value              |
| --------------- | ------------------ |
| Name            | `hello-vpc`        |
| IPv4 CIDR       | default            |
| AZs             | 2                  |
| Public subnets  | 2                  |
| Private subnets | 2                  |
| NAT gateways    | 1 (to reduce cost) |
| VPC endpoints   | None               |
```
5. Click Create VPC. Wait till status = Available

## PHASE 4 – Create EKS Cluster
1. Go to EKS
2. Click Add cluster → Create
3. Do not choose the auto-mode as it incurs extra cost, and we don't need its capabilties for this project.
4. Fill:
```
| Field                | Value                              |
| -------------------- | ---------------------------------- |
| Name                 | `hello-eks`                        |
| Kubernetes version   | default                            |
| Cluster service role | Create new role (default EKS role) |
```
> Make sure the role has the policy 'AmazonEKSClusterPolicy'
4. Click Next
**Networking Page**
- VPC: hello-vpc
- Subnets: Select all private subnets
- Endpoint access: Public and private
5. Leave every other settings/fields as is.
6. Click Create. Takes ~10mins

## PHASE 5 – Add Worker Nodes (EC2)
Your cluster currently has zero machines.
1. EKS → Clusters → hello-eks
2. Go to Compute tab
3. Click Add node group
4. Fill:
```
| Setting         | Value         |
| --------------- | ------------- |
| Node group name | `hello-nodes` |
| Node IAM role   | Create new    |
| Instance type   | `t3.medium`    |
| Desired         | 2             |
| Min             | 1             |
| Max             | 2             |
```
> Make sure the role has these polcies- AmazonEKSWorkerNodePolicy, AmazonEC2ContainerRegistryReadOnly, AmazonEKS_CNI_Policy.
> 
> If you choose instance type as t3.micro, only 4 pods can be scheduled there. Most of the EKS/k8s related pods take up those 4. So your application pod can't be scheduled then.
> Choose t3.medium for very basic use like this example.
5. Click Create. Wait till nodes become Active

## PHASE 6 – Connect kubectl to EKS
On your laptop:  
`aws eks update-kubeconfig --region ap-south-1 --name hello-eks`  
Verify:  
`kubectl get nodes`  
You should see 2 nodes in `Ready` state.  

## PHASE 7 – Deploy Your FastAPI App
Update the `image` field in the deployment.yaml file and apply it using:  
`kubectl apply -f deployment.yaml`  
Check:  
`kubectl get pods`  

## PHASE 8 – Expose App with Load Balancer
Apply the service.yaml file:  
`kubectl apply -f service.yaml`  
Get URL:  
`kubectl get svc`  
You'll see somethign like:  
<img width="975" height="78" alt="image" src="https://github.com/user-attachments/assets/885da995-55ac-427b-b276-1396c423dbae" />  

## PHASE 9 – Test
Open browser -> http://abcdef123.ap-south-1.elb.amazonaws.com/  
You should see:  
```json
{"message":"hello, world!"}
```
<img width="975" height="358" alt="image" src="https://github.com/user-attachments/assets/584221b5-b446-4d64-a8cd-20d55aefa215" />  

Boom! Done, Sir/Madam!  

## VERY IMPORTANT – COST WARNING
- EKS is not free tier.
- nor is NAT
- nor are the t3.medium EC2 instances
- nor is the load balancer
## PHASE 10 – Cleanup (Do this when done)
Console:  
- Delete EKS Node Group
- Delete EKS Cluster
- Delete VPC
- Delete ECR repo  
kubectl:
```
kubectl delete svc hello-eks-manual-way-service
kubectl delete deploy hello-eks-manual-way
```
After deleting the `svc`, the load balancer should also be ideally deleted. Just double check it. Bilkul riks nai lene ka.  

## Next Up
Automate all of the above.  
  
`Ashte`
