# platform-automation

Ansible playbooks that operate the multi-cloud MERN platform: install the tools, start and stop a working session in all three clouds with one command, put the standby clouds to sleep and wake them, deploy without Jenkins, and run the disaster-recovery and failover drills. It also contains the GitHub Actions workflow that wakes the standbys automatically during a failover.

**Ansible** 2.21 (ansible-core) in WSL Ubuntu 24.04 · collections `community.general`, `kubernetes.core`

## Playbooks

| Playbook | What it does |
|---|---|
| `toolchain.yml` | installs pinned, checksum-verified terraform, kubectl, helm, kubelogin, aws, az, gcloud (`-K` for sudo) |
| `preflight.yml` | checks tool versions, the three cloud logins, your public IP against `admin_cidr`, and reads the global secrets (hidden input) — imported by `up` and `down` |
| `up.yml` | session start: ACR mirrors → **GKE, AKS and EKS in parallel** → global layer → per cloud independently: add-ons → shop (6 services at once) → health check → standbys go to sleep (`-e keep_awake=true` to keep them running) |
| `down.yml` | session end: wake standbys → per cloud add-ons destroy → state empty → leftover check → cluster destroy (AKS/EKS, then GKE) → global layer (delete/update only) |
| `standby.yml` / `wake.yml` | sleep Azure + AWS (AKS stopped, EKS node group 0) / wake one (`-e cloud=azure`), then catch it up to GCP's versions |
| `deploy.yml` | break-glass deploy when Jenkins (GKE) is unavailable: `-e service=catalog-service -e tag=main-a1b2c3d` to every awake, reachable cloud (`-e target=azure` for one) |
| `drill-restore.yml` | runs Jenkins `restore-drill` for GCS, S3 and Azure Blob and waits for all three verdicts |
| `drill-failover.yml` | the primary fails, a sleeping standby wakes (automatic, or `-e wake_via=manual`), measures the RTO, repairs GCP, standbys back to sleep |

Every Terraform change goes through `playbooks/tasks/tf-apply.yml`: **plan → check the kind of every change against an allow-list → apply exactly that plan**. Session start may only create/update; teardown may only delete (and update).

`.github/workflows/failover-wake.yml` — started by the Cloudflare Worker when GCP's pool becomes unhealthy: logs in keylessly and starts AKS / scales the EKS node group to 2.

## Dependencies

| Depends on | For |
|---|---|
| [gcp-platform](https://github.com/ppmcad16a-cloud-platform/gcp-platform), [azure-platform](https://github.com/ppmcad16a-cloud-platform/azure-platform), [aws-platform](https://github.com/ppmcad16a-cloud-platform/aws-platform), [global-platform](https://github.com/ppmcad16a-cloud-platform/global-platform) | the Terraform stacks it applies and destroys — cloned next to this repo (`repos_root`) |
| [platform-deployments](https://github.com/ppmcad16a-cloud-platform/platform-deployments) | Helm chart + values for installing the shop; Jenkins jobs used by the drills |
| an image-tags file (`image_tags_file`) | which `main-<sha>` version of each service to install |
| cloud logins on the workstation | `gcloud`, `az`, `aws sso` |

## Usage (WSL Ubuntu)

```bash
python3 -m venv ~/.venvs/ansible && source ~/.venvs/ansible/bin/activate
pip install "ansible-core==2.21.5" "kubernetes==37.0.1" "python-debian==1.1.1"
cd /mnt/c/<path-to>/repos/platform-automation
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/toolchain.yml -K

aws sso login --profile HeroviredProfile
read -rsp "Cloudflare API token: " CLOUDFLARE_API_TOKEN; echo; export CLOUDFLARE_API_TOKEN
read -rsp "Atlas public key: " MONGODB_ATLAS_PUBLIC_KEY; echo; export MONGODB_ATLAS_PUBLIC_KEY
read -rsp "Atlas private key: " MONGODB_ATLAS_PRIVATE_KEY; echo; export MONGODB_ATLAS_PRIVATE_KEY
ansible-playbook playbooks/up.yml          # 25-40 min
ansible-playbook playbooks/down.yml        # end of session
```

Settings (paths, account ids, tool versions, standby commands, leftover checks) are in `inventory/group_vars/all.yml`; the three clouds and their kube contexts in `inventory/hosts.yml`.

Use Git from Windows, not from WSL (line endings and file modes differ). Full procedures: [operations runbook](https://github.com/ppmcad16a-cloud-platform/core-cloud-platform/blob/main/docs/05-operations-runbook.md).
