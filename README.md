# openwrt-ap-ansible

Ansible for OpenWrt access points in dumb AP mode (Cudy WR3000S): 802.11r fast roaming, guest VLAN, rollback on lost connectivity.

Each AP gets complete `/etc/config/{system,network,wireless,firewall,dhcp}` files rendered from templates. Anything not in this repo is removed on the next run. **Do not change config via LuCI.**

No Python is needed on the APs: [community.openwrt](https://galaxy.ansible.com/ui/repo/published/community/openwrt/) runs its modules with ucode.

## Layout

| Path | Content |
| :--- | :--- |
| `inventory.yml` | APs, in run order (leaf APs first, AP1 last) |
| `group_vars/aps/vars.yml` | Shared settings: SSIDs, ports, firewall, SSH keys |
| `group_vars/aps/vault.yml` | SSIDs and PSKs, ansible-vault encrypted |
| `host_vars/<ap>.yml` | Per AP: IP, hostname, channels, guest gateway, trunk ports |
| `roles/openwrt_ap/` | Tasks, handlers, UCI templates |
| `roles/openwrt_upgrade/` | Firmware upgrade (`upgrade.yml`) |

## Setup

```sh
ansible-galaxy collection install -r requirements.yml   # into ./collections
```

Put the vault password from KeePass into `.ansible_vault_pass`. To create the vault initially, see `group_vars/aps/vault.yml.example`.

## Usage

```sh
ansible-playbook site.yml --check --diff          # dry run (vault values are masked in the diff)
ansible-playbook site.yml --diff                  # apply
ansible-playbook site.yml --diff --limit ap2      # single AP
```

Changes are applied with `reload_config`. APs run one at a time.

### Upgrade

```sh
ansible-playbook upgrade.yml --limit ap2   # one AP first
ansible-playbook upgrade.yml               # all
```

Sysupgrade to `openwrt_version` (official image, checksum verified), one AP at a time, keeping the config. APs already on that version are skipped. For a major release, check the templates against the new defaults (`/etc/config`, `/etc/uci-defaults` in the image) and upgrade one AP first.

### Guest network

Guest Wi-Fi of all APs is bridged into VLAN 20 (tagged between the APs, `guest_trunk_ports`). The AP with `guest_gateway: true` routes `192.168.20.0/24`: DHCP, DNS, NAT into the LAN, no access to the LAN itself. One subnet on all APs, so clients keep their IP when roaming. The LAN is untagged on all ports.

### Rollback

Before rendering, every run backs up the config (`sysupgrade -b`) and starts a timer. If the AP is not reachable within `rollback_timeout` (300 s) after the change, the backup is restored. A failed run can therefore leave the AP unreachable for up to 5 minutes. The first rollout of risky changes (bridge, VLANs on AP1) should still be done on site.

## New AP

1. Flash OpenWrt: <https://openwrt.org/toh/cudy/wr3000s_v1>
2. Add a static DHCP lease in Pi-hole (`192.168.71.1x`, MAC from the label).
3. Add the AP to `inventory.yml` and `host_vars/`.
4. Cable: AP `wan` to the LAN (internet for package install), laptop to AP `lan1`. Do **not** plug a fresh AP's `lan` ports into the LAN (it runs its own DHCP server on `192.168.1.1`).
5. Bootstrap (fresh OpenWrt allows root SSH without password):
   ```sh
   ansible-playbook site.yml --diff --limit apN -e ansible_host=192.168.1.1 -e rollback=false
   ```
   Afterwards all ports are bridged into the LAN and the AP gets its IP from Pi-hole.
