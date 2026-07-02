# Homelab Ansible Playbook

Automates the setup of a small homelab from a fresh Debian install. It targets two kinds of hosts:

- a **network** host (Pi-hole + Unbound for DNS/ad-blocking)
- a **media** host (Plex)

Run from a control machine against one or more target hosts that have nothing more than a root account and a single normal user already on them.

## Prerequisites

**On the control node (the machine running `ansible-playbook`):**

- Ansible installed.
- The `community.docker` collection, used to manage the Docker Compose stacks:

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

**On every target host:**

- A fresh Debian install with only a root account and one normal user.  `sudo` does not need to be installed ahead of time — the playbook installs it and adds your user to the `sudo` group as part of bootstrapping.
- SSH access for that normal user (password or key-based).

**On the media host specifically:**

- An external drive (or other storage) already mounted at `/mnt/storage`. The playbook will create `/mnt/storage/tv` and `/mnt/storage/movies` underneath it, but it checks that `/mnt/storage` is an actual mount point     first and will fail rather than write to the root filesystem if it isn't mounted. Mount your drive before running the playbook.

## Before you run it

Work through this checklist first:

1. **Edit `inventory/hosts.yml`** and replace `<media-host-ip>` and `<network-host-ip>` with the real IP address or hostname of each target machine.
2. **Mount your storage drive at `/mnt/storage` on the media host**, if it isn't already.
3. **Check the timezone in the Pi-hole template** — [`playbooks/roles/network/templates/docker-compose.yml.j2`](playbooks/roles/network/templates/docker-compose.yml.j2) defaults `TZ` to `UTC`. Update it to your local timezone if you want Pi-hole's logs and schedules to reflect local time.
4. **Decide on a Pi-hole admin password** — you'll be prompted for it when you run the playbook (see below).

## Running it

From the repo root:

```bash
ansible-playbook -i inventory/hosts.yml -kK playbooks/setup.yml
```

`-kK` is shorthand for `--ask-pass --ask-become-pass`, and is needed if your target hosts use SSH password authentication and don't yet have passwordless sudo configured. If you're using SSH keys and your user can already `sudo`/`su` without a password, you can drop one or both flags.

You will also be prompted for two more things before any hosts are touched:

- **`deploy_user`** — the SSH username the playbook should use to connect to every target host, and the user that will own deployed files and be added to the `sudo`/`docker` groups.
- **`pihole_password`** — the admin password that will be set for the Pi-hole web interface.

## What it does, in order

1. **Prompt** (`localhost`) — collects `deploy_user` and `pihole_password`.
2. **Bootstrap** (`hosts: all`) — using `su` (since `sudo` isn't installed yet), updates apt, installs base packages, installs `sudo`, and adds `deploy_user` to the `sudo` group.
3. **Docker** (`hosts: all`) — using `sudo`, installs Docker and the Docker Compose plugin on every target host, and adds `deploy_user` to the `docker` group.
4. **Network host setup** (`hosts: network`) — deploys the Pi-hole + Unbound stack to `/opt/pihole` and starts it.
5. **Media host setup** (`hosts: media`) — verifies `/mnt/storage` is mounted, creates the Plex config/storage directories, deploys the Plex stack to `/opt/plex`, and starts it.

## Roles

- `base` — updates apt and installs common packages (curl, git, vim, net-tools, python3-docker).
- `users` — ensures `sudo` is installed and adds `deploy_user` to the `sudo` group.
- `docker_apps` — installs Docker via the official `get.docker.com` script and the `docker-compose-plugin` apt package; adds `deploy_user` to the `docker` group.
- `network` — deploys Pi-hole (with Unbound as its upstream resolver) using `community.docker.docker_compose_v2`.
- `media` — deploys Plex using `community.docker.docker_compose_v2`.

## Storage layout

- App configuration lives under `/opt/<app>/` (e.g. `/opt/pihole`, `/opt/plex`) on each target host.
- Plex's media library lives under `/mnt/storage/` on the media host:
  - `/mnt/storage/tv`
  - `/mnt/storage/movies`

  If you need additional media folders (music, photos, etc.), add them to both the `volumes:` section in [`playbooks/roles/media/templates/docker-compose.yml.j2`](playbooks/roles/media/templates/docker-compose.yml.j2) and the directory list in [`playbooks/roles/media/tasks/main.yml`](playbooks/roles/media/tasks/main.yml).

## After it runs

- **Pi-hole**: log in at `http://<network-host-ip>/admin` using the password you entered at the prompt. If you ever need to reset it manually:

  ```bash
  docker exec -it pihole pihole setpassword
  ```

- **Plex**: visit `http://<media-host-ip>:32400/web` to finish setup and point your libraries at `/tv` and `/movies` inside the container (these map to `/mnt/storage/tv` and `/mnt/storage/movies` on the host).

## Notes

- `PUID`/`PGID` for the Plex container are set to `1000` in [`playbooks/roles/media/templates/docker-compose.yml.j2`](playbooks/roles/media/templates/docker-compose.yml.j2), which assumes `deploy_user` is the first normal user created on a fresh Debian install (the typical default UID/GID). If that's not the case on your target host, update this file with the correct values.
- Re-running the playbook is safe — package installs, directory creation, and the Docker Compose deploys are all idempotent. You'll be re-prompted for `deploy_user` and `pihole_password` each run.
