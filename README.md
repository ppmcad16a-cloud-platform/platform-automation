# platform-automation

Ansible playbooks that run the multi-cloud MERN platform's routine work:

| Playbook | What it does |
|---|---|
| `playbooks/toolchain.yml` | Installs the pinned CLIs (terraform, kubectl, helm, kubelogin, aws, az, gcloud) into WSL Ubuntu, checksum-verified (`-K` for sudo) |
| `playbooks/preflight.yml` | Checks the tools, the 3 cloud logins, your IP vs. `admin_cidr`, and asks for the global secrets (hidden) |
| `playbooks/up.yml` | Session start: mirrors -> all 3 clusters at once -> global -> per cloud, independently: add-ons -> shop (6 services at once) -> health -> standbys sleep (~30-40 min; `-e keep_awake=true`) |
| `playbooks/deploy.yml` | Break-glass deploy when Jenkins (GKE) is down: `-e service=catalog-service -e tag=main-a1b2c3d` to every awake, reachable cloud (`-e target=azure` for one) |
| `playbooks/down.yml` | Session end: per cloud add-ons -> leftover check -> cluster, then global (delete/update only - never create or replace); stops on anything incomplete |
| `playbooks/drill-restore.yml` | Runs Jenkins' restore-drill for GCS, S3 and Azure Blob and waits for all three verdicts |
| `playbooks/standby.yml` / `playbooks/wake.yml` | Stopped standbys: Azure + AWS sleep (AKS stopped, EKS nodes 0) while GCP serves everyone; `wake.yml -e cloud=azure` starts one in ~5-10 min |
| `playbooks/drill-failover.yml` | Stopped-standby drill: closes GCP's front door, a standby wakes (automatic, or `-e wake_via=manual`), measures the RTO, then repairs GCP and puts the standbys back to sleep |

Every Terraform step runs through `playbooks/tasks/tf-apply.yml`: plan -> check the kinds of change -> apply that exact plan.

## Run (from WSL Ubuntu)
```bash
cd /mnt/c/Users/dell/Documents/HeroVired/Capstone/repos/platform-automation
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/toolchain.yml -K
```
Git is used from Windows only (line endings / file modes differ in WSL).
