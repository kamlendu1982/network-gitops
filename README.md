# Network golden config POC (IOS, EOS, Junos)

One collection (`acme.netconfig`) with one role per device class. Each role has three tasks:

| Task | When it runs | How it works |
|---|---|---|
| `base_config` | First-time onboarding only (`onboard` playbook) | One Jinja template per device class renders the full baseline, pushed with the vendor's config module |
| `banner` | Onboarding and day 2 | `*_banner` module |
| `interfaces` | Onboarding and day 2 | `*_interfaces` + `*_l3_interfaces` resource modules, `state: replaced` |

## Layout

```text
network-poc/
├── ansible.cfg
├── collections/requirements.yml
├── inventory/                                   # POC sample vars (the customer will own these)
│   ├── hosts.yml                                # groups: ios, eos, junos (under "network")
│   ├── group_vars/
│   │   ├── all/base_config.yml                  # vendor-neutral onboarding baseline
│   │   ├── all/banner.yml
│   │   ├── ios/connection.yml                   # network_cli + enable
│   │   ├── eos/connection.yml                   # network_cli + enable
│   │   └── junos/connection.yml                 # netconf
│   └── host_vars/
│       ├── ios-rtr01/device.yml                 # mgmt + interfaces
│       ├── eos-sw01/device.yml
│       └── junos-rtr01/device.yml
└── network-collection/ansible_collections/acme/netconfig/
    ├── galaxy.yml
    ├── meta/runtime.yml
    ├── playbooks/
    │   ├── onboard.yml                          # day 0: base_config + banner + interfaces
    │   └── site.yml                             # day 2: banner + interfaces
    └── roles/
        ├── ios/  eos/  junos/                   # identical layout per device class
        │   ├── defaults/main.yml
        │   ├── meta/argument_specs.yml          # shared data contract (same in all roles)
        │   ├── tasks/
        │   │   ├── main.yml
        │   │   ├── base_config.yml
        │   │   ├── banner.yml
        │   │   └── interfaces.yml
        │   └── templates/
        │       ├── base_config.j2               # the one onboarding template for this class
        │       ├── interfaces.yml.j2            # neutral model -> *_interfaces
        │       └── l3_interfaces.yml.j2         # neutral model -> *_l3_interfaces
```

## How the vars flow

`group_vars/all/base_config.yml` and `host_vars/<device>/device.yml` hold vendor-neutral intent
(domain, DNS, NTP, syslog, SNMP, management IP, interface list). Each role's templates translate
it into that vendor's syntax, so the same vars produce IOS, EOS or Junos config.
`meta/argument_specs.yml` validates the vars before anything runs.

## Usage

Run from the `network-poc/` folder.

| Purpose | Command |
|---|---|
| Install dependencies | `ansible-galaxy collection install -r collections/requirements.yml` |
| Preview onboarding offline (writes `rendered/<host>_base_config.cfg`) | `ansible-playbook acme.netconfig.onboard -e netcfg_state=rendered` |
| Onboard one device | `ansible-playbook acme.netconfig.onboard -e target=ios-rtr01` |
| Onboard one device class | `ansible-playbook acme.netconfig.onboard -e target=eos` |
| Day-2 enforcement | `ansible-playbook acme.netconfig.site` |
| Day-2 dry run | `ansible-playbook acme.netconfig.site --check --diff` |
| One feature only | `ansible-playbook acme.netconfig.site --tags interfaces` |

## Notes

- **Secrets:** the local admin user is only rendered when `base_admin_secret` is passed at run time
  (AAP credential backed by CyberArk). It must already be hashed in the vendor's format: type 9 for
  IOS, sha512 for EOS, `$6$` crypt for Junos. On IOS the AAA lines are tied to the same condition,
  so onboarding can't lock you out of a device with no local user.
- **Junos safety:** base config is committed with `commit confirmed` (5 minutes, set by
  `netcfg_junos_confirm_min`), then confirmed only if the device is still reachable.
- **Backups:** IOS, EOS and Junos all take a running-config backup into `backups/` before onboarding.
- **Banner** isn't a resource module, so in rendered mode it's only printed.
- **Junos rendered output** for interfaces is NETCONF XML, since the Junos modules use NETCONF.
- Tested offline (rendered mode) with ansible-core 2.18, cisco.ios 11.6, arista.eos 12.3 and
  junipernetworks.junos 11.0. Passes ansible-lint (production profile).
- Requires the `netaddr` Python package in the EE, plus `ncclient`, `jxmlease` and `xmltodict` for Junos.
