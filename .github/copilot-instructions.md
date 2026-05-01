# Copilot Instructions

This repository is an Ansible project for Proxmox VE 9.1 hosts.

- Prefer focused roles over one large role. Keep role boundaries aligned with repositories, Proxmox installation, updates, chrony, DNS, users, firewall, and networking.
- Use fully qualified Ansible collection names for modules.
- Keep tasks idempotent. Any `ansible.builtin.command` task must define `changed_when`, and should be guarded by a read/check task when it mutates state.
- Prefer templates for owned multi-line configuration files. Use line-oriented edits only for targeted changes to vendor-owned files.
- Treat network changes as high risk. Keep management-interface changes explicit, guarded, and applied after lower-risk roles.
- Preserve support for both plain Debian 13 bootstrap and existing Proxmox VE drift reconciliation.
- Do not store plaintext secrets in committed files. Use Ansible Vault for user passwords and checksums for remote SSH key files when possible.
- Validate changes with `yamllint .`, `ansible-playbook --syntax-check site.yml`, and `ansible-lint`.
