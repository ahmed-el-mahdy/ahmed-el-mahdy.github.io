# Environment Matrix & Gap Analysis

> **Last Updated**: July 3, 2026

---

## 1. Environment Comparison Matrix

### 1.1 Infrastructure

| Resource | Dev | QA | Staging | Prod |
|----------|-----|-----|---------|------|
| **AKS Cluster** | Recovery-aks-dev | Recovery-aks-qa | Recovery-aks-staging | Recovery-aks-prod |
| **Nodes** | 3 | 4 | 3 | 2 |
| **Node SKU** | Standard_D2s_v3 | Standard_D2s_v3 | Standard_D2s_v3 | Standard_D4ds_v5 (upgraded Jun 2026) |
| **Replicas per service** | 1 (HPA max 2) | 2 (HPA max 3) | 1 (HPA max 4) | 1-4 (HPA autoscaled) |
| **HPA** | ✅ max=2 | ✅ max=3 | ✅ max=4 | ✅ max=4 (CPU 70% / Mem 80%) |
| **Cluster Autoscaler** | ❌ | ✅ 3-6 nodes | ❌ (3 fixed) | ✅ 2-5 nodes |
| **Resource Group** | recovery-rg | recovery-rg-qa | recovery-rg-staging | Recovery-rg-prod |
| **Subscription** | 07fd1f02-* | 07fd1f02-* | 8ad13378-* | 86e6cf03-* |

### 1.2 Networking & Security

| Component | Dev | QA | Staging | Prod |
|-----------|-----|-----|---------|------|
| **APIM** | recovery-api-management-dev | recovery-api-management-qa | recovery-api-management-staging | recovery-api-management-prod |
| **APIM SKU** | Developer | Developer | Developer | StandardV2 |
| **App Gateway** | ❌ | ❌ | ❌ | ✅ WAF_v2 |
| **Firewall** | ❌ | ❌ | ❌ | ✅ Azure Firewall |
| **Bastion** | ❌ | ❌ | ❌ | ✅ Standard |
| **Ingress** | unified-ingress (default ns) | qa-unified-ingress (qa ns) | ❌ (no ingress file) | prod-unified-ingress (prod ns) |
| **TLS on Ingress** | ❌ | ❌ | ❌ | ❌ (APIM handles TLS) |
| **Network Policies** | ❌ | ❌ | ❌ | ❌ |

### 1.3 Data Layer

| Component | Dev | QA | Staging | Prod |
|-----------|-----|-----|---------|------|
| **MongoDB** | VM (same VNet) | VM (same VNet) | VM (same VNet) | VM (mongo-subnet, 10.0.2.4) |
| **Redis** | recoveryClaim-redis-cache-dev | recoveryClaim-redis-cache-qa | recoveryClaim-redis-cache-staging | recoveryClaim-redis-cache-prod |
| **Key Vault** | ❌ (secrets in configmap) | ✅ Recovery-keyvault-qa | ✅ Recovery-keyvault-staging | ✅ Recovery-keyvault-prod |
| **CosmosDB** | ❌ | ❌ | ❌ | ✅ payment-cosmos-prod |
| **Blob Storage** | recoverystoragedev | recoverystorageqa | recoverystoragestaging | recoverystorageprod |

### 1.4 Authentication (Azure AD B2C)

| Setting | Dev | QA | Staging | Prod |
|---------|-----|-----|---------|------|
| **B2C Tenant** | RecoveryClaims | RecoveryClaimsQA | RecoveryClaimsStaging | RecoveryClaimsStaging* |
| **TenantId** | ccd664dc-* | cd933260-* | 58189f61-* | 58189f61-* |
| **Auth Method** | Username/Password | B2C Policies | B2C + Auth0 | B2C + Auth0 |

> ⚠️ *Staging and Prod share the same B2C tenant* — this is a security risk for production isolation.

### 1.5 Observability

| Component | Dev | QA | Staging | Prod |
|-----------|-----|-----|---------|------|
| **App Insights** | ✅ (shared key) | ✅ (shared key) | ✅ (separate key) | ✅ (separate key) |
| **Grafana** | ❌ | ❌ | ❌ | ✅ |
| **Prometheus** | ❌ | ❌ | ❌ | ✅ |
| **Alert Rules** | ❌ | ❌ | ❌ | ✅ (4 rules) |
| **Availability Tests** | ❌ | ❌ | ❌ | ✅ (3 tests) |

### 1.6 CI/CD & Container Registry

| Component | Dev | QA | Staging | Prod |
|-----------|-----|-----|---------|------|
| **ACR** | recoveryacr | recoveryacr | recoveryacrstaging | recoveryacrprod |
| **Pipeline Trigger** | push to `dev` | push to `qa` | push to `staging` | push to `prod` |
| **APIM Import** | ✅ | ✅ | ✅ | ✅ |
| **Rollback Strategy** | Manual kubectl | Manual kubectl | Manual kubectl | Manual kubectl |
| **Blue/Green** | ❌ | ❌ | ❌ | ❌ |
| **Canary** | ❌ | ❌ | ❌ | ❌ |

---

## 2. Critical Gaps Identified

### 🔴 HIGH Priority

| # | Gap | Impact | Solution | Effort | Status |
|---|-----|--------|----------|--------|--------|
| 1 | **Dev secrets in plaintext configmaps** | Credentials exposed in git | Migrate dev to Key Vault (CSI driver already in QA/staging/prod) | 4h | ⬜ Open |
| 2 | **No staging ingress** | Services unreachable via path-based routing | Create `overlays/staging/ingress.yaml` matching QA pattern | 1h | ⬜ Open |
| 3 | ~~No HPA or autoscaling (prod)~~ | ~~Single point of failure~~ | HPA implemented all envs (24 policies) + cluster autoscaler | 8h | ✅ **Done** |
| 4 | **Shared B2C tenant (staging=prod)** | Staging changes affect prod auth | Create dedicated prod B2C tenant or separate staging | 4h | ⬜ Open |
| 5 | **No network policies** | All pods can talk to all pods | Implement deny-all + allow-list policies per service | 3h | ⬜ Open |
| 6 | ~~Terraform state not imported~~ | ~~Can't `terraform apply`~~ | Staging 0 drift, Prod aligned (MDI-184 complete) | 6h | ✅ **Done** |

### 🟡 MEDIUM Priority

| # | Gap | Impact | Solution | Effort |
|---|-----|--------|----------|--------|
| 7 | **No container image scanning** | Vulnerable images may deploy | Add Trivy/Snyk scan step in CI pipeline | 2h |
| 8 | **Dev ingress in wrong namespace** | Routing conflicts possible | Move to `dev` namespace with proper kustomize | 2h |
| 9 | **No automated rollback** | Recovery requires manual intervention | Add health check + auto-rollback in pipeline | 4h |
| 10 | **Missing Staging APIM** | Staging services not accessible via gateway | Already being provisioned (recovery-api-management-staging) | 1h |
| 11 | **No pod resource limits (dev)** | Noisy neighbor problems | Define requests/limits in dev configmaps | 2h |
| 12 | **MongoDB on VM (not managed)** | Patching, backup, HA are manual | Plan migration to Azure CosmosDB for MongoDB | 40h+ |

### 🟢 LOW Priority (Improvements)

| # | Gap | Impact | Solution | Effort |
|---|-----|--------|----------|--------|
| 13 | **No GitOps (ArgoCD/Flux)** | Drift between git and cluster possible | Implement ArgoCD for declarative deployments | 8h |
| 14 | **No service mesh** | No mTLS between services, no traffic policies | Evaluate Istio/Linkerd (may be overkill for current scale) | 16h |
| 15 | **No chaos engineering** | Untested resilience | Add Chaos Mesh for failure injection testing | 8h |
| 16 | **No database migration tooling** | Schema changes are risky | Implement MongoDB migration scripts in CI | 4h |
| 17 | ~~Single replica in prod~~ | ~~Zero fault tolerance~~ | HPA autoscales 1-4 replicas per service | 1h | ✅ **Done** |

---

## 3. Configuration Consistency Audit

### Inter-Service URLs (Claims configmap)

| Config Key | Dev | QA | Staging | Prod | Status |
|------------|-----|-----|---------|------|--------|
| `TenantAPI` | ✅ | ✅ | ✅ | ✅ | Fixed today |
| `Tenant__TenantServiceURL` | ✅ | ✅ | ✅ | ✅ | OK |
| `QuotationServiceURL` | ✅ `api/quotation/` | ✅ `api/quotation/` | ✅ `api/quotation/` | ✅ `api/quotation/` | Fixed today |
| `InvoiceServiceURL` | ✅ | ✅ | ✅ | ✅ | OK |
| `InthubServiceURL` | ✅ | ✅ | ✅ | ✅ | OK |
| `PathPrefix` | ❌ Not set | ✅ `/quotation` | ❌ Not set | ✅ `/quotation` | **GAP** |

### Entity Portal URLs

| Config Key | Dev | QA | Staging | Prod | Status |
|------------|-----|-----|---------|------|--------|
| `quotationsEndPointURL` | ✅ `api/quotation/` | ✅ `api/quotation/` | ✅ `api/quotation/` | ✅ `api/quotation/` | Fixed today |
| `claimsEndPointURL` | ✅ | ✅ | ✅ | ✅ | OK |
| `tenantsEndPointURL` | ✅ | ✅ | ✅ | ✅ | OK |
| `invoiceEndpointURL` | ✅ | ✅ | ✅ | ✅ | OK |

---

## 4. Environment Promotion Path

```
Dev → QA → Staging → Production
 │      │       │         │
 │      │       │         └── Manual approval required
 │      │       └── Demo/UAT environment
 │      └── Automated testing (future)
 └── Developer integration
```

### Current Promotion Process
1. Developer pushes to `dev` branch → auto-deploy to dev cluster
2. Merge/push to `qa` → auto-deploy to QA cluster
3. Push to `staging` → auto-deploy to staging cluster
4. Push to `prod` → auto-deploy to prod cluster (no manual gate!)

### Recommended Promotion Process
1. Push to `dev` → auto-deploy to dev
2. PR from `dev` → `qa` with automated tests → deploy to QA
3. PR from `qa` → `staging` with QA sign-off → deploy to staging
4. PR from `staging` → `prod` with **manual approval gate** + smoke tests → deploy to prod

---

## 5. Cost Optimization Notes

| Environment | Monthly Estimate | Optimization |
|-------------|-----------------|--------------|
| Dev | ~$400 | Could schedule off-hours shutdown |
| QA | ~$500 | 4 nodes seems excessive for testing; reduce to 2 |
| Staging | ~$400 | OK for demo purposes |
| Prod | ~$1,200 | Autoscaling will optimize (scale down when idle) |

> **Quick Win**: QA has 4 nodes with 2 replicas each = 12 pods on 4 nodes. This could run on 2 nodes comfortably. Save ~$200/month.
