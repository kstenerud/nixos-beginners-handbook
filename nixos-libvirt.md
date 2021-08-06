Running NixOS under libvirt
===========================

[Libvirt](https://libvirt.org/) is a toolkit to manage virtualization platforms. It supports all of the major VM and emulation platforms including KVM, QEMU, Xen, Virtuozzo, VMWare ESX, LXC, BHyve and is available on pretty much all Linux distributions these days.

There are many management tools that plug into this system (notably [Virtual Machine Manager](https://virt-manager.org/)), but we're going to do everything from the command line interface to simulate a headless NixOS build.

Prerequisites
-------------

You will need to have libvirt and virt-install on your system.

On Ubuntu:

```text
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system virtinst
```

On Redhat:

```text
sudo yum install kvm virt-manager libvirt libvirt-python python-virtinst
```

You'll also need the [NixOS minimal ISO image](https://nixos.org/download.html)


Launching the VM
----------------

Use the following as an example for launching your vm, picking a decent place to create your qcow2 disk image (you will be installing to this) so that you can find it later:

```text
virt-install --name=nixos \
--disk /path/to/my-nixos-disk-image.qcow2,device=disk,bus=virtio,size=16 \
--cdrom=/path/to/latest-nixos-minimal-x86_64-linux.iso \
--memory=2048 \
--vcpus=2 \
--os-type=generic  \
--boot=uefi \
--nographics \
--console pty,target_type=virtio
```

This launches a UEFI-enabled (`--boot=uefi`) headless (`--nographics`) VM named "nixos" (`--name=nixos`) with 2 GB of RAM (`--memory=2048`), 2 CPUs (`--vcpus=2`), and a disk image with 16 GB of space (`size=16`). It also connects to the guest's console (`--console pty,target_type=virtio`).

**Note**: The `--cdrom` entry sets the ISO image to boot from **once**. After rebooting, it will boot from your qcow2 image instead.

Due to how the console works, you won't see the boot menu or the status messages as it boots. All you'll see is this, looking like it's stuck:

```text
Starting install...
Connected to domain nixos
Escape character is ^]
```

Just be patient; it should boot within a couple of minutes:

```text
<<< Welcome to NixOS 21.05.1970.11c662074e2 (x86_64) - hvc0 >>>
The "nixos" and "root" accounts have empty passwords.

An ssh daemon is running. You then must set a password
for either "root" or "nixos" with `passwd` or add an ssh key
to /home/nixos/.ssh/authorized_keys be able to login.


Run 'nixos-help' for the NixOS manual.

nixos login: nixos (automatic login)


[nixos@nixos:~]$
```


Some tips before you continue
-----------------------------

There will invariably be problems, so here are some tips to help get you unstuck:

### Exiting the console

To exit the console, hold the CTRL key and press `]`, then press Enter.

### Reconnecting to the console:

To reconnect to the console, type `virsh console nixos`

### Deleting everything and starting over

If everything gets completely broken, here's how to start over fresh:

* Stop the VM by typing `virsh destroy nixos` (turns off the machine)
* Remove the domain by typing `virsh undefine nixos --nvram` (deletes the VM)
* If you want the disk image gone also, you must delete it manually (wherever you put `my-nixos-disk-image.qcow2`)

### Accessing the VM from your host

Use the `virsh` command to access the VM from the host. It's a good idea to [familiarize yourself with virsh](https://libvirt.org/manpages/virsh.html).

```text
$ virsh list
 Id   Name    State
-----------------------
 13   nixos   running
```

### Getting the guest's IP address

You can do this within the guest by typing `ip a`, or you can do it from the host side using virsh's `net-dhcp-leases` command:

```text
$ virsh net-dhcp-leases default
 Expiry Time           MAC address         Protocol   IP address          Hostname         Client ID or DUID
-----------------------------------------------------------------------------------------------------------------
 2021-07-23 21:04:31   52:54:00:33:0c:ee   ipv4       192.168.111.206/24   nixos            01:52:54:00:33:0c:ee

```

---------------------------------------------------------------------

Your libvirt virtual machine is now booted into the NixOS minimal installer.

[Back to the VM installer instructions](installing-vm.md#building-a-virtual-machine)
