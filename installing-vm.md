Installing NixOS in a Virtual Machine
=====================================

There are many ways to install NixOS:

* Over top of an existing system
* As a complete operating system
* In a virtual machine or container

NixOS gives you and abundance of choice, but as a beginner too much choice leads to analysis paralysis, so for learning purposes we'll just install on a VM. This is the safest option since the inevitable bumps in the road won't harm your primary operating system.

We'll be installing from the [NixOS minimal ISO image](https://nixos.org/download.html) because it's a very basic install (no windowing environment) that also allows you to work over an SSH connection.



Building a virtual machine
--------------------------

Here are instructions for installing a VM using two popular hypervisors:

* [Building a VM using VirtualBox](nixos-virtualbox.md)
* [Building a VM using libvirt](nixos-libvirt.md)



Connecting to the installer via SSH
-----------------------------------

The console can be a bit funny at times, so it's generally nicer to SSH in. We'll need a password for the `nixos` user to allow SSH logins:

1. Create a password (this is only setting a password for the **installer**, not for the OS you are about to install):

```text
[nixos@nixos:~]$ passwd
New password: 
Retype new password: 
passwd: password updated successfully
```

2. Get the IP address:

```text
[nixos@nixos:~]$ ip a
...
2: ens2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
...
    inet 192.168.111.206/24 brd 192.168.111.255 scope global dynamic noprefixroute ens2
...
```

3. SSH in to the installer:

```text
$ ssh nixos@192.168.111.206
Password: 
Last login: Sun Aug  1 05:39:07 2021

[nixos@nixos:~]$
```

Or, if you've set up host port forwarding, simply connect to localhost port 2222:

```text
$ ssh -p 2222 nixos@localhost
Password: 
Last login: Sun Aug  1 05:39:07 2021

[nixos@nixos:~]$
```



Installing NixOS
----------------

### Switch to root

Doing the installation process as root makes it less cumbersome, and it's safe since you can't damage the installer ISO image:

```text
[nixos@nixos:~]$ sudo su -

[root@nixos:~]#
```

### Setup the disk

This follows the [example in the NixOS manual](https://nixos.org/manual/nixos/stable/#sec-installation-partitioning). Your disk will be different depending on the hypervisor you're using. For example:

* In VirtualBox, you'll install to `/dev/sda`
* In libvirt, you'll install to `/dev/vda`

You can find out what your disk's device is using `fdisk`:

```text
[root@nixos:~]# fdisk -l
...
Disk /dev/sda: 16 GiB, 17179869184 bytes, 33554432 sectors
...
```

Here are the disk setup commands using `/dev/sda` (change to `/dev/vda` if you're using libvirt).

```text
dd if=/dev/zero of=/dev/sda bs=512 count=1 conv=notrunc
parted --script /dev/sda -- mklabel gpt
parted --script /dev/sda -- mkpart primary 512MiB -8GiB
parted --script /dev/sda -- mkpart primary linux-swap -8GiB 100%
parted --script /dev/sda -- mkpart ESP fat32 1MiB 512MiB
parted --script /dev/sda -- set 3 esp on
mkfs.ext4 -F -L nixos /dev/sda1
mkswap -L swap /dev/sda2
mkfs.fat -F 32 -n boot /dev/sda3
mount /dev/disk/by-label/nixos /mnt
mkdir -p /mnt/boot
mount /dev/disk/by-label/boot /mnt/boot
swapon /dev/sda2
nixos-generate-config --root /mnt
```

### Configure and install

NixOS is currently in transition (in mid-2021), and will soon move over to the new [flakes](https://www.tweag.io/blog/2020-05-25-flakes/) system, so I'll include instructions for the current installation method as well as the new flakes method:

* [Installing via configuration.nix](installing-configuration.md)
* [Installing via flakes](installing-flakes.md)


### Reboot

After installing, simply reboot and it will boot from your newly installed disk:

```text
[root@nixos:~]# reboot

Domain creation completed.
Restarting guest.
Connected to domain nixos
Escape character is ^]


<<< Welcome to NixOS 21.05.1970.11c662074e2 (x86_64) - hvc0 >>>

Run 'nixos-help' for the NixOS manual.

nixos login: 
```

Log in as the admin user you created (or you can log in via SSH if you enbled it).


### Post-install changes

If you need to make further changes to the OS after installing, simply edit your configuration and rebuild.

#### via configuration.nix

Edit `/etc/nixos/configuration.nix` and then build the new configuration:

```text
nixos-rebuild switch
```

#### via flakes

Edit `/etc/nixos/flake.nix` and then build the new configuration:

```text
nixos-rebuild switch --flake /etc/nixos/#system
```

**IMPORTANT**: Don't forget to commit your changes to the flake or else they'll be ignored! You'll see a warning like `warning: Git tree '/etc/nixos' is dirty`


### Done!

At this point, you have a functional NixOS in a virtual machine. You're at the equivalent of [chapter 3 in the NixOS manual](https://nixos.org/manual/nixos/stable/#sec-changing-config), and can now start [configuring your OS](https://nixos.org/manual/nixos/stable/index.html#ch-configuration).
