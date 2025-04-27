# EA VPN

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

    ansible-galaxy install git+https://github.com/EA31337/ansible-role-ea-vpn.git

## Role Variables

See: [./defaults/main.yml](./defaults/main.yml)

## Dependencies

This role has the following dependencies:

- `ea31337.metatrader` - Role to install trading platform.

To install dependencies manually, run:

    ansible-galaxy role install -r requirements.yml

## Example Playbook

An example on how to use this role.

1. Install role. See: [Install](#install) section.
1. Create an inventory file.
1. Create a playbook. For example:

       ---
       - name: Provision VPN
         hosts: all
         vars:
           ea_vpn_swapfile_size: 2G
         roles:
           - role: ea31337.ea_vpn

   Check other examples in: [./tests/test.yml](./tests/test.yml)
   and [./molecule/default/converge.yml](./molecule/default/converge.yml)

1. Run playbook using `ansible-playbook` command.

## License

GNU GPL v3

See: [LICENSE](./LICENSE)
