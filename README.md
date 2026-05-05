# LXD Ubuntu VM — Terraform + Ansible + Gitea Actions

Deploy Ubuntu 24.04 virtual machines on LXD via Terraform (module-based), configure them with Ansible roles, all driven by local Gitea CI/CD workflows.

---

## Table of Contents

- [Summary](#summary)
- [Prerequisites](#prerequisites)
- [Quick Steps to Deploy a VM](#quick-steps-to-deploy-a-vm)
- [Architecture Overview](#architecture-overview)
- [Repository Structure](#repository-structure)
- [Secrets & Variables](#secrets--variables)
- [Workflows](#workflows)
- [Terraform: Resource Model](#terraform-resource-model)
- [Ansible: Playbook & Roles](#ansible-playbook--roles)
- [End-to-End Deployment Flow](#end-to-end-deployment-flow)
- [Local Usage](#local-usage)
- [Destroying a VM](#destroying-a-vm)
- [Design Notes](#design-notes)
- [Created and Maintained By](#infrastructure-created-and-maintained-by)

---

## Summary

This repository implements a complete infrastructure-as-code solution that:

- Provisions Ubuntu 24.04 VMs on LXD using Terraform with a reusable `lxd-vm` module
- Configures VMs with cloud-init for automated setup
- Deploys applications via Ansible playbooks — supporting both **local roles** and **self-hosted Ansible Galaxy roles**
- Manages environments (dev/prod) with separate state management per VM in MinIO
- Provides CI/CD automation through Gitea Actions workflows, calling **shared reusable workflows**

**Two secret management strategies are supported:**

| Strategy | Repo | When to use |
|---|---|---|
| ✅ **Default** — OpenBao (Vault) | `Infra/reusable-workflows-vault` | Production-grade, dynamic short-lived secrets |
| 🔁 **Alternate** — Repo-level secrets | `Infra/reusable-workflows` | Simpler setup, static secrets in Gitea org |

**Two Terraform module strategies are supported:**

| Strategy | Source | When to use |
|---|---|---|
| ✅ **Default** — Reusable module | `Infra/reusable-modules` on Gitea | Shared, version-controlled, consistent across repos |
| 🔧 **Local** — In-repo module | `terraform/modules/lxd-vm/` | When you need to customise the module for this repo |

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Ubuntu 24.04 host | LXD installed via snap, API at `https://localhost:8443` |
| Gitea | Running at `http://gitea.local`, `Infra` org |
| Gitea Act Runner | Docker-based runner registered to the `Infra` org |
| MinIO | S3-compatible state backend at `http://10.248.42.22:9000` |
| Terraform ≥ 1.5 | Installed on the runner |
| LXD TLS certificates | Client cert/key for runner-to-LXD authentication |
| **OpenBao** *(default)* | Vault-compatible secrets manager — org secrets `VAULT_ADDR`, `VAULT_ROLE_ID`, `VAULT_SECRET_ID` configured |
| **Gitea org secrets** *(alternate)* | `LXD_ADDRESS`, `LXD_CLIENT_CERT/KEY`, `MINIO_*`, `ANSIBLE_SSH_*` set at org level |

---

## Quick Steps to Deploy a VM

1. **Clone the repo and create a feature branch**
   ```bash
   git clone http://gitea.local/infra/local-workflows-ansible-roles-modules.git
   cd local-workflows-ansible-roles-modules
   git checkout -b <env><vm_name>   # e.g. dev dev02
   ```

2. **Create a new `.tfvars` file** under `terraform/env/<env>/` — copy an existing one and adjust resources as needed (image, cpu, memory, disk)

3. **Run Terraform Check and Plan** via Gitea Actions — go to Actions → `Terraform Check` / `Terraform Plan`, select your feature branch, enter environment and VM name

4. **Run Ansible Check** via Gitea Actions — go to Actions → `Ansible Check`, select your feature branch to lint and syntax-check the playbook

5. **Open a PR and merge** to `main` — the Check workflows run automatically on the PR; merge once green

6. **Run Terraform Apply** via Gitea Actions — go to Actions → `Terraform Apply`, select `main` branch, enter environment and VM name; the VM is created on LXD

7. **Run Ansible Deploy** via Gitea Actions — go to Actions → `Ansible Deploy`, select `main` branch, enter environment and VM name; Ansible roles (local + Galaxy) are applied to the live VM

8. **Run Terraform Destroy** (when done) via Gitea Actions — go to Actions → `Terraform Destroy`, select `main` branch, enter environment, VM name, and type `yes` to confirm

---

## Architecture Overview

```
                ┌─────────────────────────────────────────────┐
                │           Gitea (gitea.local)                │
                │   local-workflows-ansible-roles-modules      │
                └───────────┬─────────────────────────────────┘
                            │ Gitea Actions triggers
                            ▼
                ┌─────────────────────────┐
                │     Gitea Act Runner    │  (Docker-based, inside LXD VM)
                └───┬─────────┬───────────┘
                    │         │
                    │ calls   │ calls
                    ▼         ▼
  ┌──────────────────────────────────────────────────────────┐
  │                  Shared Gitea Repos (Infra org)          │
  │                                                          │
  │  reusable-workflows-vault   ← Default  (OpenBao secrets) │
  │  reusable-workflows         ← Alternate (org secrets)    │
  │  reusable-modules           ← Terraform lxd-vm module    │
  │  reusable-ansible-galaxy-*  ← Ansible Galaxy roles       │
  └──────────────────────────────────────────────────────────┘
                    │         │
           Terraform│         │Ansible
                    ▼         ▼
      ┌─────────────────┐  ┌─────────────────┐
      │   LXD / KVM     │  │   Target VM     │
      │  (localhost:    │  │  (Ubuntu 24.04) │
      │    8443)        │  │  ansible user   │
      └─────────────────┘  └─────────────────┘
              │
      ┌───────────────┐    ┌───────────────┐
      │     MinIO     │    │   OpenBao     │
      │ 10.248.42.22  │    │  (secrets)    │
      │ (TF state)    │    │               │
      └───────────────┘    └───────────────┘
```

---

## Repository Structure

```
.
├── .gitea/
│   └── workflows/
│       ├── terraform-check.yml         # fmt + validate + tflint (PR trigger)
│       ├── terraform-plan.yml          # plan only (manual trigger)
│       ├── terraform-apply.yml         # apply (manual trigger)
│       ├── terraform-destroy.yml       # destroy with confirmation gate (manual trigger)
│       ├── ansible-check.yml           # lint + syntax check (PR trigger)
│       └── ansible-deploy.yml          # run playbook against VM (manual trigger)
├── terraform/
│   ├── env/
│   │   ├── dev/
│   │   │   ├── backend.tf              # S3 (MinIO) backend — empty config, filled at init
│   │   │   ├── providers.tf            # LXD provider + Terraform version constraints
│   │   │   ├── main.tf                 # Calls reusable-modules (default) or local module
│   │   │   ├── variables.tf            # All input variables for this environment
│   │   │   ├── outputs.tf              # VM name, IP, MAC, SSH command
│   │   │   └── dev01.tfvars            # Per-VM resource config (committed, no secrets)
│   │   └── prod/
│   │       ├── backend.tf
│   │       ├── providers.tf
│   │       ├── main.tf                 # Uses local module path (for customisation)
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── prod01.tfvars
│   └── modules/
│       └── lxd-vm/                     # Local module — use when customising from reusable-modules
│           ├── main.tf
│           ├── variables.tf
│           ├── outputs.tf
│           └── cloud-init/
│               └── user-data.yaml
├── ansible/
│   ├── inventory.yml                   # Dynamic inventory via VM_IP env var
│   ├── playbook.yml                    # Entry playbook: local roles + Galaxy roles
│   ├── requirements.yml                # Galaxy roles sourced from self-hosted Gitea
│   └── roles/                          # Local roles (not managed by Galaxy)
│       ├── common/
│       │   └── tasks/main.yml          # apt cache + base packages
│       └── docker/
│           └── tasks/main.yml          # Docker CE install from upstream repo
└── .gitignore
```

---

## Secrets & Variables

This repo supports two secret management strategies. The caller workflows in `.gitea/workflows/` are pre-configured for the **default** (OpenBao/Vault) approach.

### Strategy 1 — OpenBao Vault ✅ Default

Workflows call `Infra/reusable-workflows-vault` and exchange AppRole credentials for short-lived tokens at runtime. Only three secrets need to be set at the Gitea org level:

| Name | Type | Description |
|---|---|---|
| `VAULT_ADDR` | Secret | OpenBao server address (e.g. `http://10.x.x.x:8200`) |
| `VAULT_ROLE_ID` | Secret | AppRole Role ID for CI login |
| `VAULT_SECRET_ID` | Secret | AppRole Secret ID for CI login |

All other credentials (LXD certs, MinIO keys, SSH keys) are stored **inside OpenBao** at the following KV v2 paths, and fetched at runtime by the reusable workflow:

| Path | Keys stored |
|---|---|
| `homelab/data/lxd` | `address`, `client_cert`, `client_key`, `trust_password` |
| `homelab/data/minio` | `endpoint`, `access_key`, `secret_key` |
| `homelab/data/ansible` | `ssh_public_key`, `ssh_private_key` |

### Strategy 2 — Repo-level Secrets 🔁 Alternate

Workflows call `Infra/reusable-workflows` and read secrets directly from Gitea. All secrets must be configured at the **Gitea org level**: `Infra` → Settings → Secrets and Variables.

| Name | Type | Used By | Description |
|---|---|---|---|
| `LXD_ADDRESS` | Secret | Terraform | LXD API host (e.g. `192.168.x.x`) |
| `LXD_CLIENT_CERT` | Secret | Terraform | TLS client certificate (PEM) |
| `LXD_CLIENT_KEY` | Secret | Terraform | TLS client key (PEM) |
| `MINIO_ENDPOINT` | Secret | Terraform | MinIO S3 API URL |
| `MINIO_ACCESS_KEY` | Secret | Terraform | MinIO access key |
| `MINIO_SECRET_KEY` | Secret | Terraform | MinIO secret key |
| `ANSIBLE_SSH_PUBLIC_KEY` | Secret | Terraform | Injected into VM via cloud-init |
| `ANSIBLE_SSH_PRIVATE_KEY` | Secret | Ansible | Used by runner to SSH into VM |

To switch to this strategy, update the `uses:` line in each caller workflow:
```yaml
# Default (OpenBao)
uses: Infra/reusable-workflows-vault/.gitea/workflows/reusable-terraform-plan.yml@main

# Alternate (repo secrets)
uses: Infra/reusable-workflows/.gitea/workflows/reusable-terraform-plan.yml@main
```

---

## Workflows

All caller workflows live in `.gitea/workflows/` and delegate to one of the two reusable workflow repos. The `with:` inputs and `vault_path_*` parameters are the same regardless of which strategy is used.

### `terraform-check.yml` — Terraform Lint & Validate
**Trigger:** Pull Request or `workflow_dispatch`

Runs on every PR to catch issues before merge. Steps:
1. `terraform fmt -check -diff` — enforces canonical formatting across all `.tf` files
2. `terraform init -backend=false` — validates config without a real backend
3. `terraform validate` — checks HCL syntax and provider schema
4. `tflint` — additional lint rules

> No secrets required. Safe to run on any branch.

---

### `terraform-plan.yml` — Terraform Plan
**Trigger:** `workflow_dispatch`

Inputs:
| Input | Description | Example |
|---|---|---|
| `environment` | Must match env folder name | `dev` or `prod` |
| `vm_name` | Must match a `.tfvars` filename | `dev01`, `prod02` |

Steps:
1. Validates environment input (only `dev` / `prod` allowed)
2. Fetches secrets from OpenBao *(default)* or reads from Gitea org secrets *(alternate)*
3. Resolves `tfvars_file` and `state_key` paths from inputs
4. Writes LXD TLS certs to `~/.config/lxc/`
5. `terraform init` against MinIO backend, working in `terraform/env/<env>/`
6. `terraform validate`
7. `terraform plan` with the resolved `.tfvars` file

---

### `terraform-apply.yml` — Terraform Apply
**Trigger:** `workflow_dispatch`

Same inputs as Plan. Runs `terraform apply -auto-approve` after init. VM name is constructed inside the `lxd-vm` module as: `<environment>-<os_type>-<vm_name>` (e.g. `prod-ubuntu-vm-prod02`).

---

### `terraform-destroy.yml` — Terraform Destroy
**Trigger:** `workflow_dispatch`

Inputs:
| Input | Description |
|---|---|
| `environment` | `dev` or `prod` |
| `vm_name` | VM to destroy |
| `confirm_destroy` | Must type `yes` — hard gate before destroy runs |

Includes a safety check step that validates the confirmation before proceeding.

---

### `ansible-check.yml` — Ansible Lint & Syntax Check
**Trigger:** Pull Request or `workflow_dispatch`

1. Installs Ansible and `ansible-lint`
2. Installs Galaxy roles from `ansible/requirements.yml` (if present)
3. Runs `ansible-lint ansible/playbook.yml`
4. Runs `ansible-playbook --syntax-check`

> No live VM required. Safe static analysis only.

---

### `ansible-deploy.yml` — Ansible Deploy
**Trigger:** `workflow_dispatch`

Inputs:
| Input | Description |
|---|---|
| `environment` | `dev` or `prod` |
| `vm_name` | Target VM name |

Steps:
1. Validates environment
2. Fetches secrets from OpenBao *(default)* or org secrets *(alternate)*
3. Inits Terraform against MinIO to read existing state
4. Reads VM IP from `terraform output -raw vm_ip`
5. Installs Galaxy roles from `ansible/requirements.yml`
6. Polls SSH port (up to 200s) until the VM is reachable
7. Writes `ANSIBLE_SSH_PRIVATE_KEY` to `~/.ssh/id_rsa`
8. Generates an inline inventory targeting the VM
9. Runs `ansible/playbook.yml` (applies local roles and Galaxy roles)

---

## Terraform: Resource Model

### Module Strategy

Each environment root module (`terraform/env/<env>/main.tf`) calls the `lxd-vm` module. You choose which module source to use:

**Default — Reusable module from `Infra/reusable-modules`:**
```hcl
module "vm" {
  source = "git::http://gitea.local/Infra/reusable-modules.git//modules/lxd-vm?ref=main"

  vm_name                = var.vm_name
  os_type                = var.os_type
  environment            = var.environment
  ansible_ssh_public_key = var.ansible_ssh_public_key
  lxd_address            = var.lxd_address
  image                  = var.image
  network                = var.network
  storage_pool           = var.storage_pool
  disk_size              = var.disk_size
  cpu_count              = var.cpu_count
  memory_size            = var.memory_size
}
```

**Local module — for customisation:**
```hcl
module "vm" {
  source = "../../modules/lxd-vm"   # local path inside this repo

  # same inputs as above
}
```

Use the local module when you need to modify cloud-init behaviour, add new LXD device types, or iterate on the module without affecting other repos. Once stable, the changes can be promoted back to `reusable-modules`.

### VM Naming Convention

VM names are constructed inside the module's `locals` block:

```
<environment>-<os_type>-<vm_name>
```

Examples:
- `dev-ubuntu-vm-dev01`
- `prod-ubuntu-vm-prod02`

`os_type` and `vm_name` come from the `.tfvars` file; `environment` is injected via `TF_VAR_environment` from the workflow input.

### State Backend Path

State is stored in MinIO at:
```
s3://terraform-state/state/<environment>/<vm_name>/terraform.tfstate
```

Each VM gets its own isolated state file.

### cloud-init

The `cloud-init/user-data.yaml` template (in `lxd-vm` module) is rendered by Terraform's `templatefile()`. It:
- Sets hostname to the full instance name
- Configures DHCP on `enp5s0` via netplan
- Creates an `ansible` user with SSH key auth and passwordless sudo
- Installs base packages (curl, wget, git, python3, openssh-server)
- Hardens SSH (disables password auth and root login)

### tfvars Files

Each VM has a dedicated `.tfvars` file under `terraform/env/<env>/`. Sensitive values (`lxd_address`, `ansible_ssh_public_key`) are **never** stored in tfvars — they are injected as `TF_VAR_*` environment variables from OpenBao or Gitea secrets.

Example (`dev01.tfvars`):
```hcl
image        = "ubuntu-24-04-vm"
os_type      = "ubuntu-vm"
network      = "lxdbr0"
storage_pool = "default"
disk_size    = "10GiB"
cpu_count    = 2
memory_size  = "2GiB"
```

---

## Ansible: Playbook & Roles

This repo uses a **hybrid role strategy**: a mix of **local roles** (version-controlled here) and **self-hosted Ansible Galaxy roles** (sourced from `gitea.local` at deploy time).

### Inventory

`ansible/inventory.yml` is dynamic — the VM IP is passed in via the `VM_IP` environment variable set by the `ansible-deploy` workflow after reading Terraform output:

```yaml
all:
  children:
    lxd_vms:
      hosts:
        target_vm:
          ansible_host: "{{ lookup('env', 'VM_IP') }}"
```

### Playbook

`ansible/playbook.yml` applies roles in sequence — local roles first, then Galaxy roles:

| Role | Source | Tags | What it does |
|---|---|---|---|
| `common` | Local (`ansible/roles/common/`) | `common` | Updates apt cache, installs base packages |
| `docker` | Local (`ansible/roles/docker/`) | `docker` | Docker CE from upstream repo, daemon enabled |
| `curl` | Galaxy (`gitea.local/infra/reusable-ansible-galaxy-role-curl`) | `curl` | Installs curl, configurable via `curl_state` |
| `vim` | Galaxy (`gitea.local/infra/reusable-ansible-galaxy-role-vim`) | `vim` | Installs vim, configurable via `vim_state` |
| `htop` | Galaxy (`gitea.local/infra/reusable-ansible-galaxy-role-htop`) | `htop` | Installs htop, configurable via `htop_state` |

### Galaxy Roles (`ansible/requirements.yml`)

Galaxy roles are sourced from self-hosted Gitea repos. The `ansible-deploy` workflow (and `ansible-check`) runs `ansible-galaxy role install` before executing the playbook:

```yaml
# ansible/requirements.yml
roles:
  - name: curl
    src: http://gitea.local/infra/reusable-ansible-galaxy-role-curl.git
    scm: git
    version: main

  - name: vim
    src: http://gitea.local/infra/reusable-ansible-galaxy-role-vim.git
    scm: git
    version: main

  - name: htop
    src: http://gitea.local/infra/reusable-ansible-galaxy-role-htop.git
    scm: git
    version: main
```

Galaxy roles follow a standard structure (`tasks/main.yml`, `defaults/main.yml`, `meta/main.yml`) and expose configurable variables (e.g. `curl_state: absent` to uninstall). They are pinned to `main` — update `version:` to pin to a tag or commit for production stability.

### Local Roles

Local roles live under `ansible/roles/` and are not managed by Galaxy. Use local roles for:
- Infrastructure concerns that are tightly coupled to this repo (e.g. Docker, base config)
- Roles in active development before publishing to the self-hosted Galaxy server
- Roles that should not be shared across repos

---

## End-to-End Deployment Flow

```
1. Create / edit a .tfvars file under terraform/env/<env>/
2. Open a PR → terraform-check + ansible-check run automatically
   └─ Galaxy roles installed from gitea.local during ansible-check
3. Merge to main
4. Manually trigger terraform-plan  →  review output
5. Manually trigger terraform-apply →  VM is created via LXD
   └─ lxd-vm module: reusable-modules (default) or local modules
   └─ cloud-init configures ansible user, SSH, base packages on first boot
6. Manually trigger ansible-deploy  →  Ansible roles applied to the live VM
   └─ Galaxy roles installed: curl, vim, htop (from gitea.local Galaxy repos)
   └─ Local roles: common (base packages), docker (Docker CE)
```

---

## Local Usage

```bash
# 1. Copy a tfvars example and fill in secrets locally
cp terraform/env/dev/dev01.tfvars terraform/env/dev/dev02.tfvars

# 2. Init with MinIO backend (from the env directory)
cd terraform/env/dev
terraform init \
  -backend-config="endpoint=http://10.248.42.22:9000" \
  -backend-config="bucket=terraform-state" \
  -backend-config="key=state/dev/dev02/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="access_key=<key>" \
  -backend-config="secret_key=<secret>" \
  -backend-config="skip_credentials_validation=true" \
  -backend-config="skip_metadata_api_check=true" \
  -backend-config="skip_requesting_account_id=true" \
  -backend-config="force_path_style=true"

# 3. Export required TF_VAR_ secrets
export TF_VAR_lxd_address="<lxd-host-ip>"
export TF_VAR_ansible_ssh_public_key="ssh-ed25519 AAAA..."
export TF_VAR_environment="dev"
export TF_VAR_vm_name="dev02"

# 4. Plan and apply
terraform plan -var-file="dev02.tfvars"
terraform apply -var-file="dev02.tfvars"

# 5. Get SSH command
terraform output ansible_ssh_command

# 6. Install Galaxy roles locally
ansible-galaxy role install -r ansible/requirements.yml

# 7. Run Ansible manually
VM_IP=$(terraform output -raw vm_ip)
ansible-playbook -i "$VM_IP," \
  -u ansible \
  --private-key ~/.ssh/ansible_id_ed25519 \
  ../../../ansible/playbook.yml
```

---

## Destroying a VM

Via Gitea UI: trigger `terraform-destroy.yml`, enter environment, VM name, and type `yes` to confirm.

Locally:
```bash
cd terraform/env/dev
terraform destroy -var-file="dev02.tfvars"
```

---

## Design Notes

This repo is the **client** of the homelab shared infrastructure stack. It calls shared components and avoids duplicating logic:

| Concern | Default | Local fallback |
|---|---|---|
| **Workflows** | `Infra/reusable-workflows-vault` (OpenBao secrets) | `Infra/reusable-workflows` (org secrets) |
| **Terraform module** | `Infra/reusable-modules` via `git::http://` | `terraform/modules/lxd-vm/` (in-repo copy) |
| **Ansible Galaxy roles** | Self-hosted Gitea Galaxy repos | `ansible/roles/` (local, in-repo) |
| **Secrets** | OpenBao KV v2 (`homelab/data/*`) | Gitea org-level secrets |

**When to use the local module instead of `reusable-modules`:**
Switch `source` in `main.tf` from the Gitea `git::http://` reference to `../../modules/lxd-vm` when you need to customise cloud-init or add LXD-specific device config that shouldn't affect other repos. The local copy in `terraform/modules/lxd-vm/` is always kept in sync with `reusable-modules` as a starting point.

**When to use local Ansible roles instead of Galaxy:**
The `common` and `docker` roles are local because they're infra-lifecycle concerns (base OS setup, container runtime) that are closely tied to how this repo provisions VMs. Utility tool roles (curl, vim, htop) are better suited to Galaxy because they're stateless, independently testable, and reusable across many repos.

**Natural evolution paths when ready to scale:**
- Promote local roles to dedicated Galaxy repos under `Infra` org
- Add a `staging` environment by duplicating `terraform/env/dev/` and adding `vault_path_*` inputs
- Pin Galaxy role `version:` fields to git tags for production deployments
- Replace AppRole auth with short-lived tokens from OpenBao's Kubernetes or JWT auth methods

---
## Infrastructure Created and Maintained By

**Ali Ahmed**  
Building infrastructure, automation, and DevOps workflows  

**Contact**

[![GitHub](https://img.shields.io/badge/GitHub-%20ali%20ahmed-black?style=for-the-badge&logo=github)](https://github.com/jeffreyalie)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-%20ali%20ahmed-blue?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/ali-ahmed-261755252/)
---
