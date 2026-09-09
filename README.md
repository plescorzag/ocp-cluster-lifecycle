# OCP Cluster Lifecycle

Ansible playbooks to **deploy** and **destroy** OpenShift (OCP) clusters.

| Feature | Status |
|---|---|
| AWS IPI | Implemented |
| Azure IPI | Implemented |
| GCP IPI | Roadmap stub |
| UPI | Roadmap stub |
| Host OS detection | Linux, macOS, Windows |
| Cluster types | `sno`, `ha` (3 control-plane + 2 workers), `compact` |

## Prerequisites

- Ansible 2.14+ (`ansible-playbook`)
- Python 3.9+ (Ansible controller)
- Red Hat pull secret (`pull-secret.json`)
- SSH **public** key (`.pub` file, not the private key)
- **AWS:** AWS CLI configured; Route53 public hosted zone for `base_domain`
- **Azure:** Azure CLI (`az`); service principal credentials; public DNS zone in your resource group (e.g. OpenEnv)

Install Ansible collections (optional but recommended):

```bash
ansible-galaxy collection install -r requirements.yml
```

### Linux

Run the playbooks on a Linux host (or VM) with outbound HTTPS to `mirror.openshift.com`, `quay.io`, and your cloud APIs (AWS or Azure). The project uses `ansible_connection: local` — you do not need a remote inventory host.

#### Packages

| Component | Purpose |
|---|---|
| `ansible` or `ansible-core` | Run playbooks |
| `python3` | Ansible controller |
| `tar`, `gzip` | Extract downloaded `openshift-install` archive |
| `git` | Clone this repository |
| `aws` (AWS only) | Optional credential / Route53 checks |
| `az` (Azure only) | Tenant resolution, DNS preflight, RBAC checks |

**RHEL 8/9 / Fedora**

```bash
sudo dnf install -y ansible-core python3 tar gzip git

# AWS
sudo dnf install -y awscli

# Azure (Microsoft repo — see https://learn.microsoft.com/cli/azure/install-azure-cli-linux)
sudo dnf install -y azure-cli
```

**Ubuntu / Debian**

```bash
sudo apt update
sudo apt install -y ansible python3 python3-pip tar gzip git curl unzip

# AWS
sudo apt install -y awscli

# Azure
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

If your distro ships an older Ansible, install a current `ansible-core` via pip:

```bash
python3 -m pip install --user 'ansible-core>=2.14'
```

#### Verify before deploy

```bash
ansible-playbook --version    # 2.14+
python3 --version             # 3.9+
tar --version                 # GNU tar (default on Linux)
git clone https://github.com/plescorzag/ocp-cluster-lifecycle.git
cd ocp-cluster-lifecycle
ansible-galaxy collection install -r requirements.yml
```

**AWS**

```bash
aws --version
aws sts get-caller-identity    # or confirm ~/.aws/credentials / SSO
```

**Azure (OpenEnv / RHPDS)**

```bash
az version
export CLIENT_ID=... PASSWORD=... TENANT=... SUBSCRIPTION=... RESOURCEGROUP=...
az login --service-principal -u "$CLIENT_ID" -p "$PASSWORD" --tenant "$TENANT"
az account set --subscription "$SUBSCRIPTION"
az group show -n "$RESOURCEGROUP"
```

Service principal RBAC for OpenShift 4.16+ / 5.0 on Azure (see [Azure service principal RBAC](#azure-service-principal-rbac-openshift-416--50)):

```bash
az role assignment list --assignee "$CLIENT_ID" --all \
  --query "[].roleDefinitionName" -o tsv | sort -u
# Must include Storage Blob Data Contributor (or Storage Blob Data Owner).
# Subscription Owner / Contributor alone are not enough for bootstrap.ign upload.
```

If missing, an Owner SP can usually self-assign:

```bash
az role assignment create \
  --assignee "$CLIENT_ID" \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION"
```

#### Linux notes

- Set `cluster_architecture: x86_64` (default) unless you intentionally deploy ARM nodes.
- On Linux, `openshift-install` runs natively — no `arch -x86_64` wrapper (unlike Apple Silicon Macs).
- Export OpenEnv/Azure env vars in the **same shell** as `ansible-playbook`, or put values in `vars/my-azure.yml`.
- Do not commit secrets; keep `vars/my-azure.yml` and `clusters/` out of git (see `.gitignore`).

## Quick start (AWS IPI)

1. Copy the example vars file and fill in real values:

```bash
cp vars/example.yml vars/my-cluster.yml
# Edit vars/my-cluster.yml — set ocp_version, cluster_name, base_domain,
# pull_secret_file, ssh_public_key_file, aws_region, aws_profile
```

2. Deploy:

```bash
ansible-playbook playbooks/deploy.yml -e @vars/my-cluster.yml
```

3. After success, use:

- Kubeconfig: `clusters/<cluster_name>/auth/kubeconfig`
- kubeadmin password: `clusters/<cluster_name>/auth/kubeadmin-password`

4. Destroy (uses the same work directory):

```bash
ansible-playbook playbooks/destroy.yml -e @vars/my-cluster.yml
```

### AWS required variables

| Variable | Description |
|---|---|
| `ocp_version` | OpenShift version, e.g. `4.16.30` |
| `cluster_type` | `sno` \| `ha` \| `compact` |
| `platform` | `aws` |
| `provisioner` | `ipi` (default) |
| `cluster_name` | Cluster name (DNS label) |
| `base_domain` | Base DNS domain (Route53 zone) |
| `aws_region` | e.g. `eu-west-1` |
| `pull_secret_file` | Path to pull secret JSON |
| `ssh_public_key_file` | Path to SSH public key |
| `aws_profile` | AWS CLI profile name (or use env credentials) |

### AWS credentials

`aws_credentials_mode: auto` (default) picks the right install-config mode:

| Auth method | Effective mode |
|---|---|
| `~/.aws/credentials` / env keys | **Mint** (installer default) |
| AWS SSO / LoginProvider | **Passthrough** |

Override with `Passthrough`, `Manual`, or `Mint` if needed.

---

## Quick start (Azure IPI)

Works with OpenEnv / RHPDS sandboxes using a service principal.

1. Copy the example vars file:

```bash
cp vars/example-azure.yml vars/my-azure.yml
```

2. Export OpenEnv credentials (same shell as the playbook):

```bash
export GUID=shzmc
export CLIENT_ID=...
export PASSWORD=...          # service principal secret — never commit
export TENANT=...
export SUBSCRIPTION=...
export RESOURCEGROUP=openenv-shzmc
```

3. Edit `vars/my-azure.yml` — at minimum set `cluster_name`, `azure_region`, and paths to pull secret / SSH key:

```yaml
platform: azure
cluster_name: demo
azure_region: northeurope          # must be in RHPDS allow-list (see below)
azure_resource_group: openenv-shzmc
pull_secret_file: "~/pull-secret.json"
ssh_public_key_file: "~/.ssh/id_rsa.pub"
```

`base_domain` is optional if `GUID` is exported — it is auto-derived as `<guid>.azure.redhatworkshops.io`.

4. Deploy:

```bash
ansible-playbook playbooks/deploy.yml -e @vars/my-azure.yml
```

5. Destroy:

```bash
ansible-playbook playbooks/destroy.yml -e @vars/my-azure.yml
```

### Azure required variables

| Variable | Description |
|---|---|
| `platform` | `azure` |
| `ocp_version` | OpenShift version, e.g. `4.22.6` |
| `cluster_type` | `sno` \| `ha` \| `compact` |
| `cluster_name` | Cluster name (DNS label) |
| `base_domain` | Public DNS zone (e.g. `shzmc.azure.redhatworkshops.io`) |
| `azure_region` | Cluster region (e.g. `northeurope`) |
| `azure_resource_group` | RG containing the DNS zone (OpenEnv RG) |
| `pull_secret_file` | Path to pull secret JSON |
| `ssh_public_key_file` | Path to SSH **public** key |

Credentials can be set via vars (`azure_client_id`, etc.) or environment:

| Env var | Ansible var |
|---|---|
| `CLIENT_ID` | `azure_client_id` |
| `PASSWORD` | `azure_client_secret` |
| `TENANT` | `azure_tenant_id` (OpenEnv often gives a domain like `redhat0.onmicrosoft.com`; the playbook resolves it to a UUID) |
| `SUBSCRIPTION` | `azure_subscription_id` |
| `RESOURCEGROUP` | `azure_resource_group` |
| `GUID` | `azure_guid` (used to derive `base_domain`) |

### Azure regions (RHPDS / OpenEnv)

VM policy typically **allows**:

`eastus`, `eastus2`, `westus`, `centralus`, `canadacentral`, `eastasia`, `northeurope`, `westeurope`

Regions like `francecentral` are **blocked** (`RequestDisallowedByPolicy`). The playbook fails early if `azure_region` is outside the allow-list.

### Azure resource groups

Two different roles — do not confuse them:

| Purpose | Variable | Notes |
|---|---|---|
| DNS zone | `azure_resource_group` / `RESOURCEGROUP` | OpenEnv RG (e.g. `openenv-shzmc`). Can be in a different region than the cluster. |
| Cluster VMs | *(installer-created)* | Leave `azure_cluster_resource_group` empty. Installer creates a new empty RG in `azure_region`. |

Do **not** point `azure_cluster_resource_group` at the OpenEnv/DNS RG — the installer requires an empty RG in the target region.

### Azure service principal RBAC (OpenShift 4.16+ / 5.0)

The installer uploads `bootstrap.ign` (and RHCOS VHDs) to Azure Blob Storage using **OAuth**, not storage account keys. The service principal needs a **data-plane** role such as **Storage Blob Data Contributor** (or **Storage Blob Data Owner**) on the subscription or cluster resource group. Azure **subscription Owner** / **Contributor** are separate from blob data roles and produce:

`AuthorizationPermissionMismatch` on `PUT .../ignition/bootstrap.ign`

The playbook checks for this role before `openshift-install create cluster`. If your OpenEnv SP cannot self-assign roles, ask the workshop admin to grant **Storage Blob Data Contributor** at subscription scope.

### Cluster types

| Type | Control plane | Workers |
|---|---|---|
| `sno` | 1 | 0 |
| `compact` | 3 | 0 |
| `ha` | 3 | 2 |

### How it works

1. Detects the OS where Ansible runs (`linux` / `mac` / `windows`).
2. Downloads and caches `openshift-install` for `cluster_architecture` (`x86_64` default) under `~/.cache/ocp-cluster-lifecycle/`.
3. Renders `install-config.yaml` for the selected platform.
4. Runs `openshift-install create cluster` or `destroy cluster` in `clusters/<cluster_name>/`.

Use `playbooks/deploy.yml` or `playbooks/destroy.yml` to choose the action — do **not** set `ocp_lifecycle` in your vars file.

Secrets are never committed. Prefer env vars or Ansible Vault (`inventory/group_vars/all/vault.yml.example`).

---

## Beginner guide: push this project to GitHub

You only need to do this once per machine.

### 1. Install the GitHub CLI

On macOS with Homebrew:

```bash
brew install gh
```

Confirm:

```bash
gh --version
```

### 2. Log in to GitHub

```bash
gh auth login
```

Suggested answers for a first-time setup:

1. **Where do you use GitHub?** → `GitHub.com`
2. **Preferred protocol?** → `HTTPS`
3. **Authenticate Git?** → `Yes`
4. **How to authenticate?** → `Login with a web browser`

Follow the one-time code in the browser, then return to the terminal.

Check:

```bash
gh auth status
```

### 3. Create a private repo and push

From this project directory:

```bash
cd ~/Development/Projects/ocp-cluster-lifecycle

git add .
git status
git commit -m "Initial AWS IPI deploy/destroy Ansible playbooks"

gh repo create ocp-cluster-lifecycle --private --source=. --remote=origin --push
```

That creates a **private** repository under your GitHub user and uploads the code.

Open it in the browser:

```bash
gh repo view --web
```

### 4. Later updates (after you change files)

```bash
cd ~/Development/Projects/ocp-cluster-lifecycle
git add .
git commit -m "Describe your change"
git push
```

### Safety reminder

Never commit:

- `pull-secret.json`
- AWS / Azure credentials or service principal secrets
- `inventory/group_vars/all/vault.yml`
- Anything under `clusters/` (kubeconfigs and install metadata)
- Local vars files like `vars/my-cluster.yml`, `vars/my-azure.yml`

Those paths are listed in `.gitignore`.

## Roadmap

- GCP IPI
- UPI workflows per platform
- Optional Ansible Vault examples with encrypted sample structure
