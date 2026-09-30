---
name: terraform-skill
description: Design, review, troubleshoot, test, and refactor Terraform for AWS using self-contained guidance on modules, code patterns, native tests, security, and operational commands.
---

# Terraform for AWS

## Scope and operating rules

Use the Terraform CLI for AWS infrastructure. Follow the organization's Terraform Cloud/Enterprise and GitLab workflows where applicable. This file contains the core workflow and five detailed sections. Read only the sections relevant to the task; no companion Markdown files are required.

Do not fetch, clone, or update a public repository to use this skill. Do not install plugins, MCP servers, or third-party tooling. Use approved available tools and report checks that could not run. Terraform initialization, planning, and deployment may still require approved dependencies and cloud access. Attribution is not a runtime dependency.

Apply these rules throughout this skill:

- Confirm the Terraform CLI version and provider versions independently. Distinguish TFE release versions from Terraform CLI versions.
- Use a local provider schema from an already initialized, trusted working directory (`terraform providers schema -json`) or approved local documentation. Do not require Terraform MCP. Do not initialize or download dependencies merely to read this skill.
- Native tests can create infrastructure. Set `command = plan` explicitly for plan-only tests, or use supported mocks. Use real apply tests only in an authorized test environment. Sets are unordered even after apply; use comprehensions or `one()` only when singleton cardinality is guaranteed. Set type alone does not require apply; unknown values do. A computed attribute can sometimes be known in a plan; inspect actual values and schema.
- Do not treat reading a secret from Secrets Manager through a Terraform data source as keeping it out of state. Prefer application/runtime retrieval where required. Use exact provider-supported write-only arguments and supported ephemeral features; Terraform has no universal resource argument named `write_only`.
- Configure remote backend encryption, access, versioning, logging, and locking deliberately; these protections are not guaranteed merely by choosing a remote backend. Split state according to ownership, lifecycle, and impact, not a mandatory numeric resource threshold.
- Treat security and architecture examples as recommendations subject to supplied organization requirements. Public ingress can be intentional for approved public services; scope it to the required ports and architecture.
- In TFE remote execution, review and apply the same TFE run through the established workflow. Do not download a remote plan and attempt a separate local apply. GitLab may validate and orchestrate TFE runs; identify the actual execution owner first. A speculative run cannot be applied.
- Use AWS account/region boundaries, trust paths, DNS, firewall flows, observability, patching, backup/DR, and ownership when converting an HLD to an LLD or preparing a runbook. Reference centrally owned resources unless modification is in scope.
- Preserve existing authorization and organization policies. Preparing code does not authorize production deployment. Treat the embedded apply, destroy, force-unlock, import, migration, and state commands as examples requiring the correct target, review, and applicable authorization.
- Use local text search and file reads for navigation. Follow references within this file; do not assume companion Markdown files exist.
- Run validation with approved tools directly. Do not configure remote pre-commit hooks or install tooling automatically.

## Contents

- [Core Workflow](#core-workflow)
- [Module Design](#section-module-patterns)
- [Terraform Code Patterns](#section-code-patterns)
- [Testing and Validation](#section-testing-frameworks)
- [AWS Security and Compliance](#section-security-compliance)
- [Command Reference and Troubleshooting](#section-quick-reference)
- [Attribution and License](#attribution-license)

<a id="core-workflow"></a>
## Core Workflow

<a id="core-workflow-response-contract"></a>
### Response Contract

Every Terraform response must include:

1. **Assumptions & version floor** — runtime (`terraform`), exact version, providers, state backend, execution path, and environment criticality. State assumptions explicitly if the user did not provide them.
2. **Risk category addressed** — one or more of: identity churn, secret exposure, blast radius, CI drift, compliance gaps, state corruption, provider upgrade risk, testing blind spots.
3. **Chosen remediation & tradeoffs** — what was chosen, what was traded off, why.
4. **Validation plan** — exact commands (`fmt -check`, `validate`, `plan -out`) and documented AWS security review tailored to runtime and risk tier.
5. **Rollback notes** — for any destructive or state-mutating change: how to undo, what evidence to keep.

Never recommend direct production apply without a reviewed plan artifact and approval.

Never run `terraform destroy` (targeted or full) without first running `terraform plan -destroy` and showing the user every resource that will be deleted — including implicit dependents pulled in via locals or `for_each`. Get explicit confirmation before proceeding. Never use `-auto-approve` on any Terraform command.

<a id="core-workflow-workflow"></a>
### Workflow

1. **Capture execution context** — runtime+version, provider(s), backend, execution path, environment criticality.
2. **Diagnose failure mode(s)** using the routing table below. Read the relevant sections in this file.
3. **Read only the matching embedded sections** — do not preload depth the task does not need.
4. **Propose fix with risk controls** — why this addresses the mode, what could still go wrong, guardrails (tests/approvals/rollback).
5. **Generate artifacts** — HCL, migration blocks (`moved`, `import`), requested CI changes.
6. **Validate before finalizing** — run validation commands tailored to risk tier.
7. **Emit the Response Contract** at the end.

<a id="core-workflow-diagnose-before-you-generate"></a>
### Diagnose Before You Generate

| Failure category | Symptoms | Primary references |
|------------------|----------|--------------------|
| **Identity churn** | Resource addresses shift after refactor, `count` index churn, missing `moved` blocks | [Code Patterns: count vs for_each](#section-code-patterns), [Code Patterns: moved blocks](#section-code-patterns), [Code Patterns: LLM mistakes](#section-code-patterns) |
| **Secret exposure** | Secrets in defaults, state, logs, CI artifacts | [Security & Compliance](#section-security-compliance), [Code Patterns: write-only](#section-code-patterns) |
| **Blast radius** | Oversized stacks, shared prod/non-prod state, unsafe applies | [Module Patterns](#section-module-patterns) |
| **Destroy cascade** | Targeted destroy deletes more than expected; locals referencing a targeted resource make all `for_each` consumers implicit dependents | Response Contract: plan-destroy first; review the complete destroy plan and obtain explicit authorization |
| **Testing blind spots** | Plan-only validation of computed values, set-type indexing, mock/real confusion | [Testing Frameworks](#section-testing-frameworks) |
| **Provider upgrade risk** | Breaking-change provider bump, unpinned modules | [Code Patterns: versions](#section-code-patterns), [Module Patterns](#section-module-patterns) |
| **Bootstrap / orchestration misuse** | `null_resource` + `local-exec` for bootstrap, `remote-exec` for setup scripts, provisioner stdout leaking secrets in CI logs | [Code Patterns: Provisioners as Last Resort](#section-code-patterns) |

<a id="core-workflow-when-to-use-this-skill"></a>
### When to Use This Skill

**Activate when:** creating or reviewing Terraform configurations or modules, setting up or debugging tests, structuring multi-environment deployments, implementing IaC CI/CD, choosing module patterns or state organization, configuring or migrating remote state backends.

**Don't use for:** basic HCL syntax questions that do not require this workflow, provider API reference (link to docs), cloud-platform questions unrelated to Terraform.

<a id="core-workflow-core-principles"></a>
### Core Principles

<a id="core-workflow-module-hierarchy"></a>
#### Module Hierarchy

| Type | When to Use | Scope |
|------|-------------|-------|
| **Resource module** | Single logical group of connected resources | VPC + subnets, SG + rules |
| **Infrastructure module** | Collection of resource modules for a purpose | Multiple resource modules in one region/account |
| **Composition** | Complete infrastructure | Spans multiple regions/accounts |

Flow: resource → resource module → infrastructure module → composition.

<a id="core-workflow-directory-layout"></a>
#### Directory Layout

```
environments/   # prod/ staging/ dev/  — per-env configurations
modules/        # networking/ compute/ data/ — reusable modules
examples/       # minimal/ complete/ — docs + integration fixtures
```

Separate **environments** from **modules**. Use `examples/` as both documentation and test fixtures. Keep modules small and single-responsibility.

See [Module Patterns](#section-module-patterns) for architecture principles, naming conventions, variable/output contracts.

<a id="core-workflow-naming-conventions-summary"></a>
#### Naming Conventions (summary)

- Descriptive resource names (`aws_instance.web_server`, not `aws_instance.main`)
- Reserve `this` for genuine singleton resources only
- Prefix variables with context (`vpc_cidr_block`, not `cidr`)
- Standard files: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`

See [Module Patterns: Variable Naming](#section-module-patterns) and [Code Patterns: Block Ordering](#section-code-patterns) for examples.

<a id="core-workflow-block-ordering-summary"></a>
#### Block Ordering (summary)

Resource blocks: `count`/`for_each` first → arguments → `tags` → `depends_on` → `lifecycle`.
Variable blocks: `description` → `type` → `default` → `validation` → `nullable` → `sensitive`.

See [Code Patterns: Block Ordering & Structure](#section-code-patterns) for the full rules and examples.

<a id="core-workflow-testing-strategy"></a>
### Testing Strategy

<a id="core-workflow-decision-matrix-which-testing-approach"></a>
#### Decision Matrix: Which Testing Approach?

| Situation | Approach | Tools | Cost |
|-----------|----------|-------|------|
| Quick syntax check | Static analysis | `validate`, `fmt` | Free |
| Pre-commit validation | Static + lint | `validate`, `tflint` | Free |
| Terraform 1.6+, simple logic | Native test framework | `terraform test` | Free-Low |
| Terraform before 1.6 | Static checks and reviewed plans | `fmt`, `validate`, `plan` | Depends on existing infrastructure |
| Security/compliance focus | Configuration and plan review | Supplied AWS standards and review evidence | Reviewer time |
| Cost-sensitive workflow | Mock providers (1.7+) | Native tests + mocks | Free |
| Complex AWS integration | Full integration | `terraform test` + real infra | Med-High |

<a id="core-workflow-native-test-rules-16"></a>
#### Native Test Rules (1.6+)

Before writing test code, inspect available provider schemas or approved local documentation so assertions target real attributes. No Terraform MCP server is required.

- `command = plan` — fast, for input-derived values only
- `command = apply` — use when assertions need values unknown during planning; real-provider apply tests can create AWS resources
- Set-type blocks cannot be indexed with `[0]` — use `for` expressions, or `one()` for a known singleton
- Common set types: S3 encryption rules, lifecycle transitions, IAM policy statements

See [Testing Frameworks](#section-testing-frameworks) for static-analysis pipelines, native-test patterns, mock providers, and the full LLM-mistake checklist.

<a id="core-workflow-count-vs-for_each--quick-rule"></a>
### Count vs For_Each — Quick Rule

| Scenario | Use | Why |
|----------|-----|-----|
| Boolean condition (create / don't) | `count = condition ? 1 : 0` | Optional singleton toggle |
| Items may be reordered or removed | `for_each = toset(list)` | Stable resource addresses |
| Reference by key | `for_each = map` | Named access |
| Multiple named resources | `for_each` | Better identity stability |

**Never** use list index as long-lived identity — removing a middle element reshuffles every address after it. For the decision matrix, safe migration playbook, `moved` block patterns, and known-at-plan failure cases, see [Code Patterns: count vs for_each](#section-code-patterns).

<a id="core-workflow-locals-for-dependency-management"></a>
### Locals for Dependency Management

Using `try()` in a local to prefer a conditional resource's attribute over its parent is a specialized but high-value pattern — it forces correct deletion order without explicit `depends_on`. Common use: VPC + secondary CIDR associations + subnets.

See [Code Patterns: Locals for Dependency Management](#section-code-patterns) for the full pattern and worked example.

<a id="core-workflow-module-development"></a>
### Module Development

Standard layout:

```
my-module/
├── README.md       # Usage documentation
├── main.tf         # Primary resources
├── variables.tf    # Typed inputs with descriptions
├── outputs.tf      # Output values
├── versions.tf     # required_version + required_providers
├── examples/
│   ├── minimal/
│   └── complete/
└── tests/
    └── module_test.tftest.hcl   # Native Terraform test
```

**Variable contracts**: always `description`, always explicit `type`, use `validation` for complex constraints, use `sensitive = true` for secrets, prefer `optional()` with typed defaults (1.3+) over untyped `map(any)`.

**Output contracts**: always `description`, mark sensitive outputs, expose stable subsets (not whole provider objects).

See [Module Patterns](#section-module-patterns) for the full contract patterns, module release checklist, and LLM-mistake checklist.

<a id="core-workflow-cicd"></a>
### CI/CD

Pipeline stages: **validate** → **test** → **plan** → **apply** (with environment protection).

Cost control: mock providers on PR validation, real-cloud integration only on main or scheduled, tag test resources, auto-cleanup.

Drift prevention: pin runtime and providers, commit `.terraform.lock.hcl`, apply the **reviewed plan artifact** from the plan stage (do not re-run `plan` inside the apply job), complete the established AWS security review on every path to apply.


<a id="core-workflow-security--compliance"></a>
### Security & Compliance

**Essential review:** Check IAM, network exposure, encryption, secrets, and protected state against the supplied AWS standards and reviewed Terraform plan.

**Don't:** store secrets in variables or `.tfvars`, use default VPC, skip encryption, open security groups to `0.0.0.0/0`, use inline `ingress`/`egress` blocks in `aws_security_group`.

**Do:** source secrets from a cloud secret manager (AWS Secrets Manager) or use `write_only` arguments on 1.11+, create dedicated VPCs, enforce encryption at rest and TLS, least-privilege SGs, use separate `aws_vpc_security_group_{ingress,egress}_rule` resources (e.g. AWS provider v5+).

Marking a variable `sensitive = true` masks display only — the value still lives in state. Use `write_only` / `*_wo` on 1.11+, or keep secret material out of Terraform entirely via runtime lookups.

See [Security & Compliance](#section-security-compliance) for AWS security reviews, state-file hardening, compliance mappings, and the LLM-mistake checklist.

<a id="core-workflow-state-management"></a>
### State Management

**Never use local state in teams or production.** Configure locking, encryption, versioning, access control, and audit logging deliberately for the approved backend.

<a id="core-workflow-choosing-a-remote-backend"></a>
#### Choosing a Remote Backend

AWS S3 backend example:

```hcl
terraform {
  backend "s3" {
    bucket        = "my-terraform-state"
    key           = "prod/vpc/terraform.tfstate"
    region        = "us-east-1"
    encrypt       = true
    use_lockfile  = true   # Native S3 locking, 1.10+
  }
}
```

For older Terraform versions, verify the supported locking configuration against the approved backend setup. Terraform Cloud/Enterprise manages locking through its run and state workflow.

<a id="core-workflow-state-organization"></a>
#### State Organization

| Pattern | Use When | Example Path |
|---------|----------|--------------|
| **Per environment** | Different teams per env | `prod/terraform.tfstate`, `staging/...` |
| **Per component** | Independent lifecycles | `prod/vpc/`, `prod/eks/`, `prod/rds/` |
| **Hybrid** (recommended) | Both benefits | `prod/networking/`, `prod/compute/`, `staging/networking/` |

Separate state by ownership, environment, lifecycle, and change impact. Keep tightly coupled resources together when they share those boundaries.


<a id="core-workflow-version-management"></a>
### Version Management

| Component | Strategy | Example |
|-----------|----------|---------|
| Terraform runtime | Pin minor | `required_version = "~> 1.9.0"` |
| Providers | Pin major | `version = "~> 5.0"` |
| Modules (prod) | Pin exact | `version = "5.1.2"` |
| Modules (dev) | Allow patch | `version = "~> 5.1.0"` |

Commit `.terraform.lock.hcl` intentionally. Keep provider/runtime upgrades in a separate PR from functional changes. See [Code Patterns: Version Management](#section-code-patterns) for constraint syntax and upgrade workflow.

<a id="core-workflow-modern-terraform-features-10"></a>
### Modern Terraform Features (1.0+)

| Feature | Min version | Common use |
|---------|-------------|------------|
| `try()` | 0.13+ | Safe fallbacks, replaces `element(concat())` |
| `nullable = false` | 1.1+ | Prevent `null` silently overriding defaults |
| `moved` blocks | 1.1+ | Refactor without destroy/recreate |
| `optional()` with defaults | 1.3+ | Typed object attributes |
| `import` blocks | 1.5+ | Declarative imports, reviewable in VCS |
| `check` blocks | 1.5+ | Runtime assertions |
| Native `terraform test` | 1.6+ | Built-in test framework |
| Mock providers | 1.7+ | Cost-free unit testing |
| `removed` blocks | 1.7+ | Declarative resource removal |
| Provider-defined functions | 1.8+ | Provider-specific transformations (requires provider to declare functions) |
| Cross-variable validation | 1.9+ | Reference other `var.*` in `validation` blocks |
| `write_only` arguments | 1.11+ | Secrets never stored in state |
| S3 native lock-file | 1.10+ | State locking without DynamoDB |

Before emitting a feature, verify the runtime floor. See [Code Patterns: Feature Guard Table](#section-code-patterns) for the full table with common LLM error patterns per feature.

<a id="core-workflow-runtime-specific-guidance"></a>
### Runtime-Specific Guidance

- **Terraform 1.0-1.5**: static analysis + plan validation only (no native tests).
- **1.6+**: native `terraform test` available — use native plan tests and authorized apply tests as appropriate.
- **1.7+**: mock providers cut test cost — mock for unit tests, real runs for final integration.
- **1.10+**: S3 native lock-file (`use_lockfile`) is the correct default for new configurations — DynamoDB locking is no longer required.
- **1.11+**: `write_only` arguments for secret handling keep credentials out of state.

---

<a id="section-module-patterns"></a>
## Module Design

> **Purpose:** Best practices for Terraform module development

This section provides detailed guidance on creating reusable, maintainable Terraform modules. For high-level principles, see the [core workflow](#core-workflow).

---

<a id="section-module-patterns-module-hierarchy"></a>
### Module Hierarchy

<a id="section-module-patterns-module-type-classification"></a>
#### Module Type Classification

| Type | When to Use | Scope | Example |
|------|-------------|-------|---------|
| **Resource Module** | Single logical group of connected resources | Tightly coupled resources that always work together | VPC + subnets, Security group + rules, IAM role + policies |
| **Infrastructure Module** | Collection of resource modules for a purpose | Multiple resource modules in one region/account | Complete networking stack, Application infrastructure |
| **Composition** | Complete infrastructure | Spans multiple regions/accounts, orchestrates infrastructure modules | Multi-region deployment, Production environment |

**Hierarchy:** Resource → Resource Module → Infrastructure Module → Composition

<a id="section-module-patterns-resource-module"></a>
#### Resource Module

**Characteristics:**
- Smallest building block
- Single logical group of resources
- Highly reusable across projects
- Minimal external dependencies
- Clear, focused purpose

**Examples:**
```
modules/
├── vpc/                    # Resource module
│   ├── main.tf            # VPC + subnets + route tables
│   ├── variables.tf
│   └── outputs.tf
├── security-group/         # Resource module
│   ├── main.tf            # Security group + rules
│   ├── variables.tf
│   └── outputs.tf
└── rds/                    # Resource module
    ├── main.tf            # RDS instance + subnet group
    ├── variables.tf
    └── outputs.tf
```

<a id="section-module-patterns-infrastructure-module"></a>
#### Infrastructure Module

**Characteristics:**
- Combines multiple resource modules
- Purpose-specific (e.g., "web application infrastructure")
- May span multiple services
- Region or account-specific
- Moderate reusability

**Examples:**
```
modules/
└── web-application/        # Infrastructure module
    ├── main.tf            # Orchestrates multiple resource modules
    ├── variables.tf
    ├── outputs.tf
    └── README.md

# main.tf contents:
module "vpc" {
  source = "../vpc"
}

module "alb" {
  source = "../alb"
  vpc_id = module.vpc.vpc_id
}

module "ecs" {
  source = "../ecs"
  vpc_id = module.vpc.vpc_id
  subnets = module.vpc.private_subnet_ids
}
```

<a id="section-module-patterns-composition"></a>
#### Composition

**Characteristics:**
- Highest level of abstraction
- Complete environment or application
- Combines infrastructure modules
- Environment-specific (dev, staging, prod)
- Not reusable (environment-specific values)

**Examples:**
```
environments/
├── prod/                   # Composition
│   ├── main.tf            # Complete production environment
│   ├── backend.tf         # Remote state configuration
│   ├── terraform.tfvars   # Production-specific values
│   └── variables.tf
├── staging/                # Composition
│   ├── main.tf
│   ├── backend.tf
│   ├── terraform.tfvars
│   └── variables.tf
└── dev/                    # Composition
    ├── main.tf
    ├── backend.tf
    ├── terraform.tfvars
    └── variables.tf
```

<a id="section-module-patterns-decision-tree-which-module-type"></a>
#### Decision Tree: Which Module Type?

```
Question 1: Is this environment-specific configuration?
├─ YES → Composition (environments/prod/, environments/staging/)
└─ NO  → Continue

Question 2: Does it combine multiple infrastructure concerns?
├─ YES → Infrastructure Module (modules/web-application/)
└─ NO  → Continue

Question 3: Is it a focused group of related resources?
└─ YES → Resource Module (modules/vpc/, modules/rds/)
```

<a id="section-module-patterns-file-organization-standards"></a>
#### File Organization Standards

**Required files in all modules:**
```
main.tf        # Resource definitions, module calls, data sources
variables.tf   # Input variable declarations
outputs.tf     # Output value declarations
versions.tf    # Provider and Terraform version constraints
README.md      # Usage documentation
```

**Conditional files:**
```
terraform.tfvars  # ONLY at composition level (NEVER in modules)
locals.tf         # For complex local value calculations
data.tf           # Optional: Data sources (if main.tf gets large)
backend.tf        # ONLY at composition level (remote state config)
```

Required structure for Terraform Registry publishing; keeps navigation consistent across modules.

---

<a id="section-module-patterns-architecture-principles"></a>
### Architecture Principles

<a id="section-module-patterns-1-smaller-scopes--better-performance--reduced-blast-radius"></a>
#### 1. Smaller Scopes = Better Performance + Reduced Blast Radius

Faster `plan`/`apply`, isolated failures, parallel team development.

**Example:**

```hcl
# ❌ BAD - One massive composition with everything
environments/prod/
  main.tf  # 2000 lines, manages VPC, EC2, RDS, S3, IAM, everything
  # Takes 10+ minutes to plan
  # One mistake affects entire infrastructure

# ✅ GOOD - Separated by concern
environments/prod/
  networking/     # VPC, subnets, route tables
  compute/        # EC2, ASG, ALB
  data/           # RDS, ElastiCache
  storage/        # S3, EFS
  iam/            # IAM roles, policies
```

<a id="section-module-patterns-2-always-use-remote-state"></a>
#### 2. Always Use Remote State

- ❌ local `terraform.tfstate` — no locking, no backup, no team access
- ✅ remote backend — locking, versioning, encryption, audit log

```hcl
terraform {
  backend "s3" {
    bucket       = "my-terraform-state"
    key          = "prod/networking/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true   # Terraform 1.10+; native S3 locking
    # Pre-1.10 runtime: use dynamodb_table = "terraform-locks" instead
  }
}
```

<a id="section-module-patterns-3-use-terraform_remote_state-sparingly--only-at-true-ownership-boundaries"></a>
#### 3. Use terraform_remote_state Sparingly — Only at True Ownership Boundaries

**Pattern:** Connect separately-owned compositions via remote state data sources. Reserve it for genuine team/lifecycle boundaries, not as convenient glue inside a single-team stack.

**Use it when ALL of these are true:**
- Consumer and producer are owned by **different teams** or have **different release cadences**
- The producer's state is already split for lifecycle reasons (networking vs. compute vs. data)
- You cannot reasonably pass the same values as module inputs

**Do NOT use it when:**
- You control both stacks and can wire via module outputs
- You're reading values that would be better served by a cloud data source (e.g., `aws_vpc` by tag)
- You're reaching across >2 remote states in one composition — that is a signal to reshape boundaries, not add more wiring

**Common LLM mistakes:**
- reaches for `terraform_remote_state` as default integration pattern
- chains many `terraform_remote_state` reads, creating hidden cross-stack coupling
- reads values that can drift at the provider level (use cloud data sources instead)

At real boundaries, outputs from one stack become typed inputs to another — teams release independently without shared mutable state.

**Example:**

```hcl
# environments/prod/networking/outputs.tf
output "vpc_id" {
  description = "ID of the production VPC"
  value       = aws_vpc.this.id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

# environments/prod/compute/main.tf
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "prod/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

module "ec2" {
  source = "../../modules/ec2"

  vpc_id     = data.terraform_remote_state.networking.outputs.vpc_id
  subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
}
```

- ✅ document which outputs are consumed externally; version outputs, never break downstream consumers silently
- ✅ prefer cloud data sources (`aws_vpc` by tag) over `terraform_remote_state` for provider-managed resources

<a id="section-module-patterns-4-keep-resource-modules-simple"></a>
#### 4. Keep Resource Modules Simple

**Principles:**
- Don't hardcode values
- Use variables for all configurable parameters
- Use data sources for external dependencies
- Focus on single responsibility

**Example:**

```hcl
# ❌ BAD - Hardcoded values in resource module
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"  # Hardcoded
  instance_type = "t3.large"               # Hardcoded
  subnet_id     = "subnet-12345678"        # Hardcoded

  tags = {
    Environment = "production"             # Hardcoded
  }
}

# ✅ GOOD - Parameterized resource module
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id != "" ? var.ami_id : data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  tags = var.tags
}
```

<a id="section-module-patterns-aws-resource-map"></a>
#### AWS resource map

| Resource | AWS |
| ---------- | ----- |
| Network | `aws_vpc` |
| Subnet | `aws_subnet` |
| Compute instance | `aws_instance` |
| Managed relational DB | `aws_db_instance` / `aws_rds_cluster` |
| Object storage | `aws_s3_bucket` |

<a id="section-module-patterns-5-composition-layer-environment-specific-values-only"></a>
#### 5. Composition Layer: Environment-Specific Values Only

**Pattern:** Compositions provide concrete values, modules provide abstractions

```hcl
# ✅ GOOD - Composition with environment-specific values
# environments/prod/main.tf

module "vpc" {
  source = "../../modules/vpc"

  cidr_block           = "10.0.0.0/16"
  availability_zones   = ["us-east-1a", "us-east-1b", "us-east-1c"]
  enable_nat_gateway   = true
  single_nat_gateway   = false  # HA for production

  tags = {
    Environment = "production"
    ManagedBy   = "Terraform"
    CostCenter  = "engineering"
  }
}

module "rds" {
  source = "../../modules/rds"

  instance_class       = "db.r5.xlarge"  # Production sizing
  allocated_storage    = 500             # Production sizing
  multi_az             = true            # HA for production
  backup_retention     = 30              # Long retention for prod

  vpc_id               = module.vpc.vpc_id
  subnet_ids           = module.vpc.private_subnet_ids

  tags = {
    Environment = "production"
  }
}
```

---

<a id="section-module-patterns-module-structure"></a>
### Module Structure

<a id="section-module-patterns-standard-layout"></a>
#### Standard Layout

```
my-module/
├── README.md                # Usage documentation
├── LICENSE                  # MIT or Apache 2.0 (for public modules)
├── .pre-commit-config.yaml  # Pre-commit hooks configuration
├── main.tf                  # Primary resources
├── variables.tf             # Input variables with descriptions
├── outputs.tf               # Output values
├── versions.tf              # Provider version constraints
├── examples/
│   ├── simple/              # Minimal working example
│   └── complete/            # Full-featured example
└── tests/                   # Test files
    └── module_test.tft
```

<a id="section-module-patterns-file-role"></a>
#### File Role

- `README.md` — module purpose, first file users see
- `LICENSE` — legal terms for public modules (MIT or Apache 2.0)
- `.pre-commit-config.yaml` — automated validation before commits
- `main.tf` — primary resources, keep focused
- `variables.tf` — all inputs, with descriptions
- `outputs.tf` — all outputs, with descriptions
- `versions.tf` — pinned provider versions
- `examples/` — docs + test fixtures
- `tests/` — automated tests

<a id="section-module-patterns-license-files"></a>
#### License Files

- ✅ Public modules / open-source projects — include LICENSE (MIT = permissive; Apache 2.0 = permissive + patent grant)
- ❌ Private internal modules / environment-specific configs — optional
- ❌ Do NOT store LICENSE templates in this skill; generate them on demand from user preference

<a id="section-module-patterns-terraform-runtime-requirements"></a>
#### Terraform Runtime Requirements

Use the Terraform CLI for this skill. Confirm the required version from the root configuration, CI image, or TFE workspace settings. Keep commands, module documentation, and pipeline execution consistent with that version. Document Terraform and provider version requirements in the module README.

---

<a id="section-module-patterns-variable-best-practices"></a>
### Variable Best Practices

<a id="section-module-patterns-complete-example"></a>
#### Complete Example

```hcl
variable "instance_type" {
  description = "EC2 instance type for the application server"
  type        = string
  default     = "t3.micro"

  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "Instance type must be t3.micro, t3.small, or t3.medium."
  }
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}

variable "enable_monitoring" {
  description = "Enable CloudWatch detailed monitoring"
  type        = bool
  default     = true
}
```

<a id="section-module-patterns-key-principles"></a>
#### Key Principles

- ✅ **Always include `description`** - Helps users understand the variable
- ✅ **Use explicit `type` constraints** - Catches errors early
- ✅ **Provide sensible `default` values** - Where appropriate
- ✅ **Add `validation` blocks** - For complex constraints
- ✅ **Use `sensitive = true`** - For secrets (Terraform 0.14+)

<a id="section-module-patterns-variable-naming"></a>
#### Variable Naming

```hcl
# ✅ Good: Context-specific
var.vpc_cidr_block          # Not just "cidr"
var.database_instance_class # Not just "instance_class"
var.application_port        # Not just "port"

# ❌ Bad: Generic names
var.name
var.type
var.value
```

<a id="section-module-patterns-provider-requirements-and-alias-passing"></a>
#### Provider Requirements and Alias Passing

- ✅ Child module declares aliased providers: `configuration_aliases = [aws.primary, aws.replica]`
- ✅ Caller passes them explicitly: `providers = { aws.primary = aws.<caller-alias> }` on the `module` block
- ❌ Default provider inheritance applies ONLY to a single unaliased provider — never for aliases

Child module — declare aliases in `versions.tf`, bind per resource:

```hcl
# modules/replicated-s3/versions.tf
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = "~> 5.0"
      configuration_aliases = [aws.primary, aws.replica]
    }
  }
}

# in any resource:
provider = aws.primary
```

Caller — pass the `providers` map on the `module` block:

```hcl
module "bucket" {
  source      = "./modules/replicated-s3"
  bucket_name = "app-data"

  providers = {
    aws.primary = aws.us_east_2
    aws.replica = aws.us_east_1
  }
}
```

❌ DON'T — missing `providers` map on the module call:

```hcl
module "bucket" {
  source      = "./modules/replicated-s3"
  bucket_name = "app-data"
  # MISSING: providers = { aws.primary = ..., aws.replica = ... }
  # Plan fails: "No configuration for provider aws.primary"
}
```

---

<a id="section-module-patterns-output-best-practices"></a>
### Output Best Practices

<a id="section-module-patterns-complete-example-2"></a>
#### Complete Example

```hcl
output "instance_id" {
  description = "ID of the created EC2 instance"
  value       = aws_instance.this.id
}

output "instance_arn" {
  description = "ARN of the created EC2 instance"
  value       = aws_instance.this.arn
}

output "private_ip" {
  description = "Private IP address of the instance"
  value       = aws_instance.this.private_ip
  sensitive   = false  # Explicitly document sensitivity
}

output "connection_info" {
  description = "Connection information for the instance"
  value = {
    id         = aws_instance.this.id
    private_ip = aws_instance.this.private_ip
    public_dns = aws_instance.this.public_dns
  }
}
```

<a id="section-module-patterns-key-principles-2"></a>
#### Key Principles

- ✅ **Always include `description`** - Explain what the output is for
- ✅ **Mark sensitive outputs** - Use `sensitive = true`
- ✅ **Return objects for related values** - Groups logically related data
- ✅ **Document intended use** - What should consumers do with this?

---

<a id="section-module-patterns-common-patterns"></a>
### Common Patterns

<a id="section-module-patterns-iteration-for_each-vs-count"></a>
#### Iteration: `for_each` vs `count`

Use `for_each` with stable keys whenever a collection has meaningful identity — removing or reordering an element leaves unrelated addresses untouched. Reserve `count` for optional singletons (`0` or `1`) and cases where keys cannot be known at plan time.

For the decision matrix, migration playbook, and known-at-plan failure patterns, see [Code Patterns: count vs for_each](#section-code-patterns).

<a id="section-module-patterns--do-separate-root-module-from-reusable-modules"></a>
#### ✅ DO: Separate Root Module from Reusable Modules

```
# Root module (environment-specific)
prod/
  main.tf          # Calls modules with prod-specific values
  variables.tf     # Environment-specific variables

# Reusable module
modules/webapp/
  main.tf          # Generic, parameterized resources
  variables.tf     # Configurable inputs
```

Root modules are environment-specific; reusable modules are generic.

<a id="section-module-patterns--do-use-locals-for-computed-values"></a>
#### ✅ DO: Use Locals for Computed Values

```hcl
locals {
  common_tags = merge(
    var.tags,
    {
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  )

  instance_name = "${var.project}-${var.environment}-instance"
}

resource "aws_instance" "app" {
  tags = local.common_tags
  # ...
}
```

<a id="section-module-patterns--do-version-your-modules"></a>
#### ✅ DO: Version Your Modules

```hcl
# In consuming code
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"  # Pin to major version

  # module inputs...
}
```

Prevents unexpected breaking changes from upstream major bumps.

---

<a id="section-module-patterns-anti-patterns-to-avoid"></a>
### Anti-patterns to Avoid

<a id="section-module-patterns--dont-hard-code-environment-specific-values"></a>
#### ❌ DON'T: Hard-code Environment-Specific Values

```hcl
# Bad: Module is locked to production
resource "aws_instance" "app" {
  instance_type = "m5.large"  # Should be variable
  tags = {
    Environment = "production" # Should be variable
  }
}
```

**Fix:** Make everything configurable:

```hcl
resource "aws_instance" "app" {
  instance_type = var.instance_type
  tags          = var.tags
}
```

<a id="section-module-patterns--dont-create-god-modules"></a>
#### ❌ DON'T: Create God Modules

```hcl
# Bad: One module does everything
module "everything" {
  source = "./modules/app-infrastructure"

  # Creates VPC, EC2, RDS, S3, IAM, CloudWatch, etc.
}
```

**Problem:** Hard to test, hard to reuse, hard to maintain.

**Fix:** Break into focused modules:

```hcl
module "networking" {
  source = "./modules/vpc"
}

module "compute" {
  source = "./modules/ec2"
  vpc_id = module.networking.vpc_id
}

module "database" {
  source = "./modules/rds"
  vpc_id = module.networking.vpc_id
}
```

<a id="section-module-patterns--dont-use-count-or-for_each-in-root-modules-for-different-environments"></a>
#### ❌ DON'T: Use `count` or `for_each` in Root Modules for Different Environments

```hcl
# Bad: All environments in one root module
resource "aws_instance" "app" {
  for_each = toset(["dev", "staging", "prod"])

  instance_type = each.key == "prod" ? "m5.large" : "t3.micro"
}
```

**Problem:** Can't have separate state files, blast radius is huge.

**Fix:** Use separate root modules:

```
environments/
  dev/
    main.tf
  staging/
    main.tf
  prod/
    main.tf
```

<a id="section-module-patterns--dont-use-terraform_remote_state-everywhere"></a>
#### ❌ DON'T: Use `terraform_remote_state` Everywhere

Use module outputs when possible. Reserve remote state for ownership boundaries between teams. See [Use terraform_remote_state Sparingly](#section-module-patterns-3-use-terraform_remote_state-sparingly--only-at-true-ownership-boundaries) for the full rule set.

---


<a id="section-module-patterns-module-release-checklist"></a>
### Module Release Checklist

Before publishing or handing off a reusable module:

- [ ] Runtime and provider choice explicit (Terraform CLI version floor in `required_version`)
- [ ] Public vs private scope decided (affects naming + license)
- [ ] `examples/` directory with at least `minimal` and `complete`
- [ ] Tests written (native `terraform test` on 1.6+) — see [Testing and Validation](#section-testing-frameworks)
- [ ] README documents all inputs/outputs (Description → Usage → Inputs → Outputs → Requirements)
- [ ] Module source pinned with `version` in consumer code
- [ ] Run direct `terraform fmt` and `terraform validate` checks; use approved, preinstalled TFLint when available.
- [ ] `.gitignore` excludes `.terraform/`, `*.tfstate*`, `*.tfvars`, override files, and editor artifacts

---

<a id="section-module-patterns-module-testing--pointer"></a>
### Module Testing — Pointer

Module testing (what to test, tiered layers, mocking, idempotency, cost control, strategy by module type) is canonical in [Testing Frameworks](#section-testing-frameworks). Module-specific rules that belong with the module contract:

- Every reusable module must exercise its `validation` blocks in tests — reject cases are as important as happy paths.
- Tier tests by module role: **resource modules** → input validation + attribute assertions; **infrastructure modules** → composition + cross-module wiring; **compositions** → smoke-plan + production-like values + remote-state connectivity.
- Mock providers (1.7+) for unit tests; reserve real cloud runs for main-branch or scheduled jobs.

---

<a id="section-module-patterns-llm-mistake-checklist--modules"></a>
### LLM Mistake Checklist — Modules

Common model mistakes to correct when generating or reviewing modules:

- bundles unrelated resources into one "god module" instead of splitting by single responsibility
- hardcodes environment-specific values (`instance_type = "m5.large"`, `Environment = "production"`) inside a reusable module
- accepts untyped `map(any)` / `any` for core module inputs instead of typed objects with `optional()` defaults
- exposes entire provider or resource objects as outputs, leaking the whole contract instead of a stable subset
- omits `description` on inputs and outputs, forcing consumers to read the implementation
- uses `this` for multiple resources of the same type — reserve `this` for genuine singletons only
- reaches for `terraform_remote_state` inside a single team's stack instead of wiring via module outputs
- floats module sources (no `version` pin) in consumer code
- pushes environment-specific policy (prod-only allowlists, region pins) into primitive/resource modules where it cannot be overridden
- omits `configuration_aliases` in a multi-provider child module's `required_providers` — callers cannot pass aliased providers
- drops the `providers = { aws = aws.region }` map from the module call on multi-region or multi-account deploys — resources land on the default provider

---

**Back to:** [Core Workflow](#core-workflow)

---

<a id="section-code-patterns"></a>
## Terraform Code Patterns

> **Purpose:** Comprehensive patterns for Terraform code structure and modern features

This section provides detailed code patterns, structure guidelines, and modern Terraform features. For high-level principles, see the [core workflow](#core-workflow).

---

<a id="section-code-patterns-block-ordering--structure"></a>
### Block Ordering & Structure

<a id="section-code-patterns-resource-block-structure"></a>
#### Resource Block Structure

**Strict argument ordering:**

1. `count` or `for_each` FIRST (blank line after)
2. Other arguments (alphabetical or logical grouping)
3. `tags` as last real argument
4. `depends_on` after tags (if needed)
5. `lifecycle` at the very end (if needed)

```hcl
# ✅ GOOD - Correct ordering
resource "aws_nat_gateway" "this" {
  count = var.create_nat_gateway ? 1 : 0

  allocation_id = aws_eip.this[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = {
    Name        = "${var.name}-nat"
    Environment = var.environment
  }

  depends_on = [aws_internet_gateway.this]

  lifecycle {
    create_before_destroy = true
  }
}

# ❌ BAD - Wrong ordering
resource "aws_nat_gateway" "this" {
  allocation_id = aws_eip.this[0].id

  tags = { Name = "nat" }

  count = var.create_nat_gateway ? 1 : 0  # Should be first

  subnet_id = aws_subnet.public[0].id

  lifecycle {
    create_before_destroy = true
  }

  depends_on = [aws_internet_gateway.this]  # Should be after tags
}
```


<a id="section-code-patterns-variable-definition-structure"></a>
#### Variable Definition Structure

**Variable block ordering:**

1. `description` (ALWAYS required)
2. `type`
3. `default`
4. `sensitive` (when setting to true)
5. `nullable` (when setting to false)
6. `validation`

```hcl
# ✅ GOOD - Correct ordering and structure
variable "environment" {
  description = "Environment name for resource tagging"
  type        = string
  default     = "dev"
  nullable    = false

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}
```

<a id="section-code-patterns-variable-type-preferences"></a>
#### Variable Type Preferences

- Prefer **simple types** (`string`, `number`, `list()`, `map()`) over `object()` unless strict validation needed
- Use `optional()` for optional object attributes (Terraform 1.3+)
- Use `any` to disable validation at certain depths or support multiple types

**Modern variable patterns (Terraform 1.3+):**

```hcl
# ✅ GOOD - Using optional() for object attributes
variable "database_config" {
  description = "Database configuration with optional parameters"
  type = object({
    name               = string
    engine             = string
    instance_class     = string
    backup_retention   = optional(number, 7)      # Default: 7
    monitoring_enabled = optional(bool, true)     # Default: true
    tags               = optional(map(string), {}) # Default: {}
  })
}

# Usage - only required fields needed
database_config = {
  name           = "mydb"
  engine         = "mysql"
  instance_class = "db.t3.micro"
  # Optional fields use defaults
}
```

**Complex type example:**

```hcl
# For lists/maps of same type
variable "subnet_configs" {
  description = "Map of subnet configurations"
  type        = map(map(string))  # All values are maps of strings
}

# When types vary, use any
variable "mixed_config" {
  description = "Configuration with varying types"
  type        = any
}
```

<a id="section-code-patterns-output-structure"></a>
#### Output Structure

**Pattern:** `{name}_{type}_{attribute}`

```hcl
# ✅ GOOD
output "security_group_id" {  # "this_" should be omitted
  description = "The ID of the security group"
  value       = try(aws_security_group.this[0].id, "")
}

output "private_subnet_ids" {  # Plural for list
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

# ❌ BAD
output "this_security_group_id" {  # Don't prefix with "this_"
  value = aws_security_group.this[0].id
}

output "subnet_id" {  # Should be plural "subnet_ids"
  value = aws_subnet.private[*].id  # Returns list
}
```

---

<a id="section-code-patterns-count-vs-for_each-deep-dive"></a>
### Count vs For_Each Deep Dive

<a id="section-code-patterns-when-to-use-count"></a>
#### When to use count

✓ **Simple numeric replication:**
```hcl
resource "aws_subnet" "public" {
  count = 3

  cidr_block = cidrsubnet(var.vpc_cidr, 8, count.index)
}
```

✓ **Boolean conditions (create or don't):**
```hcl
# ✅ GOOD - Boolean condition
resource "aws_nat_gateway" "this" {
  count = var.create_nat_gateway ? 1 : 0
}

# Less preferred - length check
resource "aws_nat_gateway" "this" {
  count = length(var.public_subnets) > 0 ? 1 : 0
}
```

✓ **When order doesn't matter and items won't change**

<a id="section-code-patterns-when-to-use-for_each"></a>
#### When to use for_each

✓ **Reference resources by key:**
```hcl
resource "aws_subnet" "private" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.this.id
  availability_zone = each.key
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, index(var.availability_zones, each.key))
}

# Reference by key: aws_subnet.private["us-east-1a"]
```

✓ **Items may be added/removed from middle:**
```hcl
# ❌ BAD with count - removing middle item recreates all subsequent resources
resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  availability_zone = var.availability_zones[count.index]
  # If var.availability_zones[1] removed, all resources after recreated!
}

# ✅ GOOD with for_each - removal only affects that one resource
resource "aws_subnet" "private" {
  for_each = toset(var.availability_zones)

  availability_zone = each.key
  # Removing one AZ only destroys that subnet
}
```

✓ **Creating multiple named resources:**
```hcl
variable "environments" {
  default = {
    dev = {
      instance_type = "t3.micro"
    }
    prod = {
      instance_type = "t3.large"
    }
  }
}

resource "aws_instance" "app" {
  for_each = var.environments

  instance_type = each.value.instance_type

  tags = {
    Environment = each.key  # "dev" or "prod"
  }
}
```

<a id="section-code-patterns-count-to-for_each-migration"></a>
#### Count to For_Each Migration

**When to migrate:** When you need stable resource addressing or items might be added/removed from middle of list.

**Migration steps:**

1. Add `for_each` to resource
2. Use `moved` blocks to preserve existing resources
3. Remove `count` after verifying with `terraform plan`

**Complete example:**

```hcl
# Before (using count)
variable "availability_zones" {
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "private-${var.availability_zones[count.index]}"
  }
}

# Reference: aws_subnet.private[0].id

# After (using for_each)
resource "aws_subnet" "private" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, index(var.availability_zones, each.key))
  availability_zone = each.key

  tags = {
    Name = "private-${each.key}"
  }
}

# Reference: aws_subnet.private["us-east-1a"].id

# Migration blocks (prevents resource recreation)
moved {
  from = aws_subnet.private[0]
  to   = aws_subnet.private["us-east-1a"]
}

moved {
  from = aws_subnet.private[1]
  to   = aws_subnet.private["us-east-1b"]
}

moved {
  from = aws_subnet.private[2]
  to   = aws_subnet.private["us-east-1c"]
}

# Verify migration:
# terraform plan should show "moved" operations, not destroy/create
```

After migration: removing `us-east-1b` destroys only that subnet; adding an AZ does not churn existing resources; addresses are stable by AZ name.

<a id="section-code-patterns-for_each-keys-must-be-known-at-plan-time"></a>
#### `for_each` keys must be known at plan time

`for_each` (0.12+) requires its key set resolvable during plan.

| Case | Use | Why |
|------|-----|-----|
| stable key set known at plan | `for_each` over static map/var | avoids count index churn on insert/remove |
| key set unknowable at plan | `count = bool ? 1 : 0` for singleton | keys derived from values unknown until apply |

- ❌ `depends_on` does NOT fix `Invalid for_each argument` — it orders applies, not plan-time value resolution
- ❌ deriving `for_each` keys from another resource's computed attrs (IDs, ARNs)
- ✅ drive `for_each` from user-supplied variables or static locals

```hcl
# ❌ BAD - keys derived from computed IDs; plan fails
resource "aws_eip" "web" {
  for_each = toset([for i in aws_instance.web : i.id])
  instance = each.key
}

# ✅ GOOD - drive for_each from user-supplied keys
variable "instances" {
  type = map(object({ instance_type = string }))
}

resource "aws_instance" "web" {
  for_each      = var.instances
  ami           = "ami-0123"
  instance_type = each.value.instance_type
}

resource "aws_eip" "web" {
  for_each = var.instances
  instance = aws_instance.web[each.key].id
}

# ✅ GOOD - singleton when exact ID not known at plan
resource "aws_eip" "bastion" {
  count    = var.create_bastion ? 1 : 0
  instance = aws_instance.bastion[0].id
}
```

---

<a id="section-code-patterns-modern-terraform-features-10"></a>
### Modern Terraform Features (1.0+)

<a id="section-code-patterns-feature-guard-table--version-floor--common-llm-errors"></a>
#### Feature Guard Table — Version Floor & Common LLM Errors

Before emitting a feature, verify the runtime floor. Each feature here is also a known hallucination surface — the error pattern column names the mistake to avoid.

| Feature | Min version | Common LLM error pattern |
|---------|-------------|--------------------------|
| `for_each` over `count` for stable identities | 0.12+ | defaults to `count` for every collection, causing index churn |
| `try()` function | 0.12.20+ | falls back to `element(concat())` legacy pattern |
| `nonsensitive()` function | 0.15+ | used to 'unwrap' sensitive outputs into plan artifacts, effectively laundering secrets into logs |
| `nullable = false` | 1.1+ | omits it, letting `null` silently override defaults |
| `moved` blocks | 1.1+ | omitted during refactor, causing destroy/create |
| `optional()` with defaults | 1.3+ | emits wrapper variables and loose `map(any)` contracts |
| declarative `import` blocks | 1.5+ | recommends ad-hoc CLI `terraform import` only |
| `check` blocks | 1.5+ | ignores runtime assertions entirely |
| native `terraform test` | 1.6+ | treats mocked-provider tests as full integration coverage |
| mock providers | 1.7+ | asserts computed values in `command = plan` mode |
| `removed` blocks | 1.7+ | deletes resources with no lifecycle transition |
| provider-defined functions | 1.8+ | overuses data sources for simple transformations |
| cross-variable validation | 1.9+ | pushes checks into postconditions only |
| S3 native lock-file | 1.10+ | recommends DynamoDB lock table even on 1.10+ |
| `ephemeral` values | 1.10+ | treats as interchangeable with `sensitive`; ephemeral values are scrubbed from state, `sensitive` only masks display |
| `write_only` arguments | 1.11+ | uses `sensitive = true` and assumes state is safe |

If target runtime is below a feature floor, emit the pre-floor fallback explicitly instead of silently downgrading.

<a id="section-code-patterns-try-function-terraform-01220"></a>
#### try() Function (Terraform 0.12.20+)

**Use try() instead of element(concat()):**

```hcl
# ✅ GOOD - Modern try() function
output "security_group_id" {
  description = "The ID of the security group"
  value       = try(aws_security_group.this[0].id, "")
}

output "first_subnet_id" {
  description = "ID of first subnet with multiple fallbacks"
  value       = try(
    aws_subnet.public[0].id,
    aws_subnet.private[0].id,
    ""
  )
}

# ❌ BAD - Legacy pattern
output "security_group_id" {
  value = element(concat(aws_security_group.this[*].id, [""]), 0)
}
```

<a id="section-code-patterns-nullable--false-terraform-11"></a>
#### nullable = false (Terraform 1.1+)

**Set nullable = false for non-null variables:**

```hcl
# ✅ GOOD (Terraform 1.1+)
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  nullable    = false  # Passing null uses default, not null
  default     = "10.0.0.0/16"
}
```

<a id="section-code-patterns-optional-with-defaults-terraform-13"></a>
#### optional() with Defaults (Terraform 1.3+)

**Use optional() for object attributes:**

```hcl
# ✅ GOOD - Using optional() for object attributes
variable "database_config" {
  description = "Database configuration with optional parameters"
  type = object({
    name               = string
    engine             = string
    instance_class     = string
    backup_retention   = optional(number, 7)      # Default: 7
    monitoring_enabled = optional(bool, true)     # Default: true
    tags               = optional(map(string), {}) # Default: {}
  })
}

# Usage - only required fields needed
database_config = {
  name           = "mydb"
  engine         = "mysql"
  instance_class = "db.t3.micro"
  # Optional fields use defaults
}
```

<a id="section-code-patterns-moved-blocks-terraform-11"></a>
#### Moved Blocks (Terraform 1.1+)

**Rename resources without destroy/recreate.** Omitting `moved` during a refactor is one of the most common LLM mistakes — the model renames the address and silently turns the rename into destroy/create. Always emit `moved` in the same change as the rename, then verify `terraform plan` shows a move operation, not replacement.

```hcl
# Rename a resource
moved {
  from = aws_instance.web_server
  to   = aws_instance.web
}

# Rename a module
moved {
  from = module.old_module_name
  to   = module.new_module_name
}

# Move resource into for_each
moved {
  from = aws_subnet.private[0]
  to   = aws_subnet.private["us-east-1a"]
}
```

**Limits of `moved` (1.1+):**

| Limit | Can `moved` cross this? | Alternative |
|-------|-------------------------|-------------|
| Provider boundary | No | use `removed` (1.7+) + `import` (1.5+) |
| State file / backend key | No | `state mv` across backends + pre-migration backup |
| Module removal (module deleted from config) | `moved` block inside removed module silently stops working | add `moved` in the **parent**, not the removed module |

<a id="section-code-patterns-ignore_changes-lifecycle-escape-hatch"></a>
#### ignore_changes (Lifecycle Escape Hatch)

- ✅ attribute-level `ignore_changes = [tags["X"]]` with a comment naming the external system
- ❌ `ignore_changes = all` — hides real drift, turns every attribute unmanaged
- ❌ use `ignore_changes` to silence noisy plans instead of diagnosing root cause

```hcl
# ❌ BAD - blanket ignore hides all drift
resource "aws_db_instance" "this" {
  lifecycle {
    ignore_changes = all
  }
}

# ✅ GOOD - narrow ignore with justification
resource "aws_db_instance" "this" {
  lifecycle {
    # An external platform process updates this operational tag
    ignore_changes = [tags["LastReviewed"]]
  }
}
```

<a id="section-code-patterns-provider-defined-functions-terraform-18"></a>
#### Provider-Defined Functions (Terraform 1.8+)

**Use provider-specific functions for data transformation:**

```hcl
# AWS provider function example
locals {
  # provider::aws::arn_build(partition, service, region, account_id, resource)
  # S3 ARNs are global: region and account_id are empty strings.
  bucket_arn = provider::aws::arn_build("aws", "s3", "", "", "my-bucket")
}

# Check provider documentation for available functions
# Check the installed AWS provider documentation for supported provider functions.
```

<a id="section-code-patterns-cross-variable-validation-terraform-19"></a>
#### Cross-Variable Validation (Terraform 1.9+)

**Reference other variables in validation blocks:**

```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
}

variable "storage_size" {
  description = "Storage size in GB"
  type        = number

  validation {
    # Can reference var.instance_type in Terraform 1.9+
    condition = !(
      var.instance_type == "db.t3.micro" &&
      var.storage_size > 1000
    )
    error_message = "Micro instances cannot have storage > 1000 GB"
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "backup_retention" {
  description = "Backup retention period in days"
  type        = number

  validation {
    # Production requires longer retention
    condition = (
      var.environment == "prod" ? var.backup_retention >= 7 : true
    )
    error_message = "Production environment requires backup_retention >= 7 days"
  }
}
```

<a id="section-code-patterns-validation-mechanism-timing"></a>
#### Validation Mechanism Timing

Four mechanisms look similar and are routinely confused. Only three actually gate apply.

| Mechanism | When it runs | Can reference | Blocks apply? |
|-----------|--------------|---------------|---------------|
| `validation` (in `variable`) | var evaluation, before plan | the variable's own value; other vars on 1.9+ | yes |
| `precondition` (in `lifecycle`) | before resource create/update | other resources, data sources, vars | yes |
| `postcondition` (in `lifecycle`) | after apply | the resource's own computed attrs | yes |
| `check` block (1.5+) | every plan + apply | anything | **NO — advisory only, warnings not errors** |

<a id="section-code-patterns-write-only-arguments-terraform-111"></a>
#### Write-Only Arguments (Terraform 1.11+)

**Always use write-only arguments or external secret management.** A common LLM mistake is to mark a variable `sensitive = true` and assume the value is kept out of state — it is not. `sensitive` only masks display; write-only arguments (or external secret lookups at runtime) are what actually keep material out of state. Verify on 1.11+: prefer `*_wo` arguments for credentials; on older runtimes, source secrets from a secret manager and never store them in variables or tfvars.

```hcl
# ✅ GOOD - External secret with write-only argument
data "aws_secretsmanager_secret" "db_password" {
  name = "prod-database-password"
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = data.aws_secretsmanager_secret.db_password.id
}

resource "aws_db_instance" "this" {
  engine         = "mysql"
  instance_class = "db.t3.micro"
  username       = "admin"

  # password_wo keeps the resource argument out of state (1.11+),
  # but the data source still reads secret_string into state on refresh.
  # For true state exclusion: use ephemeral (1.10+), manage_master_user_password,
  # or inject via CI env var outside Terraform.
  password_wo = data.aws_secretsmanager_secret_version.db_password.secret_string
}

# ❌ BAD - Secret ends up in state file
resource "random_password" "db" {
  length = 16
}

resource "aws_db_instance" "this" {
  password = random_password.db.result  # Stored in state!
}

# ❌ BAD - Variable secret stored in state
resource "aws_db_instance" "this" {
  password = var.db_password  # Ends up in state file
}
```

<a id="section-code-patterns-nonsensitive-and-ephemeral-terraform-015--110"></a>
#### nonsensitive() and ephemeral (Terraform 0.15+ / 1.10+)

| Goal | Use | Tradeoff |
|------|-----|----------|
| derived non-secret incorrectly inferred as sensitive | `nonsensitive()` (0.15+) | only safe when provably not secret; value enters plan |
| short-lived credential that must never persist | `ephemeral` (1.10+) | never in state or plan; provider/resource must support it |
| value must persist but not display | `sensitive = true` | still in state; masks terminal only |

```hcl
# ✅ GOOD - ephemeral keeps short-lived creds out of state (1.10+)
# requires random provider >= 3.7.0
ephemeral "random_password" "session" {
  length = 32
}

# ❌ BAD - unwrapping a real secret to silence a warning
output "db_endpoint" {
  value = nonsensitive(aws_db_instance.this.password)
}
```

<a id="section-code-patterns-dynamic-blocks--iterator-shadowing--set-ordering"></a>
#### Dynamic Blocks — Iterator Shadowing + Set Ordering

| Gotcha | Cause | Fix |
|--------|-------|-----|
| outer `each.*` inside nested `dynamic` | block-name iterator shadows `each` | `iterator = rule` rename |
| non-deterministic block order | `for_each = toset([...])` on a map/object | use map keyed by stable field |

- ❌ bare `dynamic "ingress"` inside outer `for_each` — `ingress.value` shadows `each.value`
- ✅ rename inner iterator with `iterator = rule`; reference outer via `each.*`

```hcl
# ✅ GOOD - explicit iterator rename removes ambiguity
resource "aws_security_group" "this" {
  for_each = var.security_groups

  name = each.key

  dynamic "ingress" {
    for_each = each.value.rules
    iterator = rule
    content {
      from_port   = rule.value.from_port
      to_port     = rule.value.to_port
      protocol    = rule.value.protocol
      description = each.value.description  # outer iterator clear
    }
  }
}
```

---

<a id="section-code-patterns-provisioners-as-last-resort"></a>
### Provisioners as Last Resort

| Goal | Use |
|------|-----|
| Instance bootstrap | `user_data` + cloud-init via `templatefile()` |
| Orchestration with explicit re-run (1.4+) | `terraform_data` + `triggers_replace` (list; `null_resource` uses `triggers` map) |
| Ongoing OS config | External: Ansible / SSM Run Command / SSM State Manager |
| Last-resort one-shot | `terraform_data` + `provisioner` (1.4+) or `null_resource` (pre-1.4) |

**Provisioner costs (`local-exec` + `remote-exec`):**

- ❌ Non-idempotent — re-runs duplicate side effects
- ❌ Create-only — updates don't re-run; `when = destroy` is fragile
- ❌ `remote-exec` needs SSH/WinRM from runner to target
- ❌ No drift detection — Terraform can't observe what scripts changed
- ❌ Script stdout/stderr leaks to CI logs; `sensitive` won't redact it

**❌ DON'T — `null_resource` for bootstrap on 1.4+:**

```hcl
resource "null_resource" "bootstrap" {
  provisioner "local-exec" {
    command = "ssh ec2-user@${aws_instance.web.public_ip} 'bash setup.sh'"
  }
}
```

**✅ DO — bootstrap via `user_data` + cloud-init:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.al2023.id
  instance_type = "t3.small"
  user_data = templatefile("${path.module}/cloud-init.yaml", {
    app_version = var.app_version
  })
  user_data_replace_on_change = true
}
```

**✅ DO — declarative orchestration on 1.4+:**

```hcl
resource "terraform_data" "migration" {
  triggers_replace = [aws_rds_cluster.this.id, var.schema_version]

  provisioner "local-exec" {
    command = "./run-migration.sh"
  }
}
```

---

<a id="section-code-patterns-version-management"></a>
### Version Management

<a id="section-code-patterns-version-constraint-syntax"></a>
#### Version Constraint Syntax

```hcl
# Exact version (avoid unless necessary - inflexible)
version = "5.0.0"

# Pessimistic constraint (recommended for stability)
# The rightmost component is the one that's allowed to increment.
version = "~> 5.0"      # 5.x: >= 5.0, < 6.0 — allows 5.1, 5.2, 5.99
version = "~> 5.0.1"    # 5.0.x patches only: >= 5.0.1, < 5.1.0

# Range constraints
version = ">= 5.0, < 6.0"     # Any 5.x version
version = ">= 5.0.0, < 5.1.0" # Specific minor version range

# Minimum version
version = ">= 5.0"  # Any version 5.0 or higher (risky - breaking changes)

# Latest (avoid in production - unpredictable)
# No version specified = always use latest available
```

<a id="section-code-patterns-versioning-strategy-by-component"></a>
#### Versioning Strategy by Component

**Terraform itself:**
```hcl
# versions.tf
terraform {
  # Pin to minor version, allow patch updates
  required_version = "~> 1.9.0"  # Allows 1.9.x
}
```

**Providers:**
```hcl
# versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Pin major version, allow minor/patch updates
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}
```

**Modules:**
```hcl
# Production - pin exact version
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"  # Exact version for production stability
}

# Development - allow flexibility
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1.0"  # Allow patch updates in dev
}
```

<a id="section-code-patterns-update-strategy"></a>
#### Update Strategy

**Security patches:**
- Update immediately
- Test in dev → stage → prod
- Prioritize provider and Terraform core updates

**Minor versions:**
- Regular maintenance windows (monthly/quarterly)
- Review changelog for breaking changes
- Test thoroughly before production

**Major versions:**
- Planned upgrade cycles
- Dedicated testing period
- May require code changes
- Update in phases: dev → stage → prod

<a id="section-code-patterns-version-management-workflow"></a>
#### Version Management Workflow

```hcl
# Step 1: Lock versions in versions.tf
terraform {
  required_version = "~> 1.9.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Step 2: Generate lock file (commit this)
terraform init
# Creates .terraform.lock.hcl with exact versions used

# Step 3: Update providers when needed
terraform init -upgrade
# Updates to latest within constraints

# Step 4: Review and test changes before committing
terraform plan
```

<a id="section-code-patterns-example-versionstf-template"></a>
#### Example versions.tf Template

```hcl
terraform {
  # Terraform version
  required_version = "~> 1.9.0"

  # Provider versions
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
    null = {
      source  = "hashicorp/null"
      version = "~> 3.2"
    }
  }

  # Backend configuration (optional here, often in backend.tf)
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "infrastructure/terraform.tfstate"
    region = "us-east-1"
  }
}
```

---

<a id="section-code-patterns-refactoring-patterns"></a>
### Refactoring Patterns

<a id="section-code-patterns-terraform-version-upgrades"></a>
#### Terraform Version Upgrades

<a id="section-code-patterns-012013--1x-migration-checklist"></a>
##### 0.12/0.13 → 1.x Migration Checklist

**Replace legacy patterns with modern equivalents:**

- [ ] Replace `element(concat(...))` with `try()`
- [ ] Add `nullable = false` to variables that shouldn't accept null
- [ ] Use `optional()` in object types for optional attributes
- [ ] Add `validation` blocks to variables with constraints
- [ ] Migrate secrets to write-only arguments (Terraform 1.11+)
- [ ] Use `moved` blocks for resource refactoring (Terraform 1.1+)
- [ ] Consider cross-variable validation (Terraform 1.9+)

**Example migration:**

```hcl
# Before (0.12 style)
output "security_group_id" {
  value = element(concat(aws_security_group.this[*].id, [""]), 0)
}

variable "config" {
  type = object({
    name = string
    size = number
  })
}

# After (1.x style)
output "security_group_id" {
  description = "The ID of the security group"
  value       = try(aws_security_group.this[0].id, "")
}

variable "config" {
  description = "Configuration settings"
  type = object({
    name = string
    size = optional(number, 100)  # Optional with default
  })
  nullable = false  # Don't accept null
}
```

<a id="section-code-patterns-secrets-remediation"></a>
#### Secrets Remediation

Move secret material out of state into external secret management. Canonical depth lives in [AWS Security and Compliance](#section-security-compliance) — patterns below are the minimum refactor shape.

❌ BAD — both shapes land the secret in state:

```hcl
# random_password.result lives in state
resource "random_password" "db" {
  length  = 16
  special = true
}
resource "aws_db_instance" "this" {
  password = random_password.db.result
}

# var + sensitive = true still writes to state (sensitive only masks display)
variable "db_password" {
  type      = string
  sensitive = true
}
resource "aws_db_instance" "this" {
  password = var.db_password
}
```

✅ GOOD — 1.11+ write-only argument, secret created outside Terraform:

```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod-database-password"
}

resource "aws_db_instance" "this" {
  engine   = "mysql"
  username = "admin"
  # password_wo: resource argument stays out of state (1.11+).
  # Data source still reads secret_string into state on refresh.
  # For true state exclusion: ephemeral (1.10+), manage_master_user_password, or CI env var.
  password_wo = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

Pre-1.11 fallback: use the same data source without `password_wo`; rotation must happen outside Terraform.

**Migration steps:**

1. Create secret in AWS Secrets Manager outside Terraform
2. Replace `random_password` / variable with `data "aws_secretsmanager_secret_version"`
3. On 1.11+: use `password_wo`
4. Apply, then `terraform show | grep -i password` — must be empty

---

<a id="section-code-patterns-locals-for-dependency-management"></a>
### Locals for Dependency Management

**Use locals to hint explicit resource deletion order:**

```hcl
# ✅ GOOD - Forces correct deletion order
# Ensures subnets deleted before secondary CIDR blocks

locals {
  # References secondary CIDR first, falling back to VPC
  # This forces Terraform to delete subnets before CIDR association
  vpc_id = try(
    aws_vpc_ipv4_cidr_block_association.this[0].vpc_id,
    aws_vpc.this.id,
    ""
  )
}

resource "aws_vpc" "this" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc_ipv4_cidr_block_association" "this" {
  count = var.add_secondary_cidr ? 1 : 0

  vpc_id     = aws_vpc.this.id
  cidr_block = "10.1.0.0/16"
}

resource "aws_subnet" "public" {
  # Uses local instead of direct reference
  # Creates implicit dependency on CIDR association
  vpc_id     = local.vpc_id
  cidr_block = "10.1.0.0/24"
}

# Without local: Terraform might try to delete CIDR before subnets → ERROR
# With local: Subnets deleted first, then CIDR association, then VPC ✓
```

**Common use cases:**
- VPC with secondary CIDR blocks
- Resources depending on optional configurations
- Complex deletion-order requirements

---

<a id="section-code-patterns-llm-mistake-checklist--code-patterns"></a>
### LLM Mistake Checklist — Code Patterns

Common model mistakes when generating HCL. Correct these before returning code:

- defaults to `count` for every collection — prefer `for_each` with stable keys whenever identity matters
- omits `moved` blocks during rename/refactor, silently turning the change into destroy/create
- builds `for_each` keys from computed IDs not known until apply — planning will fail
- uses list index as long-lived identity (`count.index`) instead of business-meaningful keys
- marks variables `sensitive = true` and assumes the value stays out of state — on 1.11+ use `write_only` / `*_wo` arguments
- falls back to `element(concat(...))` instead of `try()` on 0.12.20+
- accepts untyped `map(any)` / `any` for long-lived module contracts instead of `optional()` with typed defaults (1.3+)
- suggests `terraform state mv` where `moved` blocks are safer and reviewable
- recommends ad-hoc CLI `terraform import` instead of declarative `import` blocks (1.5+)
- emits an exact `version = "5.0.0"` pin where `~> 5.0` would be more maintainable
- silently emits 1.11+ features (S3 native lock, `write_only`, `removed`) without checking the runtime floor
- uses `nonsensitive()` to "fix" a sensitive value appearing in plan output — this leaks secrets into CI artifacts
- conflates `sensitive = true` with `ephemeral` (1.10+); only `ephemeral` actually stays out of state
- writes a `moved` block expecting it to cross provider boundaries; it cannot
- leaves `moved` blocks inside a module that itself is being removed — the moves silently no-op, resources get destroyed
- emits CLI `terraform import` in automation when declarative `import` blocks (1.5+) give a reviewable, VCS-tracked alternative
- emits `ignore_changes = all` or broad ignore lists to silence plan output instead of diagnosing drift root cause
- uses `check` block expecting it to block apply; `check` is advisory, emits warnings only. Use `precondition`/`postcondition` to gate.
- uses `each.value` inside a `dynamic` block intending the outer iterator — shadowed by the inner block name; rename with `iterator = ...`
- emits hardcoded cloud IDs/ARNs (`vpc-0abc...`, pattern-matched `arn:aws:iam::` patterns) from training data instead of using data sources or input variables
- pairs `password_wo` with `aws_secretsmanager_secret_version` — the data source still reads `secret_string` into state on refresh. Use `ephemeral` (1.10+) or CI-injected env var.
- iterates `dynamic` blocks over `toset(...)` of maps/objects — the set's undefined ordering causes non-deterministic block ordering in the plan diff; sort the list or use a map keyed by a stable field

---

**Back to:** [Core Workflow](#core-workflow)

---

<a id="section-testing-frameworks"></a>
## Testing and Validation

> **Purpose:** Detailed guides for Terraform testing frameworks

This section provides in-depth guidance on testing frameworks for Infrastructure as Code. For the decision matrix and high-level overview, see the [core workflow](#core-workflow).

---

<a id="section-testing-frameworks-static-analysis"></a>
### Static Analysis

Start with formatting and validation using approved locally available tools.


<a id="section-testing-frameworks-what-each-tool-checks"></a>
#### What Each Tool Checks

- **`terraform fmt`** - Code formatting consistency
- **`terraform validate`** - Syntax and internal consistency
- **`TFLint`** - Best practices, provider-specific rules

<a id="section-testing-frameworks-when-to-use"></a>
#### When to Use

Run formatting and validation before committing Terraform changes. Add TFLint only when it is approved and already available.

---

<a id="section-testing-frameworks-plan-testing"></a>
### Plan Testing

<a id="section-testing-frameworks-what-terraform-plan-validates"></a>
#### What terraform plan Validates

- Verify expected resources will be created/modified/destroyed
- Catch provider authentication issues
- Validate variable combinations
- Review before applying

<a id="section-testing-frameworks-in-cicd"></a>
#### In CI/CD

```bash
terraform init
terraform plan -out=tfplan

# Optionally: Convert plan to JSON and validate with tools
terraform show -json tfplan | jq '.'
```

<a id="section-testing-frameworks-limitations"></a>
#### Limitations

- Doesn't deploy real infrastructure
- Can't catch runtime issues (IAM permissions, network connectivity)
- Won't find resource-specific bugs

---

<a id="section-testing-frameworks-native-terraform-tests"></a>
### Native Terraform Tests

**Available:** Terraform 1.6+

<a id="section-testing-frameworks-when-to-use-2"></a>
#### When to Use

- Team primarily works in HCL
- Testing logical operations and module behavior
- Want to avoid external testing dependencies

<a id="section-testing-frameworks-basic-structure"></a>
#### Basic Structure

> **Test discovery:** `terraform test` finds `*.tftest.hcl` files under `tests/` relative to the module root. Use `-filter=<path>` to scope to a specific file.

```hcl
# tests/s3_bucket.tftest.hcl
run "create_bucket" {
  command = apply

  assert {
    condition     = aws_s3_bucket.main.bucket != ""
    error_message = "S3 bucket name must be set"
  }
}

run "verify_encryption" {
  command = apply  # `rule` is a set; use `one(...)` to extract the singleton

  assert {
    condition     = one(aws_s3_bucket_server_side_encryption_configuration.main.rule).apply_server_side_encryption_by_default[0].sse_algorithm == "AES256"
    error_message = "Bucket must use AES256 encryption"
  }
}
```


<a id="section-testing-frameworks-critical-validate-resource-schemas-first"></a>
#### Critical: Validate Resource Schemas First

**Inspect the exact provider schema using available local tooling before writing tests:**

```bash
# Use only in an already initialized, trusted working directory.
terraform providers schema -json > provider-schema.json
# Inspect the target resource attributes and nested block nesting_mode.
# If providers are unavailable, use approved local schema documentation
# and report that schema verification was not executed.
```

Block-type distinctions the LLM must verify against the real schema:

- **set** — unordered, cannot index with `[0]`
- **list** — ordered, indexable
- **computed** attribute — only known after apply

**Common Schema Patterns:**

| AWS Resource | Block Type | Indexing |
|--------------|------------|----------|
| `rule` in `aws_s3_bucket_server_side_encryption_configuration` | **set** | ❌ Cannot use `[0]` |
| `transition` in `aws_s3_bucket_lifecycle_configuration` | **set** | ❌ Cannot use `[0]` |
| `noncurrent_version_expiration` in lifecycle | **nested block (MaxItems=1)** — list-of-1 | ✅ Can use `[0]` |

<a id="section-testing-frameworks-working-with-set-type-blocks"></a>
#### Working with Set-Type Blocks

**Problem:** Cannot index sets with `[0]`
```hcl
# ❌ WRONG: This will fail
condition = aws_s3_bucket_server_side_encryption_configuration.this.rule[0].bucket_key_enabled == true
# Error: Cannot index a set value
```

**Solution 1:** Use `command = apply` to materialize the set
```hcl
run "test_encryption" {
  command = apply  # Creates real/mocked resources

  assert {
    # Now the set is materialized and can be checked
    condition     = length([for rule in aws_s3_bucket_server_side_encryption_configuration.this.rule :
                             rule.bucket_key_enabled if rule.bucket_key_enabled == true]) > 0
    error_message = "Bucket key should be enabled"
  }
}
```

**Solution 2:** Check at resource level (avoid accessing nested blocks)
```hcl
run "test_encryption_exists" {
  command = plan

  assert {
    # Check that the resource exists without accessing set members
    condition     = aws_s3_bucket_server_side_encryption_configuration.this != null
    error_message = "Encryption configuration should be created"
  }
}
```

**Solution 3:** Use for expressions (works in apply mode)
```hcl
run "test_encryption_algorithm" {
  command = apply

  assert {
    condition     = alltrue([
      for rule in aws_s3_bucket_server_side_encryption_configuration.this.rule :
      alltrue([
        for config in rule.apply_server_side_encryption_by_default :
        config.sse_algorithm == "AES256"
      ])
    ])
    error_message = "Encryption should use AES256"
  }
}
```

<a id="section-testing-frameworks-command--plan-vs-command--apply"></a>
#### command = plan vs command = apply

| Goal | Mode | Why |
|------|------|-----|
| Input-derived attribute (bucket name from `var.bucket`) | `plan` | value known before refresh |
| Variable default / validation | `plan` | fast, no resource creation |
| Computed attribute (ARN, generated name, cloud ID) | `apply` | only known after provider round-trip |
| Set-type nested block | `apply` | materializes the set so `for` expressions resolve |
| Real behavior / mocked provider responses | `apply` | runs the actual create path |

```hcl
# ✅ plan — input-derived
run "test_input" {
  command = plan
  variables { bucket = "test-bucket" }
  assert {
    condition     = aws_s3_bucket.this.bucket == "test-bucket"
    error_message = "Bucket name should match input"
  }
}

# ✅ apply — computed
run "test_prefix" {
  command = apply
  variables { bucket_prefix = "test-" }
  assert {
    condition     = startswith(aws_s3_bucket.this.bucket, "test-")
    error_message = "Bucket name should start with prefix"
  }
}
```

❌ Asserting a computed value in `plan` mode → `Condition expression could not be evaluated at this time`. Fix: switch the `run` block to `command = apply`, or assert a different attribute that is known at plan.

<a id="section-testing-frameworks-with-mocking-17"></a>
#### With Mocking (1.7+)

```hcl
mock_provider "aws" {
  mock_resource "aws_instance" {
    defaults = {
      id  = "i-mock123"
      arn = "arn:aws:ec2:us-east-1:123456789:instance/i-mock123"
    }
  }
}
```

<a id="section-testing-frameworks-pros"></a>
#### Pros

- Native HCL syntax (familiar to Terraform users)
- No external dependencies
- Fast execution with mocks
- Good for unit testing module logic

<a id="section-testing-frameworks-cons"></a>
#### Cons

- Verify native-test features against the selected Terraform CLI version.
- Limited ecosystem/examples
- Mocking doesn't catch real-world AWS behavior

---

<a id="section-testing-frameworks-complete-test-examples-following-best-practices"></a>
#### Complete Test Examples (Following Best Practices)

<a id="section-testing-frameworks-example-1-s3-bucket-tests"></a>
##### Example 1: S3 Bucket Tests

```hcl
# tests/unit/s3_bucket.tftest.hcl

mock_provider "aws" {}  # Zero cost with mocks

# Test 1: Input validation (fast, plan mode)
run "validate_bucket_name" {
  command = plan

  variables {
    bucket = "my-test-bucket"
  }

  assert {
    condition     = aws_s3_bucket.this.bucket == "my-test-bucket"
    error_message = "Bucket name should match input"
  }
}

# Test 2: Encryption defaults (apply mode for set access)
run "verify_default_encryption" {
  command = apply

  variables {
    bucket = "encrypted-bucket"
  }

  assert {
    # Using for expression to check set-type block
    condition = alltrue([
      for rule in aws_s3_bucket_server_side_encryption_configuration.this.rule :
      alltrue([
        for config in rule.apply_server_side_encryption_by_default :
        config.sse_algorithm == "AES256"
      ])
    ])
    error_message = "Default encryption should be AES256"
  }

  assert {
    # Check bucket key at rule level
    condition = alltrue([
      for rule in aws_s3_bucket_server_side_encryption_configuration.this.rule :
      rule.bucket_key_enabled == true
    ])
    error_message = "Bucket key should be enabled"
  }
}

# Test 3: Computed values (apply mode required)
run "verify_generated_name" {
  command = apply

  variables {
    bucket_prefix = "test-"
  }

  assert {
    condition     = startswith(aws_s3_bucket.this.bucket, "test-")
    error_message = "Generated bucket name should have prefix"
  }

  assert {
    condition     = length(aws_s3_bucket.this.bucket) > 5
    error_message = "Bucket name should be generated"
  }
}
```

<a id="section-testing-frameworks-example-2-lifecycle-rules"></a>
##### Example 2: Lifecycle Rules

```hcl
# tests/unit/lifecycle.tftest.hcl

mock_provider "aws" {}

run "verify_lifecycle_transitions" {
  command = apply  # Authorized integration test; sets alone do not require apply

  variables {
    bucket = "lifecycle-bucket"
    lifecycle_rules = [{
      id      = "archive"
      enabled = true
      transition = [
        { days = 90, storage_class = "GLACIER" },
        { days = 180, storage_class = "DEEP_ARCHIVE" }
      ]
    }]
  }

  assert {
    # Check that both transitions exist using for expression
    condition = length([
      for rule in aws_s3_bucket_lifecycle_configuration.this.rule :
      rule.id if rule.id == "archive"
    ]) == 1
    error_message = "Lifecycle rule should exist"
  }

  assert {
    # Verify transition count using length
    condition = alltrue([
      for rule in aws_s3_bucket_lifecycle_configuration.this.rule :
      length(rule.transition) == 2
    ])
    error_message = "Should have 2 transitions"
  }
}
```


<a id="section-testing-frameworks-best-practices-summary"></a>
### Best Practices Summary

<a id="section-testing-frameworks-for-all-frameworks"></a>
#### For All Frameworks

1. **Start with static analysis** - Always free, always fast
2. **Use unique identifiers** - Prevent resource conflicts
3. **Tag test resources** - Enable tracking and cleanup
4. **Separate test accounts** - Isolate test infrastructure
5. **Implement TTL** - Automatic resource cleanup

<a id="section-testing-frameworks-framework-selection"></a>
#### Framework Selection

```
Quick syntax check? → terraform validate + fmt
AWS security review? → configuration, proposed plan, and supplied standards
Terraform 1.6+, simple logic? → Native tests
Complex AWS integration? → Authorized native apply tests in an isolated test environment
```

<a id="section-testing-frameworks-cost-optimization"></a>
#### Cost Optimization

1. Use mocking for unit tests
2. Implement resource TTL tags
3. Run integration tests only on main branch
4. Use smaller instance types in tests
5. Share test resources when safe

---

<a id="section-testing-frameworks-llm-mistake-checklist--testing"></a>
### LLM Mistake Checklist — Testing

Common model mistakes when generating test code:

- asserts computed values (ARNs, generated names, cloud-assigned IDs) in `command = plan` mode — must use `command = apply`
- indexes set-type nested blocks with `[0]` — sets are unordered, use `for` expressions or `command = apply` to materialize
- treats mocked-provider tests as integration coverage — mocks validate logic only, not provider behavior
- forgets to exercise `validation` blocks with invalid inputs — only tests the happy path
- skips idempotency (`terraform plan -detailed-exitcode` after apply) — the most common regression detector
- asserts on Terraform syntax instead of module behavior (`terraform validate` already covers syntax)
- runs expensive real-cloud integration tests on every commit instead of gating them behind main/scheduled
- omits cleanup, leaving orphaned resources billed against the test account

---

**Back to:** [Core Workflow](#core-workflow)

---

<a id="section-security-compliance"></a>
## AWS Security and Compliance

> **Purpose:** Security best practices and compliance patterns for Terraform

This section provides security hardening guidance and compliance review practices for infrastructure-as-code.

---

<a id="section-security-compliance-aws-security-review"></a>
### AWS Security Review

Review the configuration and proposed plan for least-privilege IAM, constrained trust relationships, intentional public exposure, encryption at rest and TLS, secret handling, and protected state. Use the AWS security examples below and the organization's supplied standards. Record findings and unresolved questions for review. This skill does not require a security scanner or policy engine.

Terraform fmt and validate check formatting and configuration consistency; they do not establish security or compliance. Review plans and native test results alongside the applicable architecture and standards. Preserve existing organization controls; do not invent new tool dependencies or report unperformed security checks as passed.

---

<a id="section-security-compliance-common-security-issues"></a>
### Common Security Issues

<a id="section-security-compliance--dont-store-secrets-in-variables"></a>
#### ❌ DON'T: Store Secrets in Variables

```hcl
# BAD: Secret in plaintext
variable "database_password" {
  type    = string
  default = "SuperSecret123!"  # ❌ Never do this
}
```

<a id="section-security-compliance--do-use-secrets-manager"></a>
#### ✅ DO: Use Secrets Manager

```hcl
# Good: Reference secrets from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/password"
}

resource "aws_db_instance" "this" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

<a id="secret-string-state-caveat"></a>

> **Note — data source `secret_string` persists to state:** The `aws_secretsmanager_secret_version` data source reads `secret_string` into Terraform state during refresh. `password_wo` (AWS provider v5.71+, Terraform 1.11+) keeps the **resource argument** out of state, but the data source still persists the value. For true state exclusion:
>
> - Prefer `manage_master_user_password = true` (AWS-managed, for RDS)
> - Use `ephemeral` providers/resources (Terraform 1.10+)
> - Inject via CI environment variable outside Terraform
>
> Examples below use the data-source pattern; apply one of the alternatives above when the value must not land in state.

<a id="section-security-compliance--dont-use-default-vpc"></a>
#### ❌ DON'T: Use Default VPC

```hcl
# BAD: Default VPC has public subnets
resource "aws_instance" "app" {
  ami           = "ami-12345"
  subnet_id     = "subnet-default"  # ❌ Avoid default resources
}
```

<a id="section-security-compliance--do-create-dedicated-vpcs"></a>
#### ✅ DO: Create Dedicated VPCs

```hcl
# Good: Custom VPC with private subnets
resource "aws_vpc" "this" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.this.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
}
```

<a id="section-security-compliance--dont-skip-encryption"></a>
#### ❌ DON'T: Skip Encryption

```hcl
# BAD: Unencrypted S3 bucket
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
  # ❌ No encryption configured
}
```

<a id="section-security-compliance--do-enable-encryption-at-rest"></a>
#### ✅ DO: Enable Encryption at Rest

```hcl
# Good: Enable encryption
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

> **SSE-S3 vs SSE-KMS:** `AES256` above is SSE-S3 (AWS-managed key, no per-request audit trail in CloudTrail). For regulated workloads (HIPAA/PCI/FedRAMP), prefer `aws:kms` with a customer-managed CMK + key rotation enabled.

<a id="section-security-compliance--dont-open-security-groups-to-internet"></a>
#### ❌ DON'T: Open Security Groups to Internet

```hcl
# BAD: Security group open to internet on all protocols
resource "aws_security_group_rule" "allow_all" {
  type              = "ingress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"            # ❌ All protocols (worst case)
  cidr_blocks       = ["0.0.0.0/0"]  # ❌ Never do this
  security_group_id = aws_security_group.this.id
}
```

<a id="section-security-compliance--do-use-least-privilege-security-groups"></a>
#### ✅ DO: Use Least-Privilege Security Groups

```hcl
# Good: Restrict to specific ports and sources
resource "aws_security_group_rule" "app_https" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = ["10.0.0.0/16"]  # ✅ Internal only
  security_group_id = aws_security_group.this.id
}
```

<a id="section-security-compliance--dont-use-inline-security-group-rules"></a>
#### ❌ DON'T: Use Inline Security Group Rules

```hcl
# BAD: Inline ingress/egress blocks
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web server security group"
  vpc_id      = aws_vpc.this.id

  ingress {  # ❌ Inline rules cause issues
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {  # ❌ Avoid inline rules
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

<a id="section-security-compliance--do-use-separate-security-group-rule-resources"></a>
#### ✅ DO: Use Separate Security Group Rule Resources

**Preferred (AWS provider v5+):** Use `aws_vpc_security_group_ingress_rule` / `aws_vpc_security_group_egress_rule`:

```hcl
# Best: Modern individual rule resources (AWS provider v5+)
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web server security group"
  vpc_id      = aws_vpc.this.id

  # No inline rules - managed separately
}

resource "aws_vpc_security_group_ingress_rule" "web_https" {
  security_group_id = aws_security_group.web.id
  description       = "HTTPS from internal VPC"
  cidr_ipv4         = "10.0.0.0/16"
  from_port         = 443
  to_port           = 443
  ip_protocol       = "tcp"
}

# Scope egress to needed ports when possible — avoid 0.0.0.0/0 with ip_protocol = "-1"
resource "aws_vpc_security_group_egress_rule" "web_https_out" {
  security_group_id = aws_security_group.web.id
  description       = "HTTPS to external services"
  cidr_ipv4         = "0.0.0.0/0"
  from_port         = 443
  to_port           = 443
  ip_protocol       = "tcp"
}
```

**Also acceptable:** `aws_security_group_rule` (older but still supported):

```hcl
resource "aws_security_group_rule" "web_https_ingress" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = ["10.0.0.0/16"]
  security_group_id = aws_security_group.web.id
}
```

**Why avoid inline rules:**

| Issue | Inline Rules | Separate Resources |
|-------|--------------|-------------------|
| Rule changes | Recreates entire SG (downtime) | Updates only the rule |
| Mixing approaches | Conflicts and overwrites | N/A - consistent pattern |
| Dynamic rules | Complex `dynamic` blocks needed | Native `for_each` per resource |
| State management | Rules buried in SG state | Each rule tracked separately |
| Conditional rules | Complex nested dynamics | Simple `count` or `for_each` |

---

<a id="section-security-compliance-compliance-review"></a>
### Compliance Review

Map applicable supplied requirements to the relevant Terraform resources and proposed changes. For each requirement, record the control, evidence, result, and any exception requiring review. Use native Terraform variable validation, preconditions, and tests for supported configuration assertions; these are not a substitute for runtime verification or a compliance assessment. Keep the approved review and evidence-retention process without introducing a policy engine.

---

<a id="section-security-compliance-secrets-management"></a>
### Secrets Management

<a id="section-security-compliance-aws-secrets-manager-pattern"></a>
#### AWS Secrets Manager Pattern

See the [data-source `secret_string` persistence caveat](#secret-string-state-caveat) above — both `random_password.result` and data-source reads of `secret_string` land in Terraform state. The recommended RDS pattern avoids both.

```hcl
# Recommended: let RDS generate and manage the master password in Secrets Manager
resource "aws_kms_key" "db" {
  description             = "KMS CMK for RDS-managed master password"
  enable_key_rotation     = true
  deletion_window_in_days = 30
}

resource "aws_db_instance" "this" {
  # Option 1 (recommended): AWS-managed master password in Secrets Manager
  manage_master_user_password   = true
  master_user_secret_kms_key_id = aws_kms_key.db.arn

  # Option 2 (Terraform 1.11+ + AWS provider v5.71+): write-only password
  # password_wo         = ephemeral.random_password.db.result
  # password_wo_version = 1
  # ...
}
```

If you need a manually-managed secret for a non-RDS consumer, keep the value out of state by sourcing it outside Terraform (CI env var, ephemeral resource, or a write-only argument) rather than via `random_password` + a `data` lookup:

```hcl
# Only use this shape when the consumer cannot use manage_master_user_password
# and you are comfortable with the caveat linked above.
resource "aws_secretsmanager_secret" "app_api_key" {
  name                    = "prod/app/api-key"
  description             = "Third-party API key"
  recovery_window_in_days = 30
}

# secret_string populated out-of-band (console, CLI, or a write-only argument on
# providers that support it) — not via random_password stored in state.
```

<a id="section-security-compliance-environment-variables"></a>
#### Environment Variables

```bash
# Never commit these
export TF_VAR_database_password="secret123"
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

**In .gitignore:**

```
*.tfvars
.env
secrets/
```

---

<a id="section-security-compliance-state-file-security"></a>
### State File Security

<a id="section-security-compliance-encrypt-state-at-rest"></a>
#### Encrypt State at Rest

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket       = "my-terraform-state"
    key          = "prod/vpc/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true                                             # Enables SSE on PUT
    kms_key_id   = "arn:aws:kms:us-east-1:ACCOUNT:key/KEY-ID"       # Customer-managed CMK
    use_lockfile = true                                             # Terraform 1.10+
  }
}
```

> **`encrypt = true` alone is SSE-S3 (AWS-managed AES-256 key, no per-request CloudTrail audit trail).** State often holds secrets, so pair `encrypt = true` with `kms_key_id` pointing at a customer-managed CMK. `use_lockfile = true` (Terraform 1.10+) replaces the need for a DynamoDB lock table.

<a id="section-security-compliance-secure-state-bucket"></a>
#### Secure State Bucket

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"
}

# Enable versioning (protect against accidental deletion)
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Enable encryption — customer-managed KMS CMK with bucket key to control request costs
resource "aws_kms_key" "terraform_state" {
  description             = "KMS CMK for Terraform state bucket"
  enable_key_rotation     = true
  deletion_window_in_days = 30
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
    bucket_key_enabled = true
  }
}

# Note: for regulated workloads (HIPAA/PCI/FedRAMP), customer-managed KMS with
# rotation enabled is typically required — SSE-S3 (AES256) is usually insufficient.

# Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

<a id="section-security-compliance-restrict-state-access"></a>
#### Restrict State Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListBucket",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/TerraformRole"
      },
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-terraform-state"
    },
    {
      "Sid": "AllowObjectRW",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/TerraformRole"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::my-terraform-state/*"
    },
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-terraform-state",
        "arn:aws:s3:::my-terraform-state/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

- `s3:ListBucket` must target the bucket ARN; object actions must target `/*` — splitting avoids IAM silently no-op'ing the mismatched pairings.
- `s3:DeleteObject` + `s3:GetObjectVersion` are required to rotate state objects when versioning is enabled.
- The `Deny` statement enforces TLS — any HTTP request is rejected regardless of other grants.

---

<a id="section-security-compliance-iam-best-practices"></a>
### IAM Best Practices

<a id="section-security-compliance--do-use-least-privilege"></a>
#### ✅ DO: Use Least Privilege

```hcl
# Good: Specific permissions only
resource "aws_iam_policy" "app_policy" {
  name = "app-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "arn:aws:s3:::my-app-bucket/*"
      }
    ]
  })
}
```

<a id="section-security-compliance--dont-use-wildcard-permissions"></a>
#### ❌ DON'T: Use Wildcard Permissions

```hcl
# BAD: Overly broad permissions
resource "aws_iam_policy" "bad_policy" {
  policy = jsonencode({
    Statement = [
      {
        Effect   = "Allow"
        Action   = "*"  # ❌ Never use wildcard
        Resource = "*"
      }
    ]
  })
}
```

---

<a id="section-security-compliance-aws-security-map"></a>
#### AWS security map

| Concern | AWS |
| --------- | ----- |
| Secret manager | `aws_secretsmanager_secret` |
| Network firewalling | `aws_security_group` + `aws_vpc_security_group_*_rule` |
| Identity | IAM (`aws_iam_role` / `aws_iam_policy`) |
| Encryption at rest | explicit (SSE / KMS) |

---

<a id="section-security-compliance-compliance-checklists"></a>
### Compliance Checklists

<a id="section-security-compliance-soc-2-compliance"></a>
#### SOC 2 Compliance

- [ ] Encryption at rest for all data stores
- [ ] Encryption in transit (TLS/SSL)
- [ ] IAM policies follow least privilege
- [ ] Logging enabled for all resources
- [ ] MFA required for privileged access (enforced at org/IdP level, not per-resource)
- [ ] Documented AWS configuration and plan security review before apply

<a id="section-security-compliance-hipaa-compliance"></a>
#### HIPAA Compliance

- [ ] PHI encrypted at rest and in transit
- [ ] Access logs enabled
- [ ] Dedicated VPC with private subnets
- [ ] Regular backup and retention policies
- [ ] Audit trail for all infrastructure changes

<a id="section-security-compliance-pci-dss-compliance"></a>
#### PCI-DSS Compliance

- [ ] Network segmentation (separate VPCs)
- [ ] No default passwords
- [ ] Strong encryption algorithms
- [ ] Periodic documented AWS security review
- [ ] Access control and monitoring

---

<a id="section-security-compliance-llm-mistake-checklist--security--compliance"></a>
### LLM Mistake Checklist — Security & Compliance

Common model mistakes to correct before returning security/compliance recommendations:

- assumes `sensitive = true` keeps the value out of state — it only masks display; use `write_only` / `*_wo` arguments on 1.11+ or an external secret lookup
- proposes plaintext defaults in `variable` blocks or committed `.tfvars` "for demo convenience"
- echoes secrets through `provisioner` commands or `local-exec` stdout into CI logs (see [Provisioners as Last Resort](#section-code-patterns) for the broader pattern)
- emits outputs that expose full connection strings or credentials (even when marked `sensitive`)
- mentions a compliance framework (SOC 2, PCI, HIPAA, GDPR, FedRAMP) but provides no enforceable gate — no review, no approval model, no evidence artifact
- confuses security best practices with compliance evidence (an encrypted bucket is not the same as a retained audit artifact proving it)
- omits artifact retention and access controls for plan JSON exports
- ignores data-residency obligations for GDPR/FedRAMP contexts

---

<a id="section-security-compliance-resources"></a>
### Resources

- [AWS Security Best Practices](https://aws.amazon.com/security/best-practices/)

---

**Back to:** [Core Workflow](#core-workflow)

---

<a id="section-quick-reference"></a>
## Command Reference and Troubleshooting

> **Purpose:** Command cheat sheets and decision flowcharts

This section provides quick lookup tables, command references, and decision flowcharts for rapid consultation during development.

---

<a id="section-quick-reference-command-cheat-sheet"></a>
### Command Cheat Sheet

<a id="section-quick-reference-static-analysis"></a>
#### Static Analysis

Use the following Terraform CLI commands:

```bash
# Format and validate
terraform fmt -recursive -check
terraform validate

# Linting
tflint --init && tflint

```

<a id="section-quick-reference-native-tests-16"></a>
#### Native Tests (1.6+)

```bash
# Run all tests
terraform test

# Run tests in specific directory
terraform test -test-directory=tests/unit/

# Verbose output
terraform test -verbose
```

<a id="section-quick-reference-plan-validation"></a>
#### Plan Validation

```bash
# Generate and review plan
terraform plan -out tfplan

# Convert plan to pretty JSON
terraform show -json tfplan | jq -r '.' > tfplan.json

# Check for specific changes
terraform show tfplan | grep "will be created"
```

<a id="section-quick-reference-state-management"></a>
#### State Management

```bash
# View all resources in state
terraform state list

# Show specific resource details
terraform state show aws_instance.web

# Move/rename resource in state (refactoring)
terraform state mv aws_instance.old aws_instance.new
terraform state mv aws_instance.app module.compute.aws_instance.app

# Remove resource from state (keeps actual resource)
terraform state rm aws_instance.temporary

# Import existing resource into state
terraform import aws_instance.web i-1234567890abcdef0

# Import using import blocks (1.5+)
# Define in .tf: import { to = aws_instance.web, id = "i-123..." }
# Note: File must not exist — Terraform refuses to overwrite.
terraform plan -generate-config-out=imported.tf

# Detect configuration drift
terraform plan -refresh-only

# Update state to match reality (no infrastructure changes)
terraform apply -refresh-only

# Backup state to file
terraform state pull > backup-$(date +%Y%m%d).tfstate

# Restore state from backup (DANGEROUS)
terraform state push backup.tfstate

# Force unlock stuck state lock
# Default: prompts for y/N confirmation
terraform force-unlock LOCK_ID

# CI-friendly (skips prompt):
terraform force-unlock -force LOCK_ID
```

<a id="section-quick-reference-state-backend-migration"></a>
#### State Backend Migration

```bash
# Migrate from local to remote backend
# 1. Add backend config to backend.tf
# 2. Run migration
terraform init -migrate-state

# Change backend without migrating state
terraform init -reconfigure

# Pass backend config at runtime
terraform init \
  -backend-config="key=prod/terraform.tfstate" \
  -backend-config="dynamodb_table=terraform-locks"

# Or use config file
terraform init -backend-config=backend-prod.hcl
```

---

<a id="section-quick-reference-decision-flowchart"></a>
### Decision Flowchart

<a id="section-quick-reference-testing-approach-selection"></a>
#### Testing Approach Selection

| Need | Approach |
| --- | --- |
| Formatting and syntax | `terraform fmt -check -recursive` and `terraform validate` |
| AWS security review | Review configuration and plan against supplied standards |
| Input-derived assertions, Terraform 1.6+ | Native tests with explicit `command = plan` |
| Unit tests without cloud resources, Terraform 1.7+ | Supported mock providers |
| Real AWS integration behavior | Authorized native apply tests in an isolated test environment |
| Terraform before 1.6 | Static checks and reviewed plans; native tests are unavailable |

<a id="section-quick-reference-module-development-workflow"></a>
#### Module Development Workflow

```
1. Plan
   ├─ Define inputs (variables.tf)
   ├─ Define outputs (outputs.tf)
   └─ Document purpose (README.md)

2. Implement
   ├─ Create resources (main.tf)
   ├─ Pin versions (versions.tf)
   └─ Add examples (examples/simple, examples/complete)

3. Test
   ├─ Static analysis (validate, fmt, lint)
   ├─ Native tests for supported Terraform versions
   └─ Integration tests (examples/)

4. Document
   ├─ Update README with usage
   ├─ Document inputs/outputs
   └─ Add CHANGELOG

5. Publish
   ├─ Tag version (git tag v1.0.0)
   ├─ Push to registry
   └─ Announce changes
```

---

<a id="section-quick-reference-version-specific-guidance"></a>
### Version-Specific Guidance

<a id="section-quick-reference-terraform-10-15"></a>
#### Terraform 1.0-1.5

- ❌ No native testing framework
- ✅ Use static checks and reviewed plans; native tests require Terraform 1.6+
- ✅ Focus on static analysis
- ✅ terraform plan validation

<a id="section-quick-reference-terraform-16"></a>
#### Terraform 1.6+

- ✅ NEW: Native `terraform test` framework with `.tftest.hcl` files
- ✅ Use native tests for module behavior and validation
- ✅ Use authorized native apply tests for real AWS integration behavior
- ✅ Import blocks from 1.5 available for declarative imports with `-generate-config-out`

<a id="section-quick-reference-terraform-17"></a>
#### Terraform 1.7+

- ✅ NEW: Mock providers for unit testing
- ✅ Reduce costs with mocking
- ✅ Use real integration tests for final validation
- ✅ Faster test iteration


---

<a id="section-quick-reference-troubleshooting-guide"></a>
### Troubleshooting Guide

<a id="section-quick-reference-issue-tests-fail-in-ci-but-pass-locally"></a>
#### Issue: Tests fail in CI but pass locally

**Symptoms:**
- Tests pass on your machine
- Same tests fail in GitHub Actions/GitLab CI

**Common Causes:**
1. Different Terraform/provider versions
2. Different environment variables
3. Different AWS credentials/permissions

**Solution:**

```hcl
# versions.tf - Pin versions explicitly
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"  # Pin to major version
    }
  }
}
```


<a id="section-quick-reference-issue-state-lock-is-stuck"></a>
#### Issue: State lock is stuck

**Symptoms:**
```
Error: Error acquiring the state lock
Lock Info:
  ID: a1b2c3d4-e5f6-7890-abcd-ef1234567890
  Who: user@hostname
  Created: 2026-01-20 12:00:00
```

**Common Causes:**
1. Terraform process crashed or was killed
2. Network interruption during operation
3. CI/CD job terminated unexpectedly

**Solution:**

```bash
# 1. Verify the operation is NOT actually running
# Check the host mentioned in lock info
ssh user@hostname "ps aux | grep terraform"

# Or check CI/CD job status
# GitHub Actions: Check workflow runs
# GitLab CI: Check pipeline jobs

# 2. Only if confirmed the operation is not running:
terraform force-unlock LOCK_ID

# 3. Document why you unlocked
echo "Force-unlocked due to CI job timeout" > unlock-notes.txt
```

**Prevention:**
```yaml
# GitHub Actions - Use concurrency control
concurrency:
  group: terraform-${{ github.ref }}
  cancel-in-progress: false  # Wait, don't cancel
```

<a id="section-quick-reference-issue-state-file-is-corrupted-or-lost"></a>
#### Issue: State file is corrupted or lost

**Symptoms:**
- Error: "state snapshot was created by Terraform v1.8.0"
- Error: "Failed to load state"
- State file missing or unreadable

**Solutions:**

**If versioning enabled (S3):**
```bash
# List versions
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix prod/terraform.tfstate

# Restore previous version
aws s3api get-object \
  --bucket my-terraform-state \
  --key prod/terraform.tfstate \
  --version-id PREVIOUS_VERSION_ID \
  terraform.tfstate.restored

# Push restored state
terraform state push terraform.tfstate.restored
```

**If no backup exists:**
```bash
# Recreate state by importing all resources
terraform import aws_vpc.main vpc-12345678
terraform import aws_subnet.private[0] subnet-abcd1234
# ... continue for all resources

# Or use import blocks (1.5+)
# In .tf file:
# import { to = aws_vpc.main, id = "vpc-12345678" }
# Note: File must not exist — Terraform refuses to overwrite.
terraform plan -generate-config-out=imported.tf
```

<a id="section-quick-reference-issue-configuration-drift-detected"></a>
#### Issue: Configuration drift detected

**Symptoms:**
```
Note: Objects have changed outside of Terraform
```

**Cause:** Manual changes in console or by other tools

**Solutions:**

```bash
# View drift
terraform plan -refresh-only

# Accept drift (update state to match reality)
terraform apply -refresh-only

# Or fix drift (update resources to match config)
terraform apply

# Prevent drift with detective controls
# - Enable CloudTrail
# - Use AWS Config rules
# - Regular terraform plan in CI
```

<a id="section-quick-reference-issue-cannot-migrate-state-between-backends"></a>
#### Issue: Cannot migrate state between backends

**Symptoms:**
- `terraform init -migrate-state` fails
- Backend authentication errors

**Solutions:**

```bash
# Ensure credentials are configured
export AWS_PROFILE=terraform
# or
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...

# Try migration again
terraform init -migrate-state

# If still failing, manual migration:
# 1. Pull state from old backend
terraform state pull > old-state.json

# 2. Switch backend config
# Edit backend.tf

# 3. Initialize new backend
terraform init -reconfigure

# 4. Push state to new backend
terraform state push old-state.json
```

---

<a id="section-quick-reference-migration-paths"></a>
### Migration Paths

<a id="section-quick-reference-from-manual-testing--automated"></a>
#### From Manual Testing → Automated

**Phase 1:** Static analysis
```bash
terraform validate
terraform fmt -check
```

**Phase 2:** Plan review
```bash
terraform plan -out=tfplan
# Manual review
```

**Phase 3:** Automated tests
- Native tests (1.6+)

**Phase 4:** CI/CD integration
- GitLab CI
- Reviewed plan and authorized apply through the established workflow


**Example: Native test layout**

```text
tests/
  validation.tftest.hcl  # Plan-mode validation and contract assertions
  integration.tftest.hcl # Authorized AWS integration tests when needed
```


---

<a id="section-quick-reference-pre-commit-checklist"></a>
### Pre-Commit Checklist

<a id="section-quick-reference-formatting--validation"></a>
#### Formatting & Validation

Run these commands before every commit:

```bash
# Format all Terraform files
terraform fmt -recursive

# Validate configuration
terraform validate
```

<a id="section-quick-reference-naming-convention-review"></a>
#### Naming Convention Review

- [ ] All identifiers use `_` not `-`
- [ ] No resource names repeat resource type (no `aws_vpc.main_vpc`)
- [ ] Single-instance resources named `this` or descriptive name
- [ ] Variables have plural names for lists/maps (`subnet_ids` not `subnet_id`)
- [ ] All variables have descriptions
- [ ] All outputs have descriptions
- [ ] Output names follow `{name}_{type}_{attribute}` pattern
- [ ] No double negatives in variable names

<a id="section-quick-reference-code-structure-review"></a>
#### Code Structure Review

- [ ] `count`/`for_each` at top of resource blocks (blank line after)
- [ ] `tags` as last real argument in resources
- [ ] `depends_on` after tags (if used)
- [ ] `lifecycle` at end of resource (if used)
- [ ] Variables ordered: description → type → default → sensitive → nullable → validation
- [ ] Only `#` comments used (no `//` or `/* */`)

<a id="section-quick-reference-modern-features-check"></a>
#### Modern Features Check

- [ ] Using `try()` not `element(concat())`
- [ ] Secrets use write-only arguments or external data sources (not in state)
- [ ] `nullable = false` set on non-null variables
- [ ] `optional()` used in object types where applicable (Terraform 1.3+)
- [ ] Variable validation blocks added where constraints needed
- [ ] Consider cross-variable validation for related variables (Terraform 1.9+)

<a id="section-quick-reference-architecture-review"></a>
#### Architecture Review

- [ ] `terraform.tfvars` only at composition level (not in modules)
- [ ] Remote state configured (never local state)
- [ ] Resource modules don't hardcode values (use variables/data sources)
- [ ] `terraform_remote_state` used for cross-composition dependencies
- [ ] File structure follows standard: main.tf, variables.tf, outputs.tf, versions.tf

<a id="section-quick-reference-documentation-check"></a>
#### Documentation Check

Required documentation for all modules:

- [ ] **README.md exists** with absolute links (Terraform Registry compatibility)
- [ ] **All variables documented** in README with descriptions and types
- [ ] **All outputs documented** in README with descriptions
- [ ] **Usage examples provided** showing how to use the module
- [ ] **Version requirements specified** (Terraform version, provider versions)

---

<a id="section-quick-reference-version-management-quick-reference"></a>
### Version Management Quick Reference

<a id="section-quick-reference-constraint-syntax"></a>
#### Constraint Syntax

| Syntax | Meaning | Use Case |
|--------|---------|----------|
| `"5.0.0"` | Exact version | Avoid (inflexible) |
| `"~> 5.0"` | Pessimistic (>= 5.0, < 6.0 — any 5.x) | Allow minor and patch updates within 5.x |
| `"~> 5.0.1"` | Pessimistic (>= 5.0.1, < 5.1.0 — 5.0.x patches) | Lock to 5.0.x patch updates only |
| `">= 5.0, < 6.0"` | Range | Any 5.x version |
| `">= 5.0"` | Minimum | Risky (breaking changes) |

<a id="section-quick-reference-strategy-by-component"></a>
#### Strategy by Component

| Component | Recommendation | Example |
|-----------|----------------|---------|
| **Terraform** | Pin minor, allow patch | `required_version = "~> 1.9.0"` |
| **Providers** | Pin major, allow minor/patch | `version = "~> 5.0"` |
| **Modules (prod)** | Pin exact version | `version = "5.1.2"` |
| **Modules (dev)** | Allow patch updates | `version = "~> 5.1.0"` |

<a id="section-quick-reference-update-workflow"></a>
#### Update Workflow

```bash
# Step 1: Lock versions initially
terraform init              # Creates .terraform.lock.hcl

# Step 2: Update to latest within constraints
terraform init -upgrade     # Updates providers

# Step 3: Review changes
terraform plan

# Step 4: Commit lock file
git add .terraform.lock.hcl
git commit -m "Update provider versions"
```

<a id="section-quick-reference-update-strategy"></a>
#### Update Strategy

**Security patches:**
- Update immediately
- Test: dev → stage → prod
- Prioritize Terraform core and provider updates

**Minor versions:**
- Regular maintenance (monthly/quarterly)
- Review changelog for breaking changes
- Test thoroughly before production

**Major versions:**
- Planned upgrade cycles
- Dedicated testing period
- May require code changes
- Phased rollout: dev → stage → prod

---

<a id="section-quick-reference-refactoring-quick-reference"></a>
### Refactoring Quick Reference

<a id="section-quick-reference-common-refactoring-patterns"></a>
#### Common Refactoring Patterns

<a id="section-quick-reference-pattern-1-count-to-for_each-migration"></a>
##### Pattern 1: Count to For_Each Migration

**When:** Need stable resource addressing or items might be reordered

```bash
# Step 1: Add for_each, keep count commented
# Step 2: Add moved blocks for each resource
# Step 3: Run terraform plan (should show "moved" not "destroy/create")
# Step 4: Apply changes
# Step 5: Remove commented count
```

**Key principle:** Use `moved` blocks to preserve existing resources

<a id="section-quick-reference-pattern-2-legacy-to-modern-terraform"></a>
##### Pattern 2: Legacy to Modern Terraform

**0.12/0.13 → 1.x checklist:**

- [ ] Replace `element(concat(...))` → `try()`
- [ ] Add `nullable = false` where appropriate
- [ ] Use `optional()` in object types (1.3+)
- [ ] Add `validation` blocks
- [ ] Migrate secrets to write-only arguments (1.11+)
- [ ] Use `moved` blocks for refactoring (1.1+)
- [ ] Add cross-variable validation (1.9+)

<a id="section-quick-reference-pattern-3-secrets-remediation"></a>
##### Pattern 3: Secrets Remediation

**Goal:** Move secrets out of Terraform state

```bash
# Step 1: Create secret in AWS Secrets Manager (outside Terraform)
aws secretsmanager create-secret --name prod-db-password --secret-string "..."

# Step 2: Update Terraform to use data sources
# Step 3: Use write-only argument (Terraform 1.11+)
# Step 4: Remove random_password resource or variable
# Step 5: Apply and verify secret not in state
terraform show | grep -i password  # Should not appear
```

<a id="section-quick-reference-refactoring-decision-tree"></a>
#### Refactoring Decision Tree

```
What are you refactoring?

├─ Resource addressing (count[0] → for_each["key"])
│  └─ Use: moved blocks + for_each conversion
│
├─ Secrets in state
│  └─ Use: AWS Secrets Manager + write-only arguments (1.11+)
│
├─ Legacy Terraform syntax (0.12/0.13)
│  └─ Use: Modern feature checklist above
│
└─ Module structure (rename, reorganize)
   └─ Use: moved blocks to preserve resources
```

<a id="section-quick-reference-migration-best-practices"></a>
#### Migration Best Practices

**Before refactoring:**
1. Backup state file
2. Test in development first
3. Review terraform plan carefully
4. Document what changed and why

**During refactoring:**
1. One change at a time
2. Verify each step with terraform plan
3. Use moved blocks, not destroy/recreate
4. Keep git history clean with logical commits

**After refactoring:**
1. Verify idempotency (plan shows no changes)
2. Test in staging before production
3. Update documentation
4. Communicate changes to team

**For detailed refactoring patterns, see:** [Code Patterns: Refactoring Patterns](#section-code-patterns)

---

<a id="section-quick-reference-common-patterns"></a>
### Common Patterns

<a id="section-quick-reference-resource-naming"></a>
#### Resource Naming

```hcl
# ✅ Good: Descriptive, contextual
resource "aws_instance" "web_server" { }
resource "aws_s3_bucket" "application_logs" { }

# ❌ Bad: Generic
resource "aws_instance" "main" { }
resource "aws_s3_bucket" "bucket" { }
```

<a id="section-quick-reference-variable-naming"></a>
#### Variable Naming

```hcl
# ✅ Good: Context-specific
var.vpc_cidr_block
var.database_instance_class

# ❌ Bad: Generic
var.cidr
var.instance_class
```

<a id="section-quick-reference-file-organization"></a>
#### File Organization

```
Standard module structure:
├── main.tf          # Primary resources
├── variables.tf     # Input variables
├── outputs.tf       # Output values
├── versions.tf      # Provider versions
└── README.md        # Documentation
```

---

**Back to:** [Core Workflow](#core-workflow)

---

<a id="attribution-license"></a>
## Attribution and License

Copyright 2026 Anton Babenko. Adapted from terraform-skill, upstream version 1.17.1, distributed under Apache License 2.0. Modified for local Codex use with AWS and Terraform only. This edition reorganizes the user's shortened file, removes references to deleted guides and external hooks, and uses internal section links. Source names acknowledge provenance and are not instructions to fetch a repository. The following license applies to the adapted upstream material; this is not a template to add to generated modules.

Copyright 2026 Anton Babenko

terraform-best-practices.com
Compliance.tf - Terraform Compliance for Cloud-Native Enterprise

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.


                                 Apache License
                           Version 2.0, January 2004
                        http://www.apache.org/licenses/

   TERMS AND CONDITIONS FOR USE, REPRODUCTION, AND DISTRIBUTION

   1. Definitions.

      "License" shall mean the terms and conditions for use, reproduction,
      and distribution as defined by Sections 1 through 9 of this document.

      "Licensor" shall mean the copyright owner or entity authorized by
      the copyright owner that is granting the License.

      "Legal Entity" shall mean the union of the acting entity and all
      other entities that control, are controlled by, or are under common
      control with that entity. For the purposes of this definition,
      "control" means (i) the power, direct or indirect, to cause the
      direction or management of such entity, whether by contract or
      otherwise, or (ii) ownership of fifty percent (50%) or more of the
      outstanding shares, or (iii) beneficial ownership of such entity.

      "You" (or "Your") shall mean an individual or Legal Entity
      exercising permissions granted by this License.

      "Source" form shall mean the preferred form for making modifications,
      including but not limited to software source code, documentation
      source, and configuration files.

      "Object" form shall mean any form resulting from mechanical
      transformation or translation of a Source form, including but
      not limited to compiled object code, generated documentation,
      and conversions to other media types.

      "Work" shall mean the work of authorship, whether in Source or
      Object form, made available under the License, as indicated by a
      copyright notice that is included in or attached to the work
      (an example is provided in the Appendix below).

      "Derivative Works" shall mean any work, whether in Source or Object
      form, that is based on (or derived from) the Work and for which the
      editorial revisions, annotations, elaborations, or other modifications
      represent, as a whole, an original work of authorship. For the purposes
      of this License, Derivative Works shall not include works that remain
      separable from, or merely link (or bind by name) to the interfaces of,
      the Work and Derivative Works thereof.

      "Contribution" shall mean any work of authorship, including
      the original version of the Work and any modifications or additions
      to that Work or Derivative Works thereof, that is intentionally
      submitted to Licensor for inclusion in the Work by the copyright owner
      or by an individual or Legal Entity authorized to submit on behalf of
      the copyright owner. For the purposes of this definition, "submitted"
      means any form of electronic, verbal, or written communication sent
      to the Licensor or its representatives, including but not limited to
      communication on electronic mailing lists, source code control systems,
      and issue tracking systems that are managed by, or on behalf of, the
      Licensor for the purpose of discussing and improving the Work, but
      excluding communication that is conspicuously marked or otherwise
      designated in writing by the copyright owner as "Not a Contribution."

      "Contributor" shall mean Licensor and any individual or Legal Entity
      on behalf of whom a Contribution has been received by Licensor and
      subsequently incorporated within the Work.

   2. Grant of Copyright License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      copyright license to reproduce, prepare Derivative Works of,
      publicly display, publicly perform, sublicense, and distribute the
      Work and such Derivative Works in Source or Object form.

   3. Grant of Patent License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      (except as stated in this section) patent license to make, have made,
      use, offer to sell, sell, import, and otherwise transfer the Work,
      where such license applies only to those patent claims licensable
      by such Contributor that are necessarily infringed by their
      Contribution(s) alone or by combination of their Contribution(s)
      with the Work to which such Contribution(s) was submitted. If You
      institute patent litigation against any entity (including a
      cross-claim or counterclaim in a lawsuit) alleging that the Work
      or a Contribution incorporated within the Work constitutes direct
      or contributory patent infringement, then any patent licenses
      granted to You under this License for that Work shall terminate
      as of the date such litigation is filed.

   4. Redistribution. You may reproduce and distribute copies of the
      Work or Derivative Works thereof in any medium, with or without
      modifications, and in Source or Object form, provided that You
      meet the following conditions:

      (a) You must give any other recipients of the Work or
          Derivative Works a copy of this License; and

      (b) You must cause any modified files to carry prominent notices
          stating that You changed the files; and

      (c) You must retain, in the Source form of any Derivative Works
          that You distribute, all copyright, patent, trademark, and
          attribution notices from the Source form of the Work,
          excluding those notices that do not pertain to any part of
          the Derivative Works; and

      (d) If the Work includes a "NOTICE" text file as part of its
          distribution, then any Derivative Works that You distribute must
          include a readable copy of the attribution notices contained
          within such NOTICE file, excluding those notices that do not
          pertain to any part of the Derivative Works, in at least one
          of the following places: within a NOTICE text file distributed
          as part of the Derivative Works; within the Source form or
          documentation, if provided along with the Derivative Works; or,
          within a display generated by the Derivative Works, if and
          wherever such third-party notices normally appear. The contents
          of the NOTICE file are for informational purposes only and
          do not modify the License. You may add Your own attribution
          notices within Derivative Works that You distribute, alongside
          or as an addendum to the NOTICE text from the Work, provided
          that such additional attribution notices cannot be construed
          as modifying the License.

      You may add Your own copyright statement to Your modifications and
      may provide additional or different license terms and conditions
      for use, reproduction, or distribution of Your modifications, or
      for any such Derivative Works as a whole, provided Your use,
      reproduction, and distribution of the Work otherwise complies with
      the conditions stated in this License.

   5. Submission of Contributions. Unless You explicitly state otherwise,
      any Contribution intentionally submitted for inclusion in the Work
      by You to the Licensor shall be under the terms and conditions of
      this License, without any additional terms or conditions.
      Notwithstanding the above, nothing herein shall supersede or modify
      the terms of any separate license agreement you may have executed
      with Licensor regarding such Contributions.

   6. Trademarks. This License does not grant permission to use the trade
      names, trademarks, service marks, or product names of the Licensor,
      except as required for reasonable and customary use in describing the
      origin of the Work and reproducing the content of the NOTICE file.

   7. Disclaimer of Warranty. Unless required by applicable law or
      agreed to in writing, Licensor provides the Work (and each
      Contributor provides its Contributions) on an "AS IS" BASIS,
      WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
      implied, including, without limitation, any warranties or conditions
      of TITLE, NON-INFRINGEMENT, MERCHANTABILITY, or FITNESS FOR A
      PARTICULAR PURPOSE. You are solely responsible for determining the
      appropriateness of using or redistributing the Work and assume any
      risks associated with Your exercise of permissions under this License.

   8. Limitation of Liability. In no event and under no legal theory,
      whether in tort (including negligence), contract, or otherwise,
      unless required by applicable law (such as deliberate and grossly
      negligent acts) or agreed to in writing, shall any Contributor be
      liable to You for damages, including any direct, indirect, special,
      incidental, or consequential damages of any character arising as a
      result of this License or out of the use or inability to use the
      Work (including but not limited to damages for loss of goodwill,
      work stoppage, computer failure or malfunction, or any and all
      other commercial damages or losses), even if such Contributor
      has been advised of the possibility of such damages.

   9. Accepting Warranty or Additional Liability. While redistributing
      the Work or Derivative Works thereof, You may choose to offer,
      and charge a fee for, acceptance of support, warranty, indemnity,
      or other liability obligations and/or rights consistent with this
      License. However, in accepting such obligations, You may act only
      on Your own behalf and on Your sole responsibility, not on behalf
      of any other Contributor, and only if You agree to indemnify,
      defend, and hold each Contributor harmless for any liability
      incurred by, or claims asserted against, such Contributor by reason
      of your accepting any such warranty or additional liability.

   END OF TERMS AND CONDITIONS
