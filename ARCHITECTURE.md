# Self-Hosted AI Platform Architecture

## Overview

This document outlines the architecture for a scalable, self-hosted AI platform built with open-source tools. The platform enables AI model serving, user management, monitoring, and billing capabilities while maintaining full control over infrastructure and data.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                          Load Balancer                           │
│                      (Nginx / Traefik)                          │
└────────────┬────────────────────────────────────┬───────────────┘
             │                                    │
             │                                    │
┌────────────▼────────────┐          ┌───────────▼──────────────┐
│   API Gateway            │          │   Monitoring Stack       │
│   (Kong / KrakenD)       │          │   - Prometheus           │
│   - Authentication       │          │   - Grafana              │
│   - Rate Limiting        │          │   - Loki (Logs)          │
│   - Request Routing      │          │   - Jaeger (Tracing)     │
└────────────┬────────────┘          └──────────────────────────┘
             │
             │
┌────────────▼────────────────────────────────────────────────────┐
│                    Orchestration Layer                           │
│                    (Kubernetes / Docker Swarm)                   │
└─────┬──────────┬──────────┬──────────┬──────────┬──────────────┘
      │          │          │          │          │
      │          │          │          │          │
┌─────▼────┐ ┌──▼─────┐ ┌──▼─────┐ ┌──▼─────┐ ┌──▼─────────────┐
│ Inference│ │  Auth  │ │Billing │ │ Queue  │ │   Storage      │
│ Services │ │Service │ │Service │ │Service │ │   Layer        │
│          │ │        │ │        │ │        │ │                │
│- vLLM    │ │-Keycloak│ │-Stripe │ │-Redis  │ │- PostgreSQL    │
│- Ollama  │ │- Auth0 │ │-Custom │ │-RabbitMQ│ │- MinIO (S3)   │
│- TGI     │ │-Authelia│ │ API    │ │-Kafka  │ │- Redis Cache   │
└──────────┘ └────────┘ └────────┘ └────────┘ └────────────────┘
```

## Core Components

### 1. Inference Services

The inference layer handles AI model serving and execution.

#### Recommended Tools:
- **vLLM**: High-throughput LLM inference with PagedAttention
- **Ollama**: Easy-to-use local LLM deployment
- **Text Generation Inference (TGI)**: Hugging Face's optimized inference server
- **Ray Serve**: Scalable model serving framework

#### Features:
- Multi-model support
- GPU acceleration
- Batching and caching
- Model versioning
- A/B testing capabilities

#### Configuration Example:
```yaml
inference:
  engine: vllm
  models:
    - name: llama-2-7b
      replicas: 3
      gpu_memory: 16GB
      max_batch_size: 32
    - name: mistral-7b
      replicas: 2
      gpu_memory: 16GB
      max_batch_size: 16
```

### 2. Data Storage

Multi-tier storage architecture for different data types and access patterns.

#### Components:
- **PostgreSQL**: Primary database for structured data
  - User accounts and profiles
  - API keys and tokens
  - Billing records
  - Usage metrics
  
- **MinIO (S3-compatible)**: Object storage
  - Model weights and artifacts
  - Training datasets
  - Inference results
  - Backups and archives

- **Redis**: Caching and real-time data
  - Session management
  - Rate limiting counters
  - Hot data caching
  - Pub/sub for events

#### Database Schema Highlights:
```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    subscription_tier VARCHAR(50)
);

-- API Keys table
CREATE TABLE api_keys (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    key_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW(),
    last_used_at TIMESTAMP
);

-- Usage tracking
CREATE TABLE inference_requests (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    model_name VARCHAR(100),
    tokens_used INTEGER,
    latency_ms INTEGER,
    timestamp TIMESTAMP DEFAULT NOW()
);
```

### 3. Billing Service

Flexible billing system supporting multiple pricing models.

#### Features:
- Usage-based billing (tokens, requests, compute time)
- Subscription tiers (free, pro, enterprise)
- Credit system
- Invoice generation
- Payment processing integration

#### Integration Options:
- **Stripe**: Full-featured payment processing
- **Paddle**: Merchant of record solution
- **Custom API**: For specific requirements

#### Billing Models:
```python
# Token-based pricing
PRICING = {
    'llama-2-7b': {
        'input_tokens': 0.0001,   # per token
        'output_tokens': 0.0002,  # per token
    },
    'mistral-7b': {
        'input_tokens': 0.00015,
        'output_tokens': 0.0003,
    }
}

# Subscription tiers
TIERS = {
    'free': {
        'monthly_cost': 0,
        'included_credits': 100000,  # tokens
        'rate_limit': 10,  # requests per minute
    },
    'pro': {
        'monthly_cost': 29,
        'included_credits': 1000000,
        'rate_limit': 100,
    },
    'enterprise': {
        'monthly_cost': 'custom',
        'included_credits': 'unlimited',
        'rate_limit': 'custom',
    }
}
```

### 4. Authentication & Authorization

Secure, scalable authentication system with role-based access control.

#### Recommended Solutions:
- **Keycloak**: Enterprise-grade identity and access management
- **Authelia**: Lightweight authentication and authorization server
- **Auth0**: Managed authentication service (can be self-hosted with custom setup)

#### Features:
- OAuth 2.0 / OpenID Connect
- Multi-factor authentication (MFA)
- API key management
- Role-based access control (RBAC)
- SSO integration
- Session management

#### Security Configuration:
```yaml
auth:
  provider: keycloak
  realm: ai-platform
  features:
    - oauth2
    - mfa
    - api_keys
  session:
    timeout: 3600
    refresh_enabled: true
  rbac:
    roles:
      - admin
      - developer
      - user
    permissions:
      admin: ["*"]
      developer: ["model:read", "model:write", "inference:execute"]
      user: ["inference:execute"]
```

### 5. Monitoring & Observability

Comprehensive monitoring stack for system health, performance, and debugging.

#### Components:
- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboards
- **Loki**: Log aggregation
- **Jaeger**: Distributed tracing
- **AlertManager**: Alert routing and notification

#### Key Metrics to Track:
```yaml
metrics:
  infrastructure:
    - cpu_utilization
    - memory_usage
    - gpu_utilization
    - disk_io
    - network_bandwidth
  
  application:
    - requests_per_second
    - error_rate
    - p50_latency
    - p95_latency
    - p99_latency
  
  business:
    - active_users
    - api_calls_per_user
    - tokens_processed
    - revenue_per_day
    - cost_per_request
```

#### Dashboard Examples:
- System health overview
- Model performance metrics
- User activity and engagement
- Cost tracking and optimization
- SLA compliance monitoring

### 6. Orchestration

Container orchestration for deploying and managing services at scale.

#### Kubernetes (Recommended)
```yaml
# Deployment example
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-inference
spec:
  replicas: 3
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
        resources:
          limits:
            nvidia.com/gpu: 1
            memory: 32Gi
          requests:
            nvidia.com/gpu: 1
            memory: 16Gi
        env:
        - name: MODEL_NAME
          value: "meta-llama/Llama-2-7b-hf"
```

#### Alternative: Docker Swarm
Simpler orchestration for smaller deployments:
```yaml
version: '3.8'
services:
  inference:
    image: vllm/vllm-openai:latest
    deploy:
      replicas: 3
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

## Data Flow

### Typical Inference Request Flow:

1. **Client Request**
   - User sends API request with authentication token
   - Request hits load balancer

2. **API Gateway**
   - Validates authentication token
   - Checks rate limits
   - Routes to appropriate service

3. **Request Processing**
   - Request added to queue (if batching enabled)
   - Billing service records request metadata
   - Monitoring system logs request

4. **Inference Execution**
   - Model processes request
   - Response generated
   - Tokens counted for billing

5. **Response Delivery**
   - Response returned to client
   - Usage metrics updated
   - Billing record finalized

## Scalability Considerations

### Horizontal Scaling
- **Inference Services**: Scale based on GPU availability and request volume
- **API Gateway**: Stateless design allows unlimited horizontal scaling
- **Database**: Read replicas for query scaling, sharding for write scaling
- **Cache**: Redis cluster for distributed caching

### Auto-scaling Rules
```yaml
autoscaling:
  inference:
    metric: gpu_utilization
    target: 70%
    min_replicas: 2
    max_replicas: 10
  
  api_gateway:
    metric: cpu_utilization
    target: 60%
    min_replicas: 3
    max_replicas: 20
```

## High Availability

### Redundancy
- Multi-region deployment for disaster recovery
- Database replication across availability zones
- Load balancer failover configuration
- Model replica distribution

### Backup Strategy
- Daily automated database backups
- Model weights versioned in S3
- Configuration stored in version control
- Disaster recovery runbooks

## Security Best Practices

1. **Network Security**
   - Private VPC/VLAN for internal services
   - TLS/SSL for all external communication
   - Firewall rules limiting service access

2. **Data Security**
   - Encryption at rest and in transit
   - API key hashing (never store plaintext)
   - Regular security audits
   - PII data handling compliance

3. **Access Control**
   - Principle of least privilege
   - Regular access reviews
   - Audit logging for all actions
   - MFA for administrative access

4. **Model Security**
   - Model signing and verification
   - Access controls on model artifacts
   - Inference input validation
   - Output filtering for sensitive data

## Technology Stack Summary

| Component | Primary Option | Alternative |
|-----------|---------------|-------------|
| Inference | vLLM | Ollama, TGI, Ray Serve |
| Database | PostgreSQL | MySQL, CockroachDB |
| Object Storage | MinIO | Ceph, SeaweedFS |
| Cache | Redis | Memcached, KeyDB |
| Auth | Keycloak | Authelia, Ory |
| Monitoring | Prometheus + Grafana | Netdata, VictoriaMetrics |
| Orchestration | Kubernetes | Docker Swarm, Nomad |
| Load Balancer | Nginx | Traefik, HAProxy |
| API Gateway | Kong | KrakenD, Tyk |
| Message Queue | RabbitMQ | Kafka, NATS |

## Next Steps

1. Review the [Deployment Guide](DEPLOYMENT.md) for step-by-step setup instructions
2. Consult the [Cost Analysis](COSTS.md) for budgeting and optimization
3. Customize components based on your specific requirements
4. Start with a minimal viable deployment and scale incrementally
