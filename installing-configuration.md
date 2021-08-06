Installing using configuration.nix
==================================

This is the current (in mid-2021) method for installing NixOS, but it will eventually be replaced with [flakes](install-flakes.md).



Generate a basic configuration
------------------------------

The first step is to generate a basic configuration (it will create `configuration.nix` and `hardware-configuration.nix` in `/mnt/etc/nixos`).

```text
nixos-generate-config --root /mnt
```


Customize your install
----------------------

Now you can customize your `configuration.nix` before installing. The default config comes with a number of commented-out suggestions. Normally, you'd keep your configuration files in a repository and clone or copy them in to the machine being provisioned.

To edit your configuration:

```text
[root@nixos:~]# nano /mnt/etc/nixos/configuration.nix
```

Once you're happy with your configuration, press CTRL-X and save the file to exit the editor.

### Configuration: Add an admin user

It's a good idea to create an admin user for yourself because logging in as root is dangerous. Here's an example user with admin privileges:

```text
  users.users.myuser = {
    isNormalUser = true;
    home = "/home/myuser";
    description = "My example admin user";
    # wheel allows sudo, networkmanager allows network modifications
    extraGroups = [ "wheel" "networkmanager" ];
    # For password login (works with console and SSH):
    hashedPassword = "$6$TxcQlxAlAM$Kz8iWHm..ZkLuLGq1oLshWVemgc1PIJihKPROQGZBvvSnE85pdMT7Wr7J4f50Qbq2dsUitoT0GPQ8yxhKyddM1";
    # For SSH key login (works with SSH only):
    #openssh.authorizedKeys.keys = [ "ssh-dss AAAAB3Nza... myuser@foobar" ];
  };
```

`hashedPassword` can be generated using `mkpasswd`:

```text
[root@nixos:~]# mkpasswd -m sha-512
Password: 
$6$TxcQlxAlAM$Kz8iWHm..ZkLuLGq1oLshWVemgc1PIJihKPROQGZBvvSnE85pdMT7Wr7J4f50Qbq2dsUitoT0GPQ8yxhKyddM1
```

### Configuration: Enable SSH

You can also turn on SSH so that you can connect via secure shell after rebooting (otherwise only the console will work):

```text
  services.openssh.enable = true; 
```


Run the installer
-----------------

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
