# nixos

Installs NixOS from the live installer, then manages the host declaratively in
two layers:

- **Base** (`tasks/main.yml`, the role's default entry point): admin user with
  an SSH key and passwordless sudo, console-only root password, key-only SSH
  without root login, network (DHCP or a fixed address), firewall, mDNS, base
  packages. Written to `/etc/nixos/configuration.nix` and `/etc/nixos/network.nix`.
- **Apps** (`tasks/apps.yml`, run with `tasks_from: apps.yml`): podman
  containers through `virtualisation.oci-containers`, their firewall ports and
  host directories and config files, written to `/etc/nixos/apps.nix`, and their root-only
  secret files in `/etc/nixos/secrets/`.

`configuration.nix` imports `apps.nix` when it exists, so either layer rebuilds
the whole system. The role owns these files (`hardware-configuration.nix` comes
from `nixos-generate-config`) and runs `nixos-rebuild switch` when one changes.

## How it runs

`tasks/bootstrap.yml` (run with `tasks_from`, against the placeholder name a
machine has on the live installer, e.g. `nixos.maison`): with
`nixos_install_wipe: true`, asks for the root password set on the installer's
console (unless `nixos_bootstrap_root_password` is passed) and checks it, asks
for the machine's FQDN, adds it to the inventory (`nixos_inventory_hosts_file`,
`host_vars/<fqdn>.yml`) and to `nixos_inventory_group` at the address it
answered on, with the password for this run only. The next play runs the role
on that name, one host at a time (`serial: 1`, for the prompts):

1. Logs in as `nixos_admin_user` with the SSH key, or as root with
   `nixos_bootstrap_root_password` on the live installer. NixOS has no Python by
   default, so it uses the system `python3` or fetches one with `nix-build`.
2. When the host has no `nixos_ip_address`, asks for its network (`tasks/network.yml`):
   a fixed address with its gateway and DNS servers, or DHCP. Records it in the
   host's host_vars, with `ansible_host` for a fixed address.
3. If `/` is a tmpfs (live installer): erases `nixos_install_disk`, creates a
   UEFI boot partition and an ext4 root, writes the configuration, runs
   `nixos-install`, reboots and logs back in as the admin user. This needs
   `nixos_install_wipe: true` and doesn't run in check mode.
4. Writes the base configuration and applies it. When the host's address
   changes, it applies it with a reboot and reconnects at the new address (by
   name for DHCP).

`tasks/apps.yml` logs in with the SSH key only, checks that the host has the
base configuration, then writes and applies the apps.

## Inputs

| Variable | Meaning |
| --- | --- |
| `nixos_ssh_public_key` | Admin user's authorized key (base, secret input) |
| `nixos_ssh_private_key` | Matching private key, written to `nixos_ssh_private_key_file` on the controller (secret input) |
| `nixos_bootstrap_root_password` | Root password of the live installer; asked by `tasks/bootstrap.yml` when empty, never stored |
| `nixos_root_password` | Root password of the installed system, console only (base, secret input) |
| `nixos_install_disk` | Disk erased by the install, a `/dev/disk/by-id/` path. Empty (default): the only internal disk (not USB, removable or mounted); the run stops and lists the disks when there are several |
| `nixos_install_wipe` | Must be `true` for the install to run |
| `nixos_ip_address` | `dhcp`, or a fixed address with its prefix; with `nixos_ip_interface`, `nixos_ip_gateway`, `nixos_ip_nameservers` (host_vars, asked when empty) |
| `nixos_packages` | Extra system packages, by nixpkgs attribute name (`python3` is always installed) (base) |
| `nixos_secret_files` | `name: content` files written to `/etc/nixos/secrets/<name>`, mode 0600, e.g. container `environmentFiles` (apps, secret input) |
| `nixos_containers` | `virtualisation.oci-containers.containers`, as YAML (apps) |
| `nixos_container_dirs` | Host directories created for the containers: a path, or `{path, owner, group, mode}` (apps) |
| `nixos_container_files` | `name: content` config files (not secrets) written to `/etc/apps/<name>`; a container mounts each file it uses in its `volumes` and is restarted when one changes (apps) |
| `nixos_container_networks` | Podman networks created for the containers that list them in `networks`; they reach each other by name (apps) |
| `nixos_firewall_tcp_ports`, `nixos_firewall_udp_ports` | Ports opened for the apps |
| `nixos_services` | Services listed at the end of the run (`name`, `scheme`, `port`, optional `path`), after SSH |

See `defaults/main.yml` for the others.

`nixos_containers` is converted to Nix as is: strings, numbers, booleans,
lists and mappings, so any `oci-containers` option works. Never put secrets in
it, because `/etc/nixos` and the Nix store are world-readable. Use
`environmentFiles` that point to `nixos_secret_files`.
