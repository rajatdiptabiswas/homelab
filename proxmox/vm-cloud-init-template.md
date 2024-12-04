# Create Cloud-Init VM Template

Proxmox can use cloud images meant for OpenStack.

When the VM is started for the first time, the `cloud-init` inside the VM can initialize and configure the VM with the predefined network settings, SSH keys, and more.

## Create a Cloud-Init Debian 12 Bookworm VM Template

### Download cloud image

Proxmox uses a KVM (Kernel-based Virtual Machine) so there is no need to download images meant for physical hardware.

For example, when downloading Debian cloud images, `genericcloud` is enough for Proxmox. `generic` has drivers for bare metal installations that are not required.

![](/attachments/debian-12-cloud-download.png)

`qcow2` (Quick Copy-On-Write) images are intended to be used as the root filesystem of a virtual machine.

`iso` images are used as installation media and require manual setup process.

Download `debian-12-genericcloud-amd64.qcow2` image for Debian 12.

There are no `iso` images available, so download the `qcow2` image from https://cdimage.debian.org/images/cloud/bookworm/latest/

```bash
cd /var/lib/vz/template/iso
```

```bash
wget https://cdimage.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2
```

### Install packages on the cloud image

Update the downloaded cloud image to include `qemu-guest-agent` to make it easier for the hypervisor to communicate with the VM.

> [!CAUTION]
> It is generally recommended to install packages after a VM boots and completes the `cloud-init` initialization step.
> 
> Installing packages directly to the `qcow2` image causes `/etc/machine-id` to be generated.
> 
> This creates issues like the DHCP server assigning the same IP address to every clone created from the template since they all have the same `machine-id`.
> 
> Deleting the contents of `/etc/machine-id` after installing the package has fixed the issue here.

```bash
apt install -y libguestfs-tools
```

```bash
virt-customize -a debian-12-genericcloud-amd64.qcow2 --install qemu-guest-agent
```

```bash
virt-customize -a debian-12-genericcloud-amd64.qcow2 --truncate /etc/machine-id
```

#### Terminal output

```bash
root@optiplex:/var/lib/vz/template/iso# virt-customize -a debian-12-genericcloud-amd64.qcow2 --install qemu-guest-agent
[   0.0] Examining the guest ...
[   7.4] Setting a random seed
virt-customize: warning: random seed could not be set for this type of 
guest
[   7.4] Setting the machine ID in /etc/machine-id
[   7.4] Installing packages: qemu-guest-agent
[  17.5] Finishing off
root@optiplex:/var/lib/vz/template/iso# 
```

```bash
root@optiplex:/var/lib/vz/template/iso# virt-customize -a debian-12-genericcloud-amd64.qcow2 --truncate /etc/machine-id
[   0.0] Examining the guest ...
[   3.4] Setting a random seed
virt-customize: warning: random seed could not be set for this type of 
guest
[   3.5] Truncating: /etc/machine-id
[   3.5] Finishing off
root@optiplex:/var/lib/vz/template/iso#
```

### Create the VM

Create a new VM `debian-12-cloud` with VirtIO SCSI controller.

```bash
qm create 1000 \
    --memory 2048 --core 2 \
    --name debian-12-cloud \
    --net0 virtio,bridge=vmbr0
```

### Set VM parameters

Import the downloaded image to `local-lvm` storage, attaching it as a SCSI drive. Similar to plugging in a hard drive to the motherboard SCSI port 0.

Enable SSD emulation in case the underlying disk is an SSD or NVMe. Enables TRIM support with `discard`.

Setting the SCSI Controller to 'VirtIO SCSI single' and enabling I/O Threads provide the best disk performance. [2] 

```bash
qm set 1000 \
    --scsihw virtio-scsi-single \
    --scsi0 local-lvm:0,import-from=/var/lib/vz/template/iso/debian-12-genericcloud-amd64.qcow2,ssd=1,discard=on,iothread=1
```

Add Cloud-Init CD-ROM drive to to pass Cloud-Init data to the VM. Similar to attaching a CD-ROM drive with `cloudinit` disk inserted.

```bash
qm set 1000 --ide2 local-lvm:cloudinit
```

Restrict BIOS to boot from Cloud-Init image in the SCSCI port 0 only. Speeds up booting because VM BIOS skips testing for bootable CD-ROM.

```bash
qm set 1000 --boot order=scsi0
```

Allows the VMs to get all the CPU's instruction set and capabilities instead of using a virtualized setup. Reduces portability (live migration between nodes) but generally provides better performance.

```bash
qm set 1000 --cpu host
```

Configure serial console to use as a display. Equivalent to plugging in a VGA monitor to the VM.

```bash
qm set 1000 \
    --serial0 socket \
    --vga serial0
```

Enable QEMU Guest Agent

```bash
qm set 1000 --agent enabled=1
```

### Set `cloud-init` parameters

Set Cloud-Init username and password
```bash
qm set 1000 --ciuser debian
```
```bash
qm set 1000 --cipassword $(openssl passwd -6 "bookworm")
```

Set Cloud-Init SSH keys
```bash
qm set 1000 --sshkeys /root/.ssh/id_rsa.pub
```

Set DHCP for the VM's network interface
```bash
qm set 1000 --ipconfig0 ip=dhcp
```

### Create template

Convert to created VM into a template

```bash
qm template 1000
```

## Create a new VM from template

```bash
qm clone <vm_template_id> <vm_id> --name <vm_name> --full
```

```bash
qm clone 1000 200 --name debian --full
```

```bash
qm start 200
```

If the image did not have `qemu-guest-agent` installed using `virt-customize`, install `qemu-guest-agent` after the VM boots up and `cloud-init` has completed initialization. Make sure to enable QEMU Guest Agent in the Options menu as well.

```bash
sudo apt install qemu-guest-agent
```

## References

[1] https://pve.proxmox.com/wiki/Cloud-Init_Support

[2] https://kb.blockbridge.com/technote/proxmox-aio-vs-iouring/
