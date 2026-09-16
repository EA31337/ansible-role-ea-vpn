# AGENTS.md

Persistent context for autonomous agents working on this Ansible role.

For project overview and install instructions, see [README.md](README.md).

## Setup & Environment Invariants

- Ansible role: `ea31337.ea_vpn`
- Supported OS: Alpine Linux, Debian/Ubuntu (NixOS is commented out in `meta/main.yml`)
- Driver: Docker (Molecule)
- Python required on all targets
- Collections: `community.docker`, `community.general` (>= 6.0.0), `community.windows`,
  `ansible.windows`
- Role dependency: `ea31337.metatrader`
- Vars: `defaults/main.yml` (user-facing)

## Key Files & Context Injection

| Path | Purpose |
| ---- | ------- |
| `.devcontainer/devcontainer.json` | Dev container definition (base image, Features, `onCreateCommand`) |
| `.devcontainer/provision.yml` | Ansible playbook run by `onCreateCommand` to provision the container |
| `.devcontainer/requirements.txt` | Python dependencies installed in the dev container |
| `defaults/main.yml` | Default role variables (`ea_vpn_swapfile_*`) |
| `tasks/main.yml` | Role entry point; includes the swapfile tasks |
| `tasks/swapfile.yml` | Swapfile creation, formatting, activation, and `/etc/fstab` entry |
| `molecule/default/molecule.yml` | Default Molecule scenario config |
| `molecule/default/converge.yml` | Converge playbook |
| `molecule/default/verify.yml` | Verification playbook |
| `requirements.yml` | Galaxy collection + role dependencies |
| `meta/main.yml` | Galaxy metadata + role dependencies |
| `.github/workflows/molecule.yml` | CI: Molecule test matrix |
| `.github/workflows/check.yml` | CI: pre-commit / linting |
| `.github/workflows/devcontainer-ci.yml` | CI: dev container build & test |
| `.github/workflows/test.yml` | CI: Ansible syntax/lint and Docker container tests |
| `.github/prompts/code-review.prompt.md` | Code review prompt |

## Agent Directives

- MUST run `yamllint .` and `ansible-lint` before committing YAML changes.
- MUST use FQCN for all modules (e.g. `ansible.builtin.command`, not `command`).
- MUST enforce max line length of 120 characters (`.yamllint`, `.markdownlint.yaml`).
- MUST ensure idempotency in all Ansible tasks.
- MUST update `defaults/main.yml` and `README.md` when changing role variables.
- NEVER hardcode secrets or environment-specific values.
- NEVER remove or modify tests to mask failures; fix root cause instead.
- NEVER use `git add .` without verifying staged files.
- MUST reference GitHub Actions by simple major version tags (e.g. `actions/checkout@v6`),
  not pinned patch versions (e.g. `@v6.1.0`), so minor/patch updates apply automatically.

## Docker Tests

The standalone Docker test playbooks in `tests/`, how to run them via `pipenv`, and
their troubleshooting matrix live in [tests/AGENTS.md](tests/AGENTS.md).

## Molecule Testing

Molecule scenarios, the platform matrix, how to run the tests, and Molecule-specific
troubleshooting live in [molecule/AGENTS.md](molecule/AGENTS.md).

## Testing & Verification Gates

### Dev Container Build & Test

The dev container is defined in `.devcontainer/` and is the primary development environment.
`devcontainer.json` uses the `mcr.microsoft.com/devcontainers/base:jammy` image plus devcontainer
Features; its `onCreateCommand` installs Ansible and runs `provision.yml`.

```bash
# Build the image only (base image + Features)
devcontainer build --workspace-folder .

# Build, start the container, and run onCreateCommand (provision.yml)
devcontainer up --workspace-folder .

# Run a command inside the running container
devcontainer exec --workspace-folder . bash -lc 'ansible --version'
```

- `devcontainer build` prints `{"outcome":"success",...}` on success.
- `devcontainer up` additionally returns a `containerId` and a clean Ansible recap (`failed=0`);
  it installs the apt packages, pipx Ansible, collections, and the pre-commit hook from `provision.yml`.

Requirements:

- Docker daemon reachable and the `devcontainer` CLI (v0.89+) installed.
- A working default Docker bridge (see the troubleshooting entry below).
- Outbound access to `ghcr.io`, `.github.com`, `*.githubusercontent.com`, and the apt / PyPI /
  Galaxy hosts. Host firewalls that prompt per connection (e.g. Portmaster) block the `nanolayer`
  downloads long enough to time out - pre-allow those domains.

## Common Tasks

### Before Each Commit

- Verify changes with `git diff --no-color`.
- Ensure no temporary or unrelated files are staged.
- Run `yamllint .` and `ansible-lint` for any YAML changes.
- Run `pipenv run molecule syntax` to catch playbook errors early.

### Updating Pre-commit Hooks

Run `pre-commit autoupdate`, then `pre-commit run -a`. Revert any hook that breaks and file an issue for it.

Known blockers (as of the 2026-09 update):

- `ansible-lint` v26.8.0 declares `language_version: python3.14`. Without a Python 3.14
  interpreter, either keep the ref pinned or override the hook with `language_version: python3`.
- `pre-commit-hooks` v6.0.0 removed `check-byte-order-marker`; replace it with
  `fix-byte-order-marker`.
- `markdownlint-cli` v0.49.1 needs node >= 22.20 (its dev dependency `ava@8`). If the hook pins
  `language_version: 22.14.0`, the env fails to install; pin markdownlint-cli or bump the pinned node.
- `ansible-lint` + `community.docker`: a stale, empty
  `.ansible/collections/ansible_collections/community/docker` directory shadows the real collection
  and causes `couldn't resolve module/action 'community.docker.docker_container'`. Remove it.
- `additional_dependencies` with a version range must use the block form
  (`- ansible-core>=2.16,<2.21`); the inline flow form splits on the comma into separate
  requirements, and the no-space form trips ansible-lint's `yaml[commas]` rule.

`pre-commit run -a` can also surface pre-existing failures (e.g. `yamlfix`/`black` reformatting,
`flake8` violations) unrelated to the ref bump; CI lints only changed files, so file these separately.

### Editing Files

- Max line length: 120 characters (enforced by `.yamllint` and `.markdownlint.yaml`).
- YAML indentation: 2 spaces.
- End all files with a newline.
- Keep lists and keys in lexicographical order when possible.

### Renaming/Removing Files

- Use `git mv` / `git rm` to preserve history.

## Troubleshooting

### Dev container Feature install fails

> `ERROR: Feature "..." failed to install!` with `curl: (6) Could not resolve host: github.com`
> or `urllib.error.URLError: <urlopen error [Errno 113] No route to host>`

- **Root cause (no network)**: `docker0`'s address does not match the `bridge` network's configured
  gateway, so containers on the default bridge have no working gateway and BuildKit `RUN` steps
  cannot reach the network.
  - **Check**: `ip -4 addr show docker0` vs
    `docker network inspect bridge --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'`.
  - **Fix**: `sudo systemctl restart docker` recreates `docker0` with the configured gateway.
    Non-disruptive workaround (not persistent): `sudo ip addr add <gateway>/16 dev docker0`.
- **Root cause (blocked downloads)**: a host firewall that prompts per connection (e.g. Portmaster)
  blocks `nanolayer`'s `ghcr.io` / `api.github.com` requests while the prompt is pending, and
  `nanolayer` times out first.
  - **Fix**: pre-allow `ghcr.io`, `.github.com`, `*.githubusercontent.com`, and the apt / PyPI /
    Galaxy hosts in the firewall's outgoing rules. `github.com` matches only the apex; use
    `.github.com` to match subdomains such as `api.github.com`.

## Firewall Issues

If network requests fail during molecule tests:

- Refer to <https://gh.io/copilot/firewall-config> for agent firewall setup.
- Do not work around blocked URLs; request allowlisting instead.

### Required Hosts

| Host | Purpose |
| ---- | ------- |
| `galaxy.ansible.com` | Ansible Galaxy collections |
| `github.com` | Dependency downloads |
| `raw.githubusercontent.com` | Static asset downloads |

## References

- Project documentation: [README.md](README.md)
- Agent configuration: [.github/copilot-instructions.md](.github/copilot-instructions.md)
- Org baseline: <https://github.com/Cogni-AI-OU/.github/blob/main/AGENTS.md>
- Agents.md standard: <https://agents.md/>
