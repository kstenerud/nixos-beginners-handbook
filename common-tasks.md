Common Tasks
============

Every common task will involve modifying options in your configuration, and then rebuilding your environment.

All available options are documented [here](https://nixos.org/manual/nixos/stable/options.html)

Note: You'll need to [run `nixos-rebuild`](fundamentals.md#how-do-i-apply-my-configuration-changes) after making changes!


### Tasks

* [Installing software](#installing-software)
* [Adding users](#adding-users)
* [Ensuring a directory exists](#ensuring-a-directory-exists)
* [Enabling LXD](#enabling-lxd)
* [Enabling Docker](#enabling-docker)
* [Adding a NixOS container](#adding-a-nixos-container)
* [Bridging to your LAN](#bridging-to-your-lan)
* [Enabling Samba file sharing](#enabling-samba-file-sharing)



nix-env: A word of warning
--------------------------

Many tutorials will instruct you to install software using `nix-env -i <package name>`. **Don't do this!** Always install via your configuration files if you want to maintain determinism! Save `nix-env -i` for later when you understand its implications and what's going on under the hood.



Installing Software
-------------------

Software packages are defined in `environment.systemPackages`:

```
  environment.systemPackages = with pkgs; [
    git
    vim
    wget
  ];
```



Adding users
------------

Users are defined under `users.users`:

```
  users.users.myuser = {
    isNormalUser = true;
    home = "/home/myuser";
    description = "My example admin user";
    # wheel allows sudo, networkmanager allows network modifications
    extraGroups = [ "wheel" "networkmanager" ];
    # For password login (works with console and SSH):
    hashedPassword = "$6$Jiqrq/BA.fVAogyP$3hYRaCNEX1X.3BpGEwdnGbtUIYRDZe13Le0K6RzgRY817KgfcnNCvyH6qy7pdhuYLD7ZMxu.HBOpakb9/iDqa.";
    # For SSH key login (works with SSH only):
    #openssh.authorizedKeys.keys = [ "ssh-dss AAAAB3Nza... myuser@foobar" ];
  };
```



Ensuring a directory exists
---------------------------

This one is very cryptic:

```
  systemd.tmpfiles.rules = [
    "d /path/to/dir 700 myuser mygroup -"
  ];
```



Enabling LXD
------------

```
  virtualisation.lxd.enable = true;
```

Now the `lxd` and `lxc` commnds will be avilable.



Enabling Docker
---------------
```
  virtualisation.docker.enable = true;
```

Now the `docker` command is available. To give access to a user:

```
  users.users.myuser.extraGroups = [ "docker" ];
```

**Warning**: Adding a user to the docker group is like giving [root access](https://github.com/moby/moby/issues/9976)



Adding a NixOS container
------------------------

```
  containers.http = {
    # Or start manually: nixos-container start http
    autoStart = true;

    # Use a specific bridge instead of the host's networking
    hostBridge = "br0"; # Must have set up this bridge on the host
    privateNetwork = true; # Generate an "eth0" inside the container

    # Bind mount a host directory
    bindMounts = {
      "/home" = {
        hostPath = "/home/alice";
        isReadOnly = false;
      };
    };

    config = { config, pkgs, ... }: {
      # If you've enabled privateNetwork, have eth0 use DHCP
      networking.interfaces.eth0.useDHCP = true;

      boot.isContainer = true;
      services.httpd.enable = true;
      services.httpd.adminAddr = "foo@example.org";
      networking.firewall.allowedTCPPorts = [ 80 ];
    };
  };
```

Notes:
- Setting `hostBridge`, `privateNetwork`, and `networking.interfaces.eth0.useDHCP` causes the container to participate directly in the bridge. If the bridge is configured to [bridge directly to your LAN](#bridging-to-your-lan), it will have its own peer address and name, and be accessible to anything on the LAN.

Containers live in `/var/lib/containers`

Log into container: `sudo nixos-container root-login mycontainer`



Bridging to your LAN
--------------------

Disable DHCP for your ethernet port (in this case `ens2`), set up a bridge (`br0`) to it, and turn on DHCP to the bridge.

```
  networking = {
    bridges.br0.interfaces = [ "ens2" ];

    useDHCP = false;
    interfaces.ens2.useDHCP = false;
    interfaces.br0.useDHCP = true;
  };
```


Enabling Samba file sharing
---------------------------

Open the required ports in the firewall:

```
  networking.firewall = {
    allowedTCPPorts = [ 445 139 ];
    allowedUDPPorts = [ 137 138 ];
  };
```

Enable Avahi (makes it easier to browse in Linux and Mac):

```
  services.avahi = {
    enable = true;
    nssmdns = true;
    publish = {
      enable = true;
      addresses = true;
      domain = true;
      hinfo = true;
      userServices = true;
      workstation = true;
    };
    extraServiceFiles = {
      smb = ''
        <?xml version="1.0" standalone='no'?><!--*-nxml-*-->
        <!DOCTYPE service-group SYSTEM "avahi-service.dtd">
        <service-group>
          <name replace-wildcards="yes">%h</name>
          <service>
            <type>_smb._tcp</type>
            <port>445</port>
          </service>
        </service-group>
      '';
    };
  };
```

Enable Samba:

```
  services.samba = {
    enable = true;

    # To access via username, you must generate a password: $ sudo smbpasswd -a myusername

    # [global] section config:
    extraConfig = ''
      # These next four lines are a workaround for a bug in gvfs https://bugs.launchpad.net/gvfs/+bug/1828107
      # This forces the very insecure SMB1 protocol, so remove it if you're not affected!
      workgroup = WORKGROUP
      client min protocol = NT1
      server min protocol = NT1
      name resolve order = bcast lmhosts host wins
    '';

    shares = {
      # Share user homes
      homes = {
        browseable = "no"; # "homes" is not browseable, but each home is
        "read only" = "no";
        "guest ok" = "no";
      };

      # Example public share
      public = {
        path = "/home/public";
        browseable = "yes";
        "read only" = "no";
        "guest ok" = "yes";
        "create mask" = "0644";
        "directory mask" = "0755";
        "force user" = "myusername";
        "force group" = "mygroupname";
      };
    };
  };
```








TODO

* Mounting filesystems
* Running periodic tasks
