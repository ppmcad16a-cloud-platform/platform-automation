# platform-automation

Ansible playbooks that run the multi-cloud MERN platform's routine work:

| Playbook | What it does |
|---|---|
| `playbooks/toolchain.yml` | Installs the pinned CLIs (terraform, kubectl, helm, kubelogin, aws, az, gcloud) into WSL Ubuntu, checksum-verified (`-K` for sudo) |
| `playbooks/preflight.yml` | Checks the tools, the 3 cloud logins, your IP vs. `admin_cidr`, and asks for the global secrets (hidden) |
| `playbooks/up.yml` | Session start: mirrors -> GKE -> AKS + EKS -> global -> shop -> health checks (~45-60 min) |
| `playbooks/down.yml` | Session end: per cloud add-ons -> leftover check -> cluster, then global (delete-only); stops on anything incomplete |
| `playbooks/drill-failover.yml` | `-e drill_cloud=azure` - takes one cloud's gateway away for 5 min; shop must stay up, MernServiceDown must fire |
| `playbooks/drill-restore.yml` | Runs Jenkins' restore-drill for GCS, S3 and Azure Blob and waits for all three verdicts |
| `playbooks/standby.yml` / `playbooks/wake.yml` | Stopped standbys: Azure + AWS sleep (AKS stopped, EKS nodes 0) while GCP serves everyone; `wake.yml -e cloud=azure` starts one in ~5-10 min |

Every Terraform step runs through `playbooks/tasks/tf-apply.yml`: plan -> check the kinds of change -> apply that exact plan.

## Run (from WSL Ubuntu)
```bash
cd /mnt/c/Users/dell/Documents/HeroVired/Capstone/repos/platform-automation
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/toolchain.yml -K
```
Git is used from Windows only (line endings / file modes differ in WSL).
