Role Name
=========

Upgrades an existing K3s cluster (master + workers) to a new version, one
node at a time, optionally draining and uncordoning each node around the
upgrade so running workloads are rescheduled elsewhere first.

You control the exact target version via `k3s_upgrade_version` — the play
aborts immediately if it's left empty, so an upgrade never silently jumps
to whatever "latest"/"stable" happens to resolve to. The install flags
(`k3s_channel`, `k3s_master.flags`, `k3s_worker.flags`) are still loaded
from `roles/k3s_install/vars/main.yml` at runtime, so the flags used to
reinstall the k3s binary stay in sync with a fresh install.

Requirements
------------

- A cluster already installed by the `k3s_install` role.
- A local kubeconfig at `~/.kube/config` pointing at the cluster (this is
  fetched automatically by `k3s_install` during the initial install).
- The `kubernetes.core` collection (already required by `k3s_install` /
  `k3s_deploy_apps`).

Role Variables
--------------

Defined in `defaults/main.yml`:

- `k3s_upgrade_version` (**required**): the exact version to upgrade to,
  e.g. `v1.31.5+k3s1`. The play aborts immediately if this is left empty.
- `k3s_upgrade_drain_nodes` (default `true`): cordon+drain a node before
  upgrading it. Set to `false` to skip draining entirely and accept
  downtime on that node's pods during the upgrade instead — useful for a
  homelab where the remaining nodes can't absorb the drained workload
  anyway. The node is always uncordoned after the upgrade regardless of
  this setting, so a node never gets left stuck cordoned.
- `k3s_drain_timeout` (default `120`, seconds): how long to wait for pod
  eviction when draining a node. Only relevant when
  `k3s_upgrade_drain_nodes` is `true`.
- `k3s_upgrade_wait_retries` / `k3s_upgrade_wait_delay`: polling
  retries/delay used while waiting for a node to report `Ready` again after
  being upgraded.

Read from `roles/k3s_install/vars/main.yml`:

- `k3s_channel`, `k3s_master`, `k3s_worker` (install/exec flags only — the
  version itself is *not* taken from this file).

Usage
-----

Run the upgrade playbook, passing the target version as an extra var:

```bash
ansible-playbook playbooks/pi-k3s/4_k3s-upgrade.yml \
  -e k3s_upgrade_version=v1.31.5+k3s1 \
  --ask-become-pass
```

To skip draining (accept downtime instead), add
`-e k3s_upgrade_drain_nodes=false`.

The playbook upgrades the master first, then upgrades workers one at a
time (`serial: 1`) so the cluster keeps serving from the remaining workers
during the rollout.

Once you've confirmed the cluster is healthy on the new version, remember
to also bump `k3s_version` in `roles/k3s_install/vars/main.yml` so future
fresh installs (or node replacements) provision the same version.

Example Playbook
----------------

```yaml
- hosts: master
  become: true
  roles:
    - k3s_upgrade

- hosts: workers
  become: true
  serial: 1
  roles:
    - k3s_upgrade
```

License
-------

BSD

Author Information
------------------

Franjly
