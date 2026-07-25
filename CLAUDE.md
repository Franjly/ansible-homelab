# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Ansible repository for managing a Raspberry Pi homelab: a K3s Kubernetes cluster (1 master + 3 workers) plus a Pi-hole DNS server. Secrets are managed with Ansible Vault; the vault password is read from `~/.vault_password`.

## Running Playbooks

All commands assume the repo root as the working directory. The default inventory is set in `ansible.cfg`, but some playbooks require explicit `-i` flags.

**Pi-hole:**
```bash
ansible-playbook playbooks/pi-hole/pi-hole.yml --limit pi-hole --ask-become-pass
```

**K3s cluster (ordered sequence):**
```bash
# Step 0: Change default SSH user (uses a separate inventory)
ansible-playbook -i inventories/inventory_pi_k3s_change_default_user.yml playbooks/pi-k3s/0_change-default-user.yml --limit master,workers --ask-become-pass

# Steps 1–3: Base config, K3s install, app deploy
ansible-playbook playbooks/pi-k3s/1_base-config.yml --limit master,workers --ask-become-pass
ansible-playbook playbooks/pi-k3s/2_k3s-install.yml --limit master,workers --ask-become-pass
ansible-playbook playbooks/pi-k3s/3_k3s-deploy-apps.yml --limit master,workers --ask-become-pass

# Upgrade to an explicit version (master first, then workers one at a time)
ansible-playbook playbooks/pi-k3s/4_k3s-upgrade.yml -e k3s_upgrade_version=v1.31.5+k3s1 --ask-become-pass

# Uninstall
ansible-playbook playbooks/pi-k3s/99_k3s-uninstall.yml --limit master,workers --ask-become-pass
```

**Run a subset of tasks via tags:**
```bash
ansible-playbook playbooks/pi-k3s/1_base-config.yml --limit master,workers --tags <tag> --ask-become-pass
```

**Vault operations:**
```bash
ansible-vault encrypt_string 'secret' --name 'variable_name'
ansible-vault edit inventories/inventory.yml
```

## Architecture

### Inventory
All hosts and variables live in `inventories/inventory.yml`. There are no separate `group_vars/` or `host_vars/` directories — variables are embedded directly in the inventory, with sensitive values vault-encrypted.

Host groups: `master` (192.168.30.10), `workers` (192.168.30.20–.22), `pi-hole` (192.168.40.254), `pi-dev` (192.168.40.123).

A second inventory `inventories/inventory_pi_k3s_change_default_user.yml` is used only for the initial user-change step (it connects as the factory default user before the `pi` user is configured).

### Playbooks
`playbooks/pi-k3s/` contains numbered playbooks (0–4, 99) meant to run in order. `pi-k3s.yml` is an all-in-one master playbook. `playbooks/pi-hole/` is standalone.

### Roles
Roles are in `roles/` and cover:

| Role | Purpose |
|------|---------|
| `base` | Common packages, aliases, locale (en_US.UTF-8), timezone (Europe/Madrid), /etc/hosts |
| `rpi` | Raspberry Pi specifics: disable WiFi/BT, GPU memory, cgroup flags for containers, full apt upgrade |
| `os` | Hostname, swap disable (handles both dphys-swapfile and zram), passwordless sudoers |
| `ssh` | ED25519 key generation, hardened sshd config (no passwords, no root login) |
| `network` | Static IP and VLAN config via NetworkManager/nmcli |
| `change_default_user` | Creates the `pi-k3s` user/group, configures autologin |
| `k3s_install` | K3s v1.30.3+k3s1 — master with Calico CNI, no traefik/servicelb/local-storage; workers with labels/taints |
| `k3s_uninstall` | Removes K3s from master and workers |
| `k3s_upgrade` | Upgrades K3s to an explicit version (`k3s_upgrade_version`, required, no default) one node at a time (master, then workers), draining/uncordoning around each upgrade |
| `k3s_deploy_apps` | Deploys Calico, ArgoCD, MetalLB, Ingress, Longhorn, Vault, NFS provisioner via kubectl |
| `pi_hole` | Pi-hole installation, custom dnsmasq DNS, ad/malware block lists, DoH (Cloudflare 1.1.1.1) |
| `certificate` | Self-signed TLS cert generation using `community.crypto`, delegated to localhost |
| `storage` | Checks Longhorn and NFS availability; triggers reboots when needed |

### Key Configuration Details
- **K3s cluster CIDR:** pods `10.42.0.0/16`, services `10.43.0.0/16`
- **CNI:** Calico (flannel disabled)
- **DNS wildcard:** `*.local.tecno-fly.com → 192.168.40.200` (configured in Pi-hole)
- **Longhorn:** base URL `https://longhorn.local.tecno-fly.com`
- **ArgoCD repos:** `git@github.com:Franjly/homelab-pi-k3s.git` and its argocd variant
- **Python interpreter:** `/bin/python3.10` (set in `.vscode/settings.json`)
