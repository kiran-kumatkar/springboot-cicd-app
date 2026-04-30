# 🚀 SpringBoot CI/CD Pipeline — Production-Grade Jenkins + ArgoCD + KIND

![Build Status](https://img.shields.io/badge/build-passing-brightgreen?style=for-the-badge&logo=jenkins)
![SonarQube](https://img.shields.io/badge/SonarQube-Quality%20Gate-blue?style=for-the-badge&logo=sonarqube)
![Trivy](https://img.shields.io/badge/Trivy-Security%20Scan-red?style=for-the-badge&logo=aquasecurity)
![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-orange?style=for-the-badge&logo=argo)
![Kubernetes](https://img.shields.io/badge/Kubernetes-KIND-326CE5?style=for-the-badge&logo=kubernetes)
![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?style=for-the-badge&logo=docker)
![Java](https://img.shields.io/badge/Java-17-ED8B00?style=for-the-badge&logo=openjdk)
![Maven](https://img.shields.io/badge/Maven-3.9-C71A36?style=for-the-badge&logo=apachemaven)

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Pipeline Stages](#-pipeline-stages)
- [Repository Structure](#-repository-structure)
- [Local Infrastructure Setup](#-local-infrastructure-setup)
- [How to Run](#-how-to-run)
- [Screenshots](#-screenshots)
- [GitOps Flow](#-gitops-flow)
- [Key Concepts Demonstrated](#-key-concepts-demonstrated)
- [Author](#-author)

---

## 📌 Project Overview

This project demonstrates a **fully automated, production-grade CI/CD pipeline** built from scratch on a local setup using:

- **Jenkins** as the CI orchestrator with a declarative pipeline
- **SonarQube** for static code analysis and quality gate enforcement
- **Trivy** for container image vulnerability scanning
- **Docker** with a multi-stage build for optimized, secure images
- **ArgoCD** for GitOps-based continuous delivery
- **Helm** for templated Kubernetes deployments
- **KIND** (Kubernetes IN Docker) as the local Kubernetes cluster

The application is a **Java Spring Boot REST API** with Prometheus metrics exposed via Spring Actuator — ready for Grafana monitoring integration.

> This entire setup runs locally on a Windows machine with Docker Desktop and KIND — no cloud account required.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Developer Machine                           │
│                                                                     │
│  ┌──────────┐   push    ┌─────────────────────────────────────┐    │
│  │  VS Code │ ────────► │         GitHub                      │    │
│  └──────────┘           │  ┌─────────────────────────────┐   │    │
│                          │  │  springboot-cicd-app  (CI)  │   │    │
│                          │  │  springboot-cicd-gitops(CD) │   │    │
│                          │  └─────────────────────────────┘   │    │
│                          └──────────────┬──────────────────────┘    │
│                                         │ webhook / poll             │
│                          ┌──────────────▼──────────────────────┐    │
│                          │           Jenkins                    │    │
│                          │  ┌────────────────────────────────┐ │    │
│                          │  │  Stage 1: Checkout             │ │    │
│                          │  │  Stage 2: Build + Unit Tests   │ │    │
│                          │  │  Stage 3: SonarQube Analysis   │ │    │
│                          │  │  Stage 4: Quality Gate         │ │    │
│                          │  │  Stage 5: Docker Build         │ │    │
│                          │  │  Stage 6: Trivy Security Scan  │ │    │
│                          │  │  Stage 7: Push to Registry     │ │    │
│                          │  │  Stage 8: Update GitOps Repo   │ │    │
│                          │  └────────────────────────────────┘ │    │
│                          └──────────────┬───────────────────────┘    │
│                                         │                             │
│            ┌────────────────────────────▼──────────────────────┐     │
│            │  Docker Network (cicd-network)                     │     │
│            │  ┌──────────────┐   ┌────────────────────────┐   │     │
│            │  │  SonarQube   │   │   Local Registry :5000  │   │     │
│            │  │  :9000       │   │   (simulates ECR)       │   │     │
│            │  └──────────────┘   └───────────┬────────────┘   │     │
│            └──────────────────────────────────│─────────────────┘     │
│                                               │ image pull             │
│            ┌──────────────────────────────────▼──────────────────┐    │
│            │  KIND Cluster (prod-cluster)                         │    │
│            │                                                      │    │
│            │  ┌──────────────────────────────────────────────┐   │    │
│            │  │  ArgoCD (namespace: argocd)                  │   │    │
│            │  │  Watches: springboot-cicd-gitops → main      │   │    │
│            │  └──────────────────┬───────────────────────────┘   │    │
│            │                     │ sync                           │    │
│            │  ┌──────────────────▼───────────────────────────┐   │    │
│            │  │  production namespace                         │   │    │
│            │  │  ┌────────────────────────────────────────┐  │   │    │
│            │  │  │  Deployment (2 replicas)               │  │   │    │
│            │  │  │  Service (NodePort: 30081)             │  │   │    │
│            │  │  │  HPA (min:2 max:5 cpu:70%)             │  │   │    │
│            │  │  │  ConfigMap                             │  │   │    │
│            │  │  │  ServiceAccount                        │  │   │    │
│            │  │  └────────────────────────────────────────┘  │   │    │
│            │  └──────────────────────────────────────────────┘   │    │
│            └──────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Tech Stack

| Category | Tool | Version | Purpose |
|---|---|---|---|
| **Application** | Java Spring Boot | 3.2.0 | REST API with Actuator + Prometheus |
| **Build** | Apache Maven | 3.9 | Dependency management + packaging |
| **CI Orchestration** | Jenkins | LTS | Declarative pipeline automation |
| **Code Quality** | SonarQube | LTS Community | Static analysis + quality gate |
| **Security Scan** | Trivy | Latest | CVE scanning on Docker images |
| **Containerization** | Docker | 24+ | Multi-stage image build |
| **Image Registry** | Docker Registry | v2 | Local private registry (simulates ECR) |
| **CD / GitOps** | ArgoCD | Stable | Automated sync from Git to cluster |
| **Packaging** | Helm | v3 | Kubernetes manifest templating |
| **Kubernetes** | KIND | 0.20+ | Local 4-node Kubernetes cluster |
| **Coverage** | JaCoCo | 0.8.11 | Code coverage reporting |
| **Metrics** | Micrometer + Prometheus | Latest | App metrics via /actuator/prometheus |

---

## 🔄 Pipeline Stages

```
Checkout → Build & Test → SonarQube → Quality Gate → Docker Build → Trivy Scan → Push → GitOps Update
```

### Stage Breakdown

**Stage 1 — Checkout**
Clones the application repository and prints the last 5 git commits for traceability.

**Stage 2 — Build & Unit Tests**
Runs `mvn clean test`. JUnit XML reports are published in Jenkins UI. JaCoCo generates a coverage report consumed by SonarQube.

**Stage 3 — SonarQube Analysis**
Runs full static analysis — code smells, bugs, vulnerabilities, duplications, and coverage. Results are pushed to the SonarQube server.

**Stage 4 — Quality Gate**
Waits up to 5 minutes for SonarQube to evaluate the quality gate. If the gate fails, the pipeline aborts — no image is built.

**Stage 5 — Docker Build**
Builds a multi-stage Docker image. Stage 1 uses Maven+JDK to compile. Stage 2 uses JRE-Alpine only — resulting in a minimal, non-root production image.

**Stage 6 — Trivy Security Scan**
Runs two scans: first a full report of all severities (informational), then a second scan that fails the pipeline if any `CRITICAL` CVEs are found.

**Stage 7 — Push to Registry**
Pushes the image with both `BUILD_NUMBER` tag and `latest` tag to the local registry.

**Stage 8 — Update GitOps Repo**
Clones `springboot-cicd-gitops`, updates the `tag` field in `helm/values.yaml` to the new `BUILD_NUMBER`, commits, and pushes. ArgoCD detects this change and auto-deploys.

---

## 📁 Repository Structure

### Repo 1 — `springboot-cicd-app` (this repo)

```
springboot-cicd-app/
├── src/
│   ├── main/
│   │   ├── java/com/demo/
│   │   │   ├── Application.java                  # Spring Boot entry point
│   │   │   └── controller/
│   │   │       └── AppController.java             # REST endpoints
│   │   └── resources/
│   │       └── application.properties             # App + actuator config
│   └── test/
│       └── java/com/demo/controller/
│           └── AppControllerTest.java             # MockMvc unit tests
├── Dockerfile                                     # Multi-stage production build
├── Jenkinsfile                                    # 8-stage declarative pipeline
├── pom.xml                                        # Maven + JaCoCo + Surefire
├── sonar-project.properties                       # SonarQube config
└── README.md
```

### Repo 2 — `springboot-cicd-gitops`

```
springboot-cicd-gitops/
├── helm/
│   ├── Chart.yaml                                 # Helm chart metadata
│   ├── values.yaml                                # ← Jenkins updates tag here
│   └── templates/
│       ├── deployment.yaml                        # 2 replicas, rolling update
│       ├── service.yaml                           # NodePort :30081
│       ├── hpa.yaml                               # CPU-based autoscaling
│       ├── configmap.yaml                         # App environment config
│       └── serviceaccount.yaml                    # Non-root service account
└── argocd/
    └── application.yaml                           # ArgoCD app manifest
```

---

## 🖥️ Local Infrastructure Setup

All components run locally via Docker Desktop and KIND.

### Prerequisites

| Tool | Version | Install |
|---|---|---|
| Docker Desktop | 24+ | https://docker.com |
| KIND | 0.20+ | `choco install kind` |
| kubectl | 1.27+ | `choco install kubernetes-cli` |
| Git | Latest | https://git-scm.com |

### Running Containers

| Container | Port | Purpose |
|---|---|---|
| `jenkins` | 8080 | CI pipeline orchestration |
| `sonarqube` | 9000 | Static code analysis |
| `host.docker.internal` | 5000 | Private Docker image registry |

### KIND Cluster

| Node | Role |
|---|---|
| `prod-cluster-control-plane` | Control plane |
| `prod-cluster-worker` | Worker node 1 |
| `prod-cluster-worker2` | Worker node 2 |
| `prod-cluster-worker3` | Worker node 3 |

---

## ▶️ How to Run

### 1. Start Docker containers

```bash
docker network create cicd-network

docker run -d --name host.docker.internal --network cicd-network \
  --restart always -p 5000:5000 registry:2

docker run -d --name sonarqube --network cicd-network \
  --restart always -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:lts-community

docker run -d --name jenkins --network cicd-network \
  --restart always -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts
```

### 2. Create KIND cluster

```bash
kind create cluster --name prod-cluster --config kind-config.yaml
docker network connect kind host.docker.internal
```

### 3. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl create namespace production
kubectl apply -f argocd/application.yaml
```

### 4. Trigger the pipeline

```bash
# Push any change to this repo
git commit -m "feat: trigger pipeline" --allow-empty
git push origin main
```

### 5. Access the services

| Service | URL | Credentials |
|---|---|---|
| Jenkins | http://localhost:8080 | admin / (setup password) |
| SonarQube | http://localhost:9000 | admin / Admin@123 |
| ArgoCD | https://localhost:8443 | admin / (kubectl secret) |
| Application | http://localhost:30081 | — |

### 6. Test the application

```bash
curl http://localhost:30081/api/health
curl http://localhost:30081/api/info
curl http://localhost:30081/api/ping
curl http://localhost:30081/actuator/prometheus
```

---

## 📸 Screenshots

> Screenshots are available in the `/screenshots` folder.


---

## 🔁 GitOps Flow

```
Developer pushes code
        │
        ▼
Jenkins pipeline runs (8 stages)
        │
        ▼
Docker image built → tagged with BUILD_NUMBER → pushed to host.docker.internal:5000
        │
        ▼
Jenkins updates helm/values.yaml → tag: "BUILD_NUMBER" → pushed to springboot-cicd-gitops
        │
        ▼
ArgoCD detects change in GitOps repo (polls every 3 min)
        │
        ▼
ArgoCD syncs → Helm upgrade → Rolling update in production namespace
        │
        ▼
New pods running updated image → Zero downtime deployment ✅
```

**Why two repos?**

Separating application code from deployment config is a GitOps best practice. It means:
- Deployment history is tracked in Git independently of code history
- You can roll back a deployment without touching application code
- ArgoCD is the only thing that touches the cluster — no `kubectl` in the pipeline

---

## 💡 Key Concepts Demonstrated

**Jenkins Declarative Pipeline**
- `options` block — build discarder, timeout, concurrent build prevention, timestamps
- `tools` block — JDK and Maven auto-configured
- `post` block — always/success/failure handlers with workspace cleanup
- `withSonarQubeEnv` — SonarQube environment injection
- `waitForQualityGate` — blocking quality gate check
- `withCredentials` — secure credential handling (no secrets in code)

**Docker Best Practices**
- Multi-stage build — builder image vs runtime image
- Non-root user — security compliance
- Layer caching — `dependency:go-offline` before source copy
- JVM container flags — `UseContainerSupport`, `MaxRAMPercentage`

**Kubernetes / Helm**
- Templated manifests with `values.yaml` overrides
- Rolling update strategy with `maxUnavailable: 0`
- Liveness and readiness probes
- Resource requests and limits
- HorizontalPodAutoscaler (CPU-based)
- ConfigMap for environment injection
- Dedicated ServiceAccount (least privilege)

**GitOps with ArgoCD**
- Automated sync with `selfHeal: true` — reverts manual cluster changes
- `prune: true` — removes resources deleted from Git
- `CreateNamespace=true` — namespace managed by ArgoCD
- Finalizer for clean deletion

**Security**
- Trivy CVE scanning with two-tier severity policy
- SonarQube quality gate blocking broken code
- Non-root container user
- Credentials stored in Jenkins credential store (never in code)

---

## 👤 Author

**Kiran Kumatkar**
| DevOps Engineer

