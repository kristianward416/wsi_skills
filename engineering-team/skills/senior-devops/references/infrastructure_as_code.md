# Infrastructure as Code Guide

Step-by-step processes, optimization, and troubleshooting for the Terraform modules `scripts/terraform_scaffolder.py` generates (AWS/GCP/Azure), and decision criteria for when to reach for Pulumi or a cloud-native tool instead.

## 1. Module Design

A module should have exactly one reason to change. The `ecs-service` module this skill scaffolds is a good template for that boundary: it owns the task definition, service, and its security group — nothing about the cluster, VPC, or load balancer, which live in separate modules it references by input variable.

**Recommended module layout:**
```
infra/
├── modules/
│   ├── ecs-service/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   ├── vpc/
│   └── rds-postgres/
├── environments/
│   ├── staging/
│   │   ├── main.tf        # instantiates modules with staging inputs
│   │   └── backend.tf     # staging-specific state backend config
│   └── production/
│       ├── main.tf
│       └── backend.tf
```

**Root module composition** (`environments/production/main.tf`):
```hcl
module "vpc" {
  source = "../../modules/vpc"
  cidr_block = "10.0.0.0/16"
  environment = "production"
}

module "app_service" {
  source              = "../../modules/ecs-service"
  service_name        = "app"
  cluster_id          = module.ecs_cluster.id
  container_image     = "ghcr.io/org/app:1.2.3"
  cpu                 = 512
  memory              = 1024
  desired_count        = 3
  container_port      = 8080
  private_subnet_ids  = module.vpc.private_subnet_ids
  aws_region          = "us-east-1"
  env_vars = {
    NODE_ENV = "production"
  }
}
```

**Design rules:**
- Modules take inputs and return outputs; they never hardcode environment-specific values (region, account ID, CIDR) — those live in the root module per environment.
- Pin module sources by version/tag (`source = "git::https://.../ecs-service?ref=v1.4.0"` for shared internal registries) so a change to `main` doesn't silently ripple into every environment.
- Keep `versions.tf` pinned:
```hcl
terraform {
  required_version = ">= 1.7.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## 2. State Management and Locking

State is the source of truth for what Terraform believes exists. Losing or corrupting it is the single most damaging IaC failure mode.

**Remote backend with locking (AWS — S3 + DynamoDB):**
```hcl
# environments/production/backend.tf
terraform {
  backend "s3" {
    bucket         = "acme-terraform-state"
    key            = "production/app/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```
The DynamoDB table provides distributed locking: a second `terraform apply` against the same state key blocks until the first releases the lock, preventing concurrent writers from corrupting state. As of Terraform 1.10+/OpenTofu, the S3 backend supports native locking via S3 conditional writes, removing the DynamoDB dependency — check your pinned version before assuming it's available.

**GCS backend (GCP):**
```hcl
terraform {
  backend "gcs" {
    bucket = "acme-terraform-state"
    prefix = "production/app"
  }
}
```

**Azure Blob backend:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "acmetfstate"
    container_name       = "tfstate"
    key                  = "production/app.tfstate"
  }
}
```

**State isolation rules:**
- One state file per environment, minimum. Never share a single state file between staging and production — a `terraform destroy` mistake in one blast-radius shouldn't be able to touch the other.
- Split further by blast radius for large systems: network/VPC state separate from application state, so an app-level `apply` can't touch VPC resources it has no `resource` blocks for anyway, but more importantly so plan times and lock contention stay manageable.
- Never edit state by hand. Use `terraform state mv`, `terraform import`, or `terraform state rm` for surgical fixes.

**Recovering from a stuck lock** (e.g., CI job killed mid-apply):
```bash
terraform force-unlock <LOCK_ID>
```
Only do this after confirming no other apply is actually in flight — check the CI run status first, not just the lock age.

## 3. Drift Detection

Drift is any divergence between real infrastructure and what state says exists — caused by manual console changes, another tool touching the same resources, or provider-side auto-healing.

**Manual drift check:**
```bash
terraform plan -detailed-exitcode
# exit code 0 = no changes, 1 = error, 2 = changes present (drift or pending changes)
```

**Automated drift detection in CI (scheduled, not on every push):**
```yaml
# .github/workflows/drift-detect.yml
name: Terraform Drift Detection
on:
  schedule:
    - cron: '0 6 * * *'   # daily at 06:00 UTC
jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform -chdir=infra/environments/production init
      - run: |
          terraform -chdir=infra/environments/production plan -detailed-exitcode -out=tfplan || \
          if [ $? -eq 2 ]; then
            echo "::warning::Drift detected in production"
            terraform -chdir=infra/environments/production show tfplan
          fi
```

**Reducing drift at the source:**
- Restrict console/CLI write access to the resources Terraform manages; use IAM policies that deny manual changes to tagged resources (`ManagedBy=terraform`) outside the CI role.
- For resources that legitimately need manual tuning (e.g., autoscaling adjusting `desired_count`), use `ignore_changes` explicitly rather than letting drift alerts fire on expected behavior:
```hcl
resource "aws_ecs_service" "app" {
  # ...
  lifecycle {
    ignore_changes = [desired_count]
  }
}
```
- When `scripts/terraform_scaffolder.py --verbose` runs `terraform plan`, treat any unexpected diff as a signal to investigate before merging — don't reflexively `apply` to "fix" a plan you don't understand.

## 4. Pulumi and CDK Tradeoffs

| Factor | Terraform/OpenTofu | Pulumi | AWS CDK |
|---|---|---|---|
| Language | HCL (DSL) | TypeScript, Python, Go, C# | TypeScript, Python, Go, Java, C# |
| Testing | `terraform test` (newer, limited) or Terratest (Go) | Native unit tests in the host language | Native unit tests (`cdk-assertions`) |
| State | Self-managed backend (S3/GCS/Blob + lock) | Pulumi Service (managed) or self-hosted backend | CloudFormation-managed (AWS-hosted) |
| Cloud coverage | 3,000+ providers, all major clouds equally | Same provider ecosystem as Terraform (built on it) | AWS only |
| Team fit | Ops-heavy teams, infra specialists | Dev-heavy teams that already write TS/Python and resist learning HCL | AWS-only shops wanting deep L2/L3 construct abstractions |
| Drift detection | Mature (`plan -detailed-exitcode`) | Mature (`pulumi refresh`) | Via CloudFormation drift detection |

**Decision criteria:**
- Default to Terraform/OpenTofu (per this skill's cloud-agnostic stance) unless there's a specific reason not to.
- Choose **Pulumi** when the team is primarily application developers who will maintain the infra code themselves and strongly prefer writing real loops/conditionals/functions over HCL's more limited expression language, while still needing multi-cloud parity with Terraform.
- Choose **CDK** only for AWS-only shops that want the higher-level L2/L3 constructs (e.g., `aws-ecs-patterns.ApplicationLoadBalancedFargateService` scaffolding a whole ALB+ECS+target-group stack in ~10 lines) and are willing to accept CloudFormation as the execution engine, with its slower rollback behavior on complex stacks.
- Do not run Terraform and Pulumi/CDK managing the *same* resources — pick one IaC tool per resource, ever, to avoid conflicting ownership.

## 5. HCL Patterns Worth Standardizing

**Variables with validation (fail plan, not apply, on bad input):**
```hcl
variable "environment" {
  type = string
  validation {
    condition     = contains(["staging", "production"], var.environment)
    error_message = "environment must be staging or production."
  }
}
```

**Outputs for cross-module wiring:**
```hcl
output "service_arn" {
  value       = aws_ecs_service.app.id
  description = "ARN of the ECS service, consumed by monitoring module."
}
```

**for_each over count for anything you'll later need to address by name** (count-indexed resources shift indices painfully on removal from the middle of a list):
```hcl
resource "aws_ecs_service" "app" {
  for_each = var.services   # map(object), not a list
  name     = each.key
  # ...
}
```

## 6. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Error acquiring the state lock` | Prior apply crashed without releasing lock, or a concurrent run | Confirm no run in flight, then `terraform force-unlock <ID>` |
| Plan shows unexpected recreation of a resource | Immutable attribute changed (e.g., ECS task `family`, RDS `identifier`) | Check provider docs for `ForceNew`/immutable fields; use `create_before_destroy` lifecycle if zero-downtime replacement is needed |
| `plan` differs between local and CI | Different provider version or un-pinned `required_providers` | Pin exact provider versions and commit `.terraform.lock.hcl` |
| Resource exists in cloud, not in state | Created out-of-band (console, another pipeline) | `terraform import` it, or codify then import, then investigate why the console was used |
| State file grows huge, plans get slow | Monolithic state covering too many resources/environments | Split state by environment/blast-radius (see §2) |

Once infrastructure is provisioned and validated (`terraform plan` clean, drift check green), hand off to the deployment tooling described in `deployment_strategies.md` for how application releases roll onto that infrastructure.
