# ☸️ Student Registration App — Three-Tier Architecture on Kubernetes

<div align="center">

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![Vite](https://img.shields.io/badge/Vite-646CFF?style=for-the-badge&logo=vite&logoColor=white)
![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white)

*Three-tier Student Registration app (React + Vite / Spring Boot / MariaDB) fully orchestrated on Kubernetes — StatefulSet, Deployments, HPA, Secrets, PVC, and Ingress*

</div>

---

## 📌 Overview

A three-tier **Student Registration CRUD Application** deployed on Kubernetes. The stack — **React (Vite)** frontend, **Spring Boot** backend, and **MariaDB** database — is containerized and fully orchestrated using production-grade Kubernetes patterns: StatefulSet for the database, HPA for auto-scaling, Secrets for credentials, PVC for persistent storage, and Ingress for routing.

---

## 🏗️ Architecture

```
  Browser
     │
     ▼
┌──────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                   │
│                                                       │
│  ┌────────────────────────────────────────────────┐  │
│  │  Ingress Controller                            │  │
│  │  Routes external traffic → frontend service   │  │
│  └───────────────────────┬────────────────────────┘  │
│                          │                            │
│  ┌───────────────────────▼────────────────────────┐  │
│  │  Tier 1 — Frontend                             │  │
│  │  React + Vite   Deployment   NodePort Service  │  │
│  │  HPA: auto-scales on CPU                       │  │
│  │  VITE_API_URL → backend NodePort               │  │
│  └───────────────────────┬────────────────────────┘  │
│                          │ ClusterIP :8080            │
│  ┌───────────────────────▼────────────────────────┐  │
│  │  Tier 2 — Backend                              │  │
│  │  Spring Boot   Deployment   ClusterIP Service  │  │
│  │  HPA: auto-scales on CPU                       │  │
│  │  spring.datasource.url → db StatefulSet ClusterIP│ │
│  └───────────────────────┬────────────────────────┘  │
│                          │ ClusterIP :3306            │
│  ┌───────────────────────▼────────────────────────┐  │
│  │  Tier 3 — Database                             │  │
│  │  MariaDB   StatefulSet   ClusterIP Service     │  │
│  │  Secret → credentials    PVC → persistent data │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

---

## 📁 Repository Structure

```
student-app-kubernetes/
│
├── frontend/                       # React + Vite source + Dockerfile
│   ├── src/
│   ├── .env                        # VITE_API_URL=http://<backend-ip>:<nodeport>/api
│   ├── Dockerfile
│   └── package.json
│
├── backend/                        # Spring Boot source + Dockerfile
│   ├── src/
│   ├── application.properties      # DB connection via MariaDB ClusterIP
│   └── Dockerfile
│
├── database-yaml/                  # Kubernetes manifests — database tier
│   ├── statefulset.yaml
│   ├── service.yaml
│   ├── secret.yaml
│   └── pv-pvc.yaml
│
├── simple-deploy/                  # Simplified starter manifests
│   ├── deployment.yaml
│   └── service.yaml
│
├── docker-compose.yml              # Local development stack
├── init.sql                        # Database schema initialisation
└── README.md
```

---

## ☸️ Kubernetes Resources

| Tier | Resource | Purpose |
|---|---|---|
| **Database** | `StatefulSet` | Stable pod identity + ordered startup for MariaDB |
| **Database** | `Service (ClusterIP)` | Internal access on port `3306` |
| **Database** | `Secret` | Store MariaDB credentials (base64 encoded) |
| **Database** | `PersistentVolume` + `PVC` | Persistent storage — data survives pod restarts |
| **Backend** | `Deployment` | Manage Spring Boot pods |
| **Backend** | `Service (ClusterIP)` | Internal access on port `8080` |
| **Backend** | `HPA` | Auto-scale pods based on CPU utilisation |
| **Frontend** | `Deployment` | Manage React/Vite pods |
| **Frontend** | `Service (NodePort)` | Expose frontend externally |
| **Frontend** | `HPA` | Auto-scale pods based on CPU utilisation |
| **Frontend** | `Ingress` | Route external HTTP traffic to frontend service |

---

## 🚀 Deployment Guide

### Prerequisites

- Kubernetes cluster (Minikube / EKS / k3s)
- `kubectl` configured and connected to the cluster
- Docker + DockerHub account (for building and pushing images)

---

### Step 1 — Build and Push Docker Images

**Backend image:**
```bash
cd backend/

# Edit application.properties — set the DB connection to the K8s service name
# spring.datasource.url=jdbc:mariadb://db-service:3306/student_db

docker build -t <your-dockerhub-username>/student-backend:latest .
docker login
docker push <your-dockerhub-username>/student-backend:latest
```

**Frontend image:**
```bash
cd ../frontend/

# Edit .env — point to backend NodePort
# VITE_API_URL=http://<cluster-node-ip>:<backend-nodeport>/api

docker build -t <your-dockerhub-username>/student-frontend:latest .
docker push <your-dockerhub-username>/student-frontend:latest
```

---

### Step 2 — Create Kubernetes Secret for Database Credentials

```bash
# Encode your credentials in base64
echo -n 'root'         | base64   # → cm9vdA==
echo -n 'your_password' | base64  # → replace with your encoded value
```

Edit `database-yaml/secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: student-app
type: Opaque
data:
  db-username: cm9vdA==               # base64 of 'root'
  db-password: <base64-encoded-pass>  # never commit real passwords
```

---

### Step 3 — Deploy All Tiers

```bash
# Deploy database tier first (StatefulSet must be ready before backend starts)
kubectl apply -f database-yaml/

# Wait for the database pod to be Running
kubectl get pods -n student-app -w

# Deploy backend and frontend
kubectl apply -f backend-yaml/
kubectl apply -f frontend-yaml/
```

---

### Step 4 — Verify Everything is Running

```bash
# Check all resources
kubectl get all -n student-app

# Check HPA status
kubectl get hpa -n student-app

# Check Ingress
kubectl get ingress -n student-app

# List all services across namespaces
kubectl get svc -A
```

---

### Step 5 — Access the Application

```bash
# For Minikube
minikube service frontend-service -n student-app

# For NodePort on any cluster
http://<cluster-node-ip>:<frontend-nodeport>

# For Ingress
http://<ingress-external-ip>
```

---

## ⚙️ Backend `application.properties`

```properties
server.port=8080
spring.datasource.url=jdbc:mariadb://<db-statefulset-clusterip>:3306/student_db
spring.datasource.username=root
spring.datasource.password=<injected-via-k8s-secret>
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

> In production, inject `spring.datasource.password` from the Kubernetes Secret using `valueFrom.secretKeyRef` in the Deployment manifest — never hardcode it in `application.properties`.

---

## 📋 Sample HPA — Backend

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: student-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## 💻 Local Development with Docker Compose

To run the full stack locally without Kubernetes:

```bash
docker compose up -d

# Frontend: http://localhost:3000
# Backend:  http://localhost:8080
# Database: localhost:3306
```

---

## 🔍 Useful kubectl Commands

```bash
# Watch pod status live
kubectl get pods -n student-app -w

# Stream backend logs
kubectl logs -f deployment/backend -n student-app

# Describe a pod for debugging
kubectl describe pod <pod-name> -n student-app

# Scale backend manually
kubectl scale deployment backend --replicas=4 -n student-app

# Execute into the database pod
kubectl exec -it <db-pod-name> -n student-app -- mariadb -u root -p
```

---

## ⚠️ Security Notes

| Issue | Location | Fix |
|---|---|---|
| Hardcoded DB password in `application.properties` | `backend/application.properties` | Inject via `secretKeyRef` in the backend Deployment manifest |
| Hardcoded backend IP in frontend `.env` | `frontend/.env` | Use Ingress hostname or ClusterIP service name resolved at runtime |
| Hardcoded credentials in `README.md` | Repo root `README.md` | Use placeholder values only — never commit real passwords to Git |

---

## 🔗 Related Projects

| Repo | Description |
|---|---|
| [student-app-docker-compose](https://github.com/Sanket006/student-app-docker-compose) | Same app running locally with Docker Compose |
| [crud-app-aws-ec2-rds](https://github.com/Sanket006/crud-app-aws-ec2-rds) | Same app deployed on AWS EC2 + RDS |
| [jenkins-cicd-pipelines](https://github.com/Sanket006/jenkins-cicd-pipelines) | CI/CD pipeline that builds and pushes the Docker images |
| [terraform-aws-iac](https://github.com/Sanket006/terraform-aws-iac) | Terraform to provision the EKS cluster this runs on |

---

## 👨‍💻 Author

**Sanket Ajay Chopade** — DevOps Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/sanketchopade07)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)](https://github.com/Sanket006)
