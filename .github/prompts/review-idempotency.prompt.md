---
description: "Review Proxmox Ansible changes for idempotency and operational risk"
agent: ask
---

Review the selected Ansible changes as a senior Proxmox automation reviewer.

Focus on:
- Whether each task is idempotent across repeated playbook runs.
- Whether repository, chrony, DNS, firewall, and network changes meet the host baseline requirements.
- Whether management-interface changes can interrupt SSH or leave the host without a default route.
- Whether commands use `changed_when`, `failed_when`, and pre-checks correctly.
- Whether secrets, SSH key URLs, and password hashes are handled safely.

Report findings first, with file references and concrete fixes.
