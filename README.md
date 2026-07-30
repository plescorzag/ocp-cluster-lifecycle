# OCP Cluster Lifecycle

Ansible playbooks to **deploy** and **destroy** OpenShift (OCP) clusters.

| Feature | Status |
|---|---|
| AWS IPI | Implemented |
| Azure / GCP | Roadmap stubs (fail with a clear message) |
| UPI | Roadmap stub |
| Host OS detection | Linux, macOS, Windows |
| Cluster types | `sno`, `ha` (3 control-plane + 2 workers), `compact` |

## Prerequisites

- Ansible 2.14+ (`ansible-playbook`)
- AWS CLI configured (for AWS IPI)
- Red Hat pull secret (`pull-secret.json`)
- SSH public key
- A Route53 public hosted zone for `base_domain`

Install Ansible collections (optional but recommended):

```bash
ansible-galaxy collection install -r requirements.yml
```

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

### Required variables

| Variable | Description |
|---|---|
| `ocp_version` | OpenShift version, e.g. `4.16.30` |
| `cluster_type` | `sno` \| `ha` \| `compact` |
| `platform` | `aws` (only fully implemented) |
| `provisioner` | `ipi` (default) |
| `cluster_name` | Cluster name (DNS label) |
| `base_domain` | Base DNS domain (Route53 zone) |
| `aws_region` | e.g. `eu-west-1` |
| `pull_secret_file` | Path to pull secret JSON |
| `ssh_public_key_file` | Path to SSH public key |
| `aws_profile` | AWS CLI profile name (or use env credentials) |

### Cluster types

| Type | Control plane | Workers |
|---|---|---|
| `sno` | 1 | 0 |
| `compact` | 3 | 0 |
| `ha` | 3 | 2 |

### How it works

1. Detects the OS where Ansible runs (`linux` / `mac` / `windows`).
2. Downloads and caches `openshift-install` for that OS + `ocp_version` under `~/.cache/ocp-cluster-lifecycle/`.
3. Renders `install-config.yaml` for AWS IPI.
4. Runs `openshift-install create cluster` or `destroy cluster` in `clusters/<cluster_name>/`.

Secrets are never committed. Prefer file paths + AWS profiles, or Ansible Vault (`group_vars/all/vault.yml.example`).

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
- AWS access keys
- `group_vars/all/vault.yml`
- Anything under `clusters/` (kubeconfigs and install metadata)

Those paths are listed in `.gitignore`.

## Roadmap

- Azure IPI / GCP IPI
- UPI workflows per platform
- Optional Ansible Vault examples with encrypted sample structure
