# Provision EA VPN in UpCloud

## Requirements

- Existing account at [UpCloud](https://hub.upcloud.com/).

## Steps

1. Install EA VPN role. Refer to [`README.md`](README.md) file.
1. Create playbook and inventory files.
1. Customize playbook by specifying EA to deploy.
1. Deploy a new server on UpCloud.
    - Ensure you're configured and added SSH keys.
    - Ensure you're able to connect via SSH using configured keys.
1. Test SSH connection by `ssh root@123.x.y.z`.
    It should connect using your key without typing password.
1. Run playbook (e.g. `ansible-playbook -i upcloud upcloud.yml`).
