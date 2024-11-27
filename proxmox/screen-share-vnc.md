# Set up Screen Sharing using VNC

Open the Shell of the Proxmox node.

If we want to VNC into the VM with the VM ID `100`, edit `/etc/pve/qemu-server/100.conf`.

Add the line `-args vnc 0.0.0.0:55` to `/etc/pve/qemu-server/100.conf` where `55` is the chosen display number.

VNC server will listen on `5900 + display_number`.

We can now VNC to the VM 100 using `<Proxmox IP>:5955`.

macOS Screen Sharing app requires a password to be set on the VNC server.

![](attachments/macos-screen-sharing-password.png)

Need to add `password=on` to the line `args: -vnc 0.0.0.0:55`.

![](attachments/proxmox-edit-vnc-config.png)

Go to the VM 100's 'Monitor' page and set the password using

```bash
set_password vnc <password> -d vnc2
```

The above command needs to be run again to set the password when the VM is restarted.

## References
[1] https://pve.proxmox.com/wiki/VNC_Client_Access
