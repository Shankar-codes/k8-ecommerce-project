# 🛒 k8-ecommerce-project

A fully containerized, microservices-based e-commerce platform deployed on **Kubernetes**. This repository contains the **application-layer manifests** — Dockerfiles, Deployments, and Services — for every microservice in the stack. It is designed to work alongside the companion [k8-ecommerce-database](https://github.com/Shankar-codes/k8-ecommerce-database) repository, which provisions the stateful data layer (MySQL, MongoDB, Redis, RabbitMQ) as Kubernetes StatefulSets backed by AWS EBS.

> 🐳 **Primary language: Dockerfile** — every service ships its own container image definition.

---

## 📐 Architecture Overview

The platform follows a microservices architecture where each service is independently containerized, deployed as a Kubernetes `Deployment`, and exposed via a Kubernetes `Service`. All workloads run inside the `ellamma-ecommerce` namespace.

```
                        ┌──────────────────────────────────────────┐
                        │    Kubernetes Cluster (ellamma-ecommerce) │
                        │                                           │
              ┌─────────▼──────────┐                               │
   Internet ──►  Frontend Service  │  (LoadBalancer / NodePort)    │
              └─────────┬──────────┘                               │
                        │  HTTP                                     │
          ┌─────────────┼──────────────┐                           │
          │             │              │                            │
   ┌──────▼──────┐ ┌────▼──────┐ ┌────▼──────┐                    │
   │  Catalogue  │ │   Cart    │ │   User    │                    │
   │  Service    │ │  Service  │ │  Service  │                    │
   └──────┬──────┘ └────┬──────┘ └────┬──────┘                    │
          │              │              │                           │
   ┌──────▼──────┐ ┌────▼──────┐ ┌────▼──────┐                    │
   │   MongoDB   │ │   Redis   │ │   MySQL   │  ◄── StatefulSets  │
   │(StatefulSet)│ │(StatefulSet│ │(StatefulSet   (k8-ecommerce-  │
   └─────────────┘ └───────────┘ └───────────┘    database repo)  │
                                                                    │
   ┌──────────────────────────────────────┐                        │
   │   Payment Service  │  Shipping Service│                        │
   └──────────────┬──────┴─────────────────┘                       │
                  │  AMQP                                           │
           ┌──────▼──────┐                                         │
           │  RabbitMQ   │  ◄── StatefulSet (k8-ecommerce-database)│
           └─────────────┘                                         │
                        └──────────────────────────────────────────┘
```

---

## 📦 Services

| Directory    | Service Role                                           | Data Store   |
|--------------|--------------------------------------------------------|--------------|
| `frontend/`  | Web UI — serves the e-commerce storefront              | —            |
| `user/`      | User registration, authentication, profile management  | MySQL        |
| `catalogue/` | Product listings, search, inventory display            | MongoDB      |
| `cart/`      | Shopping cart — session and item management            | Redis        |
| `payment/`   | Order payment processing (async via message queue)     | RabbitMQ     |
| `shipping/`  | Order fulfilment and shipment tracking                 | RabbitMQ     |
| `mysql/`     | MySQL connection manifests                             | Persistent   |
| `mongodb/`   | MongoDB connection manifests                           | Persistent   |
| `redis/`     | Redis cache manifests                                  | Persistent   |
| `rabbitmq/`  | RabbitMQ message broker manifests                      | Persistent   |
| `debug/`     | Ephemeral debug Pod for in-cluster troubleshooting     | —            |

---

## 🗂️ Repository Structure

```
k8-ecommerce-project/
├── 01-namespce.yaml          # Kubernetes Namespace (ellamma-ecommerce)
│
├── frontend/                 # Frontend microservice
│   ├── Dockerfile
│   ├── deployment.yaml
│   └── service.yaml
│
├── user/                     # User service
│   ├── Dockerfile
│   ├── deployment.yaml
│   └── service.yaml
│
├── catalogue/                # Catalogue service
│   ├── Dockerfile
│   ├── deployment.yaml
│   └── service.yaml
│
├── cart/                     # Cart service
│   ├── Dockerfile
│   ├── deployment.yaml
│   └── service.yaml
│
├── payment/                  # Payment service
│   ├── Dockerfile
│   ├── deployment.yaml
│   └── service.yaml
│
├── shipping/                 # Shipping service
│   ├── Dockerfile
│   ├── deployment.yaml
│   └── service.yaml
│
├── mysql/                    # MySQL connection manifests
├── mongodb/                  # MongoDB connection manifests
├── redis/                    # Redis connection manifests
├── rabbitmq/                 # RabbitMQ connection manifests
│
└── debug/                    # Debug utility pod for in-cluster troubleshooting
    └── deployment.yaml
```

---

## 🏷️ Namespace

All resources are deployed into a dedicated namespace:

```yaml
# 01-namespce.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ellamma-ecommerce
  labels:
    project: ecommerce-project
    environment: development
```

---

## 🔗 Relationship to Companion Repositories

This project is part of a three-repository ecosystem:

| Repository | Purpose |
|---|---|
| **[k8-ecommerce-project](https://github.com/Shankar-codes/k8-ecommerce-project)** ← *this repo* | Application Deployments, Services, and Dockerfiles |
| **[k8-ecommerce-database](https://github.com/Shankar-codes/k8-ecommerce-database)** | StatefulSets for MySQL, MongoDB, Redis, RabbitMQ with persistent AWS EBS volumes |
| **[k8-resources](https://github.com/Shankar-codes/k8-resources)** | Core Kubernetes concept reference manifests (learning resource) |

The database repository must be deployed **before** this one. All StatefulSets and the `ellamma-ebs` StorageClass must be up and running before application services can connect to their data stores.

---

## 🚀 Getting Started

### Prerequisites

- AWS EKS cluster (or any Kubernetes 1.24+ cluster)
- `kubectl` configured against your cluster
- Docker and access to a container registry (Docker Hub, AWS ECR, etc.)
- The [k8-ecommerce-database](https://github.com/Shankar-codes/k8-ecommerce-database) repo deployed and all StatefulSets ready

### Step 1 — Deploy the Database Layer First

```bash
git clone https://github.com/Shankar-codes/k8-ecommerce-database.git
cd k8-ecommerce-database

kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-ebs-storageclass.yaml
kubectl apply -f mysql/
kubectl apply -f mongodb/
kubectl apply -f redis/
kubectl apply -f rabbitmq/

# Wait for all StatefulSets to be ready before proceeding
kubectl rollout status statefulset --namespace ellamma-namaspace
```

### Step 2 — Build and Push Service Images

Each service directory contains a `Dockerfile`. Build and push an image for each service:

```bash
# Example — user service
cd user/
docker build -t <your-registry>/ecommerce-user:latest .
docker push <your-registry>/ecommerce-user:latest
```

Update the `image:` field in each `deployment.yaml` to reference your registry before applying.

### Step 3 — Deploy the Application Layer

```bash
git clone https://github.com/Shankar-codes/k8-ecommerce-project.git
cd k8-ecommerce-project

# 1. Create the namespace
kubectl apply -f 01-namespce.yaml

# 2. Apply data-layer connection manifests
kubectl apply -f mysql/
kubectl apply -f mongodb/
kubectl apply -f redis/
kubectl apply -f rabbitmq/

# 3. Deploy backend microservices
kubectl apply -f user/
kubectl apply -f catalogue/
kubectl apply -f cart/
kubectl apply -f payment/
kubectl apply -f shipping/

# 4. Deploy the frontend last
kubectl apply -f frontend/
```

### Step 4 — Verify Deployment

```bash
# Check all pods are Running
kubectl get pods -n ellamma-ecommerce

# Check services and their endpoints
kubectl get svc -n ellamma-ecommerce

# Check deployments rollout status
kubectl rollout status deployment --namespace ellamma-ecommerce
```

### Step 5 — Access the Application

```bash
# Get the external IP/hostname of the frontend service
kubectl get svc frontend -n ellamma-ecommerce
```

Open the `EXTERNAL-IP` in your browser. If using NodePort, navigate to `http://<NodeIP>:<NodePort>`.

---

## 🐛 Debugging

The `debug/` directory contains an ephemeral utility Pod you can deploy inside the cluster for in-network troubleshooting — useful for testing DNS resolution, inter-service connectivity, and database reachability from within the cluster network.

```bash
# Deploy the debug pod
kubectl apply -f debug/

# Exec into it
kubectl exec -it <debug-pod-name> -n ellamma-ecommerce -- /bin/sh

# Test service connectivity from inside the cluster
curl http://user/health
curl http://catalogue/health
nslookup mysql.ellamma-namaspace.svc.cluster.local
nslookup mongodb.ellamma-namaspace.svc.cluster.local
```

---

## 🔌 Inter-Service Communication

| Source        | Target        | Protocol    | Purpose                        |
|---------------|---------------|-------------|--------------------------------|
| Frontend      | User          | HTTP/REST   | Authentication, profile        |
| Frontend      | Catalogue     | HTTP/REST   | Product listings               |
| Frontend      | Cart          | HTTP/REST   | Cart management                |
| Cart          | Payment       | HTTP/REST   | Checkout initiation            |
| Payment       | RabbitMQ      | AMQP        | Publish order events           |
| Shipping      | RabbitMQ      | AMQP        | Consume order events           |
| User          | MySQL         | TCP/3306    | Persistent user data           |
| Catalogue     | MongoDB       | TCP/27017   | Product catalogue data         |
| Cart          | Redis         | TCP/6379    | Session and cart cache         |

---

## 🛠️ Tech Stack

| Layer           | Technology                       |
|-----------------|----------------------------------|
| Orchestration   | Kubernetes (AWS EKS)             |
| Containerization| Docker                           |
| Frontend        | React / Nginx                    |
| Backend Services| Node.js                          |
| Relational DB   | MySQL                            |
| Document DB     | MongoDB                          |
| Cache           | Redis                            |
| Message Queue   | RabbitMQ                         |
| Persistent Storage | AWS EBS (via CSI driver)      |

---

## 📝 Notes

- The root namespace file has a typo in its filename — `01-namespce.yaml` (missing the `a` in `namespace`). Apply it as-is; the resource name inside is correct.
- The `environment: development` label indicates this is a dev/learning environment. For production, add resource requests/limits, Horizontal Pod Autoscaling (HPA), NetworkPolicies, and proper secrets management (AWS Secrets Manager or HashiCorp Vault).
- The `debug/` Pod should be removed from the cluster once troubleshooting is complete.

---

## 👤 Author

**Shankar Thimmappa** — DevOps Trainer  
GitHub: [@Shankar-codes](https://github.com/Shankar-codes)
