---
name: proxmox-ansible
description: "Use when: working on Proxmox VE 9.1 Ansible roles, Debian 13 to Proxmox bootstrap, no-subscription repositories, chrony and DNS pinning, PVE users, Proxmox firewall, or ifupdown2 networking."
---

# Proxmox Ansible Skill

Use this skill for this repository's Proxmox VE 9.1 automation.

## Bootstrap Baseline

- The playbook must support both plain Debian 13 hosts and existing Proxmox VE 9.x hosts.
- Keep `roles/proxmox_install` idempotent: it installs missing PVE packages and starts services, but existing PVE hosts should remain unchanged except for drift.
- Debian bootstrap packages default to `proxmox-default-kernel`, `proxmox-ve`, `postfix`, `open-iscsi`, `ifupdown2`, and `chrony`.
- Proxmox requires the node IP and hostname to resolve locally in `/etc/hosts`; preserve `proxmox_manage_hosts_entry` behavior.
- Do not force reboots by default. Use `proxmox_reboot_after_kernel_install` for hosts where automated reboot is acceptable.

## Repository Baseline

- PVE 9.1 is Debian trixie.
- Disable enterprise files under `/etc/apt/sources.list.d/`, including `pve-enterprise.*`, `ceph.*`, and `ceph-enterprise.*`.
- PVE no-subscription deb822 source uses `http://download.proxmox.com/debian/pve`, suite `trixie`, component `pve-no-subscription`.
- Ceph no-subscription deb822 source uses `http://download.proxmox.com/debian/ceph-squid`, suite `trixie`, component `no-subscription`.
- Debian mirror must stay configurable and defaults to `http://debian-archive.trafficmanager.net/debian/`.

## Host Baseline

- Chrony upstream servers must be only `proxmox_ntp_servers`; disable existing `server`, `pool`, and `peer` entries in vendor configs.
- DNS upstream servers must be only `proxmox_dns_servers`; the role owns `/etc/resolv.conf` when `proxmox_manage_resolv_conf` is true.
- Users are Linux PAM users and Proxmox PAM-realm users (`<name>@pam`) when `proxmox_manage_pve_users` is true.
- PVE admin role is granted with `pveum acl modify / --users <name>@pam --roles PVEAdmin` unless user variables override path or role.
- Datacenter firewall is configured through the Proxmox cluster API with `pvesh`: `set /cluster/firewall/options -enable 1` and reconcile the `management` IPSet at `/cluster/firewall/ipset/management` from `proxmox_mgmt_networks`. Do not template `/etc/pve/firewall/cluster.fw` directly; pmxcfs disallows atomic renames and the API performs locking and validation.

## Network Baseline

- Use ifupdown2 and apply changes with `ifreload -a`.
- Keep the management interface separate from `proxmox_bridge_iface`.
- The default bridge is configurable but defaults to `vmbr0`.
- Avoid two default gateways in `/etc/network/interfaces`; prefer the management interface gateway unless a host explicitly needs another route design.
- Management-interface changes must remain guarded by `proxmox_allow_mgmt_iface_change`.
