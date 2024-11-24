# Set Dynamic IP Address using DHCP

In case it is not possible to assign a static IP to the Proxmox machine, DHCP can be enabled to assign IP addresses instead.

Edit `/etc/network/interfaces`

```bash
auto lo
iface lo inet loopback

auto enp0s31f6
iface enp0s31f6 inet dhcp

auto vmbr0
iface vmbr0 inet dhcp
	bridge-ports enp0s31f6
	bridge-stp off
	bridge-fd 0
	
source /etc/network/interfaces.d/*
 ```

Restart the networking service

```bash
systemctl restart networking
```
