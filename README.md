# Ansible Role: EA VPN

[![CodeRabbit PR Reviews](https://img.shields.io/coderabbit/prs/github/EA31337/ansible-role-ea-vpn?utm_source=oss&utm_medium=github&utm_campaign=EA31337%2Fansible-role-ea-vpn&labelColor=171717&color=FF570A&link=https%3A%2F%2Fcoderabbit.ai&label=CodeRabbit+PR+Reviews)](https://github.com/EA31337/ansible-role-ea-vpn/pulls)
[![License](https://img.shields.io/badge/license-GPLv3-brightgreen.svg)](LICENSE)

Ansible role to provision VPN (Virtual Private Server)
for Expert Advisors.

## Requirements

This role requires:

- Ansible
- Python
- Administrative/root access on target hosts
- One of the following operating systems:
  - Alpine Linux
  - Debian/Ubuntu
  - NixOS or systems with Nix package manager

For the VPN functionality:

- Open internet access from the target host.
- Sufficient system resources based on expected VPN traffic load.

## Install

To install this role, you can use the following terminal command:

```shell
ansible-galaxy install git+https://github.com/EA31337/ansible-role-ea-vpn.git
```

## Role Variables

See: [./defaults/main.yml](./defaults/main.yml)

## Dependencies

This role has the following dependencies:

- `ea31337.metatrader` - Role to install trading platform.

To install dependencies manually, run:

```console
ansible-galaxy role install -r requirements.yml
```

## Example Playbook

An example on how to use this role.

1. Install role. See: [Install](#install) section.
1. Create an inventory file. See: [`./tests/inventory`](./tests/inventory).
1. Create a playbook. For example:

      ```yaml
      ---
      - name: Provision VPN
        hosts: all
        vars:
          ea_vpn_swapfile_size: 2G
        roles:
          - role: ea31337.ea_vpn
      ```

   Check other examples in: [`./tests/test.yml`](./tests/test.yml)
   and [`./molecule/default/converge.yml`](./molecule/default/converge.yml)

1. Run playbook using `ansible-playbook` command.

## Testing

### Docker

Steps to test role on Docker containers.

1. Install the current role by running the following commands in shell:

    ```shell
    ansible-galaxy install -r requirements.yml
    jinja2 requirements-local.yml.j2 -D "pwd=$PWD" -o requirements-local.yml
    ansible-galaxy install -r requirements-local.yml
    ```

2. Ensure Docker service (e.g. Docker Desktop) is running.
3. Run playbook from `tests/`:

    ```shell
    ansible-playbook -i tests/inventory/docker-containers.yml tests/playbooks/docker-containers.yml
    ```

### Molecule

To test using Molecule, run:

```shell
molecule test
```

## License

GNU GPL v3

See: [LICENSE](./LICENSE)

<!-- Named links -->
