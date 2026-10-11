# Proxmox VE on existing Debian hosts (on-prem / air-gapped)

Turns Debian 13 (trixie) hosts — bare metal or VMs — into Proxmox VE 9 nodes with one self-contained
role, `proxmox_install`. Nothing is created or partitioned: the disk layout (including CIS partitions) is whatever the Debian installer made.

```
proxmox-onprem/
├── site.yml                      # applies roles/proxmox_install
├── roles/proxmox_install/        # install, CIS mount check, auditd, network, storage
├── group_vars/proxmox.yml        # mirrors, CIS mount check, auditd
├── host_vars/<name>.yml          # per node: network bridge, storage (you create these)
└── inventory.ini.example
```

## Requirements

- Debian 13 amd64, minimal install (no desktop / NetworkManager), static IP, SSH access with sudo.
- For full VMs inside a VM: nested virtualization on the outer hypervisor (on Proxmox: CPU type `host`).
- Air-gapped: reachable mirrors of the Debian (`trixie`, `trixie-updates`, `trixie-security`) and
  Proxmox VE (`trixie` / `pve-no-subscription`) repositories, and the Proxmox archive keyring
  (`proxmox-archive-keyring-trixie.gpg`) on the controller. `proxmox-offline-mirror` from Proxmox can
  build both mirrors and move them across the air gap on removable media.

## Run

```sh
git clone https://github.com/resumepilot360/proxmox-onprem.git
cd proxmox-onprem
cp inventory.ini.example inventory.ini        # add your hosts
$EDITOR group_vars/proxmox.yml                # mirror URLs, keyring, mount check
ansible-playbook site.yml                     # --ask-become-pass if sudo needs a password
```

The `proxmox_install` role follows the official "Install Proxmox VE on Debian 13 Trixie" steps:

1. Checks Debian 13 amd64 and makes the hostname resolve to the node IP in `/etc/hosts`.
2. Sets up the Debian and Proxmox repositories (your mirrors if given) and the Proxmox keyring.
3. Upgrades the system, installs the Proxmox kernel and reboots into it.
4. Installs `proxmox-ve`, `postfix` (local only), `open-iscsi` and `chrony`, disables the enterprise
   repositories, and removes the Debian kernels and `os-prober`.

Then, each only when its variable is set:

5. Checks the CIS mount points and options, live and in `/etc/fstab`, and fails on drift.
6. Installs and enables `auditd`.
7. Writes `/etc/network/interfaces` (e.g. a `vmbr0` bridge for guests) and reloads networking.
8. Writes `/etc/pve/storage.cfg`.

Every step is idempotent, so re-running on an installed node only applies updates and configuration.
Afterwards the web UI is at `https://<host>:8006` (log in as `root`, realm *Linux PAM*).

## Role variables (`proxmox_install`)

| Variable | Default | Purpose |
|---|---|---|
| `proxmox_install_debian_sources` | `""` (keep host's) | Debian repositories as deb822 content; also empties `sources.list` |
| `proxmox_install_pve_repo_url` | `http://download.proxmox.com/debian/pve` | Proxmox VE repository or mirror |
| `proxmox_install_pve_repo_component` | `pve-no-subscription` | Repository component |
| `proxmox_install_keyring_src` | `""` | Keyring file on the controller (air-gapped) |
| `proxmox_install_keyring_url` | Proxmox download URL | Used when `keyring_src` is empty |
| `proxmox_install_disable_enterprise_repo` | `true` | Disable `pve-enterprise` / `ceph` repositories |
| `proxmox_install_node_ip` | default-route IPv4 | IP the hostname resolves to |
| `proxmox_install_domain` | host's domain or `localdomain` | Domain for the `/etc/hosts` entry |
| `proxmox_install_required_mounts` | `{}` | Mount point → required options, e.g. `/tmp: [nodev, nosuid, noexec]` |
| `proxmox_install_auditd_enabled` | `false` | Install and enable `auditd` (rules not managed) |
| `proxmox_install_network_interfaces` | `""` | Full `/etc/network/interfaces`; empty = not managed |
| `proxmox_install_storage_cfg` | `""` | Full `/etc/pve/storage.cfg`; empty = not managed |

## Networking

Installing Proxmox replaces `ifupdown` with `ifupdown2` and keeps the installer's interface config, so
the node stays reachable. For guests you need a bridge: set `proxmox_install_network_interfaces` to the
full `/etc/network/interfaces` with a `vmbr0` that carries the node IP. The previous file is kept as a
backup next to it. For example, in `host_vars/pve-a.yml`:

```yaml
proxmox_install_network_interfaces: |
  auto lo
  iface lo inet loopback

  iface ens18 inet manual

  auto vmbr0
  iface vmbr0 inet static
  	address 192.168.10.21/24
  	gateway 192.168.10.1
  	bridge-ports ens18
  	bridge-stp off
  	bridge-fd 0
```

Get the interface name and addresses right: a wrong file here cuts the node off the network. Keep
console access to the VM while trying it the first time.

## Storage

A Debian-installed node starts with only `local` (`/var/lib/vz`, for ISOs, templates and backups). To
store VM disks elsewhere, point `proxmox_install_storage_cfg` at storage that already exists, e.g. an
LVM-thin pool you created in the installer's volume group, or a directory on its own partition:

```yaml
proxmox_install_storage_cfg: |
  dir: local
  	path /var/lib/vz
  	content iso,vztmpl,backup

  lvmthin: local-lvm
  	thinpool data
  	vgname vg0
  	content rootdir,images
```
