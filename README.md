# DevOps Engineering Portfolio
## Terraform → ArgoCD → Multi-cluster Kubernetes

> **Author:** Samuel Klin  
> **Timeline:** April 2026  
> **Status:** Phases 1–3 Complete · Capstone in progress

A self-directed, hands-on DevOps engineering program built entirely on real AWS infrastructure. No guided labs. No sandboxes. Every resource provisioned, broken, debugged, and destroyed on a live AWS account.

---

## The Stack

| Tool | Purpose |
|------|---------|
| **Terraform** | Infrastructure as Code — VPCs, EKS clusters, S3, IAM |
| **AWS EKS** | Managed Kubernetes on AWS |
| **ArgoCD** | GitOps-native continuous delivery |
| **Kustomize** | Multi-environment Kubernetes config management |
| **Argo Rollouts** | Progressive delivery — canary and blue-green |
| **External Secrets Operator** | Secrets pulled from AWS Secrets Manager at runtime |
| **AWS Secrets Manager** | Secret storage — values never touch Git |
| **GitHub** | Single source of truth for all configs |

---

## Repository Structure

```
gitops-config/                  ← this repo (GitOps config)
├── README.md
└── apps/
    ├── base/                   ← shared Kubernetes manifests
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── namespace.yaml
    │   └── kustomization.yaml
    ├── overlays/
    │   ├── dev/                ← dev: 2 replicas
    │   │   ├── kustomization.yaml
    │   │   └── external-secret.yaml
    │   ├── staging/            ← staging: 3 replicas
    │   │   └── kustomization.yaml
    │   └── prod/               ← prod: 5 replicas
    │       └── kustomization.yaml
    └── rollouts/
        └── workload/           ← Argo Rollouts canary config
            ├── namespace.yaml
            ├── rollout.yaml
            ├── service-stable.yaml
            ├── service-canary.yaml
            └── kustomization.yaml
```

---

## Phase 1 — Terraform Infrastructure as Code

**What was built:** Production-grade AWS infrastructure using reusable Terraform modules.

### Projects

**Project 1 — Modular VPC + S3**
- VPC module: public subnets across 2 AZs, internet gateway, route tables, route table associations
- S3 module: versioning enabled, AES256 encryption, public access blocked
- Root module wiring variables, locals, and outputs
- `terraform state mv` used to refactor live resources into modules with zero downtime

**Project 2 — Remote State Backend**
- S3 bucket with versioning (every state change recoverable)
- DynamoDB table for state locking (prevents concurrent applies)
- State migrated from local to remote — live, without destroying anything

**Project 3 — EKS Cluster**
- IAM roles for control plane (`eks.amazonaws.com`) and nodes (`ec2.amazonaws.com`)
- VPC with public subnets across `us-east-1a` and `us-east-1b`
- EKS control plane + managed node group (2x `t3.medium`)
- kubectl connected — both nodes `STATUS: Ready`, Kubernetes v1.31

**Project 4 — Multi-Environment Setup**
- Same modules, two completely isolated environments
- `dev`: VPC `10.0.0.0/16`, state at `environments/dev/terraform.tfstate`
- `prod`: VPC `10.1.0.0/16`, state at `environments/prod/terraform.tfstate`
- Destroying prod never touches dev

### Key HCL Patterns

```hcl
# for_each over a map — one module call, multiple environments
module "vpc" {
  source   = "./modules/vpc"
  for_each = var.clusters

  name = "${local.name_prefix}-${each.key}-vpc"
  cidr = each.value.vpc_cidr
  azs  = each.value.azs
}

# locals — compute once, use everywhere
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = {
    project    = var.project
    managed_by = "terraform"
  }
}
```

### State Key Structure

```
myapp-terraform-state-109804294707/
├── backend/terraform.tfstate
├── eks/terraform.tfstate
├── phase3/terraform.tfstate
├── environments/dev/terraform.tfstate
└── environments/prod/terraform.tfstate
```

---

## Phase 2 — ArgoCD & GitOps

**What was built:** Full GitOps delivery pipeline on EKS — Git as the single source of truth, ArgoCD as the delivery engine.

### The GitOps mental model

```
Traditional CI/CD (push)          GitOps (pull)
────────────────────────          ─────────────
Pipeline → authenticates      →   ArgoCD watches Git
           to cluster         →   detects diff
           runs kubectl apply  →   applies to its own cluster
           (credentials in CI)     (no credentials in CI)
```

### Operations Proven

| Operation | How | Result |
|-----------|-----|--------|
| Zero-touch deploy | Push YAML to GitHub | 3 pods, service, namespace created — no kubectl |
| Git-driven scaling | Change `replicas: 2` → `3` | Synced in 60 seconds |
| Self-healing | `kubectl scale --replicas=1` | ArgoCD restored to 3 within minutes |
| Rolling update | Change `nginx:1.25` → `1.26` | Zero-downtime rolling update |
| Rollback | Change image back in Git | Instant — no special commands |

### Kustomize Structure

```
apps/base/          ← shared config, written once
apps/overlays/dev/  ← namespace: myapp-dev, replicas: 2
apps/overlays/staging/ ← namespace: myapp-staging, replicas: 3
apps/overlays/prod/ ← namespace: myapp-prod, replicas: 5
```

DRY principle: change the base image once → all three environments update.

### ApplicationSet

```yaml
generators:
  - list:
      elements:
        - env: dev
        - env: staging
        - env: prod
template:
  metadata:
    name: "myapp-{{env}}"
  spec:
    source:
      path: "apps/overlays/{{env}}"
```

One template → three applications. Add `staging` by adding three lines.

### Secrets Management

```
AWS Secrets Manager          ESO              Kubernetes
───────────────────          ───              ──────────
myapp/dev/db-password   →   pulls   →    db-credentials Secret
  password: ***                            (auto-created)
  username: myapp                          (auto-refreshed 1h)
```

`ExternalSecret` resource in Git — no values, just a reference.  
Actual values live exclusively in AWS Secrets Manager.

### Deployment History

```
ID 0  2026-04-05 08:45  cbd7b8b  Initial deployment (nginx:1.25)
ID 1  2026-04-05 08:53  8f2788d  Scaled to 3 replicas
ID 2  2026-04-05 09:05  82e7b25  Upgraded to nginx:1.26
ID 3  2026-04-05 09:11  3834eab  Rolled back to nginx:1.25
```

Every deployment traceable to a Git commit, author, and timestamp.

---

## Phase 3 — Multi-cluster Kubernetes

**What was built:** Hub and spoke multi-cluster platform. ArgoCD on the management cluster driving deployments into a separate workload cluster, with Argo Rollouts canary deployments.

### Architecture

```
Management Cluster (10.0.0.0/16)      Workload Cluster (10.1.0.0/16)
────────────────────────────────       ──────────────────────────────
ArgoCD control plane           ──────► myapp-prod  (5 pods)
  App controller                        myapp-canary (Rollout)
  Repo server                           Argo Rollouts controller
  ApplicationSet controller             canary: 20%→50%→100%
```

### Clusters

| Property | Management | Workload |
|----------|-----------|---------|
| Cluster name | myapp-phase3-management | myapp-phase3-workload |
| Kubernetes | v1.31.14-eks-f69f56f | v1.31.14-eks-f69f56f |
| VPC CIDR | 10.0.0.0/16 | 10.1.0.0/16 |
| Node IPs | 10.0.0.x, 10.0.1.x | 10.1.0.x, 10.1.1.x |
| Instance type | t3.medium x2 | t3.medium x2 |
| What runs here | ArgoCD | myapp, Argo Rollouts |

### Provisioned with one Terraform config

```hcl
module "eks" {
  source   = "./modules/eks"
  for_each = var.clusters        # management + workload in parallel
  cluster_name = "${local.name_prefix}-${each.key}"
  vpc_id       = module.vpc[each.key].vpc_id
  subnet_ids   = values(module.vpc[each.key].public_subnet_ids)
}
```

`30 resources` created across both clusters in 15 minutes.

### Cross-cluster GitOps proven

```powershell
# Register workload cluster in ArgoCD
.\argocd.exe cluster add workload --name workload-cluster

# Deploy to workload cluster
.\argocd.exe app create myapp-workload `
  --dest-server https://4638BC2641EDA49A21C18C1B1B9396D1.gr7.us-east-1.eks.amazonaws.com
```

ArgoCD on management cluster → 5 pods on workload cluster.  
Workload cluster has no direct human kubectl access.

### Canary Rollout Strategy

```yaml
strategy:
  canary:
    steps:
      - setWeight: 20       # 20% traffic to new version
      - pause: {duration: 30s}
      - setWeight: 50       # 50% traffic to new version
      - pause: {duration: 30s}
      - setWeight: 100      # 100% — old pods terminate
    canaryService: myapp-canary
    stableService: myapp-stable
```

Triggered by a single `git push`. Traffic shifts automatically.  
Abort at any step to roll back instantly.

---

## Troubleshooting Hall of Fame

Real errors hit, diagnosed, and resolved during this program.

| Error | Cause | Fix |
|-------|-------|-----|
| `terraform.exe blocked` | Windows Smart App Control | Diagnosed AppLocker vs WDAC vs SAC — resolved natively |
| `state lock ConditionalCheckFailed` | Interrupted apply left stale DynamoDB lock | `terraform force-unlock <lock-id>` |
| `BucketNotEmpty on destroy` | Versioned S3 bucket has version history | Delete all versions via AWS CLI before destroy |
| `yes-dev-cluster` appeared | Typed `yes` as project name at the variable prompt | Delete node group → delete cluster → reapply with correct tfvars |
| `state mv` after module refactor | Renamed resource address broke state | `terraform state mv old_address new_address` before applying |
| Kustomize `../../base not found` | Files created in wrong GitHub folder nesting | Deleted corrupted structure, rebuilt via local git push |
| `ClusterSecretStore CRD not found` | ESO installed with wrong API version | Used `v1` not `v1beta1` after checking `kubectl api-resources` |
| `<workload-cluster-endpoint> not found` | Used placeholder text literally in command | `terraform output cluster_endpoints` to get real URL |
| `CRD annotation too long` | ArgoCD install manifest exceeds K8s annotation limit | Use `--server-side` flag on kubectl apply |

---

## Key Commands

```powershell
# Terraform
terraform init
terraform plan
terraform apply
terraform destroy
terraform state list
terraform state show <resource>
terraform state mv <old> <new>
terraform force-unlock <lock-id>
terraform output

# kubectl multi-cluster
kubectl config get-contexts
kubectl config use-context management
kubectl config use-context workload
kubectl get nodes --context=workload
kubectl get pods -n <ns> --context=workload

# ArgoCD
.\argocd.exe login localhost:8080 --username admin --password <pwd> --insecure
.\argocd.exe cluster list
.\argocd.exe cluster add workload --name workload-cluster
.\argocd.exe repo add <url> --username <u> --password <token>
.\argocd.exe app create <name> --repo <url> --path <path> --dest-server <url>
.\argocd.exe app list
.\argocd.exe app get <name>
.\argocd.exe app sync <name>
.\argocd.exe app history <name>
.\argocd.exe app rollback <name> <id>
.\argocd.exe app delete <name> --yes

# Argo Rollouts
kubectl argo rollouts list rollouts -n myapp-canary --context=workload
kubectl argo rollouts get rollout myapp -n myapp-canary --context=workload --watch
kubectl argo rollouts promote myapp -n myapp-canary --context=workload
kubectl argo rollouts abort myapp -n myapp-canary --context=workload

# AWS CLI
aws sts get-caller-identity
aws eks list-clusters --region us-east-1
aws eks update-kubeconfig --name <cluster> --region us-east-1 --alias <alias>
aws s3 ls
aws secretsmanager get-secret-value --secret-id <name> --region us-east-1
aws dynamodb scan --table-name terraform-state-locks --region us-east-1
```

---

## Rules Learned the Hard Way

```
1.  Never edit terraform.tfstate by hand
2.  Never delete backend resources from the AWS Console
3.  Always let Terraform destroy what Terraform created
4.  Always enable versioning on your state bucket
5.  Run terraform state mv BEFORE renaming resources
6.  Never put secrets in Git — not even base64
7.  Always terraform plan before terraform apply — read every +/~/−
8.  Register the workload cluster BEFORE creating apps targeting it
9.  Keep the port-forward running — closing it kills ArgoCD CLI
10. Always terraform destroy when done — EKS costs ~$4-10/day
11. Two EKS clusters cost double — destroy immediately after practice
12. Use --context= flag every time with multi-cluster kubectl
```

---

## What's Next — Capstone

Build the full platform from scratch in one shot:

```
terraform apply
  → management cluster + workload cluster provisioned

ArgoCD bootstrapped on management cluster
  → workload cluster registered
  → ApplicationSet deployed from GitHub
  → all environments synced automatically

Canary rollout triggered by git push
  → 20% → 50% → 100% traffic shift live

terraform destroy
  → everything cleaned up
```

No step-by-step guide. Just the tools and the skills.

---

## Cost Reference

| Resource | Cost |
|----------|------|
| EKS control plane | $0.10/hour (~$2.40/day) |
| t3.medium node | $0.0416/hour |
| 2x t3.medium nodes | ~$2.00/day |
| Single cluster total | ~$4.40/day |
| Two clusters (Phase 3) | ~$8.80/day |

**Always destroy when done. All infrastructure in this portfolio was destroyed after every session.**

---

*Built with real AWS infrastructure · All resources destroyed after every session · Zero ongoing charges*
