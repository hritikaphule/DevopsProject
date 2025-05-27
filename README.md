# Architecture 
![image](https://github.com/user-attachments/assets/5a644a7d-4b3c-4e7f-8923-79222e466033)

---

# Production-Ready 3-Tier Web App on AWS EKS (React + Flask + PostgreSQL)

This project demonstrates how to deploy a fully functional 3-tier web application on AWS Elastic Kubernetes Service (EKS). It includes:

* A **React frontend** for the user interface
* A **Flask backend API** for business logic
* A **PostgreSQL database** hosted on Amazon RDS (private subnet)

All infrastructure and application components are orchestrated using **Kubernetes** with declarative YAML manifests. AWS-native integrations such as **Load Balancer Controller**, **IAM roles**, **Route53**, and **OIDC** are used for secure and scalable access.

---

## 🚀 Why Kubernetes?

Kubernetes is now the industry standard for deploying and scaling containerized applications. AWS offers **EKS** — a managed Kubernetes service that removes the operational overhead of managing the control plane. This project showcases:

* EKS with **managed node groups**
* Private RDS with **secure SG configuration**
* DNS-based service discovery for RDS
* Application Load Balancer using **Ingress**

---

## 🔧 Tools Used

* AWS EKS, EC2, RDS, IAM, Route53
* `eksctl`, `kubectl`, `Helm`, `AWS CLI`
* Docker (prebuilt images)

---

## 🏗️ Cluster Setup with EKSCTL

```bash
eksctl create cluster \
  --name my-eks-cluster \
  --region us-west-2 \
  --version 1.31 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

---

## 🗄️ PostgreSQL RDS in Private Subnets

```bash
# Get VPC ID from the cluster
VPC_ID=$(aws eks describe-cluster --name my-eks-cluster --region us-west-2 --query "cluster.resourcesVpcConfig.vpcId" --output text)

# Find private subnets
PRIVATE_SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[?MapPublicIpOnLaunch==\`false\`].SubnetId" \
  --output text)

# Create DB subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name my-postgres-subnet-group \
  --db-subnet-group-description "Private subnet group" \
  --subnet-ids <subnet-ids> \
  --region us-west-2

# Create Security Group for RDS
aws ec2 create-security-group \
  --group-name rds-sg \
  --description "RDS access SG" \
  --vpc-id $VPC_ID

# Authorize ingress from EKS nodes
NODE_SG=$(aws eks describe-cluster --name my-eks-cluster --region us-west-2 \
  --query "cluster.resourcesVpcConfig.securityGroupIds[0]" --output text)

aws ec2 authorize-security-group-ingress \
  --group-id <rds-sg-id> \
  --protocol tcp \
  --port 5432 \
  --source-group $NODE_SG \
  --region us-west-2

# Launch the RDS PostgreSQL instance
aws rds create-db-instance \
  --db-instance-identifier my-postgres-db \
  --db-instance-class db.t3.small \
  --engine postgres \
  --allocated-storage 20 \
  --master-username postgresadmin \
  --master-user-password YourStrongPassword123! \
  --db-subnet-group-name my-postgres-subnet-group \
  --vpc-security-group-ids <rds-sg-id> \
  --no-publicly-accessible \
  --region us-west-2
```

---

## 🛠️ Kubernetes Setup

### Namespace

```bash
kubectl apply -f namespace.yaml
```

### RDS ExternalName Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
  namespace: 3-tier-app-eks
spec:
  type: ExternalName
  externalName: my-postgres-db.cn6kaos6emcz.us-west-2.rds.amazonaws.com
  ports:
    - port: 5432
```

### Secrets and ConfigMap

```bash
echo 'postgresadmin' | base64
# -> cG9zdGdyZXNhZG1pbg==
echo 'YourStrongPassword123!' | base64
# -> WW91clN0cm9uZ1Bhc3N3b3JkMTIzIQ==
```

**secrets.yaml** and **configmap.yaml** created and applied accordingly.

---

## 🧱 Database Migration

```bash
kubectl apply -f migration_job.yaml
kubectl logs <migration-pod> -n 3-tier-app-eks
```

This job uses `flask db upgrade` to initialize the schema.

---

## 🚀 Backend and Frontend Deployment

```bash
kubectl apply -f backend.yaml
kubectl apply -f frontend.yaml
```

Check everything:

```bash
kubectl get deployment -n 3-tier-app-eks
kubectl get svc -n 3-tier-app-eks
kubectl get pods -n 3-tier-app-eks
```

---

## 🌐 Accessing the App via Port Forward

Open two terminals:

```bash
kubectl port-forward -n 3-tier-app-eks svc/backend 8000:5000
kubectl port-forward -n 3-tier-app-eks svc/frontend 8080:80
```

Now visit:

* Backend: [http://localhost:8000](http://localhost:8000)
* Frontend: [http://localhost:8080](http://localhost:8080)

---

## 🔒 Ingress + ALB (Optional Advanced Setup)

Set up ALB ingress with:

* OIDC provider
* IAM role for service account
* AWS Load Balancer Controller (Helm)
* Ingress resource
* Route53 + Hosted Zone + Domain (optional)

See the full `ingress.yaml` for ingress class and rules.

---

## 📚 Quiz Application Functionality

This is a DevOps quiz app. Use the **Manage Questions** UI to upload questions via CSV.

CSV location:

```
backend/questions-answers/sample.csv
```

---

## 🧹 Cleanup

```bash
eksctl delete cluster --name my-eks-cluster --region us-west-2
```

---

## 🙌 Credits

This project was originally inspired by a blog tutorial and was adapted to fit my custom use case and learning goals.
