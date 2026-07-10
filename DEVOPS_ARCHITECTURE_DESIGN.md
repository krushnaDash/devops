# CPM Platform — DevOps Architecture Design
### Enterprise-Grade CI/CD & Infrastructure Strategy

> **Version:** 1.0  
> **Status:** Draft for Review  
> **Author:** DevOps Engineering  
> **Diagrams:** See `/diagrams/` folder — open `.drawio` files in [draw.io / diagrams.net](https://app.diagrams.net)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Principles](#2-architecture-principles)
3. [Technology Stack](#3-technology-stack)
4. [Application-Specific CI/CD Pipeline Design](#4-application-specific-cicd-pipeline-design)
5. [Cloud-Agnostic Framework](#5-cloud-agnostic-framework)
6. [Reusable Terraform Modules](#6-reusable-terraform-modules)
7. [Naming Conventions](#7-naming-conventions)
8. [Terraform State Management](#8-terraform-state-management)
9. [Environment Separation Strategy](#9-environment-separation-strategy)
10. [Container Image Versioning & Promotion](#10-container-image-versioning--promotion)
11. [Kubernetes Deployment Architecture](#11-kubernetes-deployment-architecture)
12. [Security, Compliance & Quality Gates](#12-security-compliance--quality-gates)
13. [GitOps Strategy](#13-gitops-strategy)
14. [Release Promotion Strategy](#14-release-promotion-strategy)
15. [Monitoring & Observability](#15-monitoring--observability)
16. [Repository & Directory Structure](#16-repository--directory-structure)
17. [Diagram References](#17-diagram-references)

---

## 1. Executive Summary

This document defines the end-to-end DevOps architecture for the CPM platform. It addresses all requirements raised, including application-specific CI/CD pipelines, cloud-agnostic infrastructure, reusable Terraform modules, standardised naming, isolated Terraform state, Kubernetes architecture decisions, GitOps deployment governance, and a multi-environment release promotion strategy.

The architecture is designed for **high scalability**, **environment isolation**, **security-by-default**, and **cloud portability** — enabling a future migration to GCP or Azure with minimal friction.

### Key Goals Addressed

| # | Requirement | Solution |
|---|-------------|----------|
| 1 | App-specific pipelines (PEH, NED, Kafka, Topics, SQS, S3) | Dedicated GitHub Actions / Jenkins pipelines per component |
| 2 | Cloud-agnostic pipeline | Abstract Terraform provider layer + parameterised CI/CD |
| 3 | Reusable infra code | Modular Terraform library (`terraform-modules/`) |
| 4 | Pipeline naming convention | `CPM-<APP>-<ENV>-CI` / `CPM-<APP>-<ENV>-CD` |
| 5 | Cloud resource naming convention | `cpm-<component>-<env>-<resource>[-suffix]` |
| 6 | TF state per component/env, no sharing | S3 backend with isolated key paths + DynamoDB locking |
| 7 | Separate workspace/dir per environment | `environments/<env>/` directories, isolated workspaces |
| 8 | Always use latest image on deploy | Image tag pinned to git SHA; CD always pulls from ECR digest |
| 9 | Per cluster / per pod design | Dedicated EKS clusters for STAGE/PERF/PROD; namespaces per app |
| 10 | CI/CD design diagram | See `diagrams/cicd-pipeline.drawio` |

---

## 2. Architecture Principles

These principles govern every design decision in this document:

1. **GitOps-First** — All deployments are driven by Git state. No manual `kubectl apply` or ad-hoc changes in production.
2. **Infrastructure as Code** — Every cloud resource is defined in Terraform. No click-ops.
3. **Immutable Artifacts** — Container images are built once, tagged with the git SHA, and promoted through environments without modification.
4. **Environment Isolation** — Each environment (DEV, INT, STAGE, PERF, PROD) has its own dedicated VPC, Terraform state file, and independent configuration.
5. **Least Privilege** — IAM roles, Kubernetes RBAC, and network policies are scoped to the minimum required permissions.
6. **Shift Left Security** — Security scanning (SAST, container scanning, dependency audits) occurs in the CI pipeline before any deployment.
7. **Cloud Portability** — Terraform modules abstract provider-specific resources. Replacing `aws` with `google` or `azurerm` providers should require only variable changes, not structural rewrites.
8. **Observability by Default** — Every deployed service emits metrics, logs, and traces from day one.

---

## 3. Technology Stack

| Layer | Tool / Service | Purpose |
|-------|---------------|---------|
| Source Control | GitHub / GitLab | App repos, GitOps config repo, Terraform IaC repo |
| CI Engine | GitHub Actions / Jenkins | Build, test, scan, package |
| CD / GitOps | ArgoCD | Kubernetes deployment reconciliation |
| IaC Deployment | Terraform + Atlantis | Infrastructure provisioning & drift detection |
| Container Registry | Amazon ECR | Immutable image storage, lifecycle policies, scan on push |
| Kubernetes | Amazon EKS | Container orchestration |
| Messaging | Amazon MSK (Kafka) | Streaming events (Kafka, Topics) |
| Queuing | Amazon SQS | Decoupled message queuing |
| Notifications | Amazon SNS | Pub/sub topics |
| Object Storage | Amazon S3 | Data lake, artifacts, static assets |
| Serverless | AWS Lambda | Event-driven compute, glue functions |
| Secrets | AWS Secrets Manager / HashiCorp Vault | Secrets injection at runtime |
| Security Scanning | SonarQube, Trivy, Checkmarx | SAST, container CVE scanning |
| Monitoring | Prometheus + Grafana, CloudWatch | Metrics, dashboards, alerting |
| Tracing | AWS X-Ray / Jaeger | Distributed tracing |
| Policy Engine | OPA / Kyverno | Kubernetes admission control |
| Load Testing | k6 / Locust | Performance environment gate |

---

## 4. Application-Specific CI/CD Pipeline Design

Each deployable component has **its own dedicated pipeline pair** — a CI pipeline and a CD pipeline. This avoids cross-component blast radius and allows independent release cadence.

### 4.1 Pipeline Naming Convention

```
CI:  CPM-<APP>-<ENV>-CI
CD:  CPM-<APP>-<ENV>-CD
```

**Examples:**

| Component | CI Pipeline Name | CD Pipeline Name |
|-----------|-----------------|-----------------|
| PEH App | `CPM-PEH-DEV-CI` | `CPM-PEH-DEV-CD` |
| NED App | `CPM-NED-INT-CI` | `CPM-NED-INT-CD` |
| Kafka / MSK | `CPM-KAFKA-DEV-CI` | `CPM-KAFKA-DEV-CD` |
| SNS Topics | `CPM-TOPICS-PROD-CI` | `CPM-TOPICS-PROD-CD` |
| SQS Queues | `CPM-SQS-STG-CI` | `CPM-SQS-STG-CD` |
| S3 Buckets | `CPM-S3-PROD-CI` | `CPM-S3-PROD-CD` |
| Lambda | `CPM-LAMBDA-PRF-CI` | `CPM-LAMBDA-PRF-CD` |

> **Note:** For infrastructure-only components (Kafka, SQS, S3, Topics), the CI pipeline validates Terraform plan and the CD pipeline applies it.

---

### 4.2 CI Pipeline Stages (Application Services: PEH, NED, Lambda)

```
┌─────────────────────────────────────────────────────────────────┐
│  CPM-<APP>-<ENV>-CI                                              │
├──────────┬──────────┬──────────┬──────────┬──────────┬──────────┤
│  Source  │  Build   │   Test   │ Security │ Package  │  Push    │
│  Checkout│ Maven /  │ Unit +   │ SAST +   │ Docker   │  ECR     │
│  + Lint  │ Gradle / │ Coverage │ Trivy +  │  Build   │  Tag:    │
│          │  npm     │  Gate    │ SonarQube│          │  SHA+#   │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

**Stage Details:**

1. **Source Checkout** — Clone repo, validate branch policy, enforce PR review before merge to main.
2. **Build** — Compile application using the appropriate build tool (Maven for Java, Gradle, npm for Node.js).
3. **Unit Tests + Coverage Gate** — Execute unit tests. Fail pipeline if coverage < 80%.
4. **SAST + Container Scan** — Run SonarQube for code quality/security. Run Trivy/Checkmarx on the Dockerfile and dependencies. Fail pipeline on Critical or High CVEs.
5. **Docker Build** — Build container image using a multi-stage Dockerfile. Minimal final image (distroless or alpine).
6. **Push to ECR** — Push image tagged as `<app>:<git-sha>-<build-number>`. Also tag `<app>:latest` (mutable, for reference only; deployments always use the SHA tag).

**CI Pipeline YAML skeleton (GitHub Actions):**
```yaml
# .github/workflows/CPM-PEH-DEV-CI.yml
name: CPM-PEH-DEV-CI
on:
  push:
    branches: [main]
    paths: ['apps/peh/**']
env:
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com
  IMAGE_NAME: cpm-peh-ecr
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: mvn clean package -DskipTests
      - name: Unit Tests
        run: mvn test
      - name: SonarQube Scan
        run: mvn sonar:sonar
      - name: Trivy Scan
        uses: aquasecurity/trivy-action@master
        with:
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
      - name: Build & Push Image
        run: |
          docker build -t $ECR_REGISTRY/$IMAGE_NAME:${{ github.sha }}-${{ github.run_number }} .
          docker push $ECR_REGISTRY/$IMAGE_NAME:${{ github.sha }}-${{ github.run_number }}
      - name: Update GitOps Repo
        run: |
          # Updates helm values in the GitOps repo with the new image tag
          scripts/update-image-tag.sh $IMAGE_NAME ${{ github.sha }}-${{ github.run_number }}
```

---

### 4.3 CD Pipeline Stages (Application Services)

```
┌──────────────────────────────────────────────────────────────────┐
│  CPM-<APP>-<ENV>-CD  (triggered by GitOps repo update)           │
├──────────┬──────────┬──────────┬──────────┬──────────────────────┤
│  ArgoCD  │  Deploy  │  Health  │  Smoke   │  Notify /            │
│  Sync    │  to NS   │  Check   │  Tests   │  Rollback            │
│  Trigger │  (Helm)  │  Gate    │          │  on Failure          │
└──────────┴──────────┴──────────┴──────────┴──────────────────────┘
```

**ArgoCD Application manifest snippet:**
```yaml
# gitops-repo/apps/peh/dev/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cpm-peh-dev
  namespace: argocd
spec:
  project: cpm-dev
  source:
    repoURL: https://github.com/org/cpm-gitops-config.git
    targetRevision: HEAD
    path: apps/peh/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: cpm-peh-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

### 4.4 CI/CD Pipeline for Infrastructure Components (Kafka, SQS, S3, Topics, Lambda)

Infrastructure components use **Terraform-based CI/CD pipelines** — no container image is built.

```
┌─────────────────────────────────────────────────────────────────┐
│  CPM-KAFKA-<ENV>-CI  (Terraform Validation Pipeline)             │
├──────────┬──────────┬──────────┬──────────┬──────────┬──────────┤
│  Checkout│ tf init  │ tf fmt   │ tf       │ tf plan  │ Cost     │
│          │          │ + lint   │ validate │  (saved) │ Estimate │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘

┌─────────────────────────────────────────────────────────────────┐
│  CPM-KAFKA-<ENV>-CD  (Terraform Apply Pipeline)                  │
├──────────┬──────────┬──────────┬──────────┬──────────┬──────────┤
│  Checkout│ tf init  │  Apply   │ Verify   │ Tag      │  Notify  │
│          │          │ (saved   │ Resources│ Resources│          │
│          │          │  plan)   │          │          │          │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

**Pipeline YAML skeleton (Terraform infra):**
```yaml
# .github/workflows/CPM-KAFKA-DEV-CI.yml
name: CPM-KAFKA-DEV-CI
on:
  push:
    branches: [main]
    paths: ['environments/dev/kafka/**']
jobs:
  terraform-plan:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: environments/dev/kafka
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init -backend-config="key=cpm-kafka-dev/terraform.tfstate"
      - run: terraform fmt -check
      - run: terraform validate
      - run: terraform plan -out=tfplan
      - uses: actions/upload-artifact@v4
        with:
          name: tfplan-kafka-dev
          path: environments/dev/kafka/tfplan
```

---

## 5. Cloud-Agnostic Framework

The framework is designed so that switching from AWS to GCP or Azure requires **provider variable changes only**, not structural rewrites.

### 5.1 Abstract Provider Layer

```hcl
# terraform-modules/providers/main.tf
variable "cloud_provider" {
  type    = string
  default = "aws"
  # Options: "aws" | "gcp" | "azure"
}

# Each environment's root module selects provider:
# environments/dev/terraform.tfvars
cloud_provider = "aws"
aws_region     = "us-east-1"
```

### 5.2 Resource Abstraction Pattern

Each reusable module exposes a cloud-neutral interface. The module internally selects the cloud-specific implementation:

```
terraform-modules/
  messaging/
    main.tf          ← switches on cloud_provider variable
    aws/
      msk.tf         ← Amazon MSK
    gcp/
      pubsub.tf      ← Google Pub/Sub
    azure/
      servicebus.tf  ← Azure Service Bus
  storage/
    main.tf
    aws/
      s3.tf
    gcp/
      gcs.tf
    azure/
      blob.tf
```

### 5.3 CI/CD Cloud Portability

CI/CD pipelines use **environment variables** for cloud-specific credentials and registry URLs. Switching providers requires updating GitHub Secrets / Jenkins credentials — the pipeline YAML structure stays identical.

```yaml
env:
  CLOUD_PROVIDER: ${{ vars.CLOUD_PROVIDER }}   # "aws" | "gcp" | "azure"
  REGISTRY_URL: ${{ secrets.REGISTRY_URL }}    # ECR URL, GCR URL, or ACR URL
  K8S_CLUSTER: ${{ secrets.K8S_CLUSTER_NAME }} # EKS, GKE, or AKS cluster name
```

---

## 6. Reusable Terraform Modules

All infrastructure is defined as **reusable Terraform modules** housed in a dedicated `terraform-modules` repository (or top-level `modules/` directory). Environments consume these modules with environment-specific variables.

### 6.1 Module Library Structure

```
terraform-modules/
├── README.md
├── networking/
│   ├── vpc/             ← VPC, subnets, route tables, NAT gateway
│   └── security-groups/ ← Standard SG definitions
├── compute/
│   ├── eks/             ← EKS cluster, node groups, OIDC, add-ons
│   └── lambda/          ← Lambda function + IAM role
├── messaging/
│   ├── msk/             ← Amazon MSK (Kafka) cluster
│   ├── sqs/             ← SQS queue + DLQ + policy
│   └── sns/             ← SNS topic + subscriptions
├── storage/
│   └── s3/              ← S3 bucket + lifecycle + versioning + encryption
├── registry/
│   └── ecr/             ← ECR repository + lifecycle policy + scan-on-push
├── security/
│   ├── iam/             ← IAM roles, policies, instance profiles
│   ├── secrets/         ← Secrets Manager secrets
│   └── kms/             ← KMS keys + aliases
├── observability/
│   ├── cloudwatch/      ← Log groups, dashboards, alarms
│   └── prometheus/      ← Prometheus + Grafana on EKS
└── state-backend/
    └── bootstrap/       ← S3 bucket + DynamoDB for Terraform state (one-time)
```

### 6.2 Module Interface Example (EKS)

```hcl
# terraform-modules/compute/eks/variables.tf
variable "cluster_name"     { type = string }    # e.g., cpm-dev-eks
variable "environment"      { type = string }    # dev | int | stg | prf | prd
variable "vpc_id"           { type = string }
variable "subnet_ids"       { type = list(string) }
variable "node_instance_type" { type = string; default = "t3.medium" }
variable "min_nodes"        { type = number; default = 2 }
variable "max_nodes"        { type = number; default = 10 }
variable "desired_nodes"    { type = number; default = 2 }
variable "tags"             { type = map(string); default = {} }
```

### 6.3 Module Consumption (Environment)

```hcl
# environments/dev/eks/main.tf
module "eks" {
  source = "../../../terraform-modules/compute/eks"

  cluster_name      = "cpm-dev-eks"
  environment       = "dev"
  vpc_id            = module.vpc.vpc_id
  subnet_ids        = module.vpc.private_subnet_ids
  node_instance_type = "t3.medium"
  min_nodes         = 2
  max_nodes         = 6
  desired_nodes     = 2

  tags = {
    Project     = "CPM"
    Environment = "dev"
    ManagedBy   = "Terraform"
    Pipeline    = "CPM-EKS-DEV-CD"
  }
}
```

---

## 7. Naming Conventions

### 7.1 Pipeline Naming

**Format:** `CPM-<COMPONENT>-<ENV>-<CI|CD>`

| Token | Values |
|-------|--------|
| `CPM` | Fixed prefix — identifies all CPM platform pipelines |
| `<COMPONENT>` | `PEH`, `NED`, `KAFKA`, `SQS`, `TOPICS`, `S3`, `LAMBDA`, `EKS`, `VPC` |
| `<ENV>` | `DEV`, `INT`, `STG`, `PRF`, `PRD` |
| `<CI\|CD>` | `CI` for build/validate pipelines; `CD` for deploy/apply pipelines |

**Examples:**
```
CPM-PEH-PRD-CI       CPM-PEH-PRD-CD
CPM-NED-INT-CI       CPM-NED-INT-CD
CPM-KAFKA-STG-CI     CPM-KAFKA-STG-CD
CPM-SQS-DEV-CI       CPM-SQS-DEV-CD
CPM-S3-PRF-CI        CPM-S3-PRF-CD
CPM-LAMBDA-PRD-CI    CPM-LAMBDA-PRD-CD
```

---

### 7.2 Cloud Resource Naming

**Format:** `cpm-<component>-<env>-<resource-type>[-suffix]`

All names are **lowercase, hyphen-separated**, starting with `cpm-`.

| Resource Type | Format | Example |
|--------------|--------|---------|
| VPC | `cpm-<env>-vpc` | `cpm-dev-vpc` |
| EKS Cluster | `cpm-<env>-eks` | `cpm-prd-eks` |
| K8s Namespace | `cpm-<app>-<env>` | `cpm-peh-prd` |
| ECR Repository | `cpm-<app>-ecr` | `cpm-peh-ecr` |
| MSK Cluster | `cpm-<env>-kafka` | `cpm-prd-kafka` |
| Kafka Topic | `cpm-<env>-<topic-name>-topic` | `cpm-prd-order-events-topic` |
| SQS Queue | `cpm-<env>-<queue-name>-queue` | `cpm-dev-payment-queue` |
| SQS DLQ | `cpm-<env>-<queue-name>-dlq` | `cpm-dev-payment-dlq` |
| SNS Topic | `cpm-<env>-<topic-name>-sns` | `cpm-prd-alerts-sns` |
| S3 Bucket | `cpm-<env>-<purpose>-<account-id>` | `cpm-dev-artifacts-123456789` |
| Lambda | `cpm-<env>-<function-name>` | `cpm-prd-invoice-processor` |
| IAM Role | `cpm-<env>-<service>-role` | `cpm-prd-peh-role` |
| Secrets | `cpm-<env>/<app>/<secret-name>` | `cpm-prd/peh/db-password` |
| TF State Bucket | `cpm-tf-state-<account-id>` | `cpm-tf-state-123456789` |
| TF Lock Table | `cpm-tf-lock-<env>` | `cpm-tf-lock-prd` |

> **Migration note:** Existing resources should be tagged with `cpm-` tag immediately and renamed during the next infrastructure refresh cycle. Use the tag `Name = "cpm-..."` as an interim step to maintain inventory.

---

## 8. Terraform State Management

### 8.1 Core Principle

> **One S3 key per component, per environment. No shared state files.**

This prevents blast radius when one component's state becomes corrupted or locked, and allows teams to work on different components simultaneously.

### 8.2 State Backend Configuration

```hcl
# environments/dev/kafka/backend.tf
terraform {
  backend "s3" {
    bucket         = "cpm-tf-state-123456789"
    key            = "dev/kafka/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "cpm-tf-lock-dev"
    encrypt        = true
    kms_key_id     = "alias/cpm-tf-state-key"
  }
}
```

### 8.3 State Key Structure

```
cpm-tf-state-123456789 (S3 Bucket)
├── dev/
│   ├── vpc/terraform.tfstate
│   ├── eks/terraform.tfstate
│   ├── kafka/terraform.tfstate
│   ├── sqs/terraform.tfstate
│   ├── sns/terraform.tfstate
│   ├── s3/terraform.tfstate
│   └── lambda/terraform.tfstate
├── int/
│   └── ... (same structure)
├── stg/
│   └── ...
├── prf/
│   └── ...
└── prd/
    └── ...
```

### 8.4 State Bootstrap (One-Time Setup)

```hcl
# terraform-modules/state-backend/bootstrap/main.tf
resource "aws_s3_bucket" "tf_state" {
  bucket = "cpm-tf-state-${data.aws_caller_identity.current.account_id}"

  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_dynamodb_table" "tf_lock" {
  for_each     = toset(["dev", "int", "stg", "prf", "prd"])
  name         = "cpm-tf-lock-${each.key}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

---

## 9. Environment Separation Strategy

### 9.1 Environment Topology

| Environment | Purpose | EKS Cluster | AWS Account |
|-------------|---------|------------|-------------|
| **DEV** | Continuous integration, developer testing | Shared | Dev Account |
| **INT** | Integration testing, contract validation | Shared | Dev Account |
| **STAGE** | Pre-prod, regression, UAT | Dedicated | Non-Prod Account |
| **PERF** | Load testing, capacity planning | Dedicated | Non-Prod Account |
| **PROD** | Live production traffic | Dedicated HA | Prod Account |

> **Recommendation:** Use **separate AWS accounts** for PROD vs non-PROD environments (AWS Organizations). This is the strongest isolation boundary — IAM errors in dev cannot affect prod.

### 9.2 Directory Structure for Environment Separation

```
environments/
├── dev/
│   ├── backend.tf          ← points to dev state
│   ├── terraform.tfvars    ← dev-specific variable values
│   ├── vpc/
│   ├── eks/
│   ├── kafka/
│   ├── sqs/
│   ├── sns/
│   ├── s3/
│   └── lambda/
├── int/
│   └── ... (same structure, different values)
├── stg/
│   └── ...
├── prf/
│   └── ...
└── prd/
    ├── terraform.tfvars    ← prod-specific (larger instances, HA, multi-AZ)
    └── ...
```

### 9.3 Variable Differentiation Example

```hcl
# environments/dev/terraform.tfvars
environment          = "dev"
eks_node_count       = 2
eks_instance_type    = "t3.medium"
kafka_broker_count   = 1
kafka_instance_type  = "kafka.t3.small"
multi_az             = false
enable_waf           = false

# environments/prd/terraform.tfvars
environment          = "prd"
eks_node_count       = 5
eks_instance_type    = "m5.2xlarge"
kafka_broker_count   = 3
kafka_instance_type  = "kafka.m5.large"
multi_az             = true
enable_waf           = true
enable_shield        = true
```

---

## 10. Container Image Versioning & Promotion

### 10.1 Tagging Strategy

Images are tagged with a **compound tag** that is deterministic and traceable:

```
<ECR_URL>/<image-name>:<git-sha>-<build-number>
```

**Example:**
```
123456789.dkr.ecr.us-east-1.amazonaws.com/cpm-peh-ecr:a1b2c3d4-42
```

- `git-sha` — first 8 characters of the commit SHA. Uniquely identifies the code version.
- `build-number` — monotonically increasing CI run number. Ensures uniqueness if the same commit is rebuilt.
- **`latest` tag** — maintained on the most recent build for developer convenience but **never used in deployments**.
- **Immutable tags** — ECR image tag immutability is enabled. Once pushed, a tag cannot be overwritten.

### 10.2 Image Promotion (Same Artifact, No Rebuild)

The same image built in CI is promoted through all environments. The CD pipeline **pulls the same image digest** — it does not rebuild.

```
DEV deploy  →  uses  cpm-peh-ecr:a1b2c3d4-42
INT deploy  →  uses  cpm-peh-ecr:a1b2c3d4-42  (promoted, same digest)
STAGE deploy →  uses  cpm-peh-ecr:a1b2c3d4-42
PERF deploy  →  uses  cpm-peh-ecr:a1b2c3d4-42
PROD deploy  →  uses  cpm-peh-ecr:a1b2c3d4-42  ← Same tested artifact
```

Only the **configuration** (environment variables, secrets, replica counts) differs between environments — not the image.

### 10.3 Always-Latest Enforcement

The CD pipeline enforces the use of the latest promoted image for a given environment by:
1. Reading the image tag from the **GitOps config repo** (the single source of truth).
2. ArgoCD reconciles the live cluster state against this tag automatically.
3. If a pod is restarted, Kubernetes always pulls the exact digest pinned in the manifest (`imagePullPolicy: Always` + pinned SHA tag).

---

## 11. Kubernetes Deployment Architecture

### 11.1 Cluster Strategy Decision

| Approach | Pros | Cons | Verdict |
|---------|------|------|---------|
| Shared cluster (all envs) | Low cost | No blast-radius isolation; noisy neighbour | ❌ Not for prod |
| Shared cluster (dev+int) | Cost-efficient for lower envs | Acceptable risk for non-prod | ✅ DEV + INT |
| Dedicated per environment | Full isolation, independent upgrades | Higher cost | ✅ STAGE, PERF, PROD |

**Recommendation:**
- **DEV + INT** → Shared EKS cluster (`cpm-dev-eks`), isolated by **namespaces**.
- **STAGE, PERF, PROD** → Dedicated EKS clusters (`cpm-stg-eks`, `cpm-prf-eks`, `cpm-prd-eks`).

### 11.2 Namespace Design

Each application gets its own namespace per environment:

```
Namespace: cpm-<app>-<env>

Examples:
  cpm-peh-dev      cpm-peh-prd
  cpm-ned-dev      cpm-ned-prd
  cpm-lambda-dev   cpm-lambda-prd
```

Kubernetes **NetworkPolicies** restrict cross-namespace traffic. Each namespace has:
- Dedicated `ServiceAccount` with IRSA (IAM Role for Service Accounts)
- `ResourceQuota` to prevent noisy-neighbour issues
- `LimitRange` to set default CPU/memory limits

### 11.3 Node Group Architecture (PROD)

```
cpm-prd-eks
├── Node Group: cpm-prd-app-ng       ← PEH, NED application pods
│   Instance: m5.2xlarge · Min:3 · Max:20
│   AZs: us-east-1a, us-east-1b, us-east-1c
├── Node Group: cpm-prd-system-ng    ← System pods (ArgoCD, Prometheus, etc.)
│   Instance: t3.large · Min:2 · Max:4
└── Node Group: cpm-prd-spot-ng      ← Batch/non-critical workloads (cost savings)
    Instance: m5.xlarge SPOT · Min:0 · Max:10
```

### 11.4 Scaling Strategy

| Mechanism | Scope | Trigger |
|-----------|-------|---------|
| **HPA** (Horizontal Pod Autoscaler) | Pod level | CPU > 70% or custom metric |
| **KEDA** (Event-driven Autoscaling) | Pod level | SQS depth, Kafka consumer lag |
| **Cluster Autoscaler** | Node level | Pending pods due to resource pressure |
| **VPA** (Vertical Pod Autoscaler) | Pod level | Recommendation mode (not auto in prod) |

### 11.5 Pod Design Principles

- All pods are **stateless** — state lives in managed services (MSK, SQS, S3, RDS).
- Pods include **liveness and readiness probes**.
- `PodDisruptionBudget` defined for all production deployments (min 50% available).
- Pods use **distroless or alpine** base images to minimise attack surface.

---

## 12. Security, Compliance & Quality Gates

### 12.1 Security Gates in CI (Shift Left)

| Gate | Tool | Failure Threshold | Stage |
|------|------|------------------|-------|
| Static Code Analysis | SonarQube | Quality Gate Fail | Post-Build |
| SAST | Checkmarx / Semgrep | Critical/High | Post-Build |
| Dependency Audit | OWASP Dependency Check | Critical CVE | Post-Build |
| Container Scan | Trivy | Critical/High CVE | Post-Docker Build |
| IaC Scan | Checkov / tfsec | Critical findings | Terraform Plan |
| Secret Detection | Gitleaks / TruffleHog | Any secret found | Pre-Commit + CI |

### 12.2 Kubernetes Security

- **OPA Gatekeeper / Kyverno** — Admission webhook policies:
  - No `latest` image tag in manifests
  - All images must come from `cpm-*-ecr` registry
  - No containers running as root (`runAsNonRoot: true`)
  - Resource limits must be defined
- **Network Policies** — Default deny-all; only explicitly allowed routes pass.
- **IRSA (IAM Roles for Service Accounts)** — No static credentials in pods. Each workload gets a fine-grained IAM role.
- **Secrets Management** — Secrets injected at runtime via AWS Secrets Manager + External Secrets Operator. No secrets in environment variables or config maps.
- **mTLS** — Service-to-service communication encrypted via AWS App Mesh or Istio (optional, for PROD).

### 12.3 Infrastructure Security

- All S3 buckets: server-side encryption (SSE-KMS), public access blocked, versioning enabled.
- All data at rest: encrypted with customer-managed KMS keys.
- All data in transit: TLS 1.2+ enforced.
- VPC: private subnets for all workloads; NAT gateway for outbound; no direct internet ingress.
- AWS GuardDuty + Security Hub enabled in all environments.
- CloudTrail enabled for all API calls (90-day retention minimum).

### 12.4 Compliance

- **PROD deployments require manual approval gate** in the CD pipeline.
- All pipeline runs are logged with full audit trail.
- Deployment artefacts (images, Terraform plans) are signed and verified.

---

## 13. GitOps Strategy

### 13.1 Repository Layout

Three separate Git repositories:

```
1. cpm-app-<name>         ← Application source code (one per service)
2. cpm-gitops-config      ← Kubernetes manifests / Helm values (GitOps truth)
3. cpm-terraform-iac      ← Terraform modules + environment configs
```

### 13.2 GitOps Workflow

```
Developer pushes code
       │
       ▼
cpm-app-peh (PR → merge to main)
       │
       ▼
CPM-PEH-<ENV>-CI (GitHub Actions)
  ├── Build + Test + Scan + Docker Build
  └── Push image: cpm-peh-ecr:a1b2c3d4-42
       │
       ▼
Update cpm-gitops-config (automated PR)
  └── apps/peh/dev/values.yaml:
        image.tag: a1b2c3d4-42
       │
       ▼
ArgoCD detects change (poll / webhook)
       │
       ▼
ArgoCD syncs cpm-peh-dev namespace
       │
       └── Health Check → Smoke Tests → Promote to INT (or wait for gate)
```

### 13.3 ArgoCD App-of-Apps Pattern

A root ArgoCD Application manages all child Applications:

```yaml
# gitops-repo/apps/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cpm-root
spec:
  source:
    path: apps/
  destination:
    namespace: argocd
```

This ensures all environments and applications are consistently managed as code.

### 13.4 Terraform GitOps (Atlantis)

For infrastructure changes, **Atlantis** provides a GitOps workflow:

1. Engineer raises PR in `cpm-terraform-iac`
2. Atlantis posts `terraform plan` output on the PR
3. Reviewer approves the plan
4. `atlantis apply` is run — pipeline applies the plan, updates state
5. PR is merged only after successful apply

---

## 14. Release Promotion Strategy

### 14.1 Promotion Flow

```
DEV  ──►  INT  ──►  STAGE  ──►  PERF  ──►  PROD
 │         │          │           │           │
Auto     Auto      Manual      Manual      Manual
         gate       gate        gate       approval
```

- **DEV → INT**: Automated after all DEV smoke tests pass.
- **INT → STAGE**: Automated after all integration tests pass.
- **STAGE → PERF**: Manual gate — QA sign-off required.
- **PERF → PROD**: Manual gate — load test results reviewed, management approval.
- **PROD deployment**: Blue/Green or Canary strategy; auto-rollback on failed health checks.

### 14.2 Deployment Strategies

| Environment | Strategy | Rollback |
|-------------|----------|----------|
| DEV / INT | Rolling Update | Automatic (ArgoCD rollback) |
| STAGE | Rolling Update | Automatic |
| PERF | Rolling Update | Automatic |
| PROD | Blue/Green | Automatic (DNS switch back in < 2 min) |

### 14.3 Rollback Procedure

1. **Automatic** — ArgoCD detects health check failure and reverts to last good Git commit.
2. **Manual** — Engineer runs `argocd app rollback cpm-peh-prd` or reverts the image tag in GitOps repo.
3. **Emergency** — Terraform `destroy` + redeploy from last known-good Terraform state version.

---

## 15. Monitoring & Observability

### 15.1 Metrics

- **Prometheus** scrapes all pod metrics (via ServiceMonitor CRDs).
- **Grafana** dashboards per application and per environment.
- **CloudWatch Metrics** for AWS managed services (MSK, SQS, Lambda, etc.).
- **Key SLIs monitored**: p99 latency, error rate, throughput, Kafka consumer lag, SQS queue depth.

### 15.2 Logging

- Application logs → **Fluent Bit** → **CloudWatch Logs** (structured JSON).
- Log groups named: `/cpm/<env>/<app>` (e.g., `/cpm/prd/peh`).
- **Log retention**: DEV 7 days · INT/STAGE 30 days · PROD 90 days (with archive to S3 Glacier).

### 15.3 Tracing

- Applications instrumented with **AWS X-Ray SDK** or **OpenTelemetry**.
- Distributed traces visualised in X-Ray console or Jaeger.
- Enables identification of latency bottlenecks across Kafka, SQS, Lambda, and pod-to-pod calls.

### 15.4 Alerting

- **PagerDuty / OpsGenie** integration for PROD alerts.
- Alerting rules defined as Prometheus Alert Rules stored in GitOps repo (treated as code).
- Critical alerts: pod crash loops, SQS DLQ messages > 0, Kafka consumer lag > threshold, error rate > 1%.

### 15.5 Deployment Monitoring

- ArgoCD notifies Slack/Teams on every sync event (success and failure).
- Post-deployment smoke test results posted to PR comments.
- **Deployment frequency**, **lead time**, **MTTR**, and **change failure rate** tracked as DORA metrics.

---

## 16. Repository & Directory Structure

```
cpm-terraform-iac/
├── terraform-modules/           ← Reusable module library
│   ├── networking/vpc/
│   ├── compute/eks/
│   ├── compute/lambda/
│   ├── messaging/msk/
│   ├── messaging/sqs/
│   ├── messaging/sns/
│   ├── storage/s3/
│   ├── registry/ecr/
│   ├── security/iam/
│   ├── security/secrets/
│   ├── observability/cloudwatch/
│   └── state-backend/bootstrap/
├── environments/
│   ├── dev/
│   │   ├── backend.tf
│   │   ├── terraform.tfvars
│   │   ├── vpc/
│   │   ├── eks/
│   │   ├── kafka/
│   │   ├── sqs/
│   │   ├── sns/
│   │   ├── s3/
│   │   └── lambda/
│   ├── int/  (same structure)
│   ├── stg/  (same structure)
│   ├── prf/  (same structure)
│   └── prd/  (same structure, HA values)
└── .github/workflows/
    ├── CPM-VPC-DEV-CI.yml
    ├── CPM-VPC-DEV-CD.yml
    ├── CPM-EKS-DEV-CI.yml
    ├── CPM-KAFKA-DEV-CI.yml
    └── ... (one pair per component per env)

cpm-gitops-config/
├── apps/
│   ├── peh/
│   │   ├── dev/   ← ArgoCD Application + Helm values
│   │   ├── int/
│   │   ├── stg/
│   │   ├── prf/
│   │   └── prd/
│   ├── ned/
│   │   └── ... (same structure)
│   └── root-app.yaml
├── charts/                      ← Shared Helm chart templates
│   ├── microservice/
│   └── lambda-function/
└── .github/workflows/
    └── validate.yml             ← Validate Helm templates on PR
```

---

## 17. Diagram References

> All diagrams are in the `/diagrams/` folder. Open `.drawio` files in [draw.io](https://app.diagrams.net) (File → Open from → Device) or using the VS Code draw.io extension.

| Diagram | File | Description |
|---------|------|-------------|
| **Deployment Architecture View** | `diagrams/deployment-view.drawio` | Full deployment topology: source control, CI/CD, registry, GitOps, all 5 environments (DEV→PROD), AWS services, monitoring, Terraform state backend |
| **CI/CD Pipeline Flow** | `diagrams/cicd-pipeline.drawio` | End-to-end pipeline: code commit → build → test → security → image push → GitOps → environment promotion → monitoring → rollback |
| **Terraform Module Structure** | `diagrams/terraform-modules.drawio` | IaC module hierarchy, state file isolation per component/env, and environment directory structure |

---

## Appendix A — Quick Reference Cheat Sheet

```
Pipeline naming:   CPM-<APP>-<ENV>-<CI|CD>
Resource naming:   cpm-<component>-<env>-<resource-type>[-suffix]

Environments:      DEV → INT → STAGE → PERF → PROD
                   (Shared EKS)    (Dedicated EKS clusters)

Image tag:         <git-sha-8>-<build-number>
                   e.g., a1b2c3d4-42

TF State key:      <env>/<component>/terraform.tfstate
                   e.g., prd/kafka/terraform.tfstate

Namespace format:  cpm-<app>-<env>
                   e.g., cpm-peh-prd
```

---

## Appendix B — Decision Log

| Decision | Rationale | Date |
|---------|-----------|------|
| Shared EKS for DEV+INT, dedicated for STAGE+ | Cost vs isolation trade-off; lower envs tolerate shared risk | 2026-07 |
| Image immutability (no rebuild on promotion) | Guarantees the exact tested artifact reaches PROD | 2026-07 |
| Separate TF state per component+env | Prevents cascading failures; enables parallel team work | 2026-07 |
| ArgoCD over Flux | Superior UI, multi-cluster support, RBAC granularity | 2026-07 |
| Blue/Green for PROD only | Doubles resource cost; justified only for PROD risk | 2026-07 |
| Atlantis for Terraform GitOps | Native PR workflow, no separate UI needed | 2026-07 |
