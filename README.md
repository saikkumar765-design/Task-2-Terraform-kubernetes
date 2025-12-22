# Task‑2: Deploy Authentik on AWS EKS using Terraform

This document provides **end‑to‑end, documentation** to provision an **AWS EKS (Managed Kubernetes) cluster using Terraform** and deploy the **Authentik identity platform** on top of it with:

* Separate pods for **Authentik Server**, **Authentik Worker**, **PostgreSQL**, and **Redis**
* **Persistent storage** for PostgreSQL using EBS
* **Automated database backups** using a Kubernetes CronJob

This guide is written so that **anyone seeing the project for the first time** can follow it successfully.

---

## 1. Architecture Overview

```
Local / EC2 Ubuntu VM
        │
        │ Terraform + kubectl
        ▼
AWS EKS Cluster (us‑east‑1)
        │
        ├─ Authentik Server
        ├─ Authentik Worker
        ├─ PostgreSQL
        ├─ Redis
        └─ CronJob
```

---

## 2. Prerequisites

### AWS Requirements

* AWS Account
* IAM **user** with `AdministratorAccess`
* AWS region: `us-east-1`

### Local / VM OS

* Ubuntu 20.04 or later (local machine or EC2 VM)

---

## 3. Install Required Tools (Ubuntu)

### 3.1 Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

Configure AWS CLI:

```bash
aws configure
```

---

### 3.2 Install Terraform

```bash
sudo apt update
sudo apt install -y gnupg software-properties-common
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt update && sudo apt install terraform -y
terraform -version
```

---

### 3.3 Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

---

### 3.4 Install eksctl (Optional but Recommended)

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

---

## 4. Project Structure

```
Task-2-Terraform-kubernetes/
│
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tf
│   └── .gitignore
│
├── kubernetes/
│   ├── secrets.yaml
│   ├── pvc.yaml
│   ├── postgres.yaml
│   ├── redis.yaml
│   ├── svc-postgres.yaml
│   ├── svc-redis.yaml
│   ├── authentik-server.yaml
│   ├── authentik-worker.yaml
│   ├── svc-authentik.yaml
│   └── backup-cronjob.yaml
│
└── README.md
```

---

## 5. Provision EKS Cluster Using Terraform

### 5.1 Initialize Terraform

```bash
cd terraform
terraform init
```

### 5.2 Validate Configuration

```bash
terraform validate
```

### 5.3 Create Infrastructure

```bash
terraform apply
```

Confirm with `yes`.

This creates:

* VPC + Subnets
* EKS Cluster
* Managed Node Group

---

## 6. Configure kubectl Access

```bash
aws eks --region us-east-1 update-kubeconfig --name <cluster-name>
```

Verify:

```bash
kubectl get nodes
```

---

## 7. Deploy Authentik Application (Kubernetes)

Change directory:

```bash
cd kubernetes
```

### 7.1 Create Secrets

```bash
kubectl apply -f secrets.yaml
```

Stores:

* `POSTGRES_PASSWORD`
* `AUTHENTIK_SECRET_KEY`

---

### 7.2 Create Persistent Storage

```bash
kubectl apply -f pvc.yaml
```

* Uses **dynamic EBS provisioning**
* No manual PV required

---

### 7.3 Deploy PostgreSQL + Service

```bash
kubectl apply -f postgres.yaml
kubectl apply -f svc-postgres.yaml
```

---

### 7.4 Deploy Redis + Service

```bash
kubectl apply -f redis.yaml
kubectl apply -f svc-redis.yaml
```

---

### 7.5 Deploy Authentik

```bash
kubectl apply -f authentik-server.yaml
kubectl apply -f authentik-worker.yaml
kubectl apply -f svc-authentik.yaml
```

---

### 7.6 Deploy Backup CronJob

```bash
kubectl apply -f backup-cronjob.yaml
```

---

## 8. Verify Deployment

```bash
kubectl get pods
kubectl get svc
kubectl get pvc
kubectl get cronjob
```

All pods should be in `Running` state.

---

## 9. Access Authentik UI

### Option 1: Port Forward (Recommended for Demo)

```bash
kubectl port-forward svc/authentik 9000:9000
```

Open browser:

```
http://localhost:9000
```



---

## 11. Backup Strategy Explained

### Where Backups Are Stored

* Location: `/backup/db.sql`
* Storage: **EBS volume attached to PostgreSQL PVC**

### How Backup Works

1. Kubernetes **CronJob** runs daily
2. Executes `pg_dump`
3. Connects to PostgreSQL via service
4. Writes SQL dump to persistent volume

### Restore Example

```bash
kubectl exec -it deployment/postgres -- psql -U authentik -d authentik < /backup/db.sql
```

---

## 12. Security & Best Practices

* PostgreSQL and Redis are **not exposed publicly**
* Secrets stored in Kubernetes Secrets
* Only Authentik server is externally accessible

---

## 13. Cleanup

```bash
cd terraform
terraform destroy
```

---

## 14. Conclusion

This project demonstrates:

* Infrastructure as Code with Terraform
* Managed Kubernetes (EKS)
* Stateful applications on Kubernetes
* Secure secret management
* Automated database backups



---

# Task‑2 (Helm): Deploy Authentik on AWS EKS using Terraform and Helm

This document provides **end‑to‑end, from‑scratch documentation** to provision an **AWS EKS cluster using Terraform** and deploy **Authentik using the official Helm chart**.

This version **replaces raw Kubernetes YAML manifests** with **Helm**, following Authentik best practices and supporting **safe PostgreSQL upgrades**.

---

## 1. Architecture Overview

```
Local / EC2 Ubuntu VM
        │
        │ Terraform + kubectl + Helm
        ▼
AWS EKS Cluster
        │
        └─ Helm Release: Authentik
            ├─ Authentik Server (Web UI)
            ├─ Authentik Worker
            ├─ PostgreSQL (StatefulSet + EBS PVC)
            └─ Redis (StatefulSet)
```

---

## 2. Prerequisites

### AWS
- AWS Account
- IAM user with AdministratorAccess
- Region (example): `ap-south-1`

### OS
- Ubuntu 20.04+

---

## 3. Tool Installation (Ubuntu)

### 3.1 AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

### 3.2 Terraform
```bash
sudo apt update
sudo apt install -y gnupg software-properties-common
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt update && sudo apt install terraform -y
```

### 3.3 kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### 3.4 eksctl (optional)
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

### 3.5 Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## 4. Project Structure

```
Task-2-Terraform-kubernetes/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tf
└── helm/
    └── authentik/
        └── values.yaml
```

---

## 5. Provision EKS using Terraform

```bash
cd terraform
terraform init
terraform validate
terraform apply
```

---

## 6. Configure kubectl

```bash
aws eks --region ap-south-1 update-kubeconfig --name <cluster-name>
kubectl get nodes
```

---

## 7. Deploy Authentik using Helm

### 7.1 Add Helm Repo
```bash
helm repo add authentik https://charts.goauthentik.io
helm repo update
```

### 7.2 Create Namespace
```bash
kubectl create namespace authentik
kubectl config set-context --current --namespace=authentik
```

### 7.3 values.yaml
```yaml
authentik:
  secret_key: "CHANGE_ME"

  postgresql:
    host: authentik-postgresql
    user: authentik
    password: authentikpassword
    name: authentik

  redis:
    host: authentik-redis

server:
  replicas: 1
  service:
    type: LoadBalancer
    port: 9000

worker:
  replicas: 1

postgresql:
  enabled: true
  auth:
    username: authentik
    password: Authentik@123
    database: authentik
  primary:
    persistence:
      enabled: true
      size: 10Gi

redis:
  enabled: true
  master:
    persistence:
      enabled: true
      size: 5Gi
```

### 7.4 Install Helm Chart
```bash
helm install authentik authentik/authentik -f values.yaml
```

---

## 8. Verification

```bash
kubectl get pods
kubectl get svc
kubectl get pvc
```

---

## 9. Access Authentik UI

```bash
kubectl get svc authentik
```

Open:
```
http://<EXTERNAL-IP>:9000
```

---

## 10. PostgreSQL Access

```bash
kubectl port-forward svc/authentik-postgresql 5432:5432
psql -h localhost -U authentik -d authentik
```

---

## 11. Helm Upgrade (PostgreSQL Safe)

```bash
helm upgrade authentik authentik/authentik -f values.yaml
```

Rollback:
```bash
helm rollback authentik 1
```

---

## 12. Cleanup

```bash
helm uninstall authentik
cd terraform
terraform destroy
```

---

## 13. Conclusion

This project demonstrates:
- Terraform-based EKS provisioning
- Helm-based application lifecycle management
- Stateful workloads on Kubernetes
- Upgrade-safe PostgreSQL management

**End of Documentation**

