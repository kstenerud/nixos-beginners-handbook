Installing NixOS in a Virtual Machine
=====================================

NixOS, being ultra-configurable, has seemingly endless installation methods, making it hard to decide which is best for your needs. The primary setups are:

* Over top of an existing system
* As its own complete operating system
* In a virtual machine or container

As a beginner, you'll want a sandbox to play around in and break things without breaking your main OS or environment, and you'll want to be able to quickly destroy and rebuild it from scratch after you break it too much. For this purpose, virtual machines are ideal.

We'll be installing from the [NixOS minimal ISO image](https://nixos.org/download.html) because it's a very basic install (no windowing environment) that also allows you to work over an SSH connection.



Building a virtual machine
--------------------------

There are many virtual machine technologies to choose from depending on your host OS. Pick whichever you're most comfortable with. Here are instructions for some of them:

| Technology                        | GUI | TUI | Linux | Mac | Windows | Other   |
| --------------------------------- | --- | --- | ----- | --- | ------- | ------- |
| [VirtualBox](nixos-virtualbox.md) |  Y  |  Y  |   Y   |  Y  |    Y    | Solaris |
| [libvirt](nixos-libvirt.md)]      |  Y  |  Y  |   Y   |  Y  |    Y    | FreeBSD |



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

This follows the [example in the NixOS manual](https://nixos.org/manual/nixos/stable/#sec-installation-partitioning). Your disk's device will be different depending on the hypervisor you're using. For example:

* In VirtualBox, you'll install to `/dev/sda`
* In libvirt, you'll install to `/dev/vda`

You can find out the device name for your disk using `parted`:

```text
[root@nixos:~]# parted -m -l
BYT;                                                                      
/dev/sr0:697MB:scsi:2048:2048:msdos:QEMU QEMU DVD-ROM:;
2:25.4MB:113MB:88.1MB:::esp;

BYT;                                                                      
/dev/vda:17.2GB:virtblk:512:512:unknown:Virtio Block Device:;
```

Here are the disk setup commands using `/dev/sda` (change to `/dev/vda` if you're using libvirt). This builds a 511MB ESP boot partition at the beginning of the disk (`mkpart ESP fat32 1MiB 512MiB`), an 8GB swap partition at the end of the disk (`mkpart primary linux-swap -8GiB 100%`), and uses the remainder for the root fs (`mkpart primary 512MiB -8GiB`). The logical partition ordering is 1: root, 2: swap, 3: boot. We also clear the first block of the drive in case it has a partition table already.

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
```

### Configure and install

NixOS is currently in transition (in mid-2021), and will soon move over to the new [flakes](https://www.tweag.io/blog/2020-05-25-flakes/) system, so I'll include instructions for the current installation method as well as the new flakes method:

* [Installing via flakes](installing-flakes.md) (new way)
* [Installing via configuration.nix](installing-configuration.md) (old way)


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

#### via flakes

Edit `/etc/nixos/flake.nix` and then build the new configuration:

```text
nixos-rebuild switch --flake /etc/nixos/#system
```

**IMPORTANT**: Don't forget to commit your changes to the flake or else they'll be ignored! You'll see a warning like `warning: Git tree '/etc/nixos' is dirty`

#### via configuration.nix

Edit `/etc/nixos/configuration.nix` and then build the new configuration:

```text
nixos-rebuild switch
```


### Done!

At this point, you have a functional NixOS in a virtual machine. You're at the equivalent of [chapter 3 in the NixOS manual](https://nixos.org/manual/nixos/stable/#sec-changing-config), and can now start [configuring your OS](https://nixos.org/manual/nixos/stable/index.html#ch-configuration).
