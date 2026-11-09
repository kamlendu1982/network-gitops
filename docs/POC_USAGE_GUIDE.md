# Network GitOps PoC — Usage Guide

This guide explains how to run the Network GitOps proof of concept with **Ansible Automation Platform (AAP)** and lab/network devices.

**Assumptions**

- Customer has AAP (Controller) and a Network-capable Execution Environment.
- At least one network device is reachable from the AAP EE (SSH).
- Device intent (banner, interfaces, base config) lives in **Git**; AAP applies it.
- Credentials (device username/password) are stored in **AAP**, not in Git.

---

## 1. What this PoC demonstrates

| Stage | What happens |
|---|---|
| **Day 0 — Onboard** | First-time baseline (`base_config` Jinja) + banner + interfaces |
| **Day 2 — Banner** | Change MOTD in Git → webhook/JT enforces banner on device |
| **Day 2 — Interfaces** | Change interface intent in Git → webhook/JT enforces interfaces |
| **Verify** | `test-connection` checks SSH + shows clock and MOTD banner |

**Design principles**

- One collection role per OS class (`ios`, `nxos`, `eos`, `junos`).
- Vendor-neutral intent in `inventory/`; templates/modules translate per OS.
- Day 0 vs day 2 are separate playbooks.
- NetBox (or any CMDB inventory) can supply **connection** data; Git supplies **golden-config intent**. This PoC commonly uses a **project-sourced static inventory** for demos.

---

## 2. Repository layout (what to edit)

```text
network-poc/
├── inventory/
│   ├── hosts.yml                          # device connectivity (ansible_host/port)
│   ├── group_vars/all/
│   │   ├── banner.yml                     # <-- Day-2 banner intent
│   │   └── base_config.yml                # <-- Day-0 shared baseline intent
│   └── host_vars/<device>/device.yml      # <-- Day-2 interfaces (+ mgmt for day 0)
└── network-collection/ansible_collections/acme/netconfig/playbooks/
    ├── onboard.yml                        # Day 0
    ├── day2_banner.yml                    # Day 2 banner only
    ├── day2_interfaces.yml                # Day 2 interfaces only
    ├── site.yml                           # Day 2 banner + interfaces (manual)
    └── check_reachability.yml             # SSH / show clock / show banner
```

| Goal | Edit this file in Git |
|---|---|
| Change MOTD banner | `inventory/group_vars/all/banner.yml` |
| Change interfaces | `inventory/host_vars/<hostname>/device.yml` (`interfaces:` list) |
| Change day-0 baseline (DNS/NTP/etc.) | `inventory/group_vars/all/base_config.yml` |
| Change how AAP reaches the device | `inventory/hosts.yml` (`ansible_host` / `ansible_port`) |

---

## 3. One-time AAP setup

### 3.1 Project

1. **Projects → Add** (or use existing).
2. SCM type: Git; URL = this repository.
3. Enable **Update Revision on Launch** (recommended for GitOps demos).
4. Sync the project until status is successful.

Example used in the lab: project `gitops-networking-kam`.

### 3.2 Inventory (project-sourced — recommended for PoC)

1. **Inventories → Add** (e.g. `kam-gitops-networking`).
2. **Sources → Add** → **Sourced from a Project**.
3. Project = GitOps project above.
4. Inventory file = `inventory/hosts.yml`.
5. **Sync** the source.
6. Confirm hosts appear (e.g. `leaf4`) with correct `ansible_host` / `ansible_port`.

**Reachability tip:** The AAP EE often **cannot** reach containerlab management IPs (e.g. `172.20.20.x`). Prefer the lab hypervisor/DNS name plus published SSH port, for example:

```yaml
leaf4:
  ansible_host: containerlab   # or the lab host IP reachable from the EE
  ansible_port: 2225           # host port mapped to device :22
```

Keep the device’s real mgmt CIDR in `host_vars/.../device.yml` under `mgmt:` (intent only).

### 3.3 Credential

1. Create a **Machine** or **Network** credential with device username/password (or SSH key).
2. Do **not** put passwords in Git.
3. Attach this credential to every Job Template below.

### 3.4 Execution Environment

Use a Network EE that includes:

- `cisco.nxos` / `cisco.ios` / `arista.eos` / `junipernetworks.junos` as needed  
- `ansible.netcommon`, `ansible.utils`  
- Python libs as required (`netaddr`, etc.)

---

## 4. Job Templates to create

Create **four** Job Templates (names can match your org; examples below).

### 4.1 `test-connection` — verify SSH and read banner

| Field | Value |
|---|---|
| Playbook | `network-collection/ansible_collections/acme/netconfig/playbooks/check_reachability.yml` |
| Inventory | Project static inventory |
| Limit | device hostname (e.g. `leaf4`) |
| Credential | Network credential |
| Update revision on launch | Yes |

**What it does:** `show clock` and `show banner motd` over SSH (uses AAP credential).  
**Use when:** validating reachability or confirming a banner change without a local SSH password.

### 4.2 `gitops-network-kam` (or `gitops-onboard`) — Day 0

| Field | Value |
|---|---|
| Playbook | `.../playbooks/onboard.yml` |
| Inventory | Project static inventory |
| Limit | e.g. `leaf4` |
| Credential | Network credential |
| Update revision on launch | Yes |
| Extra vars (optional) | `target: leaf4` |

**What it does:** `base_config` (Jinja baseline) → banner → interfaces.

### 4.3 `gitops-day2-banner` — Day 2 banner only

| Field | Value |
|---|---|
| Playbook | `.../playbooks/day2_banner.yml` |
| Inventory | Project static inventory |
| Limit | e.g. `leaf4` |
| Credential | Network credential |
| Update revision on launch | Yes |
| Webhook | Enable → **GitHub** (optional but recommended for demo) |

**Git file to change:** `inventory/group_vars/all/banner.yml`

### 4.4 `gitops-day2-interfaces` — Day 2 interfaces only

| Field | Value |
|---|---|
| Playbook | `.../playbooks/day2_interfaces.yml` |
| Inventory | Project static inventory |
| Limit | e.g. `leaf4` |
| Credential | Network credential |
| Update revision on launch | Yes |
| Webhook | Enable → **GitHub** (optional) |

**Git file to change:** `inventory/host_vars/<hostname>/device.yml`

---

## 5. GitHub webhook (Day 2 automation)

For each Day-2 Job Template:

1. Open the JT → **Enable Webhook** → select **GitHub** → Save.
2. Copy **Webhook URL** and **Webhook Key**.
3. In GitHub: repo **Settings → Webhooks → Add webhook**:
   - Payload URL = JT Webhook URL  
   - Content type = `application/json`  
   - Secret = JT Webhook Key  
   - Which events = **Just the push event**  
4. Save.

**PoC note:** If both banner and interfaces JTs have push webhooks, **both** may run on every push. That is acceptable for a lab demo. For production, use path filters (e.g. GitHub Actions calling AAP) or a Workflow.

---

## 6. Demo / test procedures

### 6.1 Prerequisites checklist

- [ ] Device SSH works from the **lab host** (e.g. `ssh -p 2225 admin@127.0.0.1`).
- [ ] AAP inventory shows correct `ansible_host` / `ansible_port` (not an unroutable `172.20.20.x` from the EE).
- [ ] Project and inventory source are synced.
- [ ] Network credential attached to JTs.

### 6.2 Verify connectivity (no local device password needed)

1. Launch **`test-connection`** with Limit = device.
2. Confirm job succeeds.
3. In output, confirm **show clock** and **Banner on device**.

### 6.3 Day 0 — onboard a device

1. Ensure `host_vars/<device>/device.yml` and `group_vars/all/base_config.yml` are filled in.
2. Launch **onboard** JT (Limit = device).
3. Expect: base_config applied → banner → interfaces.
4. Platform notes (NX-OS / n9kv examples from this PoC):
   - Do not set arbitrary interface MTU unless system jumbo MTU is configured.
   - Routed Ethernet ports need L3 mode (`no switchport`) before `ip address`.
   - Prefer not rewriting mgmt0 over the same SSH session (`netcfg_configure_mgmt: false`).

### 6.4 Day 2 — banner change (GitOps)

1. Edit `inventory/group_vars/all/banner.yml` (change `banner.motd`).
2. Commit and push to the Git branch AAP tracks.
3. Either:
   - Webhook auto-starts **`gitops-day2-banner`**, or  
   - Manually **Launch** that JT (after project sync / update on launch).
4. Confirm in job output: `MOTD banner` is **changed**.
5. Optional: run **`test-connection`** and verify **Banner on device** matches Git.

### 6.5 Day 2 — interface change (GitOps)

1. Edit `inventory/host_vars/<device>/device.yml` under `interfaces:`.
2. Commit and push.
3. Webhook or manual launch of **`gitops-day2-interfaces`**.
4. Confirm interface / L3 tasks succeed in the job output.

---

## 7. Operational flow (summary diagram)

```text
  GitHub (intent YAML)
        │
        │  push / commit
        ▼
  AAP Project sync  ─────────────────────────────┐
        │                                          │
        ▼                                          ▼
  Day-2 JT (banner or interfaces)          Day-0 JT (onboard)
  [webhook or manual launch]               [manual launch]
        │                                          │
        ▼                                          ▼
  AAP Network credential ── SSH ──►  Network device
        │
        ▼
  test-connection JT (optional verify)
```

---

## 8. Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| `Connection timed out` to `172.20.20.x` | EE cannot route to containerlab mgmt net | Use lab host DNS/IP + published SSH port in `hosts.yml` |
| `kex_exchange_identification` / banner hang | Device SSH stuck / mid-boot | Restart/redeploy lab node; wait for local SSH password prompt |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | Expected after lab redeploy | `ssh-keygen -R '[host]:port'` (local only; AAP JTs disable strict host key checks) |
| `no hosts matched` / ignoring `network` | Inventory has no `network` group | Set JT **Limit** to hostname or use `target` extra var |
| `MTU can only be default or system jumbo MTU` | NX-OS n9kv restriction | Omit interface `mtu` or set system jumbo MTU first |
| `ip address ... Invalid command` on Ethernet | Port still switchport (L2) | Ensure L3 mode / `no switchport` before addressing (role sets `mode: layer3`) |
| Webhook did not run JT | Webhook not enabled / wrong secret / project not updated | Check GitHub delivery log; enable Update revision on launch |
| Ad-hoc `ping` works but playbook SSH fails | Different connection path or device SSH unhealthy | Use `test-connection` (raw CLI), not ICMP/Python ping alone |

---

## 9. Security reminders

- Never commit device passwords or API tokens to Git.
- Prefer AAP credentials (and CyberArk / external secret backends where required).
- Rotate any token that was shared outside AAP.
- Limit webhook JTs to the intended inventory and host Limit for demos.

---

## 10. Quick reference — playbooks

| Playbook | When to use |
|---|---|
| `check_reachability.yml` | SSH health + show clock + show banner |
| `onboard.yml` | First-time device onboarding |
| `day2_banner.yml` | GitOps banner enforcement (webhook) |
| `day2_interfaces.yml` | GitOps interface enforcement (webhook) |
| `site.yml` | Manual day-2 banner + interfaces together |
