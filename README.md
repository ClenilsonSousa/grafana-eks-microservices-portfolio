# Observability Platform — Microservices no Amazon EKS

![Status](https://img.shields.io/badge/Status-Produção-success)
![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazonaws)
![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?logo=terraform)
![EKS](https://img.shields.io/badge/AWS-EKS_1.35-FF9900?logo=amazonaws)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.35-326CE5?logo=kubernetes)
![Istio](https://img.shields.io/badge/Istio-1.23-466BB0?logo=istio)
![Grafana](https://img.shields.io/badge/Grafana-35+_Dashboards-F46800?logo=grafana)
![Prometheus](https://img.shields.io/badge/Prometheus-Monitoring-E6522C?logo=prometheus)
![Kong](https://img.shields.io/badge/Kong-API_Gateway-003459?logo=kong)
![WAF](https://img.shields.io/badge/AWS-WAFv2-DD344C?logo=amazonaws)
![Helm](https://img.shields.io/badge/Helm-3.x-0F1689?logo=helm)
![ARM64](https://img.shields.io/badge/ARM64-Graviton-FF6600)

Plataforma de observabilidade **100% como código com Terraform** baseada em **arquitetura de microserviços** no **Amazon EKS**. Stack completa com **6 microserviços**, **Istio Service Mesh** (mTLS, Circuit Breaker, Traffic Management), **Prometheus + Grafana + Loki + Jaeger** para observabilidade full-stack, **Kong API Gateway**, comunicação assíncrona via **SNS/SQS** (Event-Driven + CQRS + Saga Pattern), **3 bancos de dados dedicados** (PostgreSQL, MySQL, DynamoDB) + **Redis Multi-AZ**, e segurança com **WAFv2** (OWASP Top 10 + Rate Limiting).

> ⚠️ **Nota:** O código-fonte Terraform é mantido em repositório privado. Este repositório público serve como portfólio documental com arquitetura, prints e resultados.

---

# 📑 Índice

- [Visão Geral](#-visão-geral)
- [Arquitetura](#-arquitetura)
- [Microserviços](#-microserviços)
- [Stack Tecnológica](#-stack-tecnológica)
- [Serviços AWS](#-serviços-aws)
- [Event-Driven Architecture](#-event-driven-architecture)
- [Service Mesh (Istio)](#-service-mesh-istio)
- [Observabilidade](#-observabilidade)
- [Segurança](#-segurança)
- [Prints da Infraestrutura](#-prints-da-infraestrutura)
- [Deploy](#-deploy)
- [Custos Mensais Estimados](#-custos-mensais-estimados)
- [Autor](#-autor)

---

# 📖 Visão Geral

Este projeto implementa uma **plataforma de observabilidade distribuída** no **Amazon EKS**, seguindo o **AWS Well-Architected Framework** e princípios de **Domain-Driven Design**. Toda a infraestrutura — da VPC aos dashboards — é criada por **Terraform** (~290 recursos), sem dependências de recursos pré-existentes:

| Pilar | Implementação |
|---|---|
| **Excelência Operacional** | IaC com Terraform (50+ arquivos), S3 backend, deploy em 2 fases, zero intervenção manual |
| **Segurança** | Istio mTLS STRICT, WAFv2 (4 regras), Secrets Manager, IRSA, KMS encryption, Security Groups restritivos |
| **Confiabilidade** | 6 microserviços com HPA (2-15 réplicas), Circuit Breaker, 3 AZs, Redis Multi-AZ, DLQs, Saga Pattern |
| **Eficiência** | ARM64 Graviton (t4g.medium), DynamoDB on-demand, Redis cache (CQRS), connection pooling |
| **Otimização de Custos** | ARM64 (~20% mais barato), NAT x2, DynamoDB PAY_PER_REQUEST, single-AZ RDS (portfolio) |

---

# 🏗️ Arquitetura

![Arquitetura](arquitetura/diagrama_microservicos.png)

```
User ─► Route 53 ─► ACM TLS ─► WAFv2 (4 rules) ─► ALB
                                                      │
                                              ┌───────▼────────┐
                                              │   Kong Gateway │
                                              │   (API Gateway)│
                                              └───────┬────────┘
                                                      │
                                              ┌───────▼────────┐
                                              │  Istio Gateway │
                                              │  (Service Mesh)│
                                              └───────┬────────┘
                          ┌───────────────────────────┼──────────────────────────┐
                          │                           │                          │
                ┌─────────▼──────────┐    ┌───────────▼──────────┐   ┌───────────▼──────────┐
                │   Auth Service     │    │  Dashboard Service   │   │   Metrics Service    │
                │   FastAPI :8000    │    │       :8080          │   │       :8080          │
                │   PostgreSQL 16    │    │     MySQL 8.0        │   │     DynamoDB         │
                └────────────────────┘    └──────────────────────┘   └──────────────────────┘
                          │                           │                           │
                ┌─────────▼──────────┐   ┌────────────▼──────────┐   ┌────────────▼──────────┐
                │  Alerting Service  │   │ Notification Service  │   │   Query Service       │
                │       :8080        │   │       :8080           │   │       :8080           │
                │   PostgreSQL 16    │   │    (stateless)        │   │  Redis 7.1 Multi-AZ   │
                └────────────────────┘   └───────────────────────┘   └───────────────────────┘
```

---

# 🔧 Microserviços

| # | Serviço | Porta | Database | Responsabilidade |
|---|---------|-------|----------|------------------|
| 1 | **Auth Service** | 8000 | PostgreSQL 16 | Autenticação JWT, signup/login, verificação de tokens |
| 2 | **Dashboard Service** | 8080 | MySQL 8.0 | CRUD de dashboards, templates, visualizações |
| 3 | **Metrics Service** | 8080 | DynamoDB | Ingestão de métricas time-series (gauge, counter, histogram) |
| 4 | **Alerting Service** | 8080 | PostgreSQL 16 | Regras de alerta, avaliação de thresholds, disparo |
| 5 | **Notification Service** | 8080 | — (stateless) | Envio de notificações (email, Slack, webhook) |
| 6 | **Query Service** | 8080 | Redis 7.1 | CQRS read model, cache de queries, agregações |

### Padrões Aplicados
- **Database per Service** — cada serviço tem seu banco dedicado
- **Event-Driven Architecture** — comunicação assíncrona via SNS/SQS
- **CQRS** — separação de escrita (DynamoDB) e leitura (Redis cache)
- **Saga Pattern** — orquestração de transações distribuídas com compensação
- **Circuit Breaker** — Istio DestinationRules com outlier detection
- **API Gateway** — Kong + Istio VirtualService para roteamento

---

# 🛠️ Stack Tecnológica

| Camada | Tecnologia |
|--------|-----------|
| **IaC** | Terraform 1.15+ (S3 backend, ~290 recursos) |
| **Orquestração** | EKS 1.35, ARM64 Graviton (t4g.medium), 4 nodes |
| **Service Mesh** | Istio 1.23 (mTLS, Circuit Breaker, Traffic Mirror, A/B Test) |
| **API Gateway** | Kong Ingress Controller |
| **Load Balancer** | AWS ALB (via AWS Load Balancer Controller) |
| **Monitoring** | Prometheus + Alertmanager |
| **Dashboards** | Grafana (35+ dashboards, sidecar provisioning) |
| **Logs** | Loki (DaemonSet, integrado ao Grafana) |
| **Tracing** | Jaeger + AWS X-Ray |
| **Databases** | PostgreSQL 16 (x2), MySQL 8.0, DynamoDB, Redis 7.1 |
| **Messaging** | SNS (5 topics) + SQS (7 queues + 7 DLQs) |
| **Security** | WAFv2, KMS, Secrets Manager, ACM, IRSA |
| **Storage** | EFS (Grafana PV), S3 (TF state), ECR (container images) |
| **Backup** | AWS Backup (EFS, 30 dias retenção) |
| **Container Build** | Docker Buildx + QEMU (cross-compile ARM64) |

---

# ☁️ Serviços AWS

| Serviço | Uso |
|---------|-----|
| **EKS** | Cluster Kubernetes 1.35 (ARM64) |
| **ECR** | Registry para imagem do Auth Service |
| **RDS** | 2x PostgreSQL 16 + 1x MySQL 8.0 (gp3, encrypted) |
| **ElastiCache** | Redis 7.1 (3 nodes, Multi-AZ, failover automático) |
| **DynamoDB** | Metrics time-series (on-demand, TTL, PITR, KMS) |
| **SQS** | 7 filas + 7 DLQs (event-driven communication) |
| **SNS** | 5 tópicos (metrics-created, alert-triggered, dashboard-updated, user-activity, system-events) |
| **ALB** | Application Load Balancer (AWS LBC) |
| **WAFv2** | 4 regras (OWASP, SQLi, IP Reputation, Rate Limit) |
| **VPC** | 10.0.0.0/16, 3 AZs, 6 subnets, NAT x2 |
| **Route 53** | DNS |
| **ACM** | Certificado TLS wildcard |
| **Secrets Manager** | Credenciais DB + Grafana admin |
| **KMS** | Encryption (RDS, DynamoDB, EFS) |
| **CloudWatch** | Logs + Alarms (CPU, connections, storage, DLQ) |
| **AWS Backup** | EFS daily backup, 30 dias retenção |
| **EFS** | Persistent Volume para Grafana |
| **S3** | Terraform state backend |
| **IAM** | IRSA (Service Accounts com roles dedicadas) |

---

# 📨 Event-Driven Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         SNS Topics (Pub/Sub)                             │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  metrics-created ──► alerting-events (SQS) ──► Alerting Service          │
│         │                                                                │
│         └──────────► query-events (SQS) ──────► Query Service (CQRS)     │
│                                                                          │
│  alert-triggered ──► notification-events (SQS) ► Notification Service    │
│                                                                          │
│  dashboard-updated ► metrics-events (SQS) ──► Metrics Service            │
│         │                                                                │
│         └──────────► query-events (SQS) ──────► Query Service (cache)    │
│                                                                          │
│  user-activity ────► auth-events (SQS) ──────► Auth Service              │
│                                                                          │
│  system-events ────► saga-events (SQS) ──────► Saga Orchestrator         │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

Cada fila tem DLQ (Dead Letter Queue) com 14 dias de retenção e alarme CloudWatch.
```

**Fluxo típico:**
1. **Metrics Service** ingere métricas → publica `metrics-created`
2. **Alerting Service** avalia regras → se threshold violado → publica `alert-triggered`
3. **Notification Service** recebe alerta → envia notificação
4. **Query Service** atualiza read model no Redis (CQRS)

---

# 🕸️ Service Mesh (Istio)

| Feature | Configuração |
|---------|-------------|
| **mTLS** | STRICT mode (toda comunicação pod-to-pod criptografada) |
| **Circuit Breaker** | Outlier Detection (3-10 erros → eject 30-60s) |
| **Connection Pool** | TCP: 100-200 max | HTTP: 50-200 pending requests |
| **Traffic Mirroring** | Dashboard Service: 10% espelhado para canary v2 |
| **A/B Testing** | Metrics Service: header `x-version: beta` → v2 |
| **Rate Limiting** | Global: 10.000 tokens, 1.000/s refill (EnvoyFilter) |
| **Retry + Timeout** | Automático com backoff exponencial |
| **Telemetry** | Prometheus metrics (port 15090) + Zipkin tracing |

---

# 📊 Observabilidade

| Ferramenta | Função | Integração |
|-----------|--------|-----------|
| **Prometheus** | Coleta de métricas (PodMonitor + ServiceMonitor) | Scrape a cada 15-30s |
| **Grafana** | 35+ dashboards (Kubernetes, Istio, CoreDNS, API Server) | Datasources: Prometheus, Loki, Jaeger |
| **Loki** | Agregação de logs (DaemonSet) | Label-based querying |
| **Jaeger** | Distributed tracing | 100% sampling via Istio |
| **X-Ray** | AWS-native tracing | DaemonSet no kube-system |
| **CloudWatch** | Alarms + Log Groups (9 serviços) | CPU, connections, storage, DLQ |

### Dashboards Istio Customizados
- **Istio Service Dashboard** — Request Volume, Success Rate, Duration p50/p90/p99
- **Istio Workload Dashboard** — Inbound/Outbound Traffic, Response Codes
- **Istio Control Plane** — XDS Pushes, Connected Proxies, Pilot Errors, Certificates

---

# 🔒 Segurança

| Camada | Proteção |
|--------|----------|
| **Perímetro** | WAFv2: OWASP Top 10 + SQLi + IP Reputation + Rate Limit (1000/5min em /login) |
| **Rede** | VPC isolada, Security Groups restritivos, subnets privadas, NAT Gateway |
| **Transport** | Istio mTLS STRICT (zero-trust), ACM TLS wildcard |
| **Auth** | JWT HS256 (30min expiry), bcrypt password hashing |
| **Dados** | KMS encryption (RDS, DynamoDB, EFS, Redis at-rest + in-transit) |
| **Secrets** | AWS Secrets Manager (auto-generated passwords, 32 chars) |
| **IAM** | IRSA (least-privilege per service), no shared credentials |

---

# 📸 Prints da Infraestrutura

### VPC & Networking
| VPC | Subnets | Route Tables |
|-----|---------|-------------|
| ![VPC](arquitetura/VPC.png) | ![Subnets](arquitetura/subredes.png) | ![Routes](arquitetura/route-tables.png) |

| Internet Gateway | NAT Gateway | Security Groups |
|-----------------|-------------|-----------------|
| ![IGW](arquitetura/InternetGateway.png) | ![NAT](arquitetura/NatGateway.png) | ![SGs](arquitetura/securitygroups.png) |

### Compute & Containers
| EKS Cluster | ECR Repository | Pods (1/4) |
|-------------|---------------|------------|
| ![EKS](arquitetura/eks.png) | ![ECR](arquitetura/ecr.png) | ![Pods](arquitetura/workloads-pods1_de_4.png) |

| Pods (2/4) | Pods (3/4) | Pods (4/4) |
|------------|------------|------------|
| ![Pods2](arquitetura/workloads-pods2_de_4.png) | ![Pods3](arquitetura/workloads-pods3_de_4.png) | ![Pods4](arquitetura/workloads-pods4_de_4.png) |

### Databases
| RDS (PostgreSQL + MySQL) | ElastiCache Redis | DynamoDB |
|--------------------------|-------------------|----------|
| ![RDS](arquitetura/rds.png) | ![Redis](arquitetura/elasticache-redis.png) | ![DynamoDB](arquitetura/dynamoDB.png) |

### Messaging (Event Bus)
| SNS Topics | SQS Queues |
|-----------|-----------|
| ![SNS](arquitetura/sns.png) | ![SQS](arquitetura/sqs.png) |

### Security & Observability
| WAFv2 | IAM | Secrets Manager |
|-------|-----|-----------------|
| ![WAF](arquitetura/waf.png) | ![IAM](arquitetura/IAM.png) | ![Secrets](arquitetura/secrets-manager.png) |

| KMS | CloudWatch | AWS Backup |
|-----|-----------|-----------|
| ![KMS](arquitetura/kms.png) | ![CW](arquitetura/cloudwatch.png) | ![Backup](arquitetura/aws-backup.png) |

| ACM Certificate |
|-----------------|
| ![ACM](arquitetura/certificate-manager.png) |

### Grafana
| Login | Datasources | Dashboard 1 |
|-------|------------|-------------|
| ![Login](arquitetura/login-grafana.png) | ![DS](arquitetura/datasources-grafana.png) | ![D1](arquitetura/grafana-dashboard1.png) |

| Dashboard 2 | Dashboard 3 | Dashboard 4 |
|-------------|-------------|-------------|
| ![D2](arquitetura/grafana-dashboard2.png) | ![D3](arquitetura/grafana-dashboard3.png) | ![D4](arquitetura/grafana-dashboard4.png) |

| Istio Service | Istio Workload | Istio Control Plane |
|--------------|---------------|-------------------|
| ![IS](arquitetura/dashboard-istio1.png) | ![IW](arquitetura/dashboard-istio3.png) | ![ICP](arquitetura/dashboard-istio4.png) |

| Kubernetes API Server | Kubernetes Compute Resources |
|----------------------|------------------------------|
| ![API](arquitetura/dashboard-kubernetes-api-server.png) | ![Compute](arquitetura/dashboard-kubernetes-compute-resources-nodes.png) |

---

# 🚀 Deploy

O deploy é 100% automatizado via Terraform (2 fases):

```bash
# Fase 1: Infraestrutura base (VPC + EKS + IAM + Nodes)
terraform apply -target=aws_eks_cluster.main -target=aws_eks_access_entry.console -target=aws_eks_node_group.main

# Fase 2: Tudo (microserviços, Helm charts, Istio, monitoring, databases)
terraform apply
```

**Pré-requisitos:**
- Docker com QEMU (cross-compile ARM64): `docker run --privileged multiarch/qemu-user-static --reset -p yes`
- AWS CLI configurado
- Terraform 1.15+

**Total de recursos Terraform:** ~290

---

# 💰 Custos Mensais Estimados

| Recurso | Custo/mês |
|---------|-----------|
| EKS Cluster | $73 |
| EC2 Nodes (4x t4g.medium) | ~$108 |
| RDS PostgreSQL x2 (t4g.medium) | ~$130 |
| RDS MySQL (t4g.medium) | ~$65 |
| ElastiCache Redis (3x t4g.micro) | ~$40 |
| DynamoDB (on-demand) | ~$5-25 |
| NAT Gateway x2 | ~$65 |
| ALB | ~$22 |
| EFS + S3 + ECR | ~$10 |
| **Total estimado** | **~$520-540/mês** |

> *Valores para us-east-2, uso low-traffic (portfólio). Produção seria ~2-3x com Multi-AZ RDS + mais nodes.*

---

# 👤 Autor

**Clenilson Sousa** — Engenheiro DevOps / Cloud

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?logo=linkedin&logoColor=white)](https://www.linkedin.com/in/clenilsonsousa/)
[![GitHub](https://img.shields.io/badge/GitHub-181717?logo=github&logoColor=white)](https://github.com/ClenilsonSousa)

---

### Projetos relacionados:
- [Grafana EKS Kubernetes (Monolítico)](https://github.com/ClenilsonSousa/grafana-eks-k8s-portfolio) — Grafana HA no EKS com Aurora MySQL
- [Grafana ECS Fargate](https://github.com/ClenilsonSousa/grafana-ecs-fargate-portfolio) — Grafana HA no ECS Fargate
