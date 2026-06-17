# Homelab Ansible Playbook

This repository sets up two different target hosts:
- a **network** host (Pi-hole / DNS-related services)
- a **media** host (Plex / media stack)

## Overview

The main playbook runs in this order:
1. Prompt for the deploy user and the Pi-hole admin password
2. Bootstrap the target machines with the `base` role
3. Configure the network host
4. Configure the media host

## How the playbook works

The main entrypoint is [playbooks/setup.yml](playbooks/setup.yml).

### Prompted variables
When you run the playbook, you will be asked for:
- `deploy_user`: the username that should be used for SSH and file ownership
- `pihole_password`: the admin password used by Pi-hole

These values are stored for later plays so the templates can use them.

### Roles used
- `base`: installs system prerequisites and bootstrap tools
- `users`: configures the deploy user
- `docker_apps`: installs Docker and Docker Compose support
- `network`: deploys the network stack (including Pi-hole)
- `media`: deploys the media stack (including Plex)

## Inventory

The inventory file defines the hosts that should be managed:
- `media`
- `network`

Each group contains the hostname/IP for the matching machine.

## Templates

The compose files are managed with templates so values like the Pi-hole password can be filled in at runtime.

Relevant template files:
- [playbooks/roles/network/templates/docker-compose.yml.j2](playbooks/roles/network/templates/docker-compose.yml.j2)
- [playbooks/roles/media/templates/docker-compose.yml.j2](playbooks/roles/media/templates/docker-compose.yml.j2)

## Example run

If your hosts require SSH password authentication and a privilege escalation password, run the playbook with explicit prompt flags:

```bash
ansible-playbook -i inventory/hosts.yml \
  --ask-pass \
  --ask-become-pass \
  playbooks/setup.yml
```

The shorthand `-kK` is equivalent.

You will also be prompted for the deploy user and Pi-hole password before the playbook continues.

## Notes

- The playbook assumes the target machines are reachable over SSH.
- The pihole template needs updated with your timezone.
- If your setup uses SSH keys and passwordless sudo/su, you may be able to omit `--ask-pass` and `--ask-become-pass`.
- The bootstrap stage is designed to install the base system tools needed before the rest of the setup runs.
- If you need to customize the Plex volume mount, you can override the corresponding variable when running the playbook.
- If you can't log into pihole run this command: docker exec -it pihole pihole setpassword
