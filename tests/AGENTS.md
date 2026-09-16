# Docker Tests

Agent guidance for the standalone Docker test playbooks in `tests/`.

These are the non-Molecule test path: they start real distro containers, install the role into each,
and are what the `Test` CI workflow runs. For the Molecule path, see
[../molecule/AGENTS.md](../molecule/AGENTS.md).

## Layout

| Path | Purpose |
| --- | --- |
| `inventory/docker-containers.yml` | Six-host distro matrix (alpine, debian, nixos, ubuntu jammy/noble/latest) |
| `playbooks/docker-containers.yml` | Starts distro containers, then installs the role on each |
| `test.yml` | Minimal localhost playbook that applies the role from the working tree |
| `upcloud.yml` | Playbook that applies the role to the hosts listed in `upcloud` |
| `upcloud` | Plain-text inventory holding the UpCloud VM address |

`docker-containers.yml` uses `ansible_connection: docker`, so the containers are addressed by name
over the Docker socket rather than SSH.

## Running Tests

Run everything through `pipenv`, matching [../molecule/AGENTS.md](../molecule/AGENTS.md):

```bash
# Install/refresh dependencies from Pipfile.lock (first run)
pipenv sync

# Distro matrix (pulls images, ~2 min)
pipenv run ansible-playbook -i tests/inventory/docker-containers.yml tests/playbooks/docker-containers.yml
```

Syntax check only (what CI runs first):

```bash
pipenv run ansible-playbook --syntax-check \
  -i tests/inventory/docker-containers.yml tests/playbooks/docker-containers.yml
```

`pipenv` auto-detects the project's `.venv` directory (see `is_venv_in_project` in pipenv's
`project.py`), so `pipenv run <cmd>` and `.venv/bin/<cmd>` resolve to the same interpreter. Prefer
`pipenv run` - it does not depend on the caller's `PATH` or on the venv being activated.

### Why pipenv

`Pipfile.lock` pins the exact versions the tests were validated against (`ansible-core 2.17.9`,
`ansible-compat 25.1.4`, `molecule 25.3.1`, `molecule-docker 2.1.0`), so `pipenv sync` reproduces the
environment deterministically. Installing `ansible`/`ansible-lint` ad hoc instead pulls the latest
`ansible-core` (2.21.x), which is a different runtime than CI uses.

`pipenv` does **not** provide `ansible-lint` - it is not in the `Pipfile`. Run lint via
`pre-commit run ansible-lint -a`, or install it separately.

## Prerequisites

- Docker daemon reachable and a working default bridge (see the troubleshooting entry below).
- Ansible collections installed: `ansible-galaxy collection install -r requirements.yml`
  (`community.docker`, `community.general >= 6.0.0`, `community.windows`, `ansible.windows`).
- The role resolvable as `ea31337.ea_vpn`. The playbooks use `ansible.builtin.import_role`, which
  resolves from `~/.ansible/roles/` - not from the working tree. Symlink it for development:

    ```bash
    ln -vs "$PWD" ~/.ansible/roles/ea31337.ea_vpn
    ```

- The role dependency `ea31337.metatrader` installed as well, since `meta/main.yml` pulls it in.

## What the Playbooks Do

`docker-containers.yml` runs two plays:

1. **Configure Docker container** - starts each container with `sleep infinity`, waits for it to be
   running, bootstraps Python 3 with inline `raw` tasks, gathers facts, and installs the extra
   Ubuntu/Debian Python packages. The NixOS host uses the `nixos/nix` image directly - there is no
   custom image build.
2. **Install ea31337.ea_vpn role** - applies the role to every container, then stops the containers.

Unlike the xvfb playbook, this one does not recreate containers, so a re-run reuses the containers
that are already there. It does stop them at the end of every run, which is why the two-run check
below cannot report `changed=0` (see Idempotency).

`test.yml` applies the role to `localhost` from the working tree, and `upcloud.yml` applies it to the
hosts in the `upcloud` inventory (a real VM, not a container).

## Verifying the Result

The playbooks only *install* the role - they never assert that the swapfile came up, so a green run
does not by itself prove the role works. Check the container directly:

```bash
docker exec ea-vpn-on-ubuntu-latest swapon --show
# NAME                TYPE SIZE USED PRIO
# /var/cache/swapfile file   2G   0B   -2

docker exec ea-vpn-on-ubuntu-latest grep swapfile /etc/fstab
# /var/cache/swapfile none swap sw 0 0
```

The swapfile path comes from `ea_vpn_swapfile_path` (default `/var/cache/swapfile`), and activation
is gated on `ea_vpn_swapfile_activate`. Activating swap needs privileges a plain container may not
have, so the `swapon` step can be skipped or fail to take effect on some hosts.

### Idempotency

`AGENTS.md` requires idempotent tasks, but running `docker-containers.yml` twice is **not** a valid
idempotency check, and the second run can never report `changed=0`. Two things guarantee changes:

- **`Stop Docker containers`** (post-task) unconditionally stops the containers, so it reports
  `changed` on every run.
- **A container restart resets running services.** The next run's pre-tasks start the containers
  again, so anything started as a process is no longer running. This role depends on
  `ea31337.metatrader`, which pulls in `ea31337.xvfb`; three tasks in its `tasks/supervisord.yml` are
  gated on `supervisord_status.rc != 0` and so re-fire: `Remove stale supervisor socket if not
  running`, `Remove stale supervisor pid if not running`, and `Start supervisord daemon`.

To test idempotency, keep the containers up between runs and apply the role twice. The stop post-task
inherits the play's `tags: always`, so `--skip-tags` cannot drop it without dropping the whole play;
use a probe playbook that omits it and confirm the second run reports `changed=0`:

```yaml
---
- name: Idempotency check
  hosts: docker_containers
  gather_facts: true
  vars:
    controller_python: '{{ ansible_playbook_python }}'
  tasks:
    - name: Installs ea31337.ea_vpn role
      ansible.builtin.import_role:
        name: ea31337.ea_vpn
```

```bash
# The playbook leaves the containers stopped, so start them first.
docker start ea-vpn-on-ubuntu-latest

# Run 1 may change the xvfb start-up tasks; run 2 must report changed=0.
pipenv run ansible-playbook -i tests/inventory/docker-containers.yml /tmp/idempotency.yml
pipenv run ansible-playbook -i tests/inventory/docker-containers.yml /tmp/idempotency.yml
```

## Troubleshooting Matrix

### `ansible-lint` cannot resolve `community.docker.*`

> `syntax-check[unknown-module]: couldn't resolve module/action 'community.docker.docker_image'`

- **Root cause**: a stale, empty `.ansible/collections/ansible_collections/community/docker`
  directory shadows the real collection. This is the blocker documented in the root `AGENTS.md`.
- **Check**: `ls -la .ansible/collections/ansible_collections/community/docker/` - if it contains
  only an empty `roles/` directory, this is the cause.
- **Fix**: `rm -rf .ansible/collections/ansible_collections/community/docker`. `.ansible` is
  gitignored and holds no tracked files, so this is safe.

### Docker bridge has no gateway (containers cannot reach the network)

> `apk update` / `apt-get update` fails, or `getent hosts` returns nothing, while the host resolves
> and routes fine.

- **Root cause**: `docker0`'s address does not match the `bridge` network's configured gateway, so
  containers get a default route pointing at an address that is not on the bridge.
- **Check**: `ip -4 addr show docker0` vs
  `docker network inspect bridge --format '{{range .IPAM.Config}}{{.Subnet}} {{.Gateway}}{{end}}'`.
  If the gateway from the second command is missing from the first, the bridge is broken.
- **Fix**: `sudo systemctl restart docker` recreates `docker0` with the configured gateway. This
  stops running containers, including an in-flight test run.
- **Note**: `docker0` may carry more than one address. As long as the gateway reported by
  `docker network inspect bridge` is present on `docker0`, the bridge works even if the subnet
  differs from `bip` in `/etc/docker/daemon.json`.

### `community.general does not support Ansible version 2.17.9`

> `[WARNING]: Collection community.general does not support Ansible version 2.17.9`

- **Root cause**: the installed `community.general` expects a newer `ansible-core` than the one
  pinned in `Pipfile.lock` (2.17.9).
- **Impact**: warning only; the tests pass. Do not "fix" it by upgrading `ansible-core` ad hoc - that
  diverges from the pinned environment CI uses.

### `molecule` fails with `FileNotFoundError: 'ansible-config'`

> Running a venv binary directly (`.venv/bin/molecule`) raises
> `FileNotFoundError: [Errno 2] No such file or directory: 'ansible-config'`.

- **Root cause**: `ansible_compat` shells out to `ansible-config` by name. Invoking the binary
  directly does not put the venv's `bin` on `PATH`, so the lookup fails.
- **Fix**: use `pipenv run molecule ...` - pipenv prepends the venv's `bin` to `PATH`. Activating
  the venv (`source .venv/bin/activate`) works too. This is the main reason to prefer `pipenv run`
  over calling `.venv/bin/*` directly.

### `pipenv` warns that Python 3.10 was not found

> `Warning: Python 3.10 was not found on your system...`

- **Root cause**: `Pipfile` pins `python_version = "3.10"`, but the project `.venv` was created with
  a different Python version.
- **Impact**: harmless when `.venv` already exists, because pipenv reuses it. Only relevant when
  creating a fresh environment.
- **Fix**: install Python 3.10 (`pyenv`/`asdf`), or create the venv explicitly with
  `python3 -m venv .venv` and use `pipenv run` / `.venv/bin` directly.
