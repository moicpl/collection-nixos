# moicpl.nixos

The NixOS machines of lab1: installing them, their base system and their
podman containers. What runs inside the containers (logins, settings) is
[moicpl.app_config](https://github.com/moicpl/collection-app-config).

Run through [nixos_playbooks](https://github.com/moicpl/nixos_playbooks), which
holds the inventory link, `ansible.cfg` and the thin playbooks that import these.

## Playbooks

| Playbook | Hosts | What it does |
| --- | --- | --- |
| `moicpl.nixos.bootstrap_nix` | `nixos_bootstrap`, `nixos` | Installs a machine from the live installer and keeps the base system of every NixOS machine. |
| `moicpl.nixos.home_management` | `nixos_home` | UniFi, Home Assistant, Frigate and the portal. |
| `moicpl.nixos.arr_nix` | `nixos_arr` | Sonarr, Radarr, Lidarr, Prowlarr, Transmission (PIA), Jellyfin, Immich and the portal. |

`playbooks/templates/` holds the portal's Caddyfile and page.

## Roles

| Role | What it does |
| --- | --- |
| `moicpl.nixos.nixos` | Install, base system (`tasks/main.yml`), containers (`tasks_from: apps.yml`), login (`tasks_from: connect.yml`, also used by moicpl.app_config). See [its README](roles/nixos/README.md). |

## Requirements

- The `bitwarden` role (https://github.com/moicpl/bitwarden) in the roles path:
  every secret comes from Bitwarden through it. It isn't a collection, so it is
  not a galaxy dependency; nixos_playbooks clones and links it.
- The inventory (https://github.com/moicpl/lab1inventory) as the first
  inventory source: the playbooks record hosts and secret IDs in it.
