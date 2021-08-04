Installing NixOS
================

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

The console can be a bit funny at times (especially if you resize your window), so it's generally nicer to SSH in. We'll use a password-based SSH login since it's only the installer.

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

* In VirtualBox, it's `/dev/sda`
* In libvirt, it's `/dev/vda`

You can find out what your disk's device is using `fdisk`:

```text
[root@nixos:~]# fdisk -l
...
Disk /dev/sda: 16 GiB, 17179869184 bytes, 33554432 sectors
...
```

Here are the disk setup commands using `/dev/sda` (change to `/dev/vda` if you're using libvirt).

```text
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

### Customize your install

At this point, you can customize your configuration before installing. Configuration happens via `configuration.nix`, and the default config comes with a number of commented-out suggestions. Normally, you'd keep your configuration files in a repository and clone or copy them in to the machine being provisioned.

To edit your configuration:

```text
[root@nixos:~]# nano /mnt/etc/nixos/configuration.nix
```

Once you're happy with your configuration, press CTRL-X and save the file to exit the editor.

#### Configuration: Add an admin user

It's a good idea to create an admin user for yourself because logging in as root is dangerous. Here's an example user with admin privileges:

```text
  users.users.myuser = {
    isNormalUser = true;
    home = "/home/myuser";
    description = "My example admin user";
    # wheel allows sudo, networkmanager allows network modifications
    extraGroups = [ "wheel" "networkmanager" ];
    # For password login (works with console and SSH):
    hashedPassword = "$6$Cc5l1Gyv2gP$Mw0RKFkH719QCZAggQDTJIDcE4HoHFEYUqS71H0FVA/AHR4BJEWhfyPaR3RKiz3WsMsDp1di4oPX3b1s3s6Jt.";
    # For SSH key login (works with SSH only):
    openssh.authorizedKeys.keys = [ "ssh-dss AAAAB3Nza... myuser@foobar" ];
  };
```

`hashedPassword` can be generated using `mkpasswd`:

```text
[root@nixos:~]# mkpasswd -m sha-512
Password: 
$6$Cc5l1Gyv2gP$Mw0RKFkH719QCZAggQDTJIDcE4HoHFEYUqS71H0FVA/AHR4BJEWhfyPaR3RKiz3WsMsDp1di4oPX3b1s3s6Jt.
```

#### Configuration: Enable SSH

You can also turn on SSH so that you can connect via secure shell after rebooting (otherwise only the console will work):

```text
  services.openssh.enable = true; 
```

### Run the installer

Once you're happy with your configuration, it's time to install the OS:

```text
[root@nixos:~]# nixos-install
```

If you've made any mistakes, it will print out error messages detailing what you need to fix in your `configuration.nix`.

The last installer step will ask you to set the root password (you can use `nixos-install --no-root-passwd` to disable this and leave it blank):

```text
setting root password...
Enter new UNIX password: ***
Retype new UNIX password: ***
```

### Reboot

This will cause it to reboot into your newly installed disk:

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

Log in as the admin user you created (or you can log in via SSH if you enbled it). If you need to make further changes to the configuration, edit `/etc/nixos/configuration.nix` and then build the new configuration:

```text
nixos-rebuild switch
```

At this point, you have a functional NixOS in a virtual machine. You're at the equivalent of [chapter 3 in the NixOS manual](https://nixos.org/manual/nixos/stable/#sec-changing-config), and can now start [configuring your OS](https://nixos.org/manual/nixos/stable/index.html#ch-configuration).
