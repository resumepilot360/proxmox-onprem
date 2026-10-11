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

- Debian 13 amd64, minimal install (no desktop / NetworkManager), static IP, and a user that can log
  in over SSH and become root, with `sudo` or with `su` and the root password.
- For full VMs inside a VM: nested virtualization on the outer hypervisor (on Proxmox: CPU type `host`).
- Air-gapped, either:
  - the Proxmox VE installer ISO (e.g. `proxmox-ve_9.2-1.iso`) copied to the host. Nothing else is
    needed, see [Air-gapped with the ISO](#air-gapped-with-the-proxmox-ve-iso); or
  - reachable mirrors of the Debian (`trixie`, `trixie-updates`, `trixie-security`) and
  Proxmox VE (`trixie` / `pve-no-subscription`) repositories, and the Proxmox archive keyring
  (`proxmox-archive-keyring-trixie.gpg`) on the controller. `proxmox-offline-mirror` from Proxmox can
  build both mirrors and move them across the air gap on removable media.

## Run

```sh
git clone https://github.com/resumepilot360/proxmox-onprem.git
cd proxmox-onprem
cp inventory.ini.example inventory.ini        # add your hosts
$EDITOR group_vars/proxmox.yml                # ISO path or mirrors, mount check
ansible all -m ping                           # check SSH
ansible-playbook site.yml --ask-become-pass   # root password (su) or your sudo password
```

The `proxmox_install` role follows the official "Install Proxmox VE on Debian 13 Trixie" steps:

1. Checks Debian 13 amd64 and makes the hostname resolve to the node IP in `/etc/hosts`.
2. Sets up the package source: the Proxmox VE ISO on the host, or the Debian and Proxmox
   repositories (your mirrors if given) and the Proxmox keyring.
3. Upgrades the system (with the ISO: to the ISO's versions), installs the Proxmox kernel and reboots
   into it.
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
| `proxmox_install_iso_path` | `""` | Proxmox VE ISO on the host, used as the only package source; empty = repositories |
| `proxmox_install_iso_store` | `/var/lib/proxmox-iso/proxmox-ve.iso` | Where the ISO is moved to (on disk, survives reboots) |
| `proxmox_install_iso_mount` | `/mnt/pve-iso` | Where the ISO is mounted (also at boot, via `/etc/fstab`) |
| `proxmox_install_iso_disable_other_sources` | `true` | With the ISO, disable the host's other apt sources (backups kept) |
| `proxmox_install_local_debs_dir` | `""` | Directory on the controller with extra `.deb` files to install (e.g. auditd) |
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

## Air-gapped with the Proxmox VE ISO

The Proxmox VE installer ISO contains two apt repositories that together hold Proxmox VE and every
Debian package it needs, so the ISO alone is enough; no mirror or internet access.

1. Copy the ISO to the host, e.g. `scp proxmox-ve_9.2-1.iso youruser@192.168.10.21:/tmp/`.
2. Set `proxmox_install_iso_path: /tmp/proxmox-ve_9.2-1.iso` in `group_vars/proxmox.yml`.
3. Run the playbook.

What the role does with it:

- Moves the ISO to `/var/lib/proxmox-iso/` (needs ~2 GB free there), because `/tmp` is a tmpfs on
  Debian 13 and is emptied by the reboot into the Proxmox kernel. It's mounted at `/mnt/pve-iso`, now
  and at boot, and stays the node's package source.
- Adds the ISO's repositories (they are unsigned, so they're marked trusted) and pins them above
  everything else. Some ISO packages need exact versions of Debian packages (e.g. `perl` needs the
  same `perl-base`), so the host is first aligned with the ISO's versions, which can mean small
  downgrades if the host was installed from a newer Debian ISO. The result matches a node installed
  from the Proxmox ISO directly.
- Disables the host's other apt sources (internet mirrors, the Debian DVD), which can't be reached
  and would make `apt update` fail. `sources.list` is kept as `sources.list.pre-pve-iso`.

Packages that aren't on the ISO, such as `auditd` (and `sudo`), have to be brought in as `.deb` files.
On a machine with internet access and Debian 13, download them with their dependencies, copy the
directory next to the playbook and point the role at it:

```sh
mkdir debs && cd debs && apt-get download auditd audispd-plugins libauparse0t64
```
```yaml
proxmox_install_local_debs_dir: files/debs
proxmox_install_auditd_enabled: true
```

## Logging in without sudo

When the Debian installer is given a root password, it doesn't install `sudo` and the user it creates
isn't an administrator. Ansible then logs in as that user and switches to root with `su`: set
`ansible_become_method=su` in the inventory (as in `inventory.ini.example`) and run with
`--ask-become-pass`, entering the root password. SSH as root isn't needed.

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
