# Deployment Guide

This guide provides step-by-step instructions for deploying the self-hosted AI platform. Choose the deployment path that best fits your requirements and infrastructure.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start (Docker Compose)](#quick-start-docker-compose)
3. [Production Deployment (Kubernetes)](#production-deployment-kubernetes)
4. [Component-by-Component Setup](#component-by-component-setup)
5. [Post-Deployment Configuration](#post-deployment-configuration)
6. [Troubleshooting](#troubleshooting)

## Prerequisites

### Hardware Requirements

#### Minimum (Development/Testing):
- CPU: 8 cores
- RAM: 32 GB
- Storage: 500 GB SSD
- GPU: 1x NVIDIA GPU with 16GB VRAM (for inference)

#### Recommended (Production):
- CPU: 16+ cores per node
- RAM: 64+ GB per node
- Storage: 1+ TB NVMe SSD
- GPU: 2-4x NVIDIA A100/H100 GPUs per inference node
- Network: 10 Gbps+ interconnect

### Software Requirements

- Operating System: Ubuntu 22.04 LTS (recommended) or compatible Linux distribution
- Docker: 24.0+
- Docker Compose: 2.20+
- Kubernetes: 1.28+ (for production deployment)
- kubectl: Latest version
- Helm: 3.12+
- NVIDIA Container Toolkit (for GPU support)
- Git

### Cloud Provider Options

The platform can be deployed on:
- **Self-hosted**: Bare metal or private cloud
- **AWS**: EC2 with EKS
- **GCP**: Compute Engine with GKE
- **Azure**: VMs with AKS
- **Hybrid**: Mix of on-premise and cloud resources

## Quick Start (Docker Compose)

Perfect for development, testing, and small-scale deployments.

### Step 1: Clone and Configure

```bash
# Clone the repository (or create deployment directory)
mkdir ai-platform && cd ai-platform

# Create environment file
cat > .env << 'EOF'
# WARNING: All placeholder values below MUST be replaced with strong, unique secrets before deployment!
# Never use these example values in production.

# Database
POSTGRES_DB=aiplatform
POSTGRES_USER=admin
POSTGRES_PASSWORD=REPLACE_WITH_SECURE_PASSWORD

# Redis
REDIS_PASSWORD=REPLACE_WITH_SECURE_PASSWORD

# MinIO
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=REPLACE_WITH_SECURE_PASSWORD

# Keycloak
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=REPLACE_WITH_SECURE_PASSWORD

# Application
JWT_SECRET=REPLACE_WITH_SECURE_SECRET_KEY
API_KEY_SALT=REPLACE_WITH_SECURE_SALT

# Billing (optional)
STRIPE_API_KEY=REPLACE_WITH_YOUR_STRIPE_KEY
EOF

# Secure the environment file
chmod 600 .env
```

### Step 2: Create Docker Compose File

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    container_name: ai-platform-db
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: ai-platform-redis
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # MinIO Object Storage
  minio:
    image: minio/minio:latest
    container_name: ai-platform-minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # Keycloak Authentication
  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    container_name: ai-platform-keycloak
    command: start-dev
    environment:
      KEYCLOAK_ADMIN: ${KEYCLOAK_ADMIN}
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB}
      KC_DB_USERNAME: ${POSTGRES_USER}
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  # vLLM Inference Service
  vllm:
    image: vllm/vllm-openai:latest
    container_name: ai-platform-vllm
    command: --model meta-llama/Llama-2-7b-hf --dtype auto
    environment:
      HUGGING_FACE_HUB_TOKEN: ${HF_TOKEN}
    volumes:
      - ./models:/root/.cache/huggingface
    ports:
      - "8000:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped

  # Prometheus Monitoring
  prometheus:
    image: prom/prometheus:latest
    container_name: ai-platform-prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    restart: unless-stopped

  # Grafana Dashboards
  grafana:
    image: grafana/grafana:latest
    container_name: ai-platform-grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_INSTALL_PLUGINS: grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    restart: unless-stopped

  # Nginx Load Balancer
  nginx:
    image: nginx:alpine
    container_name: ai-platform-nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - vllm
      - keycloak
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  minio_data:
  prometheus_data:
  grafana_data:

networks:
  default:
    name: ai-platform-network
EOF
```

### Step 3: Create Supporting Configuration Files

```bash
# Database initialization
cat > init-db.sql << 'EOF'
-- Users table
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    subscription_tier VARCHAR(50) DEFAULT 'free',
    credits_balance INTEGER DEFAULT 100000
);

-- API Keys table
CREATE TABLE IF NOT EXISTS api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    key_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    last_used_at TIMESTAMP,
    is_active BOOLEAN DEFAULT true
);

-- Inference requests tracking
CREATE TABLE IF NOT EXISTS inference_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    api_key_id UUID REFERENCES api_keys(id),
    model_name VARCHAR(100),
    input_tokens INTEGER,
    output_tokens INTEGER,
    total_tokens INTEGER,
    latency_ms INTEGER,
    status VARCHAR(50),
    error_message TEXT,
    timestamp TIMESTAMP DEFAULT NOW()
);

-- Billing records
CREATE TABLE IF NOT EXISTS billing_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    amount_cents INTEGER NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    description TEXT,
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW(),
    paid_at TIMESTAMP
);

-- Indexes for performance
CREATE INDEX IF NOT EXISTS idx_api_keys_user ON api_keys(user_id);
CREATE INDEX IF NOT EXISTS idx_inference_user ON inference_requests(user_id);
CREATE INDEX IF NOT EXISTS idx_inference_timestamp ON inference_requests(timestamp);
CREATE INDEX IF NOT EXISTS idx_billing_user ON billing_records(user_id);
EOF

# Prometheus configuration
cat > prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'vllm'
    static_configs:
      - targets: ['vllm:8000']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres:5432']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']
EOF

# Nginx configuration
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream inference_backend {
        least_conn;
        server vllm:8000;
    }

    upstream auth_backend {
        server keycloak:8080;
    }

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    server {
        listen 80;
        server_name _;

        # Redirect to HTTPS (uncomment when SSL is configured)
        # return 301 https://$server_name$request_uri;

        # API endpoints
        location /v1/ {
            limit_req zone=api_limit burst=20 nodelay;
            proxy_pass http://inference_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_read_timeout 300s;
        }

        # Authentication
        location /auth/ {
            proxy_pass http://auth_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Health check
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
EOF
```

### Step 4: Launch the Platform

```bash
# Start all services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f

# Wait for all services to be healthy
docker-compose ps
```

### Step 5: Verify Deployment

```bash
# Check PostgreSQL
docker-compose exec postgres psql -U admin -d aiplatform -c "\dt"

# Check Redis
docker-compose exec redis redis-cli -a $REDIS_PASSWORD ping

# Check MinIO
curl http://localhost:9000/minio/health/live

# Check vLLM (once model is loaded)
curl http://localhost:8000/health

# Check Prometheus
curl http://localhost:9090/-/healthy

# Access web interfaces:
# - Grafana: http://localhost:3000 (admin/admin)
# - Keycloak: http://localhost:8080 (admin/<password>)
# - MinIO Console: http://localhost:9001 (<user>/<password>)
# - Prometheus: http://localhost:9090
```

## Production Deployment (Kubernetes)

For scalable, production-ready deployments.

### Step 1: Prepare Kubernetes Cluster

```bash
# For cloud providers, create cluster:
# AWS EKS
eksctl create cluster \
  --name ai-platform \
  --region us-west-2 \
  --nodegroup-name gpu-nodes \
  --node-type p3.2xlarge \
  --nodes 3

# GCP GKE
gcloud container clusters create ai-platform \
  --zone us-central1-a \
  --machine-type n1-standard-8 \
  --num-nodes 3 \
  --accelerator type=nvidia-tesla-v100,count=1

# Azure AKS
az aks create \
  --resource-group ai-platform-rg \
  --name ai-platform \
  --node-count 3 \
  --node-vm-size Standard_NC6s_v3 \
  --generate-ssh-keys

# Install NVIDIA GPU Operator
kubectl create ns gpu-operator
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm install gpu-operator nvidia/gpu-operator -n gpu-operator
```

### Step 2: Install Dependencies with Helm

```bash
# Add Helm repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add kong https://charts.konghq.com
helm repo update

# Create namespace
kubectl create namespace ai-platform

# Install PostgreSQL
helm install postgres bitnami/postgresql \
  --namespace ai-platform \
  --set auth.username=admin \
  --set auth.password=secure_password \
  --set auth.database=aiplatform \
  --set primary.persistence.size=100Gi

# Install Redis
helm install redis bitnami/redis \
  --namespace ai-platform \
  --set auth.password=secure_password \
  --set master.persistence.size=10Gi

# Install MinIO
helm install minio bitnami/minio \
  --namespace ai-platform \
  --set auth.rootUser=admin \
  --set auth.rootPassword=secure_password \
  --set persistence.size=500Gi

# Install Keycloak
helm install keycloak bitnami/keycloak \
  --namespace ai-platform \
  --set auth.adminUser=admin \
  --set auth.adminPassword=secure_password \
  --set postgresql.enabled=false \
  --set externalDatabase.host=postgres-postgresql \
  --set externalDatabase.database=aiplatform

# Install Prometheus & Grafana
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace ai-platform \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi

# Install Kong API Gateway
helm install kong kong/kong \
  --namespace ai-platform \
  --set ingressController.enabled=true \
  --set proxy.type=LoadBalancer
```

### Step 3: Deploy Inference Services

```bash
# Create deployment manifest
cat > vllm-deployment.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: vllm-config
  namespace: ai-platform
data:
  MODEL_NAME: "meta-llama/Llama-2-7b-hf"
  MAX_MODEL_LEN: "4096"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-inference
  namespace: ai-platform
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vllm-inference
  template:
    metadata:
      labels:
        app: vllm-inference
    spec:
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        args:
          - --model
          - $(MODEL_NAME)
          - --dtype
          - auto
          - --max-model-len
          - $(MAX_MODEL_LEN)
        envFrom:
        - configMapRef:
            name: vllm-config
        resources:
          limits:
            nvidia.com/gpu: 1
            memory: 32Gi
          requests:
            nvidia.com/gpu: 1
            memory: 16Gi
        ports:
        - containerPort: 8000
          name: http
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 300
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 60
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-inference
  namespace: ai-platform
spec:
  selector:
    app: vllm-inference
  ports:
  - port: 8000
    targetPort: 8000
    name: http
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vllm-hpa
  namespace: ai-platform
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm-inference
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
EOF

kubectl apply -f vllm-deployment.yaml
```

### Step 4: Configure Ingress and TLS

```bash
# Install cert-manager for TLS certificates
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create ingress with TLS
cat > ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-platform-ingress
  namespace: ai-platform
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.yourdomain.com
    secretName: ai-platform-tls
  rules:
  - host: api.yourdomain.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: vllm-inference
            port:
              number: 8000
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: keycloak
            port:
              number: 8080
EOF

kubectl apply -f ingress.yaml
```

### Step 5: Initialize Database Schema

```bash
# Get PostgreSQL pod name
POSTGRES_POD=$(kubectl get pod -n ai-platform -l app.kubernetes.io/name=postgresql -o jsonpath="{.items[0].metadata.name}")

# Copy init script and execute
kubectl cp init-db.sql ai-platform/$POSTGRES_POD:/tmp/
kubectl exec -n ai-platform $POSTGRES_POD -- psql -U admin -d aiplatform -f /tmp/init-db.sql
```

## Component-by-Component Setup

### GPU Driver Setup

```bash
# Ubuntu 22.04 NVIDIA driver installation
sudo apt update
sudo apt install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo reboot

# Verify installation
nvidia-smi

# Install NVIDIA Container Toolkit
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo systemctl restart docker

# Test GPU in Docker
docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi
```

### Database Optimization

```bash
# PostgreSQL performance tuning
cat >> /etc/postgresql/16/main/postgresql.conf << 'EOF'
# Memory settings
shared_buffers = 8GB
effective_cache_size = 24GB
maintenance_work_mem = 2GB
work_mem = 64MB

# Checkpoint settings
checkpoint_completion_target = 0.9
wal_buffers = 16MB
max_wal_size = 4GB

# Connection settings
max_connections = 200

# Query optimization
random_page_cost = 1.1  # For SSD
effective_io_concurrency = 200
EOF

sudo systemctl restart postgresql
```

### Model Caching Strategy

```bash
# Pre-download models to persistent storage
mkdir -p /mnt/models
export HF_HOME=/mnt/models

# Download models
python3 << 'EOF'
from transformers import AutoTokenizer, AutoModelForCausalLM

models = [
    "meta-llama/Llama-2-7b-hf",
    "mistralai/Mistral-7B-v0.1",
]

for model_name in models:
    print(f"Downloading {model_name}...")
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    model = AutoModelForCausalLM.from_pretrained(model_name)
    print(f"Completed {model_name}")
EOF

# Create persistent volume in Kubernetes
cat > model-pvc.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-storage
  namespace: ai-platform
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 500Gi
EOF

kubectl apply -f model-pvc.yaml
```

## Post-Deployment Configuration

### 1. Configure Keycloak

```bash
# Access Keycloak admin console
# http://keycloak-url:8080/auth/admin

# Create realm
# 1. Click "Add Realm"
# 2. Name: ai-platform
# 3. Enable "User Registration"

# Create client
# 1. Clients -> Create
# 2. Client ID: ai-platform-api
# 3. Client Protocol: openid-connect
# 4. Access Type: confidential
# 5. Valid Redirect URIs: https://your-domain.com/*

# Create roles
# 1. Roles -> Add Role
# 2. Create: admin, developer, user

# Get client secret for application configuration
```

### 2. Set Up Grafana Dashboards

```bash
# Import pre-built dashboards
# Dashboard IDs:
# - Node Exporter: 1860
# - PostgreSQL: 9628
# - NVIDIA GPU: 12239
# - Kubernetes Cluster: 7249

# Access Grafana: http://grafana-url:3000
# Default credentials: admin/admin (change immediately)
```

### 3. Configure Monitoring Alerts

```bash
cat > alerting-rules.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alerts
  namespace: ai-platform
data:
  alerts.yml: |
    groups:
    - name: ai-platform
      rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        annotations:
          summary: "High error rate detected"
      
      - alert: GPUMemoryHigh
        expr: nvidia_gpu_memory_used_bytes / nvidia_gpu_memory_total_bytes > 0.9
        for: 5m
        annotations:
          summary: "GPU memory usage above 90%"
      
      - alert: DatabaseConnectionPoolExhausted
        expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.8
        for: 5m
        annotations:
          summary: "Database connection pool near limit"
EOF

kubectl apply -f alerting-rules.yaml
```

### 4. Create Initial API Keys

```bash
# Connect to database
kubectl exec -n ai-platform -it $POSTGRES_POD -- psql -U admin -d aiplatform

-- Create admin user
INSERT INTO users (email, subscription_tier, credits_balance)
VALUES ('admin@example.com', 'enterprise', 999999999);

-- Create API key (hash the key before storing)
INSERT INTO api_keys (user_id, key_hash, name)
VALUES (
  (SELECT id FROM users WHERE email = 'admin@example.com'),
  'hashed_api_key_value',
  'Admin API Key'
);
```

## Troubleshooting

### Common Issues

**Issue: vLLM out of memory**
```bash
# Solution: Reduce max_model_len or use smaller batch size
docker-compose exec vllm env | grep CUDA_VISIBLE_DEVICES
# Reduce model length in configuration
```

**Issue: PostgreSQL connection refused**
```bash
# Check if PostgreSQL is running
docker-compose ps postgres
# Check logs
docker-compose logs postgres
# Verify network connectivity
docker-compose exec api-service ping postgres
```

**Issue: Slow inference**
```bash
# Check GPU utilization
nvidia-smi
# Monitor vLLM metrics
curl http://localhost:8000/metrics
# Check if batching is enabled
```

**Issue: Keycloak not starting**
```bash
# Check database connection
docker-compose logs keycloak
# Verify PostgreSQL is ready
docker-compose ps
# Reset Keycloak data if needed
docker-compose down
docker volume rm ai-platform_keycloak_data
docker-compose up -d
```

### Performance Tuning

```bash
# Enable vLLM performance optimizations
export VLLM_USE_MODELSCOPE=False
export VLLM_ATTENTION_BACKEND=FLASH_ATTN
export CUDA_VISIBLE_DEVICES=0,1  # Multiple GPUs

# PostgreSQL connection pooling
# Use PgBouncer for better connection management

# Redis persistence tuning
# Disable persistence for pure cache usage
redis-cli CONFIG SET save ""
```

### Monitoring Commands

```bash
# Check all services health
docker-compose ps
kubectl get pods -n ai-platform

# View resource usage
docker stats
kubectl top nodes
kubectl top pods -n ai-platform

# Check logs
docker-compose logs -f --tail=100
kubectl logs -n ai-platform -l app=vllm-inference --tail=100 -f

# Database queries
docker-compose exec postgres psql -U admin -d aiplatform -c "SELECT COUNT(*) FROM users;"
kubectl exec -n ai-platform $POSTGRES_POD -- psql -U admin -d aiplatform -c "SELECT * FROM inference_requests ORDER BY timestamp DESC LIMIT 10;"
```

## Next Steps

1. Review the [Architecture Document](ARCHITECTURE.md) for system design details
2. Consult the [Cost Analysis](COSTS.md) for optimization strategies
3. Set up automated backups and disaster recovery
4. Implement CI/CD pipelines for updates
5. Configure custom monitoring dashboards
6. Scale based on actual usage patterns
