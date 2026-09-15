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

## Molecule Scenarios

| Scenario | Notes |
| -------- | ----- |
| `default` | Swapfile provisioning tests |

### Platforms (default scenario)

| Container | Image | Notes |
| --------- | ----- | ----- |
| `ubuntu-jammy` | `ubuntu:jammy` | Uses apt |
| `ubuntu-noble` | `ubuntu:noble` | Uses apt |

### Running Tests

Molecule and Ansible are installed via the project `Pipfile`, so run every command through `pipenv`
(they are not on `PATH`).

```bash
# Full test (all scenarios)
pipenv run molecule test

# Single scenario
pipenv run molecule test -s default

# Individual steps
pipenv run molecule create -s default
pipenv run molecule converge -s default
pipenv run molecule verify -s default
pipenv run molecule destroy -s default

# Syntax check only
pipenv run molecule syntax
```

### Sandboxed / firewalled environments

In sandboxed or firewalled environments the default Docker bridge may have no outbound NAT, and the
resolver may return IPv6 addresses that are not routable. Both behaviours are opt-in via environment
variables consumed by `molecule/default/create.yml`:

- `MOLECULE_DOCKER_NETWORK` (default `default`): Docker network used for test containers. Set to
  `host` in sandboxed environments where the default bridge has no outbound NAT.
- `MOLECULE_DOCKER_FORCE_IPV4` (default unset): Prefer IPv4 for DNS resolution inside containers.
  Set to `true` when the resolver returns IPv6 addresses that are not routable.

Example invocation:

```bash
MOLECULE_DOCKER_NETWORK=host MOLECULE_DOCKER_FORCE_IPV4=true pipenv run molecule test -s default
```

## Testing & Verification Gates

- `pipenv run molecule syntax` - YAML + playbook syntax validation
- `pipenv run molecule converge` - full role execution on all containers
- `pipenv run molecule idempotence` - re-run must produce zero changes
- `pipenv run molecule verify` - asserts role functionality
- `yamllint .` - YAML lint (config: `.yamllint`)
- `ansible-lint` - Ansible best practices (config: `.ansible-lint`)
- `pre-commit run -a` - all pre-commit hooks

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
