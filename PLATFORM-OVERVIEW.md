# Mosadad Recovery Claims Platform — Architecture Overview

> **Last Updated**: July 3, 2026

---

## 1. Platform Identity

**Mosadad Recovery Claims** is an insurance subrogation recovery platform built for the UAE market. It automates the complete lifecycle of vehicle insurance claims — from filing through investigation, quotation, invoicing, settlement, and closure — serving both insurance companies (admin portal) and repair entities (entity portal).

---

## 2. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                              PRODUCTION ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  Users ──► app.recoveryclaims.ae (DNS)                                                  │
│                 │                                                                        │
│                 ▼                                                                        │
│  ┌─────────────────────────────┐                                                        │
│  │   Azure App Gateway (WAF)   │  TLS termination, WAF rules, rewrite rules            │
│  │   4.161.44.242              │  Injects APIM subscription key for portal traffic      │
│  └──────────────┬──────────────┘                                                        │
│                 │                                                                        │
│       ┌─────────┴─────────┐                                                            │
│       │                   │                                                            │
│       ▼                   ▼                                                            │
│  ┌──────────┐    ┌────────────────────────┐                                            │
│  │  Portals │    │  APIM v2 (StandardV2)  │  Rate limiting, CORS, TLS 1.2+            │
│  │ (Direct) │    │  7 APIs registered     │  Subscription keys, Security headers       │
│  └──────────┘    └───────────┬────────────┘                                            │
│                              │                                                          │
│                              ▼                                                          │
│  ┌────────────────────────────────────────────────────────────────────────────────┐     │
│  │                    AKS Cluster (2× Standard_D4ds_v5)                           │     │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │     │
│  │  │  NGINX Ingress Controller (20.233.167.149)                              │   │     │
│  │  │  Path-based routing: /claims, /tenant, /quotation, /invoice, etc.       │   │     │
│  │  └──────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┘   │     │
│  │         │          │          │          │          │          │               │     │
│  │         ▼          ▼          ▼          ▼          ▼          ▼               │     │
│  │  ┌──────────┐┌──────────┐┌──────────┐┌──────────┐┌──────────┐┌──────────┐    │     │
│  │  │  Claims  ││  Tenant  ││Quotation ││ Invoice  ││Settlement││  IntHub  │    │     │
│  │  │  .NET 8  ││  .NET 8  ││  .NET 8  ││  .NET 8  ││  .NET 8  ││  .NET 8  │    │     │
│  │  └────┬─────┘└────┬─────┘└────┬─────┘└────┬─────┘└────┬─────┘└────┬─────┘    │     │
│  │       └───────────┴───────────┴───────────┴───────────┴───────────┘           │     │
│  │                              │                                                 │     │
│  └──────────────────────────────┼─────────────────────────────────────────────────┘     │
│                                 │                                                        │
│                    ┌────────────┴────────────────────┐                                  │
│                    │         Data Layer              │                                  │
│                    │  ┌─────────┐  ┌─────────────┐  │                                  │
│                    │  │ MongoDB │  │ Azure Redis  │  │                                  │
│                    │  │  (VM)   │  │   Cache      │  │                                  │
│                    │  └─────────┘  └─────────────┘  │                                  │
│                    │  ┌─────────┐  ┌─────────────┐  │                                  │
│                    │  │  Blob   │  │  CosmosDB   │  │                                  │
│                    │  │ Storage │  │  (Payment)  │  │                                  │
│                    │  └─────────┘  └─────────────┘  │                                  │
│                    └────────────────────────────────┘                                  │
│                                                                                          │
│  ┌───────────────────────────────── Supporting ──────────────────────────────────┐      │
│  │  Key Vault │ App Insights │ Grafana+Prometheus │ Firewall │ Bastion │ ACR    │      │
│  └──────────────────────────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Service Inventory

### 3.1 Backend Microservices

| Service | Repo | Purpose | API Path | DB |
|---------|------|---------|----------|-----|
| **Claims** | `backend-Mosadad-claims` | Core claims lifecycle, dashboards, SLA management | `/api/claims` | MongoDB (MotoriDB) |
| **Tenant** | `backend-mosadad-tenant` | Auth, users, entities, B2C integration, refresh tokens | `/api/tenant` | MongoDB |
| **Quotation** | `backend-mosadad-quotation` | Repair/replacement quotes, bulk approval, negotiations | `/api/quotation` | MongoDB |
| **Invoice** | `backend-mosadad-invoice` | Billing, LPO invoices, payment tracking | `/api/invoice` | MongoDB |
| **Settlement** | `backend-mosadad-settlement` | Final settlement processing, payment reconciliation | `/api/settlement` | MongoDB |
| **IntHub** | `backend-mosadad-inthub` | External integrations (SMS, email, third-party APIs) | `/api/inthub` | MongoDB |
| **TriggerHub** | `backend-triggerhub` | Azure Functions — scheduled jobs, event-driven tasks | N/A (timer/queue) | MongoDB |

### 3.2 Frontend Applications

| App | Repo | Framework | Purpose | Hosting |
|-----|------|-----------|---------|---------|
| **Admin Portal** | `WebApps-admin-portal` | Angular 18 | Insurance company users — claims management, dashboards | Azure App Service |
| **Entity Portal** | `WebApps-entity-portal` | Angular 18 | Repair entities — quotation submission, status tracking | Azure App Service |

### 3.3 Infrastructure Repos

| Repo | Purpose |
|------|---------|
| `terraform` | IaC for all Azure resources (modularized) |
| `kubernetes` | K8s manifests — Kustomize base + overlays for 4 envs |

---

## 4. Technology Stack

| Layer | Technology |
|-------|-----------|
| **Compute** | AKS (Kubernetes 1.28+), Azure App Service |
| **Runtime** | .NET 8 (ASP.NET Core Web API) |
| **Frontend** | Angular 19, TypeScript |
| **Database** | MongoDB (self-hosted VM), Azure CosmosDB (payment) |
| **Cache** | Azure Redis Cache |
| **API Gateway** | Azure API Management (StandardV2) |
| **WAF/LB** | Azure Application Gateway (WAF_v2) |
| **Auth** | Azure AD B2C (per-environment tenants) |
| **Secrets** | Azure Key Vault (CSI driver mount in K8s) |
| **Storage** | Azure Blob Storage |
| **Monitoring** | Application Insights + Grafana + Prometheus |
| **CI/CD** | GitHub Actions |
| **IaC** | Terraform (modularized — api_management, aks, app_service, etc.) |
| **Container Registry** | Azure Container Registry (per-env) |
| **Networking** | Azure Firewall, Bastion, VNet + subnets, NSGs |
| **Serverless** | Azure Functions (Isolated Worker, .NET 8) |

---

## 5. Security Posture (Production)

| Control | Status |
|---------|--------|
| TLS 1.2+ enforced | ✅ |
| WAF enabled (App Gateway) | ✅ |
| Rate limiting (100/60s/IP) | ✅ |
| CORS restricted (5 origins) | ✅ |
| Subscription keys on all APIs | ✅ |
| Security headers (HSTS, X-Frame, nosniff) | ✅ |
| Key Vault for secrets | ✅ (QA/Staging/Prod) |
| Network segmentation (subnets) | ✅ |
| Azure Firewall (ThreatIntel: Alert) | ✅ |
| Bastion for SSH access | ✅ |
| Container image scanning | ⚠️ Not automated |
| Network policies in K8s | ⚠️ Not configured |
| Pod security standards | ⚠️ Not enforced |

---

## 6. Observability

| Component | Tool | Status |
|-----------|------|--------|
| APM & Distributed Tracing | Application Insights | ✅ Active (all 6 services) |
| Metrics | Prometheus + Grafana | ✅ Connected |
| Alerting | Azure Monitor (4 rules) | ✅ Enabled |
| Availability Tests | 3 ping tests | ✅ Enabled |
| Logging | Application Insights + Log Analytics | ✅ |
| Real-time Notifications | SignalR (via APIM WebSocket) | ✅ |

---

## 7. CI/CD Architecture

### Pipeline Pattern (Backend Services)
```
git push → GitHub Actions → dotnet build → Docker build → Push to ACR → kubectl set image → APIM swagger import
```

### Pipeline Pattern (Frontend Portals)
```
git push → GitHub Actions → npm install → ng build --configuration={env} → Docker build (nginx) → Push to ACR → az webapp deploy
```

### Branch → Environment Mapping

| Branch | AKS Namespace | Cluster | ACR | APIM |
|--------|---------------|---------|-----|------|
| `dev` | dev | Recovery-aks-dev | recoveryacr | recovery-api-management-dev |
| `qa` | qa | Recovery-aks-qa | recoveryacr | recovery-api-management-qa |
| `staging` | staging | Recovery-aks-staging | recoveryacrstaging | recovery-api-management-staging |
| `prod` | prod | Recovery-aks-prod | recoveryacrprod | recovery-api-management-prod |

### Key CI/CD Features
- **DRY Workflow Pattern**: Single workflow file uses GitHub Environment variables
- **Guardrails**: Branch-to-environment validation prevents wrong deployments
- **Image Tags**: `{env}-{short_sha}` for traceability
- **APIM Import**: Automatic Swagger/OpenAPI import post-deploy
- **Rollback**: Previous image tag stored, manual rollback via `kubectl set image`

---

## 8. Current State Summary (June 11, 2026)

### All Environments Healthy

| Environment | Pods | Nodes | Status |
|-------------|------|-------|--------|
| Dev | 6/6 Running | 3 | ✅ Healthy |
| QA | 12 (2 replicas each) | 4 | ✅ Healthy |
| Staging | 6/6 Running | 3 | ✅ Healthy |
| Production | 6/6 Running | 2 | ✅ Healthy |

### Recent Fixes Applied (This Session)
1. ✅ Quotation API path standardized (`api/quotation` singular) — all environments
2. ✅ Portal CI/CD workflows unified to DRY pattern — all branches
3. ✅ Missing `TenantAPI` config added to dev/qa claims
4. ✅ Entity portal environment URLs corrected — all branches
5. ✅ Claims pods restarted with correct configmaps
