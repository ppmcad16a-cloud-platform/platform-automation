# platform-automation

Ansible playbooks that run the multi-cloud MERN platform's routine work:

| Playbook | What it does |
|---|---|
| `playbooks/toolchain.yml` | Installs the pinned CLIs (terraform, kubectl, helm, kubelogin, aws, az, gcloud) into WSL Ubuntu, checksum-verified |
| `playbooks/preflight.yml` | (next) Checks logins, IP and required secrets before anything runs |
| `playbooks/down.yml` | (next) Safe teardown: add-ons -> leftover checks -> cluster -> global |
| `playbooks/up.yml` | (next) Whole session start in one command |

## Run (from WSL Ubuntu)
```bash
cd /mnt/c/Users/dell/Documents/HeroVired/Capstone/repos/platform-automation
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/toolchain.yml -K
```
Git is used from Windows only (line endings / file modes differ in WSL).
