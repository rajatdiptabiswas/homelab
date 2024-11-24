# Post Installation

Running the following script sets useful defaults for the Proxmox VE installation.

- Disables the message reminding to purchase a subscription every time when logging in to the web interface
- Updates the Proxmox VE sources in `/etc/apt/sources.list` to allow the `apt` package manager use the correct sources to update and install packages
- Disables the `pve-enterprise` repository since it is only available after purchasing a Proxmox VE subscription
- Enables the `pve-no-subscription` repository which provides access to all the open-source components of Proxmox VE
- Corrects the Ceph package repositories to provide access to the `no-subscription` and `enterprise` repositories
- Adds the `pvetest` repository to provide access to new features and updates before they are officially released
- Disables the Proxmox high availability services to reclaim system resources when using a single node instead of a cluster

https://community-scripts.github.io/ProxmoxVE/scripts?id=post-pve-install

```bash
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/misc/post-pve-install.sh)"
```
