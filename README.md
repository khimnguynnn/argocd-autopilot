# Bootstrap Folder - ArgoCD Autopilot Architecture

> Documentation cho GitOps repository: https://github.com/khimnguynnn/argocd-autopilot

## Tổng quan

Bootstrap folder là **trái tim của GitOps system** - nơi ArgoCD tự quản lý chính nó và orchestrate toàn bộ infrastructure thông qua **App of Apps pattern**.

```
bootstrap/
├── argo-cd.yaml              # ArgoCD tự quản lý chính nó
├── argo-cd/
│   └── install.yaml          # Full ArgoCD installation manifests
├── root.yaml                 # App of Apps - quản lý tất cả projects
├── cluster-resources.yaml    # ApplicationSet quản lý clusters
└── cluster-resources/
    ├── in-cluster.json       # Cluster metadata
    └── in-cluster/
        └── argocd-ns.yaml    # Cluster-specific resources
```

---

## Kiến trúc tổng quan

```
┌─────────────────────────────────────────────────────┐
│  Bootstrap Applications (Self-Managing)             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  argo-cd.yaml ──sync──> argo-cd/install.yaml       │
│       │                       │                     │
│       │                       └──> ArgoCD pods      │
│       └──selfHeal──> Keep ArgoCD running           │
│                                                     │
│  root.yaml ──sync──> projects/*.yaml               │
│       │                    │                        │
│       │                    ├──> AppProjects         │
│       │                    └──> ApplicationSets     │
│       └──discover new projects automatically       │
│                                                     │
│  cluster-resources.yaml                             │
│       └──scan──> cluster-resources/*.json          │
│             │                                       │
│             └──> Generate per-cluster Apps         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 1. `argo-cd.yaml` - Self-Managing ArgoCD

### Mục đích
ArgoCD **tự deploy và manage chính nó** từ git. Đây là nền tảng của GitOps - control plane cũng là workload.

### Manifest
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-cd
  namespace: argocd
spec:
  source:
    path: bootstrap/argo-cd        # ← Trỏ đến install.yaml
    repoURL: https://github.com/khimnguynnn/argocd-autopilot.git
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true               # ← ArgoCD tự heal chính nó!
      prune: true
```

### Flow
```
argo-cd.yaml (Application)
    ↓ sync
bootstrap/argo-cd/install.yaml (CRDs + Deployments + Services)
    ↓ apply to cluster
ArgoCD running + self-healing
```

### Hành vi
- **Self-healing**: Nếu ai đó `kubectl delete deployment argocd-server`, ArgoCD tự recreate
- **GitOps upgrade**: Upgrade ArgoCD = commit new version vào `install.yaml`
- **Immutable**: Cluster state = git state, không drift

### Ví dụ: Upgrade ArgoCD
```bash
# Download new version
wget https://raw.githubusercontent.com/argoproj/argo-cd/v2.10.0/manifests/install.yaml \
  -O bootstrap/argo-cd/install.yaml

# Commit
git add bootstrap/argo-cd/install.yaml
git commit -m "Upgrade ArgoCD to v2.10.0"
git push

# ArgoCD tự upgrade chính nó trong vài phút!
```

---

## 2. `root.yaml` - App of Apps Pattern

### Mục đích
Root application là **parent của tất cả projects**. Nó scan `projects/` folder và tự động discover projects mới.

### Manifest
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io  # ← Cascade delete protection
spec:
  source:
    path: projects                  # ← Scan toàn bộ projects/ folder
    repoURL: https://github.com/khimnguynnn/argocd-autopilot.git
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
```

### Flow
```
root.yaml (Application)
    ↓ sync
projects/backend.yaml (AppProject + ApplicationSet)
    ↓ ApplicationSet generates
backend-api (Application)
backend-worker (Application)
```

### Cascade Discovery
```
root.yaml
  ↓ discovers
projects/backend.yaml (ApplicationSet)
  ↓ generates from
apps/**/backend/config.json
  ↓ creates
backend-api, backend-worker, backend-frontend (Applications)
  ↓ deploys
Actual workloads in cluster
```

### Ví dụ: Tạo project mới
```bash
# Tạo project
argocd-autopilot project create payments

# Điều gì xảy ra:
# 1. Tạo file projects/payments.yaml
# 2. root.yaml tự động discover
# 3. ApplicationSet trong payments.yaml scan apps/**/payments/config.json
# 4. Tạo Applications cho mỗi app tìm được

# Không cần kubectl apply gì cả!
```

---

## 3. `cluster-resources.yaml` - Cluster Discovery

### Mục đích
ApplicationSet tự động tạo Application cho **mỗi cluster** được đăng ký. Mỗi cluster có folder riêng chứa cluster-scoped resources.

### Manifest
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-resources
spec:
  generators:
  - git:
      files:
      - path: bootstrap/cluster-resources/*.json  # ← Auto-discover
      repoURL: https://github.com/khimnguynnn/argocd-autopilot.git
      requeueAfterSeconds: 20
  template:
    metadata:
      name: cluster-resources-{{name}}
    spec:
      source:
        path: bootstrap/cluster-resources/{{name}}  # ← Per-cluster folder
      destination:
        server: '{{server}}'
```

### Flow
```
cluster-resources.yaml (ApplicationSet)
    ↓ scan every 20s
bootstrap/cluster-resources/in-cluster.json
bootstrap/cluster-resources/prod-gke.json
    ↓ generates Applications
cluster-resources-in-cluster
cluster-resources-prod-gke
    ↓ sync from folders
bootstrap/cluster-resources/in-cluster/argocd-ns.yaml
bootstrap/cluster-resources/prod-gke/namespaces.yaml
```

### Multi-cluster Pattern
```
bootstrap/cluster-resources/
├── in-cluster.json              # {"name":"in-cluster","server":"https://kubernetes.default.svc"}
├── in-cluster/
│   ├── argocd-ns.yaml          # Namespace cho ArgoCD
│   └── README.md
├── prod-gke.json                # {"name":"prod-gke","server":"https://prod-gke:443"}
├── prod-gke/
│   ├── namespaces.yaml         # Production namespaces
│   └── network-policies.yaml   # NetworkPolicies
└── staging-eks.json             # Staging cluster
    └── staging-eks/
        └── resources.yaml
```

### Ví dụ: Add cluster mới
```bash
# Tạo cluster metadata
cat > bootstrap/cluster-resources/prod-gke.json <<EOF
{
  "name": "prod-gke",
  "server": "https://35.123.45.67:443"
}
EOF

# Tạo cluster resources folder
mkdir -p bootstrap/cluster-resources/prod-gke

cat > bootstrap/cluster-resources/prod-gke/namespaces.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
EOF

# Commit
git add bootstrap/cluster-resources/
git commit -m "Add prod-gke cluster"
git push

# ApplicationSet tự tạo cluster-resources-prod-gke Application!
```

---

## 4. `cluster-resources/in-cluster.json` - Cluster Metadata

### Mục đích
Metadata về cluster để ApplicationSet generate Applications. `in-cluster` là cluster mà ArgoCD đang chạy trong đó.

### Format
```json
{
  "name": "in-cluster",
  "server": "https://kubernetes.default.svc"
}
```

### Các fields
- `name`: Tên cluster (human-readable, unique)
- `server`: Kubernetes API server URL

### Multi-cluster scenario
```json
// in-cluster.json - ArgoCD control plane
{"name":"in-cluster","server":"https://kubernetes.default.svc"}

// prod-us-east.json
{"name":"prod-us-east","server":"https://prod-us-east-k8s.aws.com:443"}

// staging-gcp.json
{"name":"staging-gcp","server":"https://35.123.45.67:443"}
```

---

## 5. `cluster-resources/in-cluster/` - Cluster-Specific Resources

### Mục đích
Chứa **cluster-scoped resources** cho mỗi cluster:
- Namespaces
- CRDs
- ClusterRoles / ClusterRoleBindings
- NetworkPolicies (cluster-wide)
- StorageClasses
- PriorityClasses

### Structure
```
bootstrap/cluster-resources/in-cluster/
├── argocd-ns.yaml          # Namespace cho ArgoCD
├── monitoring-ns.yaml      # Namespace cho Prometheus/Grafana
├── custom-crds.yaml        # Custom Resource Definitions
└── README.md               # Documentation
```

### Ví dụ: `argocd-ns.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
  labels:
    name: argocd
```

### Best practices
- **Cluster-scoped only**: Không đặt Deployments/Services ở đây
- **Per-cluster customization**: Mỗi cluster có config riêng
- **Infrastructure resources**: CRDs, namespaces, base policies

---

## 6. `argo-cd/install.yaml` - ArgoCD Installation

### Mục đích
Full ArgoCD installation manifests bao gồm:
- CRDs (Application, AppProject, ApplicationSet)
- Deployments (argocd-server, argocd-repo-server, argocd-application-controller)
- Services (argocd-server, argocd-metrics)
- ConfigMaps (argocd-cm, argocd-rbac-cm)
- Secrets (argocd-secret)

### Customization options
```yaml
# High Availability
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-server
spec:
  replicas: 3  # ← HA setup

---
# Custom RBAC
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
data:
  policy.csv: |
    p, role:org-admin, applications, *, */*, allow
    g, eng-team, role:org-admin
```

---

## Bootstrap Flow - Step by Step

### 1. Initial Bootstrap
```bash
argocd-autopilot repo bootstrap \
  --repo https://github.com/khimnguynnn/argocd-autopilot.git
```

**Các bước xảy ra:**

```
1. Create git structure
   ├── bootstrap/
   │   ├── argo-cd.yaml
   │   ├── argo-cd/install.yaml
   │   ├── root.yaml
   │   ├── cluster-resources.yaml
   │   └── cluster-resources/in-cluster.json
   ├── projects/ (empty)
   └── apps/ (empty)

2. Apply to cluster
   kubectl apply -f bootstrap/argo-cd/install.yaml
   kubectl apply -f bootstrap/argo-cd.yaml
   kubectl apply -f bootstrap/root.yaml
   kubectl apply -f bootstrap/cluster-resources.yaml

3. Self-healing activates
   - ArgoCD manages itself
   - Root app watches projects/
   - Cluster resources watches *.json
```

### 2. Add Project
```bash
argocd-autopilot project create backend
```

**Flow:**
```
1. Create projects/backend.yaml
   ├── AppProject
   └── ApplicationSet (scan apps/**/backend/config.json)

2. root.yaml discovers projects/backend.yaml

3. ApplicationSet waits for apps
```

### 3. Deploy Application
```bash
argocd-autopilot app create api \
  --app github.com/org/api \
  --project backend
```

**Flow:**
```
1. Create structure:
   apps/api/
   ├── base/kustomization.yaml
   └── overlays/backend/config.json

2. ApplicationSet discovers config.json

3. Generate Application:
   backend-api (reads apps/api/overlays/backend/)

4. Deploy to cluster
```

---

## Key Concepts

### 1. Self-Managing ArgoCD
- ArgoCD vừa là **controller** vừa là **workload**
- Upgrade = git commit, không cần manual intervention
- Self-healing tự động recovery từ failures

### 2. App of Apps Pattern
- Hierarchy: `root → projects → applications`
- Cascade discovery: Add file → auto-create resources
- Declarative management: Git = source of truth

### 3. Cluster Auto-Discovery
- Add `*.json` file → ApplicationSet generates Application
- Scale to hundreds of clusters
- Per-cluster customization trong dedicated folders

### 4. GitOps Principles
- **Git as single source of truth**: Cluster state = git state
- **Automated sync**: Changes auto-deploy
- **Self-healing**: Drift tự động correct
- **Auditability**: Git history = change log

---

## Troubleshooting

### ArgoCD không sync
```bash
# Check application status
kubectl get application -n argocd

# Check logs
kubectl logs -n argocd deploy/argocd-application-controller

# Force sync
argocd app sync argo-cd --force
```

### Application stuck OutOfSync
```bash
# Check diff
argocd app diff <app-name>

# Manual sync với prune
argocd app sync <app-name> --prune
```

### Cluster không được discover
```bash
# Verify JSON file tồn tại
ls bootstrap/cluster-resources/*.json

# Check ApplicationSet
kubectl get applicationset -n argocd cluster-resources -o yaml

# Force ApplicationSet refresh
kubectl annotate applicationset cluster-resources -n argocd \
  argocd.argoproj.io/refresh=true
```

---

## Advanced Patterns

### Multi-environment setup
```
bootstrap/
└── cluster-resources/
    ├── dev-cluster.json
    ├── dev-cluster/
    │   └── dev-namespaces.yaml
    ├── staging-cluster.json
    ├── staging-cluster/
    │   └── staging-namespaces.yaml
    └── prod-cluster.json
        └── prod-cluster/
            ├── prod-namespaces.yaml
            └── prod-network-policies.yaml
```

### Hub-and-spoke (multi-region)
```
bootstrap/
└── cluster-resources/
    ├── us-east-hub.json       # Regional ArgoCD
    ├── us-east-spoke1.json    # Workload clusters
    ├── us-east-spoke2.json
    ├── eu-central-hub.json
    └── eu-central-spoke1.json
```

---

## Security Best Practices

### 1. Least Privilege
```yaml
# AppProject với restricted permissions
spec:
  namespaceResourceWhitelist:
  - group: 'apps'
    kind: Deployment
  clusterResourceDenylist:
  - group: '*'
    kind: '*'
```

### 2. Git Commit Signing
```bash
# Require GPG signatures
git config --global commit.gpgsign true

# ArgoCD verify signatures
argocd-cm:
  repository.verify: "true"
```

### 3. Credential Management
- KHÔNG commit secrets vào git
- Dùng External Secrets Operator
- Hoặc Sealed Secrets

---

## Maintenance

### Backup
```bash
# Backup bootstrap folder
tar -czf bootstrap-backup-$(date +%Y%m%d).tar.gz bootstrap/

# Backup ArgoCD resources
kubectl get applications -n argocd -o yaml > argocd-apps-backup.yaml
```

### Disaster Recovery
```bash
# 1. Re-bootstrap cluster
argocd-autopilot repo bootstrap --repo <git-repo>

# 2. ArgoCD tự restore tất cả từ git
# 3. Verify
kubectl get applications -n argocd
argocd app list
```

---

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD Autopilot](https://argocd-autopilot.readthedocs.io/)
- [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ApplicationSet Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)

---

**Last Updated**: 2026-06-24  
**Repository**: https://github.com/khimnguynnn/argocd-autopilot  
**Maintained by**: @khimnguynnn
