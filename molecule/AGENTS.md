# Molecule Testing

## Molecule Scenarios

| Scenario | Notes |
| -------- | ----- |
| `default` | Swapfile provisioning tests |

### Platforms (default scenario)

| Container | Image | Notes |
| --------- | ----- | ----- |
| `ea-vpn-default-ubuntu-jammy` | `ubuntu:jammy` | Uses apt |
| `ea-vpn-default-ubuntu-noble` | `ubuntu:noble` | Uses apt |

Platform names follow the `<role>-<scenario>-<platform>` convention (here
`ea-vpn-default-`) because Molecule's Docker driver names each container exactly
after its platform. Generic names such as `ubuntu-noble` would collide with
concurrent Molecule runs of other roles, and role-only names would collide across
scenarios of the same role.

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

## Molecule Gates

- `pipenv run molecule syntax` - YAML + playbook syntax validation
- `pipenv run molecule converge` - full role execution on all containers
- `pipenv run molecule idempotence` - re-run must produce zero changes
- `pipenv run molecule verify` - asserts role functionality
- `yamllint .` - YAML lint (config: `.yamllint`)
- `ansible-lint` - Ansible best practices (config: `.ansible-lint`)
- `pre-commit run -a` - all pre-commit hooks

## Troubleshooting

### Molecule prepare fails with DNS resolution errors

> `Temporary failure resolving 'deb.debian.org'` (or `azure.archive.ubuntu.com`), followed by
> `E: Unable to locate package python3` during the `prepare` step.

- **Root cause**: `docker0` has lost the `bridge` network's configured gateway address, so containers
  on the default bridge have no working gateway and cannot resolve DNS or reach the network.
- **Check**: `ip -4 addr show docker0` vs
  `docker network inspect bridge --format '{{range .IPAM.Config}}{{.Gateway}}{{end}}'`.
  If the gateway address is missing from `docker0`, this is the cause.
- **Fix (durable)**: `sudo systemctl restart docker` recreates `docker0` with the configured gateway.
  This restarts the daemon and stops any running containers.
- **Fix (non-disruptive, not persistent)**: `sudo ip addr add 172.17.0.1/16 dev docker0`
  (use the gateway reported by the check above).
- **Workaround (no sudo)**: run the tests on the host network, which bypasses the broken bridge.

```bash
MOLECULE_DOCKER_NETWORK=host \
MOLECULE_DOCKER_FORCE_IPV4=true \
MOLECULE_XVFB_DISPLAY_BASE=90 \
pipenv run molecule test
```
