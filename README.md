# Voting Application CI/CD Pipeline on Kubernetes

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28-blue.svg)](https://kubernetes.io)
[![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-red.svg)](https://jenkins.io)
[![Docker](https://img.shields.io/badge/Docker-24.0-blue.svg)](https://docker.com)
[![GitHub](https://img.shields.io/badge/GitHub-Webhooks-black.svg)](https://github.com)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

A complete DevOps implementation of a microservices-based voting application with a fully automated CI/CD pipeline. Users vote for **Cats** or **Dogs** and see live results — showcasing Kubernetes orchestration, Jenkins pipelines, Docker Hub integration, and Infrastructure as Code with Vagrant.

---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Infrastructure Setup](#infrastructure-setup)
- [Kubernetes Cluster Setup](#kubernetes-cluster-setup)
- [Jenkins Pipeline](#jenkins-pipeline)
- [Application Components](#application-components)
- [CI/CD Workflow](#cicd-workflow)
- [Accessing the Application](#accessing-the-application)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

---

## Architecture

```
GitHub ──► Jenkins ──► Docker Hub ──► Kubernetes Cluster ──► Users
  ▲            │             │                │
  └────────────┴─────────────┴────────────────┘
           Webhook Triggers Pipeline
```

### Component Data Flow

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Vote    │────►│  Redis   │     │  Worker  │────►│ Postgres │
│  (Flask) │     │  (Cache) │     │  (.NET)  │     │   (DB)   │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                      ▲                                   ▲
                      └───────── Votes Flow ──────────────┘

┌──────────┐     ┌──────────┐
│  Result  │────►│ Postgres │
│ (Node.js)│     │   (DB)   │
└──────────┘     └──────────┘
```

---

## Tech Stack

| Component | Technology | Version |
|-----------|------------|---------|
| Container Orchestration | Kubernetes | 1.28 |
| CI/CD Server | Jenkins | Latest |
| Container Runtime | Docker | 24.0+ |
| Container Registry | Docker Hub | — |
| Infrastructure | Vagrant | 2.3+ |
| Hypervisor | VirtualBox | 7.x |
| Vote Service | Python / Flask | 3.11 |
| Result Service | Node.js / Express | 18 |
| Worker Service | .NET | 7.0 |
| Database | PostgreSQL | 15 |
| Cache | Redis | Alpine |
| Networking | Flannel CNI | Latest |
| OS | Ubuntu | 22.04 LTS |

---

## Prerequisites

**Hardware**

- CPU: 4+ cores
- RAM: 16 GB minimum
- Storage: 40 GB free space

**Software — install on your host machine**

- VirtualBox 7.x
- Vagrant 2.3+
- Git
- `kubectl` (optional, for host-level cluster access)

---

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/Kiran-Kumar-K17/voting-app-k8s-jenkins.git
cd voting-app-k8s-jenkins
```

### 2. Start the VMs

```bash
# Start all 4 VMs (takes 10–15 minutes on first run)
vagrant up

# Or start individual VMs
vagrant up k8s-master
vagrant up k8s-worker-1
vagrant up k8s-worker-2
vagrant up jenkins
```

### 3. Verify the Cluster

```bash
vagrant ssh k8s-master
kubectl get nodes
kubectl get pods -A
```

### 4. Deploy the Application

```bash
kubectl create namespace vote
kubectl apply -f example-voting-app/k8s-specifications/ -n vote
kubectl get all -n vote
```

### 5. Access the Application

```bash
# Check NodePorts
kubectl get svc -n vote

# Open in your browser
Voting App:  http://192.168.56.11:30001
Results App: http://192.168.56.11:30002
```

---

## Infrastructure Setup

### VM Configuration

| VM | IP Address | Role | Specs |
|----|------------|------|-------|
| `k8s-master` | 192.168.56.10 | Kubernetes Master | 4 GB RAM, 2 vCPU |
| `k8s-worker-1` | 192.168.56.11 | Kubernetes Worker | 2 GB RAM, 1 vCPU |
| `k8s-worker-2` | 192.168.56.12 | Kubernetes Worker | 2 GB RAM, 1 vCPU |
| `jenkins` | 192.168.56.13 | CI/CD Server | 4 GB RAM, 3 vCPU |

All VMs are on the `192.168.56.0/24` host-only network. See the `Vagrantfile` in the repository root for the full configuration.

---

## Kubernetes Cluster Setup

### Master Node

```bash
# Initialize the cluster
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.56.10

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel CNI
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Worker Nodes

Run the join command printed by `kubeadm init` on each worker:

```bash
sudo kubeadm join 192.168.56.10:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

## Jenkins Pipeline

The pipeline uses parallel stages and smart change detection to avoid rebuilding services that haven't changed.

```groovy
pipeline {
  agent any
  environment {
    DOCKER_HUB_USER = 'kirankumark17'
    NAMESPACE       = 'vote'
    K8S_SERVER      = 'https://192.168.56.10:6443'
  }
  stages {
    stage('Checkout') {
      steps {
        git 'https://github.com/Kiran-Kumar-K17/voting-app-k8s-jenkins.git'
      }
    }

    stage('Login to Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh 'docker login -u $DOCKER_USER -p $DOCKER_PASS'
        }
      }
    }

    stage('Detect Changes') {
      steps {
        script {
          def changed = sh(script: 'git diff --name-only HEAD~1', returnStdout: true)
          env.VOTE_CHANGED   = changed.contains('vote')   ? 'true' : 'false'
          env.RESULT_CHANGED = changed.contains('result') ? 'true' : 'false'
          env.WORKER_CHANGED = changed.contains('worker') ? 'true' : 'false'
        }
      }
    }

    stage('Build & Push Images') {
      parallel {
        stage('Vote') {
          when { expression { env.VOTE_CHANGED == 'true' } }
          steps {
            dir('example-voting-app/vote') {
              sh 'docker build -t ${DOCKER_HUB_USER}/voting:${BUILD_NUMBER} . && docker push ${DOCKER_HUB_USER}/voting:${BUILD_NUMBER}'
            }
          }
        }
        stage('Result') {
          when { expression { env.RESULT_CHANGED == 'true' } }
          steps {
            dir('example-voting-app/result') {
              sh 'docker build -t ${DOCKER_HUB_USER}/result:${BUILD_NUMBER} . && docker push ${DOCKER_HUB_USER}/result:${BUILD_NUMBER}'
            }
          }
        }
        stage('Worker') {
          when { expression { env.WORKER_CHANGED == 'true' } }
          steps {
            dir('example-voting-app/worker') {
              sh 'docker build -t ${DOCKER_HUB_USER}/worker:${BUILD_NUMBER} . && docker push ${DOCKER_HUB_USER}/worker:${BUILD_NUMBER}'
            }
          }
        }
      }
    }

    stage('Update Manifests') {
      steps {
        sh 'sed -i "s|image:.*voting.*|image: ${DOCKER_HUB_USER}/voting:${BUILD_NUMBER}|g" k8s-specifications/*.yaml'
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        withKubeConfig([serverUrl: env.K8S_SERVER]) {
          sh 'kubectl apply -f k8s-specifications/ -n ${NAMESPACE}'
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        withKubeConfig([serverUrl: env.K8S_SERVER]) {
          sh 'kubectl rollout status deployment/voting -n ${NAMESPACE}'
        }
      }
    }
  }
}
```

---

## Application Components

| Service | Language | Internal Port | NodePort | Dependencies |
|---------|----------|--------------|----------|--------------|
| Vote | Python / Flask | 80 | 30001 | Redis |
| Result | Node.js / Express | 80 | 30002 | PostgreSQL |
| Worker | .NET | — (no external) | — | Redis, PostgreSQL |
| PostgreSQL | — | 5432 (ClusterIP) | — | — |
| Redis | — | 6379 (ClusterIP) | — | — |

Default PostgreSQL credentials: `postgres / postgres`, database: `votes`.

---

## CI/CD Workflow

```
Git Push
   │
   ▼
GitHub Webhook
   │
   ▼
Jenkins Pipeline
   │
   ├── Detect Changes
   │       ├── vote/    changed? ──► Build & Push vote image
   │       ├── result/  changed? ──► Build & Push result image
   │       └── worker/  changed? ──► Build & Push worker image
   │
   ├── Update K8s manifests with new image tags
   │
   ├── kubectl apply → Kubernetes cluster
   │
   └── Verify rollout status
```

---

## Accessing the Application

**From your host machine:**

```
Voting:  http://192.168.56.11:30001  or  http://192.168.56.12:30001
Results: http://192.168.56.11:30002  or  http://192.168.56.12:30002
```

**Via port-forward (if NodePort is not reachable):**

```bash
kubectl port-forward -n vote svc/voting 8081:80
# Then open: http://localhost:8081
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `kubectl get nodes` shows `NotReady` | Flannel CNI not running | `kubectl get pods -n kube-flannel` — reinstall if missing |
| `CrashLoopBackOff` on pods | Bad config or missing env var | `kubectl logs -n vote <pod>` |
| Worker logs show "Waiting for db" | Missing `POSTGRES_DB` env var | Add `POSTGRES_DB=votes` to the db deployment |
| NodePort unreachable from host | iptables blocking traffic | See network fix below |
| `ImagePullBackOff` | Wrong image tag in manifest | Verify tag in deployment YAML matches Docker Hub |
| Jenkins build fails | Bad credentials or disk full | Check Docker Hub credentials and available disk space |

### Network Fix for NodePort Access

Run on each worker node:

```bash
sudo iptables -I INPUT   -s 192.168.56.0/24 -j ACCEPT
sudo iptables -I FORWARD -s 192.168.56.0/24 -j ACCEPT
sudo netfilter-persistent save
```

### Useful Diagnostic Commands

```bash
# Cluster overview
kubectl get nodes -o wide
kubectl get pods -A -o wide

# Logs
kubectl logs -n vote deployment/voting  --tail=50
kubectl logs -n vote deployment/worker  --tail=50 -f

# Describe a failing pod
kubectl describe pod -n vote -l app=voting

# Scale manually
kubectl scale deployment -n vote voting --replicas=3
```

---

## Project Status

**Completed**
- Kubernetes cluster (1 master + 2 workers)
- Jenkins CI/CD pipeline with GitHub webhooks
- Docker Hub image registry integration
- Smart per-service change detection
- Multi-language microservices (Python, Node.js, .NET)
- PostgreSQL + Redis data layer
- NodePort external access
- Automated rolling deployments

**In Progress**
- Ingress controller (NGINX)
- Prometheus metrics collection
- Grafana dashboards
- Horizontal Pod Autoscaling (HPA)

---

## License

MIT — see [LICENSE](LICENSE) for details.

---

## Acknowledgments

- [Docker Samples](https://github.com/dockersamples) for the original voting app
- Kubernetes, Jenkins, and Flannel open-source communities
- All contributors and testers
