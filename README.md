# ansible-homelab

## Playbooks
Install and configure pi-hole
```sh
ansible-playbook -i inventories/inventory.yml playbooks/pi-hole/pi-hole.yml --limit pi-hole --ask-become-pass
```