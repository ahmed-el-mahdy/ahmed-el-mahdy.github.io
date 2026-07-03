# Actionable Roadmap — Detailed Implementation Guide

> **Purpose**: Step-by-step instructions for every planned improvement  
> **Format**: Each item includes why, how, commands, validation, and estimated effort  
> **Priority Order**: Critical → High → Medium → Enhancement  
> **Last Updated**: July 3, 2026

---

## ✅ Completed (June-July 2026)

| Item | Status | Date |
|------|--------|------|
| Terraform staging alignment (0 drift) | ✅ Done | Jun 2026 |
| Terraform prod alignment (0 drift) | ✅ Done | Jun 2026 |
| HPA policies (24 total, all envs) | ✅ Done | Jun 2026 |
| Prod AKS upgrade (B2s → D4ds_v5) | ✅ Done | Jun 2026 |
| Prod APIM upgrade (Developer → StandardV2) | ✅ Done | Jun 2026 |
| Cluster autoscaler (QA 3-6, Prod 2-5) | ✅ Done | Jun 2026 |
| Production monitoring (4 alerts + 3 avail tests) | ✅ Done | Jun 2026 |
| VNet redesign (8 subnets, per-subnet NSGs) | ✅ Done | Jun 2026 |
| Performance analysis (MongoDB + Redis audit) | ✅ Done | Jul 2026 |

---

## Phase 1: Critical Fixes (Week 1-2)

### 1.1 Migrate Dev Secrets to Key Vault

**Why**: Dev configmap has Redis passwords, B2C secrets in plaintext committed to git.

**Steps**:
```bash
# 1. Create Key Vault for dev
az keyvault create --name Recovery-keyvault-dev \
  --resource-group recovery-rg \
  --location uaenorth \
  --sku standard

# 2. Add secrets
az keyvault secret set --vault-name Recovery-keyvault-dev \
  --name "ConnectionStrings--DefaultConnection" \
  --value "<mongodb-connection-string>"

az keyvault secret set --vault-name Recovery-keyvault-dev \
  --name "ConnectionStrings--Redis" \
  --value "<redis-connection-string>"

az keyvault secret set --vault-name Recovery-keyvault-dev \
  --name "AzureAdB2C--ClientSecret" \
  --value "<b2c-secret>"

# 3. Enable CSI driver on dev AKS
az aks enable-addons --addons azure-keyvault-secrets-provider \
  --name Recovery-aks-dev \
  --resource-group recovery-rg

# 4. Create SecretProviderClass (copy from QA pattern)
# File: overlays/dev/patches/keyvault-dev.yaml
# Already exists! Just needs correct Key Vault name and secrets list

# 5. Update dev configmap to remove secrets
# Remove: ConnectionStrings__Redis, AzureAdB2C__ClientSecret, etc.

# 6. Apply and verify
kubectl apply -k overlays/dev/ --context Recovery-aks-dev
kubectl get secretproviderclasspodstatus -n dev --context Recovery-aks-dev
```

**Validation**: Pods start successfully, no plaintext secrets in configmap, Key Vault audit shows access.

**Effort**: 4 hours

---

### 1.2 Create Staging Ingress

**Why**: Staging has no ingress file, meaning services can only be accessed by port-forwarding or internal ClusterIP.

**Steps**:
```bash
# 1. Create the file
cat > overlays/staging/ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: staging-unified-ingress
  namespace: staging
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /inthub(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: staging-inthub
                port:
                  number: 80
          - path: /claims(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: staging-claims
                port:
                  number: 80
          - path: /invoice(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: staging-invoice
                port:
                  number: 80
          - path: /quotation(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: staging-quotation
                port:
                  number: 80
          - path: /settlement(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: staging-settlement
                port:
                  number: 80
          - path: /tenant(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: staging-tenant
                port:
                  number: 80
EOF

# 2. Add to kustomization.yaml
# In overlays/staging/kustomization.yaml, add under resources:
#   - ingress.yaml

# 3. Apply
kubectl apply -f overlays/staging/ingress.yaml --context Recovery-aks-staging

# 4. Verify
kubectl get ingress -n staging --context Recovery-aks-staging
```

**Effort**: 1 hour

---

### 1.3 Add PathPrefix to Dev and Staging Quotation

**Why**: QA and Prod have `PathPrefix: "/quotation"` but Dev and Staging don't. This means Swagger URLs are wrong in those environments.

**Steps**:
```bash
# Add to overlays/dev/quotation/dev-configmap-patch.yaml:
#   PathPrefix: "/quotation"

# Add to overlays/staging/quotation/staging-configmap-patch.yaml:
#   PathPrefix: "/quotation"

# Apply and restart
kubectl apply -k overlays/dev/quotation/ --context Recovery-aks-dev -n dev
kubectl rollout restart deployment dev-quotation -n dev --context Recovery-aks-dev
kubectl apply -k overlays/staging/quotation/ --context Recovery-aks-staging -n staging
kubectl rollout restart deployment staging-quotation -n staging --context Recovery-aks-staging
```

**Effort**: 30 minutes

---

### 1.4 Increase Prod Replicas (Critical Services)

**Why**: Claims and Tenant are critical path — single replica = single point of failure.

**Steps**:
```bash
# Edit overlays/prod/claims/prod-patch.yaml
# Change: replicas: 1 → replicas: 2

# Edit overlays/prod/tenant/prod-patch.yaml  
# Change: replicas: 1 → replicas: 2

# Apply
kubectl apply -k overlays/prod/ --context Recovery-aks-prod
kubectl get pods -n prod --context Recovery-aks-prod
```

**Effort**: 30 minutes

---

## Phase 2: Autoscaling (Week 2-3) — MDI-185

### 2.1 Enable Cluster Autoscaler (MDI-186)

```bash
# Enable autoscaling on the prod node pool
az aks nodepool update \
  --resource-group Recovery-rg-prod \
  --cluster-name Recovery-aks-prod \
  --name nodepool1 \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 5

# Verify
az aks nodepool show \
  --resource-group Recovery-rg-prod \
  --cluster-name Recovery-aks-prod \
  --name nodepool1 \
  --query '{autoscaling: enableAutoScaling, min: minCount, max: maxCount}'
```

### 2.2 Configure HPA (MDI-187)

```yaml
# For each service, create HPA:
# File: overlays/prod/claims/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: prod-claims-hpa
  namespace: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: prod-claims
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### 2.3 Define Resource Requests/Limits (MDI-188)

```yaml
# In each prod-patch.yaml, add:
spec:
  template:
    spec:
      containers:
        - name: claims
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
```

**Recommended values** (based on typical .NET 8 service):
| Service | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---------|------------|-----------|----------------|--------------|
| Claims | 250m | 1000m | 256Mi | 512Mi |
| Tenant | 200m | 500m | 256Mi | 512Mi |
| Quotation | 200m | 500m | 256Mi | 512Mi |
| Invoice | 150m | 500m | 256Mi | 512Mi |
| Settlement | 150m | 500m | 256Mi | 512Mi |
| IntHub | 200m | 500m | 256Mi | 512Mi |

---

## Phase 3: Terraform Import (Week 3-4) — MDI-184

### Full Import Command List

```bash
cd d:/Mosadad/terraform/prod

# APIM v2
terraform import module.api_management.azurerm_api_management.apim \
  /subscriptions/86e6cf03-b5d9-4e18-83a0-6d90c266e39f/resourceGroups/Recovery-rg-prod/providers/Microsoft.ApiManagement/service/recovery-apim-v2-prod

# Redis Cache
terraform import module.redis.azurerm_redis_cache.redis \
  /subscriptions/86e6cf03-b5d9-4e18-83a0-6d90c266e39f/resourceGroups/Recovery-rg-prod/providers/Microsoft.Cache/redis/recoveryClaim-redis-cache-prod

# Bastion
terraform import azurerm_bastion_host.bastion \
  /subscriptions/86e6cf03-b5d9-4e18-83a0-6d90c266e39f/resourceGroups/Recovery-rg-prod/providers/Microsoft.Network/bastionHosts/Recovery-bastion-prod

# Grafana
terraform import azurerm_dashboard_grafana.grafana \
  /subscriptions/86e6cf03-b5d9-4e18-83a0-6d90c266e39f/resourceGroups/Recovery-rg-prod/providers/Microsoft.Dashboard/grafana/recovery-grafana-prod

# App Gateway
terraform import module.app_gateway.azurerm_application_gateway.appgw \
  /subscriptions/86e6cf03-b5d9-4e18-83a0-6d90c266e39f/resourceGroups/Recovery-rg-prod/providers/Microsoft.Network/applicationGateways/Recovery-appgw-prod

# Firewall
terraform import azurerm_firewall.firewall \
  /subscriptions/86e6cf03-b5d9-4e18-83a0-6d90c266e39f/resourceGroups/Recovery-rg-prod/providers/Microsoft.Network/azureFirewalls/Recovery-firewall-prod

# Run plan after each import to check alignment
terraform plan -out=plan.tfplan
```

**Critical**: Always run `terraform plan` after importing to check for drift. Fix any differences before `terraform apply`.

---

## Phase 4: Network Policies (Week 4-5)

### Default Deny-All Policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Allow Specific Traffic
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-claims-ingress
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: claims
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: ingress-nginx
      ports:
        - port: 80
```

---

## Phase 5: GitOps with ArgoCD (Week 5-7)

### Install ArgoCD
```bash
kubectl create namespace argocd --context Recovery-aks-prod
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --context Recovery-aks-prod

# Expose via port-forward initially
kubectl port-forward svc/argocd-server -n argocd 8080:443 --context Recovery-aks-prod

# Get initial password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Create Application
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prod-mosadad
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Mosadad/kubernetes.git
    targetRevision: main
    path: overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Phase 6: Container Scanning in CI (Week 3)

### Add to GitHub Actions workflow
```yaml
- name: Scan Docker image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'

- name: Upload Trivy scan results
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'

- name: Fail on critical vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}'
    exit-code: '1'
    severity: 'CRITICAL'
```

---

## Phase 7: Production Approval Gate (Week 2)

### Add Environment Protection Rule
```
GitHub → Settings → Environments → prod → Protection Rules:
  ✅ Required reviewers: Ahmed Almahdi, Team Lead
  ✅ Wait timer: 5 minutes
  ✅ Deployment branches: only `prod` branch
```

### Update workflow to use environment
```yaml
deploy:
  runs-on: ubuntu-latest
  environment: prod  # This triggers the approval gate
  steps:
    - name: Deploy to production
      # ... deployment steps
```

---

## Phase 8: Monitoring Enhancements (Week 4)

### Add PagerDuty/Teams Integration
```bash
# Create Azure Action Group with Teams webhook
az monitor action-group create \
  --name Recovery-Critical-AG \
  --resource-group Recovery-rg-prod \
  --short-name CriticalAG \
  --action webhook "Teams-Webhook" "https://outlook.office.com/webhook/..."
```

### Add SLO Monitoring
```kql
// Claims API SLO (99.9% availability, <500ms p95)
requests
| where timestamp > ago(30d)
| where cloud_RoleName == "claims"
| summarize 
    total = count(),
    successful = countif(success == true),
    p95_duration = percentile(duration, 95)
| extend availability = (successful * 100.0) / total
| project availability, p95_duration
```

---

## Timeline Summary

| Week | Phase | Effort | Impact |
|------|-------|--------|--------|
| 1 | Dev Key Vault + Staging Ingress + PathPrefix | 6h | Security + Accessibility |
| 2 | Prod replicas + Approval gate + Container scanning | 4h | Reliability + Security |
| 2-3 | Autoscaling (HPA + Cluster Autoscaler) | 10h | Scalability |
| 3-4 | Terraform import | 6h | IaC completeness |
| 4-5 | Network policies | 4h | Zero-trust security |
| 5-7 | ArgoCD GitOps | 8h | Drift prevention |
| 8+ | MongoDB migration planning | Planning only | Long-term reliability |

**Total Estimated Effort**: ~38 hours over 7 weeks
