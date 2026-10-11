# Proxmox VE on existing Debian hosts (on-prem / air-gapped)

Turns Debian 13 (trixie) hosts — bare metal or VMs — into Proxmox VE 9 nodes with one self-contained
role, `proxmox_install`. Nothing is created or partitioned: the disk layout (including CIS partitions) is whatever the Debian installer made.

```
proxmox-onprem/
├── site.yml                      # existing Debian hosts -> Proxmox VE (roles/proxmox_install)
├── vm.yml                        # new VMs on a Proxmox server -> Debian -> Proxmox VE, one run
├── baremetal.yml                 # bare-metal servers -> Debian -> Proxmox VE, from a built ISO
├── node.yml                      # second stage of vm.yml / baremetal.yml (Proxmox VE install)
├── roles/proxmox_install/        # install, CIS mount check, auditd, network, storage
├── roles/proxmox_autoinstall/    # unattended Debian 13 install (CIS partitions): VMs or ISOs
├── group_vars/proxmox.yml        # ISO or mirrors, CIS mount check, auditd
├── group_vars/hypervisor.yml     # the VMs to create (vm.yml)
├── group_vars/builder.yml        # the bare-metal servers (baremetal.yml)
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

## New nodes from scratch: VMs or bare metal

Instead of installing Debian by hand first, `vm.yml` and `baremetal.yml` start from nothing and end
with Proxmox VE nodes. Both use the role `proxmox_autoinstall`, which installs Debian 13 unattended
from the Debian **DVD** (no mirror needed) with a generated preseed file: the CIS partitions below, a
static IP and root's SSH key. Then `node.yml` installs Proxmox VE on the new nodes from the Proxmox
ISO with `proxmox_install` (same as `site.yml`, air-gapped), moves the IP to a `vmbr0` bridge and
turns the rest of the disk into the `local-lvm` thin pool.

Both need a root password hash for the new nodes (`openssl passwd -6`; also the web UI login).
Ansible logs in to them as root with your SSH key (`~/.ssh/id_ed25519.pub` by default, see
`proxmox_autoinstall_ssh_public_key`). `group_vars/proxmox.yml` (CIS mount check, auditd) applies to
the new nodes too.

### As VMs on a Proxmox server (`vm.yml`)

1. **On the server** (inventory group `hypervisor`): for each entry in `proxmox_autoinstall_nodes`,
   creates the VM (CPU type `host` for nested virtualization), boots the Debian installer directly
   with the preseed file in its initrd, and installs Debian. All VMs install in parallel. When an
   install is done the VM powers off; the role boots it from its disk with the Proxmox ISO in its
   CD-ROM drive.
2. **On each new VM**: `node.yml`, after copying the Proxmox ISO from the CD-ROM.

Needs, on the Proxmox server in storage `local` (`/var/lib/vz/template/iso/`): the Debian 13 DVD ISO
(`debian-13.x.y-amd64-DVD-1.iso`) and the Proxmox VE ISO (`proxmox-ve_9.2-1.iso`).

```sh
cp inventory.ini.example inventory.ini     # [hypervisor]: your Proxmox server, as root
$EDITOR group_vars/hypervisor.yml          # ISOs, password hash, VMs (vmid, name, address, gateway)
ansible-playbook vm.yml
```

If the server can't run VMs with hardware virtualization (e.g. it is itself a VM without nested
virtualization), set `kvm: false` on the VM: it then runs fully emulated, which works but is many
times slower. If the controller can't reach the VMs' network (e.g. a NAT bridge on the server), set
`proxmox_autoinstall_ssh_jump` (e.g. `root@pve-host`).

### On bare-metal servers (`baremetal.yml`)

1. **On the build host** (inventory group `builder`, the controller by default; needs `xorriso` and
   ~6 GB per server): builds one ISO per server, `files/autoinstall/pve-autoinstall-<name>.iso`: the
   Debian DVD with the server's preseed file, a boot menu that starts the install by itself (BIOS
   and UEFI) and the Proxmox ISO inside.
2. **You boot each server from its ISO**: write it to a USB stick (`dd if=<iso> of=/dev/<stick>
   bs=4M`) or attach it as BMC virtual media (iDRAC, iLO, IPMI). Debian installs on the disk given as
   `disk` (**erased**, check with `lsblk`), copies the Proxmox ISO from the media, and the server
   powers off. **Remove the ISO before powering it on again**, or it reinstalls.
3. **Meanwhile the play waits** (up to `proxmox_autoinstall_baremetal_wait`, default 4 h) for each
   server to answer over SSH, then runs `node.yml` on it.

Put the Debian DVD and Proxmox ISOs in `files/` (or set their paths), then:

```sh
cp inventory.ini.example inventory.ini     # [builder] localhost is enough
$EDITOR group_vars/builder.yml             # password hash, servers (name, address, gateway, disk)
ansible-playbook baremetal.yml             # builds the ISOs, then waits for the servers
```

The ISOs contain the root password hash and your SSH public key: keep them like secrets. If the
servers boot from their disk after the install anyway (typical with UEFI, where the installer puts
Debian first in the boot order), `proxmox_autoinstall_baremetal_reboot: true` reboots instead of
powering off.

### Disk layout

Sizes in `proxmox_autoinstall_partitions` (GiB); the default VM disk is 100 GiB:

| Mount | Size | Options |
|---|---|---|
| `/boot/efi` (UEFI only) | 0.5 | |
| `/boot` (partition) | 1 | nodev, nosuid |
| `/` | 20 | |
| `/home` | 2 | nodev, nosuid |
| `/tmp` | 4 | nodev, nosuid, noexec |
| `/var` | 20 | nodev, nosuid |
| `/var/tmp` | 4 | nodev, nosuid, noexec |
| `/var/log` | 8 | nodev, nosuid, noexec |
| `/var/log/audit` | 4 | nodev, nosuid, noexec |
| `/dev/shm` (tmpfs) | | nodev, nosuid, noexec |
| swap | 4 | |
| thin pool `pve/data` (`local-lvm`) | rest | |

### Re-running and troubleshooting

Re-running is safe: existing VMs and installed servers are not reinstalled (only `node.yml` runs
again). To rebuild a VM, delete it on the server (`qm stop <vmid>; qm destroy <vmid>`); for any
reinstalled node, remove its old SSH host key (`ssh-keygen -R <ip>`). If a VM install doesn't finish
within `proxmox_autoinstall_install_timeout` (default 1 h), the installer is waiting on a question:
open the VM's console in the web UI (on bare metal: the server's console).

## Role variables (`proxmox_install`)

| Variable | Default | Purpose |
|---|---|---|
| `proxmox_install_iso_path` | `""` | Proxmox VE ISO on the host, used as the only package source; empty = repositories |
| `proxmox_install_iso_store` | `/var/lib/proxmox-iso/proxmox-ve.iso` | Where the ISO is moved to (on disk, survives reboots) |
| `proxmox_install_iso_mount` | `/mnt/pve-iso` | Where the ISO is mounted (also at boot, via `/etc/fstab`) |
| `proxmox_install_debian_iso_path` | `""` | Debian 13 DVD ISO on the host: second package source (auditd, sudo, ...) |
| `proxmox_install_debian_iso_store` | `/var/lib/proxmox-iso/debian-dvd.iso` | Where the DVD is moved to |
| `proxmox_install_debian_iso_mount` | `/mnt/debian-iso` | Where the DVD is mounted |
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

The Proxmox ISO only holds what Proxmox VE needs (~860 packages). For other Debian packages, such as
`auditd` or `sudo`, copy the **Debian 13 DVD** (`debian-13.x.y-amd64-DVD-1.iso`, ~4 GB, several
thousand packages, signed by Debian) to the host as well and set
`proxmox_install_debian_iso_path: /tmp/debian-13.7.0-amd64-DVD-1.iso`. It's kept and mounted like the
Proxmox ISO (`/mnt/debian-iso`) and added as a second source, pinned below the Proxmox ISO so
Proxmox's exact-version dependencies keep winning.

`apt update` prints `N: Missing Signed-By ...` for the Proxmox ISO's repositories: a notice, not an
error. They are unsigned on the ISO, so they're marked trusted instead.

Without the DVD, packages that aren't on the Proxmox ISO have to be brought in as `.deb` files.
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
