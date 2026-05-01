---
description: "Add an idempotent task to one of the Proxmox Ansible roles"
agent: agent
---

Add or modify a task in this Proxmox VE 9.1 Ansible project.

Requirements:
- Keep the change inside the most specific existing role.
- Use fully qualified module names.
- Preserve idempotency and avoid unguarded `command` or `shell` tasks.
- Add handlers only when a service or generated configuration actually needs them.
- Update README variables only when the user-facing configuration surface changes.
- Run `yamllint .`, `ansible-playbook --syntax-check site.yml`, and `ansible-lint` when tooling is available.
