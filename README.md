# Portfolio Workflow

## 1. **Overall Architecture**

1. **GitHub Actions**
   - Builds and pushes your application Docker image to ECR using **OIDC** (no static credentials).
   - Runs Terraform “locally” (in CI) with a **S3 + DynamoDB** backend for remote state.
   - Controls plan/apply steps for each environment (dev, staging, prod), passing in `.tfvars` files for environment-specific variables.

2. **Amazon EKS** + **Argo CD**
   - EKS hosts your Kubernetes cluster. Argo CD is installed in the cluster (provisioned via Terraform).
   - Argo CD provides **GitOps**: it continuously pulls from your `k8s/<env>` folder and deploys any changes.

3. **AWS Parameter Store + External Secrets Operator**
   - Parameter Store stores sensitive values as SecureStrings.
   - External Secrets Operator pulls them into Kubernetes Secrets at runtime.
   - Non-sensitive config (feature flags, environment toggles, etc.) go into ConfigMaps or your Kustomize overlays.

4. **Remote State with S3 + DynamoDB**
   - Terraform state is stored in an S3 bucket with a DynamoDB table for locking.
   - GitHub Actions uses an **OIDC** IAM role to authenticate and read/write from S3 + DynamoDB, ensuring concurrency control and versioned state.

5. **`.tfvars` for Environment Overrides**
   - Each environment folder (dev, staging, prod) has a `.tfvars` file specifying environment-specific variables (like instance sizes, CIDR blocks, replica counts).
   - GitHub Actions includes `-var-file="<env>.tfvars"` in the plan/apply commands.

> **Disclaimer**: In a larger or more complex org, you might use **Terraform Cloud** or **Spacelift** to manage state, apply policy checks, or enforce RBAC. Here we use **S3+Dynamo** for simplicity and cost-effectiveness, which is very common in AWS shops—and well-suited for a **portfolio**.

---

## 2. **Repository Structure**

A recommended **mono-repo** layout:

```
my-portfolio-repo/
├── infra/
│   ├── modules/
│   │   ├── vpc/
│   │   │   ├── main.tf
│   │   │   └── variables.tf
│   │   ├── eks/
│   │   │   ├── main.tf
│   │   │   └── variables.tf
│   │   ├── argocd/
│   │   │   ├── main.tf
│   │   │   └── variables.tf
│   │   └── my_app_infra/
│   │       ├── main.tf
│   │       └── variables.tf
│   └── envs/
│       ├── dev/
│       │   ├── main.tf       # Instantiates modules for dev
│       │   ├── backend.tf    # S3+DDB remote state config
│       │   ├── variables.tf  # Variable declarations
│       │   └── dev.tfvars    # Dev-specific values
│       ├── staging/
│       │   ├── main.tf
│       │   ├── backend.tf
│       │   ├── variables.tf
│       │   └── staging.tfvars
│       └── prod/
│           ├── main.tf
│           ├── backend.tf
│           ├── variables.tf
│           └── prod.tfvars
├── k8s/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── dev-overlay.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── staging-overlay.yaml
│   └── prod/
│       ├── kustomization.yaml
│       └── prod-overlay.yaml
├── app/
│   ├── Dockerfile
│   └── ... (Rails/Node/etc. code)
├── .github/
│   └── workflows/
│       ├── build-app.yml
│       └── terraform.yml
└── README.md
```

### Explanation

1. **`infra/modules/*`**
   - Each major infrastructure component is in its own module (VPC, EKS, Argo CD, app-specific resources).
2. **`infra/envs/<env>`**
   - `main.tf` instantiates these modules.
   - `backend.tf` configures remote state in S3+Dynamo.
   - **`<env>.tfvars`** holds environment-specific values (e.g. instance sizes).
3. **`k8s/<env>`**
   - Kustomize overlays for environment-specific manifests (like scaling, configmaps).
4. **`.github/workflows`**
   - **`build-app.yml`**: CI pipeline building/pushing Docker images to ECR (OIDC-based).
   - **`terraform.yml`**: Runs Terraform plan/apply for each environment folder, referencing `.tfvars`.

> **In real production**: You might separate out each environment (or microservice) into its own repo/workspace for stricter ownership boundaries. But for a **portfolio**, it’s easier to keep them in one place.

---

## 3. **AWS Authentication: GitHub Actions OIDC**

- Define an IAM role (e.g., `GitHubOIDC-TerraformRole`) in AWS that trusts GitHub’s OIDC provider.
- In `terraform.yml`, use [`aws-actions/configure-aws-credentials@v2`](https://github.com/aws-actions/configure-aws-credentials) to assume that role at runtime, eliminating long-lived credentials.

**IAM Policy Requirements**:

- **S3**: `GetObject`, `PutObject`, `DeleteObject`, `ListBucket` for remote state.
- **DynamoDB**: `GetItem`, `PutItem`, `DeleteItem`, `Query` for lock table operations.
- **Any other AWS resources** relevant for your Terraform code (e.g., ECR push, VPC creation).

---

## 4. **Terraform Workflow**

### 4.1 **S3 + DynamoDB Setup**

In `backend.tf` for each environment:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-remote-state"
    key            = "envs/dev/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "my-terraform-lock-table"
    encrypt        = true
  }
}
```

### 4.2 **Environment’s `main.tf` + `.tfvars`**

```hcl
# main.tf
module "vpc" {
  source      = "../../modules/vpc"
  environment = "dev"
  # Dev-specific settings can be passed as variables
}

module "eks" {
  source      = "../../modules/eks"
  vpc_id      = module.vpc.vpc_id
  environment = "dev"
  # ...
}

module "argocd" {
  source           = "../../modules/argocd"
  cluster_endpoint = module.eks.cluster_endpoint
  environment      = "dev"
}

module "my_app_infra" {
  source           = "../../modules/my_app_infra"
  cluster_endpoint = module.eks.cluster_endpoint
  environment      = "dev"
  # ...
}
```

**Variables** might be declared in `variables.tf`:

```hcl
variable "instance_type" {
  type = string
}
variable "replica_count" {
  type    = number
  default = 1
}
```

**`dev.tfvars`** could then look like:

```hcl
instance_type  = "t3.small"
replica_count  = 1
```

### 4.3 **GitHub Actions** (`terraform.yml`)

Below shows a *matrix* example passing the appropriate `.tfvars` to each environment’s folder:

```yaml
name: Terraform CI
on:
  push:
    paths:
      - 'infra/envs/dev/**'
      - 'infra/envs/staging/**'
      - 'infra/envs/prod/**'
  workflow_dispatch:

jobs:
  terraform:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]

    steps:
      - name: Check out
        uses: actions/checkout@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubOIDC-TerraformRole
          aws-region: us-east-1

      - name: Set up Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Terraform Init
        run: terraform init
        working-directory: infra/envs/${{ matrix.environment }}

      - name: Terraform Plan
        run: terraform plan -input=false -var-file="${{ matrix.environment }}.tfvars"
        working-directory: infra/envs/${{ matrix.environment }}

      - name: Terraform Apply
        # example gating condition for merges to main
        if: github.ref == 'refs/heads/main'
        run: terraform apply -input=false -auto-approve -var-file="${{ matrix.environment }}.tfvars"
        working-directory: infra/envs/${{ matrix.environment }}
```

**Key Points**:
1. **Matrix** runs dev, staging, prod in parallel (or sequentially).
2. We pass `-var-file="<env>.tfvars"` to override environment-specific values.
3. If you prefer separate workflows for dev/staging/prod, that’s also valid.

---

## 5. **Application Build & Deploy**

1. **`build-app.yml`**
   - Triggers on commits to `app/`.
   - Uses OIDC to assume a role with ECR push permissions.
   - Builds + pushes Docker images.

2. **Argo CD**
   - Installed via `argocd` module in Terraform.
   - Watches `k8s/<env>` for changes.
   - Deploys or updates your Deployment, Service, etc. whenever you commit to the `k8s/<env>` folder.
   - Automated image updates are possible with an add-on like Argo CD Image Updater or by manually updating image tags in the Kustomize overlay.

### **Parameter Store & External Secrets**

- Store **sensitive** config in Parameter Store (SecureString).
- **External Secrets Operator** fetches them and creates a K8s `Secret`.
- **Non-sensitive** env vars in ConfigMaps or Kustomize overlays.

---

## 6. **Scaling to Production / Larger Systems**

> **Disclaimer**: This portfolio structure merges everything in a single repo for clarity. In a **true production** environment:

1. **Terraform Cloud / Spacelift**
   - Instead of GH Actions running Terraform, teams might store state in TFC or Spacelift for features like policy enforcement, RBAC, a web UI for runs, etc.

2. **Multiple Repos or Workspaces**
   - “Platform” (VPC, EKS, Argo) might be in one repo/workspace, while each application (and environment) is in another.
   - Minimizes risk of a dev misconfig affecting production.

3. **Enhanced Governance**
   - Strict approvals and gating steps. Possibly separate AWS accounts for dev vs. prod.

4. **More Modules & Versioning**
   - Large teams often keep a registry of versioned modules so all microservices are consistent.

For **small teams or a personal portfolio**, the single repo + environment subfolders + `.tfvars` approach is straightforward, cost-effective, and still illustrates strong DevOps/GitOps foundations.
