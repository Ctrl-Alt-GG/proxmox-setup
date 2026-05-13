# proxmox-setup

Ansible playbook for turning Debian 13 hosts into Proxmox VE 9.1 hosts and bringing existing Proxmox VE installs back onto a consistent baseline.

## What It Configures

- Proxmox no-subscription and Ceph no-subscription repositories.
- Disabled Proxmox and Ceph enterprise repositories when present.
- Proxmox VE packages installed on plain Debian 13 hosts.
- Debian packages from `http://debian-archive.trafficmanager.net/debian/` by default.
- Full package upgrades with autoremove and autoclean.
- Chrony with only `192.168.130.1` and `192.168.130.53` as upstream NTP servers by default.
- Host DNS with only `192.168.130.1` and `192.168.130.53` as upstream DNS servers by default.
- Configurable Linux PAM users with passwords and SSH keys downloaded from configurable URLs.
- Matching Proxmox PAM users with configurable PVE roles, defaulting to `PVEAdmin` at `/`.
- Datacenter-level Proxmox firewall enabled with a `management` IPSet containing `192.168.0.0/16` by default.
- Configurable management interface and a separate configurable interface bound to `vmbr0`.

## Layout

- `site.yml` applies focused roles in dependency order.
- `group_vars/proxmox.yml` contains the shared defaults.
- `inventory/hosts.yml` is an example inventory.
- `roles/proxmox_repositories` manages Debian, PVE, and Ceph repositories.
- `roles/proxmox_install` installs Proxmox VE packages on Debian 13 and starts PVE services.
- `roles/proxmox_updates` upgrades packages.
- `roles/proxmox_chrony` pins upstream NTP servers.
- `roles/proxmox_dns` pins upstream DNS servers.
- `roles/proxmox_users` manages Linux and Proxmox users.
- `roles/proxmox_firewall` reconciles the datacenter firewall options and the `management` IPSet via the Proxmox cluster API (`pvesh`).
- `roles/proxmox_network` manages `/etc/network/interfaces` and runs `ifreload -a`.

## Usage

Install collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

Edit `inventory/hosts.yml` for your nodes, then run:

```bash
ansible-playbook site.yml
```

For a safer first pass:

```bash
ansible-playbook site.yml --check --diff
```

## Important Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `proxmox_install_ve` | `true` | Install Proxmox VE packages when missing. Existing PVE hosts remain idempotent. |
| `proxmox_install_packages` | `proxmox-default-kernel`, `proxmox-ve`, `postfix`, `open-iscsi`, `ifupdown2`, `chrony` | Packages needed to bootstrap Debian 13 into Proxmox VE. |
| `proxmox_reboot_after_kernel_install` | `false` | Reboot automatically into the Proxmox kernel after package installation. |
| `proxmox_node_ip` | `""` | Optional IP used for the Proxmox `/etc/hosts` entry; falls back to management IP or discovered default IPv4. |
| `proxmox_node_hostname` | `""` | Optional hostname used in `/etc/hosts`; falls back to gathered facts. |
| `proxmox_node_fqdn` | `""` | Optional FQDN used in `/etc/hosts`; falls back to gathered facts. |
| `proxmox_debian_mirror` | `http://debian-archive.trafficmanager.net/debian/` | Debian package mirror. |
| `proxmox_ntp_servers` | `192.168.130.1`, `192.168.130.53` | Only allowed chrony upstream servers. |
| `proxmox_dns_servers` | `192.168.130.1`, `192.168.130.53` | Only allowed DNS upstream servers. |
| `proxmox_admin_users` | `[]` | Linux and PVE admin users to create. |
| `proxmox_ssh_key_default_url` | `""` | Default SSH authorized keys URL for users. |
| `proxmox_mgmt_networks` | `192.168.0.0/16` | Networks added to the Proxmox firewall `management` IPSet. |
| `proxmox_mgmt_iface` | `""` | Management NIC name. |
| `proxmox_mgmt_address_cidr` | `""` | Management interface address, including prefix length. |
| `proxmox_mgmt_gateway` | `""` | Management default gateway. |
| `proxmox_bridge_name` | `vmbr0` | Proxmox bridge to configure. |
| `proxmox_bridge_iface` | `""` | Physical NIC bound to the bridge. |
| `proxmox_bridge_address_cidr` | `""` | Optional bridge address, including prefix length. |
| `proxmox_allow_mgmt_iface_change` | `false` | Must be `true` before the management NIC is templated. |

Example user entry:

```yaml
proxmox_admin_users:
  - name: adminuser
    password_hash: "$y$j9T$replace-with-a-vaulted-hash"
    ssh_key_url: https://github.com/adminuser.keys
    ssh_key_checksum: sha256:replace-with-real-checksum
    groups:
      - sudo
    pve_role: PVEAdmin
    pve_path: /
```

Plain `password` values are supported, but committed inventories should use Ansible Vault or `password_hash` values.

## Debian Bootstrap Notes

On plain Debian 13, the playbook installs the Proxmox repository, Proxmox kernel, `proxmox-ve`, `postfix`, `open-iscsi`, `ifupdown2`, and `chrony` before the reconciliation roles run. The playbook does not reboot by default; after the first bootstrap run, reboot manually or set `proxmox_reboot_after_kernel_install` to `true` for hosts where an automated reboot is acceptable.

The `proxmox-ve` package ships `/etc/apt/sources.list.d/pve-enterprise.sources` (and the matching Ceph enterprise file). The playbook re-runs the `proxmox_repositories` role immediately after `proxmox_install` so those package-shipped enterprise sources are removed and the apt cache is refreshed before the `proxmox_updates` role runs. When `proxmox_perform_upgrade` is `true`, the update role uses Ansible's `ansible.builtin.apt` module with `update_cache: true`, which would otherwise fail against the unauthenticated enterprise repository.

In check mode on a plain Debian host, PVE-specific reconciliation roles are skipped because `proxmox-ve`, `/etc/pve`, `pveum`, and `ifreload` do not exist until a real bootstrap run installs them. Existing Proxmox VE hosts still run the full drift check.

## Network Safety

The network role owns `/etc/network/interfaces`. It refuses to run unless the management interface and bridge-bound interface are different. It also refuses to manage the management interface unless `proxmox_allow_mgmt_iface_change` is set to `true` for that host.

Only one default gateway should be configured. The role fails if both the management interface and bridge are given gateways.

## Validation

Run the same checks as CI:

```bash
yamllint .
ansible-playbook --syntax-check site.yml
ansible-lint
```

## Dev Container / Codespaces

A [Development Container](https://containers.dev) definition is provided in
`.devcontainer/devcontainer.json`. It works with GitHub Codespaces, the
VS Code Dev Containers extension, and any other devcontainer-compatible tool.

The container is based on a Python image and, on creation, installs the same
tooling used by CI (`ansible`, `ansible-lint`, `yamllint`) and the Ansible
collections from `requirements.yml`. The Red Hat Ansible and YAML extensions
are preinstalled for editor support.

To use it:

- **GitHub Codespaces:** open the repository in Codespaces; the container is
  built automatically.
- **VS Code locally:** install the *Dev Containers* extension and run
  *Dev Containers: Reopen in Container*.

Once the container is ready you can run the validation commands above
directly inside it.
