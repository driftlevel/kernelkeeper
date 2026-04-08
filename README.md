# kernelkeeper

### Script for maintaining kernels on Debian-based distros

A consistent issue with Debian-based distros when using `unattended-upgrades` is that they often don't remove old kernels at all, or don't remove enough of them. This can quickly fill up your filesystem — especially if you have a separate `/boot` partition. **kernelkeeper** is a simple shell script that intelligently purges unused kernels while protecting the ones that matter.

## What it does

For every kernel "variant" installed on the system (e.g. `generic`, `rt`, `pve`, `lowlatency`), kernelkeeper keeps:

- The **currently running** kernel (`uname -r`), and
- The **newest installed** kernel for that variant (the N-1 fallback)

Everything older is purged via `apt-get purge`. If a metapackage (e.g. `linux-image-generic`, `proxmox-default-kernel`) gets pulled out as a dependency during the purge, kernelkeeper detects this and reinstalls it automatically so future upgrades continue to work.

## Supported distros

Any Debian/Ubuntu-family system, including:

- Debian, Devuan, Ubuntu, Linux Mint, Pop!_OS, Zorin, elementary, Raspbian
- Armbian
- Proxmox VE (both PVE 7 with `pve-kernel-*` and PVE 8+ with `proxmox-kernel-*`, including signed variants and systems mid-migration where both naming schemes coexist)

The distro is verified via `lsb_release` before any action is taken.

## Usage

```
kernelkeeper              # normal run — purges old kernels
kernelkeeper --dry-run    # show what would be removed, make no changes
kernelkeeper --help       # print usage
```

Must be run as root.

## Installation

Drop the script somewhere on root's `PATH` and make it executable:

```sh
sudo install -m 0755 kernelkeeper /usr/local/sbin/kernelkeeper
```

### Run it on a schedule

The simplest setup — symlink it into one of the cron directories:

```sh
sudo ln -s /usr/local/sbin/kernelkeeper /etc/cron.weekly/kernelkeeper
```

Daily or monthly works too, depending on how aggressively your system installs new kernels.

### Run it automatically on every apt operation

To have kernelkeeper run before every apt install/upgrade, create `/etc/apt/apt.conf.d/99-kernelkeeper`:

```
Dpkg::Pre-Install-Pkgs {"/usr/local/sbin/kernelkeeper";};
```

This is especially useful on systems with a small `/boot` partition that can fail mid-upgrade if there isn't enough free space for the new kernel.

## How it handles Proxmox

Proxmox VE changed its kernel package naming between major versions:

| PVE version | Kernel packages              | Header packages               |
|-------------|------------------------------|-------------------------------|
| 7 and older | `pve-kernel-*`               | `pve-headers-*`               |
| 8+          | `proxmox-kernel-*`           | `proxmox-headers-*`           |

kernelkeeper queries both, deduplicates versions across the two schemes (so a system upgraded from PVE 7 to PVE 8 doesn't get confused), and also handles the `-signed` virtual packages used for secure boot.

The `proxmox-default-kernel` and `proxmox-default-headers` metapackages are tracked and restored if removed.

## Safety notes

- Runs `set -euo pipefail` and aborts on the first error.
- Never touches the running kernel or the newest installed kernel for any variant.
- Skips kernel-series metapackages (e.g. `pve-kernel-5.15`, `proxmox-kernel-6.8`) that have no patch-level component.
- Use `--dry-run` first if you're unsure what will be removed.
