# 🎯 TERRAFORM ASSOCIATE 004 - COMMAND & STRATEGY CHEAT SHEET
**For Test-Takers with 48 Hours to Exam**

---

## 📌 TABLE OF CONTENTS
1. [Core Workflow Commands](#core-workflow-commands)
2. [Advanced Commands](#advanced-commands)
3. [HCP Terraform (Cloud) Section](#hcp-terraform-cloud-section)
4. [Exam Quick-Reference](#exam-quick-reference)

---

# CORE WORKFLOW COMMANDS

---

## 1. `terraform init`

### SYNTAX & EXAM-CRITICAL FLAGS
```bash
# Base syntax
terraform init [options]

# Exam-relevant flags:
terraform init -backend=false          # Skip backend init (useful for testing)
terraform init -upgrade                # Upgrade provider versions in .terraform.lock.hcl
terraform init -reconfigure            # Reconfigure backend without destroying state
```

### UNDER THE HOOD - FILES & OPERATIONS

| File/Component | What Happens |
|---|---|
| `.terraform/` | Created; stores downloaded providers (binary plugins) |
| `.terraform.lock.hcl` | Created/updated; locks exact provider versions (ALWAYS commit to Git) |
| `backend config` | Remote backend initialized if configured in `terraform {}` block |
| Network | **YES** - downloads providers from Terraform Registry |

**State Impact:** No state file created until `terraform apply`

**Key Logic:**
- Downloads provider plugins matching version constraints
- Creates `.terraform.lock.hcl` (CRITICAL for reproducibility)
- Initializes backend (creates remote state location if needed)
- Validates backend configuration (fails if backend unreachable)

### 004 SPECIFIC UPDATES
- **Provider dependency lock**: The lock file is now **mandatory** for reproducible builds (exam heavily tests this)
- **Backend migration**: `-reconfigure` flag used when changing backends without destroying state
- **Ephemeral values**: Introduced in 1.8; don't affect init, but init must succeed before using ephemeral resources

### EXAM GOTCHA

**Question**: "Can you run `terraform validate` before `terraform init`?"

**Answer**: ❌ NO (Common trap!)
- Before `init`, the `.terraform/` directory doesn't exist
- Terraform doesn't know which providers to use
- `validate` will fail with "module source not found" or similar
- **Order is ALWAYS**: `init` → `validate` → `plan` → `apply`

**Exam variant**: "What's the first command you run in a new Terraform repository?"
- Answer: `terraform init` (always)

---

## 2. `terraform validate`

### SYNTAX & EXAM-CRITICAL FLAGS
```bash
# Base syntax
terraform validate

# Exam-relevant flags:
terraform validate -json              # Output in JSON (for parsing in scripts/CI)

# Note: validate has very few flags (intentional)
```

### UNDER THE HOOD - FILES & OPERATIONS

| Operation | Details |
|---|---|
| **Prerequisites** | Requires `.terraform/` (must run `init` first) |
| **Network** | NO - purely local, doesn't contact providers |
| **State File** | NO - doesn't read or modify state |
| **Files Checked** | All `.tf` files in working directory + modules |

**What it validates:**
- HCL syntax correctness
- Resource block structure
- Variable type constraints
- Output definitions
- Module source validity
- **Does NOT validate**: Actual cloud permissions, API connectivity, variable values

### 004 SPECIFIC UPDATES
- **Check blocks validation**: `validate` now confirms syntax of `check {}` blocks (new in 1.7)
- **Moved blocks**: Validates that `moved {}` block target resources exist
- **Ephemeral values**: Validates that ephemeral values are used only within same resource/data source

### EXAM GOTCHA

**Question**: "You run `terraform validate` and it passes. What does this guarantee?"

**Answer**: ❌ Only syntax is correct
- ✅ HCL syntax is valid
- ❌ Variables exist in `terraform.tfvars`
- ❌ Cloud resources can actually be created
- ❌ Authentication will work
- ❌ Required providers are installed (wait, this is wrong—if they're not, init fails first)

**Real-world scenario**: "Your CI/CD runs `validate` before `plan`. Why might `plan` still fail?"
- Missing variables, wrong credentials, quota limits, insufficient permissions

---

## 3. `terraform plan`

### SYNTAX & EXAM-CRITICAL FLAGS
```bash
# Base syntax
terraform plan [options] [plan file]

# EXAM-CRITICAL FLAGS:
terraform plan -out=tfplan              # Save plan to file (must do before apply)
terraform plan -destroy                 # Plan resource destruction (equivalent to apply -destroy)
terraform plan -refresh-only            # Reconcile state with real infrastructure (no changes to apply)
terraform plan -replace="resource.id"   # Force replacement of specific resource
terraform plan -target="resource.id"    # Plan changes for specific resource only
terraform plan -var="key=value"         # Pass variables on command line
terraform plan -input=false             # Fail if variables prompt needed (CI/CD requirement)
```

### UNDER THE HOOD - FILES & OPERATIONS

| Operation | Details |
|---|---|
| **State File** | READ ONLY - doesn't modify state (unless `-refresh-only`) |
| **Refresh Phase** | By default, refreshes state to match reality; `-refresh=false` skips this |
| **Lock** | Acquires read lock on state (or full lock with `-lock=true`, which is default) |
| **Network** | YES - queries cloud provider API to check current state |
| **Plan File** | `-out=tfplan` saves plan in binary format (required for reproducible apply) |
| **Variables** | Prompted interactively unless `-input=false` (required in CI/CD) |

**Execution Flow:**
```
1. Load configuration
2. Load state file (if exists)
3. Refresh state (query real infrastructure) — skipped with -refresh=false
4. Compare state vs configuration
5. Generate execution plan (add, change, destroy)
6. Display plan (if no -out flag)
```

### 004 SPECIFIC UPDATES

- **Terraform Cloud plan policy**: Plans in HCP Terraform are checked against policy-as-code before apply
- **Check blocks**: `plan` output now includes results of `check {}` blocks (visually marked in output)
- **Moved blocks**: If resource moved via `moved {}`, plan shows "no changes" (not a destroy+create)
- **Sensitive output**: Terraform now masks sensitive values in plan output by default
- **Conditions with sensitive data**: Custom validation failure messages hide sensitive data

### EXAM GOTCHA

**Question**: "You run `terraform plan` and see no changes. What's guaranteed?"

**Answer**: ❌ NOT that `apply` will be safe
- ✅ Current state matches desired configuration
- ❌ BUT—if someone changed the plan file (`plan -out=file`) manually
- ❌ OR if they run `apply` without the plan file, variables might be different
- ❌ OR if state lock expires and someone modifies state during plan→apply

**Real exam question variant**: "You save a plan with `-out=tfplan`, then run `terraform apply tfplan`. What's true?"
- ✅ Apply uses exact plan from the file (reproducible)
- ✅ No additional prompts for variable values
- ✅ Plan can't be changed mid-execution

**Trick question**: "Can you run `terraform plan -out=tfplan` twice and use the second file for apply?"
- ✅ YES, but the first plan is wasted (you'd typically only save the final plan)
- This tests understanding that plan files are binary snapshots

---

## 4. `terraform apply`

### SYNTAX & EXAM-CRITICAL FLAGS
```bash
# Base syntax
terraform apply [options] [plan-file]

# EXAM-CRITICAL FLAGS:
terraform apply tfplan                  # Apply specific plan file (recommended safe approach)
terraform apply -auto-approve           # Skip interactive approval (use ONLY in CI/CD with plan saved)
terraform apply -refresh-only           # Reconcile state without modifying resources
terraform apply -destroy                # Destroy all resources (equivalent to terraform destroy)
terraform apply -target="resource.id"   # Apply changes for specific resource
terraform apply -replace="resource.id"  # Replace (destroy then create) specific resource
terraform apply -lock-timeout=5m        # Wait max 5 min for state lock (default 0 timeout)
```

### UNDER THE HOOD - FILES & OPERATIONS

| Operation | Details |
|---|---|
| **State File** | WRITE - modifications only on successful completion |
| **State Lock** | Acquires full lock (blocks other terraform commands) |
| **Atomicity** | NOT atomic—if apply fails mid-execution, state is partially updated |
| **Rollback** | NO automatic rollback (manual fix required) |
| **Network** | YES - makes actual API calls to cloud provider |
| **Approval** | Interactive prompt unless `-auto-approve` or plan file provided |

**Execution Flow:**
```
1. Acquire state lock
2. Compare current state to desired configuration
3. Execute changes in dependency order (DAG)
4. Update state file after EACH successful resource change
5. Release state lock
6. Display output values
```

**Critical**: If apply fails partway:
- ✅ Some resources created, others not
- ✅ State file reflects what was created
- ❌ NO rollback happens
- You must manually fix the problem and re-apply

### 004 SPECIFIC UPDATES

- **Terraform Cloud integration**: Apply must be triggered from HCP Terraform UI (if workspace configured for remote runs)
- **Policy as Code enforcement**: HCP Terraform enforces Sentinel policies before apply
- **Check blocks reporting**: `apply` output shows pass/fail status of all checks
- **Sensitive data handling**: `-json` output properly redacts sensitive values
- **Moved blocks**: Apply doesn't show destroy→create cycle for moved resources

### EXAM GOTCHA

**Question**: "You run `terraform apply` without a saved plan file, and you provided variables via `-var` during plan. What happens?"

**Answer**: ⚠️ **TRAP** - Variables are re-prompted or use defaults
- If you don't use saved plan file, `apply` regenerates the plan
- If you don't re-provide `-var` values, Terraform prompts you (or uses defaults)
- The actual infrastructure created might DIFFER from what you planned

**Safe approach**: ALWAYS save plan, then apply plan file
```bash
terraform plan -out=tfplan
terraform apply tfplan  # Guaranteed to use exact plan
```

**Exam question**: "You run `terraform apply -auto-approve -var='instance_count=5'` in CI/CD. What's the risk?"

**Answer**: If the pipeline later uses `-var='instance_count=3'`, a new plan is generated and applied immediately without review → **unexpected infrastructure changes**

**Another gotcha**: "Can you run apply twice simultaneously on same state file?"

**Answer**: ❌ NO
- First apply acquires state lock
- Second apply waits (or fails if `-lock=false`)
- This is why state locking is critical

---

## 5. `terraform destroy`

### SYNTAX & EXAM-CRITICAL FLAGS
```bash
# Base syntax
terraform destroy [options]

# EXAM-CRITICAL FLAGS:
terraform destroy -auto-approve        # Skip confirmation (use ONLY in CI/CD)
terraform destroy -target="resource"   # Destroy only specific resource
terraform destroy -refresh-only        # Update state without destroying (rare)
terraform destroy -lock=false          # Don't acquire state lock (risky, for emergencies)
```

### UNDER THE HOOD - FILES & OPERATIONS

| Operation | Details |
|---|---|
| **State File** | WRITE - removes destroyed resources from state |
| **Confirmation** | Interactive prompt (unless `-auto-approve`) |
| **Execution Order** | Reverse dependency order (destroys dependent resources first) |
| **API Calls** | YES - actual deletion calls to cloud provider |
| **State Lock** | Acquired (can be skipped with `-lock=false` for emergency) |

**What destroy does:**
```
1. Load current state
2. Generate destroy plan (all resources marked for destruction)
3. Request confirmation (unless -auto-approve)
4. Destroy resources in reverse dependency order
5. Update state file
6. Complete—state file remains but is empty
```

**Critical difference from manual deletion:**
- ✅ Terraform removes from state file
- ✅ Terraform respects dependencies (destroys in correct order)
- ❌ Manual deletion doesn't update state → state drift

### 004 SPECIFIC UPDATES

- **Terraform Cloud**: Destroy via `terraform destroy` is often disabled in HCP Terraform for safety
- **Workspace deletion**: Destroying doesn't delete the workspace; use HCP UI to delete workspace
- **Protected resources**: `lifecycle { prevent_destroy = true }` blocks destroy of specific resources
- **Check blocks**: Pre-destroy checks don't prevent destruction (checks are informational)

### EXAM GOTCHA

**Question**: "You deleted an AWS EC2 instance manually from the console, then run `terraform destroy`. What happens?"

**Answer**: ✅ No error—Terraform destroys what's in the state file
- State file says instance exists
- Terraform tries to destroy it
- AWS returns "resource not found"
- Terraform removes from state (treats as success)

**Exam variant**: "You want to stop using an AWS instance but keep it running. What command?"

**Answer**: `terraform destroy -target="aws_instance.example"`
- BUT this removes from state only; instance keeps running
- To truly keep it managed separately: use `terraform state rm` instead

**Dangerous scenario**: "You run `terraform destroy -auto-approve` in wrong environment"
- ❌ Deletes EVERYTHING in that infrastructure
- This is why `-auto-approve` requires explicit use (not default)

---

## 6. `terraform state`

### SYNTAX & EXAM-CRITICAL FLAGS
```bash
# State management commands
terraform state list                    # List all resources in state
terraform state show resource_type.name # Show state of specific resource
terraform state rm resource_type.name   # Remove resource from state (NOT delete from cloud)
terraform state mv old.name new.name    # Rename/move resource in state
terraform state replace-provider old new # Replace provider in state (version migration)
terraform state pull > backup.tfstate   # Save state locally
terraform state push backup.tfstate     # Restore state from backup

# EXAM-CRITICAL: Advanced flags (less common but testable)
terraform state list -state=/path/file  # Read state from specific file (useful for backups)
terraform state mv -state-out=/path     # Move resource to different state file
```

### UNDER THE HOOD - FILES & OPERATIONS

| Operation | Details |
|---|---|
| **State Lock** | Acquired for `rm`, `mv`, `replace-provider` (not for `list`, `show`, `pull`) |
| **Cloud API** | NO API calls made; purely state file manipulation |
| **Backup** | Terraform creates automatic backup (`terraform.tfstate.backup`) before modifications |
| **Remote State** | Works with remote backends (pulls/modifies remote state) |
| **Sensitive Data** | State file contains ALL values including passwords (NEVER commit to Git) |

**Key operations:**

| Command | Use Case | Risk Level |
|---|---|---|
| `list` | Audit resources | LOW—read-only |
| `show` | Inspect single resource attributes | LOW—read-only |
| `rm` | Stop managing resource (keep cloud resource) | HIGH—breaks terraform tracking |
| `mv` | Refactor resource names | MEDIUM—requires careful naming |
| `replace-provider` | Migrate provider (e.g., `-/aws` to `aws`) | MEDIUM—can cause issues if misnamed |

### 004 SPECIFIC UPDATES

- **Terraform Cloud integration**: `state` commands work with remote state seamlessly
- **State locking precedence**: Remote state commands respect backend locking configuration
- **Sensitive values**: `state show` displays sensitive values (unlike `plan` output which masks them) — exam tests this
- **Resource move tracking**: `moved` blocks are preferred over `terraform state mv` (declarative vs imperative)

### EXAM GOTCHA

**Question**: "You run `terraform state rm aws_instance.web`. What happens?"

**Answer**: ❌ Resource is NOT deleted from AWS
- ✅ Instance is REMOVED from Terraform state file
- ✅ Next `terraform apply` won't manage it
- Instance keeps running in AWS (orphaned)
- If you re-apply with same resource definition, new instance is created → 2 instances

**Recovery scenario**: "You accidentally ran `state rm` on production database. How do you fix it?"

**Answer**: Use `terraform state push` to restore from backup
```bash
terraform state push backup.tfstate.backup  # Restore previous state
terraform plan  # Verify nothing changes
terraform apply  # Reconcile state
```

**Exam trick**: "Can you use `terraform state` commands on a local state file while apply is running?"

**Answer**: ❌ NO
- Apply acquires state lock
- `state` commands respect that lock and wait/fail
- This is why state locking is critical

---

## 7. `terraform workspace`

### SYNTAX & EXAM-CRITICAL FLAGS
```bash
# Workspace operations
terraform workspace list                # List all workspaces (current one marked with *)
terraform workspace new dev             # Create new workspace
terraform workspace select dev           # Switch to workspace
terraform workspace delete dev           # Delete workspace (must not be active)
terraform workspace show                # Show current workspace name

# EXAM-CRITICAL: Limited to local state
terraform workspace list -state=/path   # List workspaces in specific state directory
```

### UNDER THE HOOD - FILES & OPERATIONS

| Component | Details |
|---|---|
| **Storage** | Local: `.terraform/workspace/` directory; Remote: HCP Terraform projects/workspaces |
| **State Files** | Each workspace has separate state: `.terraform/env/workspace_name/terraform.tfstate` |
| **Default Workspace** | `default` always exists and can't be deleted |
| **Variables Scope** | Variables in `.tfvars` apply to all workspaces (unless using HCP Terraform variable sets) |
| **Backend Requirement** | Workspaces work with LOCAL backend; remote backends use different isolation |

**Workspace isolation:**
```
/project/
├── .terraform/
│   └── workspace/
│       ├── default/
│       │   └── terraform.tfstate
│       ├── dev/
│       │   └── terraform.tfstate
│       └── prod/
│           └── terraform.tfstate
```

Each workspace = **separate state file** = **separate infrastructure**

### 004 SPECIFIC UPDATES

- **HCP Terraform preferred**: HashiCorp recommends HCP Terraform workspaces over local workspaces (more features, better governance)
- **Local workspaces limitation**: Can't enforce policies; no team access control
- **Migration path**: Moving to HCP Terraform? State must be migrated from local workspaces
- **Workspace interpolation**: `${terraform.workspace}` variable is available in configuration for conditional logic

### EXAM GOTCHA

**Question**: "You create a workspace `prod` and deploy infrastructure. A colleague on the team doesn't see your infrastructure. Why?"

**Answer**: 🎯 **Workspaces are LOCAL** (in local backend)
- Your colleague's `.terraform/workspace/prod/` is empty
- They need to run `terraform apply` independently
- Workspaces are NOT shared across team members

**This is why HCP Terraform is recommended:**
- Workspaces in HCP are SHARED
- Team members see same workspace and state
- Only real solution for team collaboration

**Exam question**: "You want dev and prod to use same configuration but different variable values. Recommended approach?"

**Answer**: ❌ Don't use local workspaces (not shared)
- ✅ Use HCP Terraform workspaces with different variable sets per workspace
- OR ✅ Use different directories with separate backends
- OR ✅ Use `-var-file=dev.tfvars` and `-var-file=prod.tfvars` with same state backend

**Critical constraint**: "Can you use workspaces with remote backends (e.g., Terraform Cloud)?"

**Answer**: ✅ YES, but differently
- HCP Terraform workspaces are separate from local workspaces
- One HCP workspace = ONE local backend directory
- Workspaces are managed via HCP UI, not CLI (mostly)

---

## 8. `terraform import`

### SYNTAX & EXAM-CRITICAL FLAGS
```bash
# Base syntax
terraform import resource_type.name resource_id

# Examples
terraform import aws_instance.web i-1234567890abcdef0
terraform import aws_security_group.allow sg-12345678
terraform import aws_rds_cluster.main my-db-cluster-name

# EXAM-CRITICAL FLAGS:
terraform import -input=false           # Fail if provider config requires input
terraform import -lock=true             # Acquire state lock (default)
terraform import -lock-timeout=5m       # Wait for lock (default: indefinite)
```

### UNDER THE HOOD - FILES & OPERATIONS

| Operation | Details |
|---|---|
| **Network** | YES - queries cloud provider for resource details |
| **State File** | WRITE—adds resource to state |
| **Configuration** | Requires stub definition; import fills in state only, NOT configuration |
| **Provider** | Must be configured and authenticated |
| **ID Format** | Cloud-provider-specific (ARN for some, name for others) |

**Import process:**
```
1. Resource definition exists in HCL (as stub):
   resource "aws_instance" "web" {}  # Empty—just structure

2. Run: terraform import aws_instance.web i-1234567890abcdef0

3. Terraform queries AWS: "Tell me about instance i-1234567890abcdef0"

4. AWS returns: {instance_type: t2.micro, tags: {...}, ...}

5. Terraform writes to state file ONLY (NOT configuration)

6. You manually write HCL to match state (IDE hints help)
```

### 004 SPECIFIC UPDATES

- **No automatic code generation**: Import adds to state; you MUST manually write HCL (Terraform doesn't generate it)
- **Resource IDs vary**: Each provider has different ID format; docs required
- **Check blocks**: Imported resources can't have `check` blocks that reference non-imported attributes
- **Ephemeral values**: Imported resources can't populate ephemeral values (by definition—those are read-only computed)
- **Import blocks (1.7+)**: NEW feature—`import {}` blocks allow declarative imports (alternative to CLI)

### EXAM GOTCHA

**Question**: "You run `terraform import aws_instance.web i-123`. Now you run `terraform plan`. What happens?"

**Answer**: ❌ Plan shows resource exists (no changes) BUT configuration might be incomplete
- ✅ State has resource details
- ❌ HCL is just a stub `resource "aws_instance" "web" {}`
- ❌ Attributes aren't specified in HCL yet
- ❌ Next `apply` might change attributes to match HCL (currently empty)

**Safe procedure:**
```bash
terraform import aws_instance.web i-123
# Now manually write HCL configuration:
resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t2.micro"
  # ... match all attributes from state
}
terraform plan  # Should show no changes
```

**Exam question**: "You have 500 existing AWS resources to manage with Terraform. Best approach?"

**Answer**: ❌ Manually import each one (too slow)
- ✅ Use `import` blocks (declarative, faster, no CLI needed)
- ✅ Use third-party tools (Terraformer, AWS CloudFormation import)
- ✅ Use module registry (if common patterns)

**Critical point**: "Import only updates state, not configuration. Why is this a problem?"

**Answer**: 
- State might have attributes you don't want
- Next `apply` could change those attributes
- Requires careful manual HCL writing post-import
- This is why Terraform prefers greenfield (new) infrastructure

---

---

# ADVANCED COMMANDS

---

## 9. `terraform fmt`

### SYNTAX & EXAM-CRITICAL FLAGS
```bash
# Format current directory
terraform fmt

# EXAM-CRITICAL FLAGS:
terraform fmt -recursive                # Format all .tf files in subdirectories
terraform fmt -check                   # Check if formatting needed (fail if yes)
terraform fmt -write=false             # Check without modifying files
terraform fmt -diff                    # Show differences (useful in CI/CD)
```

### UNDER THE HOOD
- Local operation—no network, no state access
- Modifies `.tf` files in place (unless `-write=false`)
- Standardizes indentation (2 spaces), quotes, spacing
- **Exam context**: Used in CI/CD pipelines to enforce style before commit

### EXAM GOTCHA
**Question**: "You run `terraform fmt -check` in CI/CD. What does it do?"

**Answer**: Returns exit code 0 if formatted correctly; non-zero if formatting needed
- Can be used as gate: reject PR if formatting wrong
- This enforces code style consistency

---

## 10. `terraform taint` / `terraform untaint` (DEPRECATED)

### ⚠️ DEPRECATED BUT STILL TESTABLE
```bash
# Old way (marked for replacement)
terraform taint aws_instance.web      # Mark for replacement
terraform untaint aws_instance.web    # Unmark

# Modern replacement (Terraform 1.6+)
terraform apply -replace="aws_instance.web"
```

### WHY DEPRECATED?
- Imperative (marks resource, requires apply step)
- Newer way: `terraform apply -replace=` is cleaner
- Or use `replaced_by` in `moved` blocks (declarative)

### EXAM GOTCHA
**Question**: "You want to force replacement of an instance. Which command?"

**Answer**: ✅ `terraform apply -replace="aws_instance.web"`
- ❌ `terraform taint` is outdated (might appear as distractor)

---

---

# HCP TERRAFORM (CLOUD) SECTION

---

## 🏢 HCP TERRAFORM (formerly Terraform Cloud) - Critical for Exam

### WORKSPACE vs PROJECT Comparison

| Feature | **Workspace** | **Project** |
|---|---|---|
| **Definition** | Terraform working directory; maps to state file and variables | Container for organizing multiple workspaces |
| **How Many** | 1000s per organization | Organize by team/app/environment |
| **State File** | ONE state per workspace | Multiple workspaces (multiple state files) |
| **Variables** | Workspace-scoped (can be inherited from project) | Project-level variables (inherited by ALL workspaces) |
| **Access Control** | Fine-grained (per workspace) | Team-level (controls who sees project) |
| **Use Case** | dev, staging, prod (separate infrastructure) | Organize multiple environments or applications |
| **API** | Full API support | Limited API (newer feature) |
| **Cost** | Each workspace counts toward plan limit | Container only (no additional cost) |
| **Typical Structure** | `myapp-dev`, `myapp-staging`, `myapp-prod` workspaces | `my-app-project` contains those workspaces |

**Visual hierarchy:**
```
Organization
├── Project (my-application)
│   ├── Workspace (dev)
│   │   └── state file, variables, VCS link
│   ├── Workspace (staging)
│   │   └── state file, variables, VCS link
│   └── Workspace (prod)
│       └── state file, variables, VCS link
├── Project (infrastructure)
│   ├── Workspace (shared-services)
│   └── Workspace (networking)
```

---

### VARIABLE SET PRECEDENCE (CRITICAL FOR EXAM)

**Variable Sets** are reusable variable collections assigned to workspaces.

#### Precedence Order (Highest to Lowest Priority):

```
1. Workspace Variables (explicit in workspace)
2. Variable Set Variables (assigned to workspace)
3. Environment Variables (TF_VAR_* in system)
4. Terraform Variables (defaults in configuration)
```

**Example scenario:**

```hcl
# Configuration (default)
variable "region" {
  default = "us-west-1"
}

variable "instance_count" {
  default = 1
}
```

**HCP Terraform Variable Sources:**

| Source | Value | Priority |
|---|---|---|
| `variable "region" { default }` | `us-west-1` | Lowest (4) |
| Variable Set `vpc-settings` | `region = "us-east-1"` | Medium (2) |
| Workspace variable | `region = "eu-west-1"` | Highest (1) |
| Environment variable | `TF_VAR_region=ap-south-1` | (depends on context) |

**Result**: `eu-west-1` (workspace variable wins)

#### Variable Set Assignment Logic:

```
If variable exists in:
- Workspace variables → USE IT (override all)
- Variable Set → USE IT (override defaults)
- Configuration default → USE IT
- NOT FOUND → ERROR
```

#### EXAM SCENARIO:

**Setup:**
- Variable Set `prod-settings`: `instance_type = "t3.xlarge"`, `region = "us-east-1"`
- Workspace `myapp-prod`: inherits from project, which has Variable Set assigned
- Workspace variable: `instance_type = "t2.micro"` (NOT in set, workspace-specific)

**Run `terraform apply` in workspace:**

| Variable | Where Found | Final Value |
|---|---|---|
| `instance_type` | Workspace var | `t2.micro` |
| `region` | Variable Set | `us-east-1` |
| `aws_region` (no such var) | Default or error | ERROR if not in defaults |

**Key insight**: Workspace-level variables ALWAYS override Variable Sets.

#### ANTI-PATTERN (Exam gotcha):

❌ **DON'T DO THIS:**
```hcl
# In HCP Terraform workspace
variable "database_password" {
  default = "hardcoded-secret"  # NEVER in config!
}
```

✅ **DO THIS:**
- Set in HCP Terraform Variable Set with `Sensitive` flag checked
- OR use Vault integration for secrets
- Or use GitHub Actions secrets (if using VCS)

---

### HCP TERRAFORM FEATURES SUMMARY

| Feature | Purpose | Exam Relevance |
|---|---|---|
| **State Locking** | Prevents concurrent applies (automatic) | HIGH—know that it's built-in |
| **Remote Runs** | Plan/apply triggered from HCP UI | HIGH—understand workflow differences |
| **VCS Integration** | Auto-trigger on Git push | MEDIUM—know it exists |
| **Sentinel Policies** | Policy-as-code enforcement | MEDIUM—appears in scenario questions |
| **Team Management** | Role-based access (Owner, Developers, etc.) | MEDIUM—understand 3 permission levels |
| **Module Registry** | Host private modules | LOW—less heavily tested |
| **API Tokens** | Programmatic access | MEDIUM—used in CI/CD |
| **Cost Estimation** | Estimate monthly costs of runs | LOW—not critical |

#### Key Differences from Open-Source Terraform:

| Feature | Open-Source | HCP Terraform |
|---|---|---|
| State file | Local or remote backend | Managed by HCP (automatic) |
| Locking | Manual setup (DynamoDB, Consul, etc.) | Built-in |
| Runs | Local machine or CI/CD | HCP-hosted (remote runs) |
| Team Collaboration | Manual state sharing | Native support |
| Policy Enforcement | External tools | Sentinel policies (built-in) |
| Approval Workflow | Not built-in | Built-in (Cost & Policy checks) |

---

## 🎓 HCP TERRAFORM EXAM QUESTIONS

### Question: Variable Set Inheritance

**Scenario**: You have:
- Organization `ACME`
- Project `web-app`
- Workspace `web-app-prod` (in project)
- Variable Set `global-prod` assigned to workspace

Variable Set contains: `environment = "production"`, `region = "us-east-1"`
Workspace contains: `region = "eu-west-1"`
Configuration default: `region = "us-west-2"`

**Q: What's the value of `var.region` during apply?**

A) `us-west-2` (config default)
B) `eu-west-1` (workspace variable wins)
C) `us-east-1` (variable set)
D) Error (conflicting values)

**Answer**: ✅ **B** - Workspace variables have highest priority

---

### Question: Remote Runs in HCP Terraform

**Scenario**: You create a HCP Terraform workspace and push code to GitHub. Every time you push, HCP automatically runs `terraform plan`.

**Q: Where is the plan executed?**

A) On your local machine
B) In the GitHub Actions runner
C) On HCP Terraform servers
D) In an AWS EC2 instance

**Answer**: ✅ **C** - HCP Terraform servers execute remote runs

---

---

# EXAM QUICK-REFERENCE

## Command Decision Tree

```
Need to create/modify infrastructure?
├─ First time? → terraform init
├─ Check syntax? → terraform validate
├─ Preview changes? → terraform plan -out=tfplan
├─ Apply changes? → terraform apply tfplan
└─ Done

Need to destroy infrastructure?
├─ Destroy everything? → terraform destroy
├─ Destroy one resource? → terraform destroy -target=resource
└─ Keep in cloud, remove from Terraform? → terraform state rm

Need to manage existing infrastructure?
├─ Add to Terraform? → terraform import
├─ Fix state? → terraform state [list|show|rm|mv]
└─ Check drift? → terraform plan -refresh-only

Code style issues?
└─ Fix formatting → terraform fmt -recursive

Team collaboration?
└─ Switch to HCP Terraform (local workspaces are NOT shared)

Multiple environments?
├─ Local: terraform workspace [new|select]
└─ Team: HCP Terraform projects & workspaces
```

---

## Flags to Memorize (by priority)

### TIER 1 (Definitely on exam):
- `plan -out=tfplan` (save plan)
- `apply tfplan` (apply saved plan)
- `apply -auto-approve` (skip confirmation)
- `-target=resource` (single resource)
- `-input=false` (CI/CD requirement)

### TIER 2 (Likely on exam):
- `init -upgrade` (update providers)
- `plan -refresh-only` (check drift)
- `destroy -auto-approve` (skip confirmation)
- `import` (add existing resources)
- `state rm` (orphan resource)

### TIER 3 (Possible exam questions):
- `-lock=false` (emergency only)
- `-lock-timeout` (wait for lock)
- `workspace list/new/select` (local only)
- `-replace=` (force replacement)
- `fmt -check` (CI/CD validation)

---

## Critical Files to Know

| File | Purpose | Commit to Git? |
|---|---|---|
| `.terraform.lock.hcl` | Provider version lock | ✅ YES (ALWAYS) |
| `terraform.tfstate` | State file (local) | ❌ NO (secrets!) |
| `terraform.tfstate.backup` | Previous state backup | ❌ NO |
| `.terraform/` | Provider binaries | ❌ NO (gitignore) |
| `.terraform.tfvars` | Secret variables | ❌ NO (gitignore) |
| `*.tfvars` | Variable values | ⚠️ Depends (public ok, secrets no) |
| `.terraform/workspace/` | Workspace state | ❌ NO |

---

## Gotchas Summary

| Gotcha | Reality |
|---|---|
| "Can I validate before init?" | ❌ NO—need `.terraform/` first |
| "Does plan modify state?" | ❌ NO—read-only (unless `-refresh-only`) |
| "Does apply guarantee safe changes?" | ❌ NO—someone might change state mid-plan |
| "Does import copy configuration?" | ❌ NO—imports state only; write HCL manually |
| "Are local workspaces shared with team?" | ❌ NO—each person gets separate workspaces |
| "Can I run plan and apply simultaneously?" | ❌ NO—state lock blocks concurrency |
| "Does destroy rollback if it fails?" | ❌ NO—partial destroy leaves broken state |
| "Does taint command still work?" | ✅ YES but DEPRECATED—use `-replace=` instead |

---

## Things That Require Network (in exam):

✅ `init` (downloads providers)
✅ `plan` (queries cloud API)
✅ `apply` (makes changes via API)
✅ `destroy` (deletes via API)
✅ `import` (fetches resource details)
✅ `state pull/push` (if remote backend)

❌ `validate` (local only)
❌ `fmt` (local only)
❌ `state list/show/rm/mv` (if local; remote varies)
❌ `workspace list` (if local)

---

## 48-HOUR EXAM PREP FOCUS

**Day 1 (Today - 6 hours):**
1. ✅ Memorize command flags (Tier 1 + Tier 2)
2. ✅ Understand state lock implications
3. ✅ Practice: `plan -out → apply plan` workflow
4. ✅ Know when to use `-auto-approve` (CI/CD only)

**Day 2 (Next 6 hours):**
1. ✅ HCP Terraform: Workspace vs Project differences
2. ✅ Variable Set precedence (practice scenarios)
3. ✅ Import workflow (stub config + manual HCL)
4. ✅ Workspace limitations (local, not shared)

**Last 6 Hours (Day before exam):**
1. ✅ Take practice exam (focus on command flags)
2. ✅ Review gotchas (state lock, import config)
3. ✅ Practice decision tree (when to use each command)
4. ✅ Rest & review critical flags

---

## FINAL EXAM STRATEGY

**When you see command-based questions:**
1. Identify what the question wants (create? destroy? import? check?)
2. Recall the decision tree
3. Check if flags change behavior (especially `-auto-approve`, `-target`, `-refresh-only`)
4. Verify if network is needed (rules out some options)
5. Remember: **Saved plan is ALWAYS safer than unsaved**

**When you see state questions:**
1. Understand: State = truth of infrastructure
2. Workspace = separate state files
3. Lock = prevents concurrent changes
4. Drift = reality differs from state

**When you see HCP Terraform questions:**
1. Workspace = where infrastructure runs
2. Project = organizational container
3. Variable Set = reusable variables (lower priority than workspace vars)
4. Remote runs = HCP executes (not your machine)

---

## QUICK COMMANDS REFERENCE

```bash
# ABSOLUTE BASICS
terraform init                          # Start here
terraform validate                      # Check syntax
terraform plan -out=tfplan              # Preview
terraform apply tfplan                  # Execute (safe)
terraform destroy -auto-approve         # Delete all

# STATE INSPECTION
terraform state list                    # What resources exist?
terraform state show aws_instance.web   # What attributes?

# IMPORT WORKFLOW
terraform import aws_instance.web i-123 # Add to state
# (then manually write HCL)
terraform plan                          # Verify

# RECOVERY
terraform state push backup.tfstate     # Restore old state
terraform plan -refresh-only            # Check what changed

# TEAM COLLABORATION
terraform workspace list                # (Local only—limited)
# Better: Switch to HCP Terraform
```

---

**GOOD LUCK ON YOUR EXAM! 🚀**

*Last updated: Terraform 1.8+ | HCP Terraform (current)*