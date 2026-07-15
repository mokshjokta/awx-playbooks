# SoftAge Ubuntu Workstation Management (AWX / Ansible)

GitHub is the source of truth. AWX pulls this repo, runs the playbooks against
domain-joined Ubuntu 22/24 LTS workstations, and enforces desktop, security,
browser, software, network, AD and compliance policy.

```
GitHub (source of truth)
      -> AWX / Ansible Automation Platform
      -> Ubuntu 22/24 LTS
      -> Domain joined (AD + SSSD)
      -> Policy enforcement + compliance reporting
```

---

## 1. Repository layout

```
awx-playbooks/
├── ansible.cfg
├── collections/requirements.yml     # community.general, ansible.posix, community.crypto
├── inventories/
│   ├── production/ (hosts.ini + group_vars/all.yml)  <- all tunables live here
│   ├── testing/
│   └── staging/
├── playbooks/
│   ├── onboarding.yml     # full first-run provisioning (runs everything)
│   ├── offboarding.yml    # leave domain + strip company config (guarded)
│   ├── desktop.yml        # wallpaper, lockscreen, idle-lock, browsers
│   ├── security.yml       # pwpolicy, usb, bluetooth, firewall, ssh, sudo, av, audit
│   ├── software.yml       # install allowed, remove blocked, updates
│   └── compliance.yml     # logging, monitoring, backup, compliance report
├── roles/                 # 31 roles, one policy area each
└── README.md
```

**Almost everything you tune lives in one file:**
`inventories/production/group_vars/all.yml`
(company name, wallpaper path, password policy, allowed/blocked software,
AD domain, DNS, proxy, firewall ports, etc.)

---

## 2. Before you push: things you must fill in

These are placeholders in the repo — replace with your real values/assets.

1. **Company image assets**
   - `roles/wallpaper/files/wallpaper.png`  (replace placeholder)
   - `roles/lockscreen/files/lockscreen.png`
2. **Internal CA certificates** — drop `*.crt` (PEM) into `roles/certificates/files/`
3. **Group vars** — edit `inventories/production/group_vars/all.yml`:
   - `ad_domain`, `ad_realm`, `ad_ou`, `ad_computer_ou`, `ad_admin_group`
   - `dns_servers`, `dns_search`, `ntp_servers`
   - `allowed_software` / `blocked_software`
   - `browser_homepage`, bookmarks, blocked URLs
   - `remote_syslog_host`, `backup_paths`
4. **Software map** — `roles/software/defaults/main.yml` maps friendly names to
   apt packages vs snaps. VS Code, Teams and Zoom ship as snaps on Ubuntu; adjust
   to match how you distribute them.
5. **AD join credentials — NEVER commit these.** See section 4.

---

## 3. Secrets (Ansible Vault or AWX credential)

The AD join needs `ad_join_user` and `ad_join_password`. Two options:

**Option A — Ansible Vault file (committed, encrypted):**
```bash
ansible-vault create inventories/production/group_vars/vault.yml
# put inside:
#   ad_join_user: "svc-linux-join"
#   ad_join_password: "REDACTED"
```
Then give AWX the vault password as a **Vault credential**.

**Option B — AWX extra vars / custom credential (nothing in git):**
Define `ad_join_user` / `ad_join_password` in the Job Template survey or as a
custom credential injected as extra vars. Recommended.

`.gitignore` already excludes `.vault_pass` and `vault_pass.txt`.

---

## 4. Push to GitHub

From the folder you downloaded:

```bash
cd awx-playbooks
git init
git branch -M main
git add .
git commit -m "Initial AWX Ubuntu workstation management repo"
git remote add origin git@github.com:YOUR_ORG/awx-playbooks.git
git push -u origin main
```

Use a deploy key or a machine user with **read-only** access — AWX only needs to
pull.

---

## 5. Configure AWX

Assuming AWX is already hosted and running.

### 5.1 Credentials (Resources -> Credentials -> Add)
1. **Source Control** credential = your GitHub SSH deploy key (or PAT).
2. **Machine** credential = the SSH user + key AWX uses to log in to the
   workstations. That user must have passwordless sudo (or supply a become
   password). Set *Privilege Escalation* = `sudo`.
3. *(If using Vault)* **Vault** credential = your vault password.

### 5.2 Project (Resources -> Projects -> Add)
- SCM Type: **Git**
- SCM URL: your repo URL
- SCM Branch: `main`
- Source Control Credential: the one from 5.1
- Tick **Update Revision on Launch** and **Clean**.
- Save, then **Sync** — AWX reads `collections/requirements.yml` and installs
  `community.general`, `ansible.posix`, `community.crypto` automatically.

### 5.3 Inventory (Resources -> Inventories)
Create an inventory (e.g. `Ubuntu Production`). Add hosts, or better, attach a
**dynamic inventory / smart inventory**. Create groups matching departments
(`dept_engineering`, `dept_hr`, ...) so department-based policy can target them.
You can also point AWX at the `inventories/production` source in the project.

### 5.4 Job Templates (Resources -> Templates -> Add -> Job Template)
Create one per playbook. For each:
- Inventory: `Ubuntu Production`
- Project: your project
- Playbook: pick from dropdown (e.g. `playbooks/security.yml`)
- Credentials: Machine (+ Vault if used)
- Enable **Privilege Escalation**

| Template name        | Playbook                  | Purpose                    |
|----------------------|---------------------------|----------------------------|
| WS - Onboard         | playbooks/onboarding.yml  | full provisioning          |
| WS - Desktop         | playbooks/desktop.yml     | wallpaper/lock/browsers    |
| WS - Security        | playbooks/security.yml    | security baseline          |
| WS - Software        | playbooks/software.yml    | allowed/blocked software   |
| WS - Compliance      | playbooks/compliance.yml  | reporting + monitoring     |
| WS - Offboard        | playbooks/offboarding.yml | decommission (guarded)     |

**Offboard template:** add a survey variable `offboard_confirm` (boolean) —
the playbook refuses to run unless it is `true`.

### 5.5 Surveys (optional but recommended)
On the Onboard/Software templates add survey fields for things like
`ad_join_user`, `ad_join_password` (mark as password), and department, so
operators fill them at launch instead of editing YAML.

### 5.6 Schedules
- **WS - Compliance** — every 6 hours (feeds the dashboard data).
- **WS - Security** and **WS - Desktop** — nightly, to re-enforce drift.
- **WS - Software** — daily.

### 5.7 Workflow (optional)
Build a **Workflow Template** for onboarding:
`Network/DNS/Time -> AD Join -> Security -> Desktop -> Software -> Compliance`
so a single launch provisions a fresh machine end-to-end with ordering.

---

## 6. Dashboard / reporting

The `compliance` role gathers per-machine facts (name, serial, RAM, CPU, disk,
OS/kernel, firewall, antivirus, domain-joined, encryption, USB, bluetooth,
wallpaper applied, firefox policy applied, reboot required), computes a
compliance score, writes `/var/log/softage-compliance/report.json` on each host,
prints the summary to the job output, and can POST it to a central endpoint
(set `compliance_report_endpoint` in group_vars).

To power a fleet dashboard (500 machines, healthy/failed, USB disabled,
wallpaper applied, updates pending, cert expiry, disk usage, compliance score):
- Simplest: read the `report.json` from each host (backup role or a fetch task),
  or set `compliance_report_endpoint` to a small collector.
- Or scrape the `node_exporter` the `monitoring` role installs with Prometheus +
  Grafana for CPU/RAM/disk/uptime panels.
- AWX's own dashboard already shows job success/failure counts per template.

---

## 7. Safe rollout order

1. Point a **testing** inventory at 1-2 lab machines.
2. Run **Onboard** there, verify AD login, wallpaper lock, USB block, firewall.
3. Promote to **staging** (a department), watch compliance reports.
4. Roll to **production** in waves via inventory groups.

> USB/Bluetooth blocking blacklists kernel modules and rebuilds initramfs; a
> reboot fully applies it. Offboarding is destructive and gated behind
> `offboard_confirm=true`. Always test against staging first.

---

## 8. Local validation (optional)

```bash
pip install ansible-core
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook --syntax-check -i inventories/production/hosts.ini playbooks/security.yml
ansible-lint            # if installed
```
