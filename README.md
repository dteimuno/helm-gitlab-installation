# Self-Managed GitLab 19.1.1 on Kubernetes (kubeadm) — AWS Home Lab

A complete guide to deploying a self-managed GitLab Enterprise Edition on a two-node kubeadm Kubernetes cluster on AWS, with external RDS PostgreSQL 17, Redis via Helm, S3 object storage, cert-manager TLS, and F5 NGINX Ingress Controller.

> **Author:** Dennis Teimuno  
> **Domain:** [gitlab.dtmprojects.net](https://gitlab.dtmprojects.net)  
> **GitLab Version:** 19.1.1  
> **Kubernetes Version:** v1.36.0  
> **Date:** July 2026

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [AWS Infrastructure](#aws-infrastructure)
  - [EC2 Instances](#ec2-instances)
  - [Security Groups](#security-groups)
  - [RDS PostgreSQL 17](#rds-postgresql-17)
  - [S3 Buckets](#s3-buckets)
  - [Route 53 DNS](#route-53-dns)
- [Kubernetes Cluster Setup (kubeadm)](#kubernetes-cluster-setup-kubeadm)
  - [System Prerequisites (Both Nodes)](#system-prerequisites-both-nodes)
  - [Install containerd](#install-containerd)
  - [Install kubeadm kubelet kubectl](#install-kubeadm-kubelet-kubectl)
  - [Initialize Control Plane](#initialize-control-plane)
  - [Install Flannel CNI](#install-flannel-cni)
  - [Join Worker Node](#join-worker-node)
  - [Verify Cluster](#verify-cluster)
- [Install Cluster Essentials](#install-cluster-essentials)
  - [Metrics Server](#metrics-server)
  - [Local Path Provisioner](#local-path-provisioner)
- [Install kube-prometheus-stack](#install-kube-prometheus-stack)
- [Install Redis via Helm](#install-redis-via-helm)
  - [Redis values.yaml](#redis-valuesyaml)
  - [Redis Validations](#redis-validations)
- [Install cert-manager](#install-cert-manager)
- [Install F5 NGINX Ingress Controller](#install-f5-nginx-ingress-controller)
- [Install GitLab via Helm](#install-gitlab-via-helm)
  - [Prerequisites](#gitlab-prerequisites)
  - [Create Kubernetes Secrets](#create-kubernetes-secrets)
  - [Create S3 Connection Secret](#create-s3-connection-secret)
  - [Create Registry S3 Secret](#create-registry-s3-secret)
  - [Install PostgreSQL Extensions](#install-postgresql-extensions)
  - [GitLab values.yaml](#gitlab-valuesyaml)
  - [Deploy GitLab](#deploy-gitlab)
- [Post-Installation](#post-installation)
  - [Configure Worker Node Port Forwarding](#configure-worker-node-port-forwarding)
  - [Access GitLab](#access-gitlab)
- [Cost Optimization](#cost-optimization)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    AWS us-east-1f                        │
│                                                          │
│  ┌──────────────────────┐  ┌───────────────────────┐    │
│  │   Control Plane      │  │    Worker Node         │    │
│  │   m6i.2xlarge        │  │    m6i.4xlarge         │    │
│  │   8 vCPU / 32 GB     │  │    16 vCPU / 64 GB     │    │
│  │   Elastic IP         │  │    Elastic IP           │    │
│  │                      │  │    100.51.62.238        │    │
│  │  - API Server        │  │                         │    │
│  │  - etcd              │  │  - GitLab pods          │    │
│  │  - Scheduler         │  │  - Redis pods           │    │
│  │  - Controller Mgr    │  │  - Prometheus/Grafana   │    │
│  └──────────────────────┘  └───────────────────────┘    │
│                                                          │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────┐  │
│  │  RDS          │  │  S3 Buckets   │  │  Route 53   │  │
│  │  PostgreSQL17 │  │  (7 buckets)  │  │  DNS        │  │
│  │  db.m5.large  │  │  Object Store │  │  dtmprojects│  │
│  └──────────────┘  └───────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Stack:**

| Component | Technology |
|---|---|
| Kubernetes | kubeadm v1.36 |
| Container Runtime | containerd |
| CNI | Flannel |
| GitLab | 19.1.1 (Helm chart 10.1.1) |
| Database | AWS RDS PostgreSQL 17 |
| Cache | Redis 8.8.0 (Bitnami Helm) |
| Object Storage | AWS S3 |
| Ingress | F5 NGINX Ingress Controller 2.6.1 |
| TLS | cert-manager v1.20.3 |
| Monitoring | kube-prometheus-stack |
| Storage Class | local-path-provisioner |

---

## Prerequisites

- AWS account with Community Builder credits or billing configured
- AWS CLI configured (`aws configure`)
- `kubectl`, `helm`, `ssh` on your local machine
- Domain name with Route 53 hosted zone
- SSH key pair for EC2

---

## AWS Infrastructure

### EC2 Instances

| Node | Instance | vCPU | RAM | Role |
|---|---|---|---|---|
| controlplane | m6i.2xlarge | 8 | 32 GB | Kubernetes control plane |
| kube-workernode | m6i.4xlarge | 16 | 64 GB | Kubernetes worker |

**Storage:**

| Node | Volume | Size | Type | IOPS |
|---|---|---|---|---|
| Control Plane | Root | 50 GB | gp3 | 3,000 |
| Worker | Root | 30 GB | gp3 | 3,000 |
| Worker | Data | 200 GB | gp3 | 6,000 |

Both nodes use **Elastic IPs** so they retain their public IPs across stop/start cycles.

### Security Groups

**Worker node inbound rules (minimum required):**

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | Your IP | SSH |
| 80 | TCP | 0.0.0.0/0 | HTTP (GitLab) |
| 443 | TCP | 0.0.0.0/0 | HTTPS (GitLab) |
| 6443 | TCP | 0.0.0.0/0 | Kubernetes API |
| 10250 | TCP | 172.31.0.0/16 | Kubelet |
| 8472 | UDP | 172.31.0.0/16 | Flannel VXLAN |
| 30080 | TCP | 0.0.0.0/0 | NGINX NodePort HTTP |
| 30443 | TCP | 0.0.0.0/0 | NGINX NodePort HTTPS |
| All traffic | All | 172.31.0.0/16 | VPC internal |

### RDS PostgreSQL 17

- **Instance:** db.m5.large
- **Engine:** PostgreSQL 17.x
- **Storage:** 50 GiB gp3
- **Multi-AZ:** No (dev/personal cluster)
- **Public access:** No
- **Database name:** `gitlabhq_production`
- **Username:** `gitlab`

> **Important:** GitLab 19.x requires PostgreSQL **17 minimum**. PostgreSQL 16 will cause migrations to fail.

**Create required extensions after RDS is available:**

```bash
psql -h <your-rds-endpoint>.rds.amazonaws.com \
     -U gitlab \
     -d gitlabhq_production

# Inside psql:
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE EXTENSION IF NOT EXISTS amcheck;
\q
```

### S3 Buckets

Create seven S3 buckets for GitLab object storage:

```bash
aws s3 mb s3://gitlab-artifacts-dtm --region us-east-1
aws s3 mb s3://gitlab-lfs-dtm --region us-east-1
aws s3 mb s3://gitlab-uploads-dtm --region us-east-1
aws s3 mb s3://gitlab-packages-dtm --region us-east-1
aws s3 mb s3://gitlab-registry-dtm --region us-east-1
aws s3 mb s3://gitlab-backups-dtm --region us-east-1
aws s3 mb s3://gitlab-tmp-dtm --region us-east-1
```

### Route 53 DNS

Create A records in your hosted zone pointing to your worker's Elastic IP:

| Record | Type | Value |
|---|---|---|
| gitlab.dtmprojects.net | A | 100.51.62.238 |
| gitlab-registry.dtmprojects.net | A | 100.51.62.238 |
| gitlab-kas.dtmprojects.net | A | 100.51.62.238 |
| gitlab-pages.dtmprojects.net | A | 100.51.62.238 |
| gitlab-smartcard.dtmprojects.net | A | 100.51.62.238 |

---

## Kubernetes Cluster Setup (kubeadm)

### System Prerequisites (Both Nodes)

Run on **both** control plane and worker:

```bash
# Disable swap permanently
sudo swapoff -a
sudo sed -i '/\bswap\b/d' /etc/fstab

# Verify
free -h | grep Swap
# Expected: Swap: 0B 0B 0B

# Load kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Verify
lsmod | grep br_netfilter
sysctl net.ipv4.ip_forward
```

### Install containerd

Run on **both** nodes:

```bash
# Install dependencies
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg

# Add Docker GPG key and repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt-get update -y
sudo apt-get install -y containerd.io

# Configure containerd — critical: SystemdCgroup must be true
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

# Verify
grep SystemdCgroup /etc/containerd/config.toml
# Expected: SystemdCgroup = true

sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

### Install kubeadm kubelet kubectl

Run on **both** nodes:

```bash
K8S_VERSION="1.36"

curl -fsSL "https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/Release.key" \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl

# Pin versions — never auto-upgrade
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet

# Verify
kubeadm version
```

### Initialize Control Plane

Run on **control plane only**:

```bash
PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
PUBLIC_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)

sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=$PRIVATE_IP \
  --apiserver-cert-extra-sans=$PUBLIC_IP,$PRIVATE_IP \
  --node-name=$(hostname)

# Set up kubeconfig
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Save the join command — needed for worker
sudo kubeadm token create --print-join-command
```

**Copy kubeconfig to your local machine:**

```bash
scp -i ~/.ssh/your-key.pem \
  ubuntu@<CONTROL_PLANE_IP>:~/.kube/config \
  ~/.kube/config
```

### Install Flannel CNI

Run on **control plane only**:

```bash
kubectl apply -f \
  https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Watch flannel pods come up
kubectl get pods -n kube-flannel -w
# Wait until all show Running

kubectl get nodes
# Control plane should show Ready
```

> **Common issue:** If Flannel pods crash with `br_netfilter` errors, ensure `br_netfilter` is loaded on **both** nodes and restart the Flannel daemonset.

### Join Worker Node

SSH into the worker node and run the join command from the previous step:

```bash
sudo kubeadm join <CONTROL_PLANE_PRIVATE_IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Verify Cluster

```bash
kubectl get nodes
# Both nodes should show Ready

kubectl get pods -A
# All system pods should be Running
```

---

## Install Cluster Essentials

### Metrics Server

Required for `kubectl top` and HPA:

```bash
kubectl apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for kubeadm self-signed kubelet certs
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-",
        "value":"--kubelet-insecure-tls"}]'

kubectl top nodes
```

### Local Path Provisioner

Required for dynamic PVC provisioning (no default StorageClass on kubeadm):

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# Set as default StorageClass
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":
    {"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl get storageclass
# local-path should show (default)
```

---

## Install kube-prometheus-stack

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=changeme123 \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=local-path \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=20Gi

# Access Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana \
  -n monitoring 3000:80
# Open http://localhost:3000 — admin / changeme123
```

---

## Install Redis via Helm

### Redis values.yaml

See [`redis-values.yaml`](./redis-values.yaml) in this repository.

**Install:**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

kubectl create namespace gitlab

helm install redis bitnami/redis \
  --version 27.0.13 \
  --namespace gitlab \
  -f redis-values.yaml
```

**Verify:**

```bash
kubectl get pods -n gitlab | grep redis
# redis-master-0      1/1 Running
# redis-replicas-0    1/1 Running
# redis-replicas-1    1/1 Running
# redis-replicas-2    1/1 Running
```

**Create the Redis password secret for GitLab:**

```bash
REDIS_PASS=$(kubectl get secret redis -n gitlab \
  -o jsonpath="{.data.redis-password}" | base64 -d)

kubectl create secret generic redis-password \
  --namespace gitlab \
  --from-literal=password="$REDIS_PASS"
```

### Redis Validations

**1. Ping test:**

```bash
kubectl run redis-test --rm -it \
  --image=redis:alpine \
  --restart=Never \
  -n gitlab -- \
  redis-cli -h redis-master.gitlab.svc.cluster.local \
  -a $REDIS_PASS ping
# Expected: PONG
```

**2. Read/write test:**

```bash
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASS SET testkey "hello"

kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASS GET testkey
# Expected: hello
```

**3. Replication health:**

```bash
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASS info replication \
  | grep -E "role|connected_slaves"
# Expected: role:master, connected_slaves:3
```

**4. Replica sync test:**

```bash
# Write to master
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASS SET replication-test "confirmed"

# Read from replica — should show the same value
kubectl exec -it redis-replicas-0 -n gitlab -- \
  redis-cli -a $REDIS_PASS GET replication-test
# Expected: confirmed
```

**5. Persistence test (data survives pod restart):**

```bash
kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASS SET persist-test "survived"

kubectl delete pod redis-master-0 -n gitlab
# Wait for it to restart

kubectl exec -it redis-master-0 -n gitlab -- \
  redis-cli -a $REDIS_PASS GET persist-test
# Expected: survived
```

**6. Performance benchmark:**

```bash
kubectl run redis-bench --rm -it \
  --image=redis:alpine \
  --restart=Never \
  -n gitlab -- \
  redis-benchmark \
  -h redis-master.gitlab.svc.cluster.local \
  -a $REDIS_PASS \
  -n 10000 -q
```

---

## Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true

# Verify
kubectl get pods -n cert-manager
# All 3 pods should be Running

# Create Let's Encrypt ClusterIssuer
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

kubectl get clusterissuer
# letsencrypt-prod should show Ready: True
```

> **Note:** If cert-manager CRDs are owned by another Helm release, delete them first:
> ```bash
> kubectl delete crd \
>   challenges.acme.cert-manager.io \
>   orders.acme.cert-manager.io \
>   certificates.cert-manager.io \
>   certificaterequests.cert-manager.io \
>   clusterissuers.cert-manager.io \
>   issuers.cert-manager.io
> ```

---

## Install F5 NGINX Ingress Controller

> **Note:** The community `kubernetes/ingress-nginx` was retired in March 2026. This project uses the F5/NGINX Inc controller (`nginxinc/kubernetes-ingress`) which is actively maintained.

```bash
helm install ingress-nginx \
  oci://ghcr.io/nginx/charts/nginx-ingress \
  --version 2.6.1 \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.hostPort.enable=true \
  --set controller.service.type=NodePort \
  --set controller.service.httpPort.nodePort=30080 \
  --set controller.service.httpsPort.nodePort=30443 \
  --set controller.ingressClass.name=nginx

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

**Configure port forwarding on worker node** (so ports 80/443 reach the NodePort):

```bash
# SSH into worker node
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 30080
sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 30443

# Persist across reboots
sudo apt-get install -y iptables-persistent
sudo netfilter-persistent save

# Verify
sudo iptables -t nat -L PREROUTING -n | grep -E "80|443"
```

---

## Install GitLab via Helm

### GitLab Prerequisites

Ensure the following are ready before installing:

- [ ] RDS PostgreSQL 17 is running and accessible
- [ ] Database extensions created (`pg_trgm`, `btree_gist`, `amcheck`)
- [ ] All 7 S3 buckets created
- [ ] Redis is running in the `gitlab` namespace
- [ ] cert-manager is running with a ClusterIssuer
- [ ] F5 NGINX Ingress Controller is running
- [ ] DNS A records created in Route 53

### Create Kubernetes Secrets

```bash
# PostgreSQL password
kubectl create secret generic posgres-password \
  --namespace gitlab \
  --from-literal=password='YOUR_RDS_PASSWORD'

# Redis password (already created above in Redis section)
# Verify it exists:
kubectl get secret redis-password -n gitlab
```

### Create S3 Connection Secret

```bash
cat <<EOF > /tmp/s3-connection.yaml
provider: AWS
region: us-east-1
aws_access_key_id: YOUR_AWS_ACCESS_KEY
aws_secret_access_key: YOUR_AWS_SECRET_KEY
EOF

kubectl create secret generic gitlab-s3-connection \
  --namespace gitlab \
  --from-file=connection=/tmp/s3-connection.yaml

rm /tmp/s3-connection.yaml
```

### Create Registry S3 Secret

The container registry requires a different secret format:

```bash
cat <<EOF > /tmp/registry-s3.yaml
s3:
  accesskey: YOUR_AWS_ACCESS_KEY
  secretkey: YOUR_AWS_SECRET_KEY
  region: us-east-1
  bucket: gitlab-registry-dtm
  encrypt: false
  secure: true
  v4auth: true
EOF

kubectl create secret generic gitlab-registry-s3 \
  --namespace gitlab \
  --from-file=config=/tmp/registry-s3.yaml

rm /tmp/registry-s3.yaml
```

### Install PostgreSQL Extensions

```bash
sudo apt-get install -y postgresql-client

psql -h gitlab-postgres.<your-endpoint>.rds.amazonaws.com \
     -U gitlab \
     -d gitlabhq_production

# Inside psql:
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE EXTENSION IF NOT EXISTS amcheck;
\dx
\q
```

### GitLab values.yaml

See [`gitlab-values.yaml`](./gitlab-values.yaml) in this repository.

### Deploy GitLab

```bash
helm repo add gitlab https://charts.gitlab.io
helm repo update

helm install gitlab gitlab/gitlab \
  --version 10.1.1 \
  --namespace gitlab \
  --create-namespace \
  --timeout 600s \
  -f gitlab-values.yaml
```

**Monitor the installation:**

```bash
# Watch all pods
kubectl get pods -n gitlab -w

# Watch migrations specifically
MPOD=$(kubectl get pods -n gitlab -l app=migrations \
  --no-headers | head -1 | awk '{print $1}')
kubectl logs -n gitlab $MPOD -f
```

**Expected final pod state:**

```
gitlab-gitaly-0                    1/1 Running    ✅
gitlab-gitlab-exporter-*           1/1 Running    ✅
gitlab-gitlab-shell-*              1/1 Running    ✅
gitlab-kas-*                       1/1 Running    ✅
gitlab-migrations-*                0/1 Completed  ✅
gitlab-registry-*                  1/1 Running    ✅
gitlab-sidekiq-*                   1/1 Running    ✅
gitlab-toolbox-*                   1/1 Running    ✅
gitlab-webservice-default-*        2/2 Running    ✅
```

---

## Post-Installation

### Access GitLab

```bash
# Get the initial root password
kubectl get secret gitlab-initial-root-password \
  -n gitlab \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

Open your browser: `https://gitlab.dtmprojects.net`

- **Username:** `root`
- **Password:** output from above command

> The site uses GitLab's self-signed TLS certificate. Click **Advanced → Proceed** to bypass the browser warning. For proper Let's Encrypt certificates, configure the `letsencrypt-prod` ClusterIssuer annotation on the ingress.

### Verify Certificates

```bash
kubectl get certificates -n gitlab
# All should show READY: True

kubectl get clusterissuer
# letsencrypt-prod should show READY: True
```

---

## Cost Optimization

**Stop instances when not in use — biggest cost lever:**

```bash
# Stop (pay only for EBS storage ~$37/month)
aws ec2 stop-instances \
  --instance-ids <control-plane-id> <worker-id>

# Stop RDS too
aws rds stop-db-instance \
  --db-instance-identifier gitlab-postgres

# Start (always start RDS first)
aws rds start-db-instance \
  --db-instance-identifier gitlab-postgres

# Wait for RDS to be available before starting EC2
aws rds wait db-instance-available \
  --db-instance-identifier gitlab-postgres

aws ec2 start-instances \
  --instance-ids <control-plane-id> <worker-id>
```

**Estimated monthly costs:**

| Scenario | Cost |
|---|---|
| Stopped (storage only) | ~$37/month |
| Running 4 hrs/day | ~$150/month |
| Running 8 hrs/day | ~$296/month |
| Always on | ~$846/month |

---

## Troubleshooting

### Nodes NotReady — Flannel crashloop

```bash
kubectl logs -n kube-flannel -l app=flannel --all-containers=true
```

**Fix:** Load `br_netfilter` on all nodes:

```bash
sudo modprobe br_netfilter
sudo modprobe overlay
sudo sysctl --system
kubectl rollout restart daemonset/kube-flannel-ds -n kube-flannel
```

### GitLab Migrations Failing

Check logs:

```bash
MPOD=$(kubectl get pods -n gitlab -l app=migrations \
  --no-headers | head -1 | awk '{print $1}')
kubectl logs -n gitlab $MPOD --tail=50
```

Common causes:
- **PostgreSQL version < 17** → Upgrade RDS to PostgreSQL 17
- **CI database conflict** → Set `databaseTasks: false` on the `ci` psql config
- **Wrong secret name** → Verify secret name matches values.yaml exactly

### Registry CrashLoopBackOff

Check logs:

```bash
kubectl logs -n gitlab -l app=registry --tail=30
```

**Fix:** Ensure the registry uses S3 format (not Rails fog format):

```yaml
# registry-s3 secret must use this format:
s3:
  accesskey: KEY
  secretkey: SECRET
  region: us-east-1
  bucket: gitlab-registry-dtm
```

### Site Not Reachable

```bash
# 1. Verify DNS
dig gitlab.dtmprojects.net +short
# Should return worker Elastic IP

# 2. Test ingress directly
curl -v http://<WORKER_IP>:30080

# 3. Check ingress-nginx
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

# 4. Verify port forwarding on worker
sudo iptables -t nat -L PREROUTING -n | grep -E "80|443"

# 5. Check security group has ports 80, 443, 30080, 30443 open
```

### Control Plane IP Changed After Restart

Update kubeconfig after restarting EC2:

```bash
NEW_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=kube-controlplane" \
            "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)

sed -i "s|server: https://.*:6443|server: https://$NEW_IP:6443|" \
  ~/.kube/config

kubectl get nodes
```

---

## Repository Structure

```
.
├── README.md               # This file
├── redis-values.yaml       # Bitnami Redis Helm values
└── gitlab-values.yaml      # GitLab Helm values
```

---

## References

- [GitLab Helm Chart Documentation](https://docs.gitlab.com/charts/)
- [kubeadm Installation Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Flannel CNI](https://github.com/flannel-io/flannel)
- [F5 NGINX Ingress Controller](https://github.com/nginx/nginx-ingress-helm-chart)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Bitnami Redis Helm Chart](https://github.com/bitnami/charts/tree/main/bitnami/redis)
- [Ingress NGINX Retirement (March 2026)](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)

---

*Built with ❤️ as part of Dennis Teimuno's DevOps home lab — [@dtmprojects](https://gitlab.dtmprojects.net)*
