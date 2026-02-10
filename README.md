# Ansible MikroTik Upgrade

Automated two-phase upgrade playbook for MikroTik RouterOS devices. Brings older routers up to a minimum baseline version, then switches them to the **long-term (LTS)** release channel and installs the latest available LTS firmware.

## How It Works

The playbook executes in two sequential phases against every host in the `mikrotik` inventory group.

### Phase 1 — Baseline Upgrade (conditional)

Runs **only** when the current RouterOS version is below `7.12.1`.

1. Checks for available updates on the currently configured channel.
2. Installs the update and waits for the device to reboot.
3. Upgrades RouterBOARD firmware if `current-firmware` differs from `upgrade-firmware`, then reboots.

### Phase 2 — LTS Channel Upgrade (always)

1. Reads the current update channel; switches to `long-term` only if it is not already set (idempotent).
2. Checks for updates on the LTS channel.
3. Installs the update if a newer LTS version is available.
4. Upgrades RouterBOARD firmware if needed, then reboots.

Both phases reuse the same roles — `mikrotik_routeros_upgrade` and `mikrotik_routerboard_upgrade` — so the upgrade logic is defined once.

## Project Structure

```
.
├── ansible.cfg                          # Ansible configuration
├── requirements.yml                     # Collection dependencies
├── playbooks/
│   └── upgrade.yml                      # Main playbook (two-phase logic)
├── inventory/
│   └── prod.ini                         # Inventory file (create your own)
└── roles/
    ├── mikrotik_facts/                  # Gathers RouterOS and RouterBOARD versions
    │   └── tasks/main.yml
    ├── mikrotik_routeros_upgrade/       # RouterOS upgrade (check → install → verify)
    │   ├── defaults/main.yml            #   ros_wait_seconds: 120
    │   └── tasks/main.yml
    └── mikrotik_routerboard_upgrade/    # RouterBOARD firmware upgrade + reboot
        ├── defaults/main.yml            #   rb_wait_seconds: 60
        └── tasks/main.yml
```

## Prerequisites

- **Ansible** 2.14+
- **community.routeros** collection

Install the required collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Inventory

Create `inventory/prod.ini` (or pass `-i` at runtime) with a `[mikrotik]` group. Hosts must be reachable via the RouterOS API or SSH, depending on your `community.routeros` connection settings.

```ini
[mikrotik]
router1  ansible_host=192.168.1.1
router2  ansible_host=192.168.1.2
```

Refer to the [community.routeros documentation](https://docs.ansible.com/ansible/latest/collections/community/routeros/) for connection parameters (`ansible_connection`, `ansible_network_os`, etc.).

## Usage

Upgrade all devices in the `mikrotik` group:

```bash
ansible-playbook playbooks/upgrade.yml
```

Limit to a single host:

```bash
ansible-playbook playbooks/upgrade.yml -l router1
```

Use an external inventory:

```bash
ansible-playbook playbooks/upgrade.yml -i /etc/ansible/hosts -l router1
```

## Configuration

| Variable | Default | Defined in | Description |
|---|---|---|---|
| `ros_wait_seconds` | `120` | `mikrotik_routeros_upgrade/defaults` | Seconds to wait after RouterOS install (reboot time) |
| `rb_wait_seconds` | `60` | `mikrotik_routerboard_upgrade/defaults` | Seconds to wait after RouterBOARD upgrade + reboot |

Override via `group_vars`, `host_vars`, or `--extra-vars`:

```bash
ansible-playbook playbooks/upgrade.yml -e ros_wait_seconds=180
```

## Design Notes

- **Idempotent** — safe to run multiple times. Roles compare the installed version against the latest available and skip the install when already up to date. The channel switch only fires when the channel is not already `long-term`.
- **Reboot-safe** — tasks that trigger a device reboot use `failed_when: false` to gracefully handle the expected SSH disconnect without polluting the output with `fatal...ignoring`.
- **Shared facts** — roles communicate through `ros_ver` and `rb_fw` host facts. Intermediate `_tmp` variables and conditional `set_fact` prevent skipped `register` tasks from overwriting these facts with empty objects.
