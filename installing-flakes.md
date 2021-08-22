Installing using Flakes
=======================

Flakes are an experimental feature (in mid-2021) that will soon enter mainstream. [This blog post](https://www.tweag.io/blog/2020-05-25-flakes/) gives a good overview of the problems flakes solve.



Enable flakes in the installer
------------------------------

We must specifically enable flakes because they're currently experimental, and we'll also need git:

```text
cat << 'EOF' > /etc/nixos/configuration.nix
{ config, pkgs, ... }:
{
  imports = [ <nixpkgs/nixos/modules/installer/cd-dvd/installation-cd-minimal.nix> ];
  nix.package = pkgs.nixUnstable;
  nix.extraOptions = ''
    experimental-features = nix-command flakes
  '';
  environment.systemPackages = with pkgs; [
    git
  ];
}
EOF
nixos-rebuild test
```

Git needs to have some info before `git commit` will work:

```text
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```



Build a flake
-------------

A flake package is based around the `flake.nix` file, which looks similar to the old `configuration.nix` file. We'll create a basic flake that has SSH enabled and an admin user (because logging in as root all the time is dangerous).

We'll be putting the flake directly in the usual `/etc/nixos` dir (basically, `flake.nix` will replace `configuration.nix`). If you already have a flake set up in a repository for your machine, you could just boot the installer image, git clone your flake repo into `/mnt/etc/nixos`, and then run `nixos-install --no-root-passwd --flake /mnt/etc/nixos/#system`.


### Generate configuration.nix and hardware-configuration.nix

Just use `nixos-generate-config` to generate a suitable base config in /mnt/etc/nixos for us:

```text
nixos-generate-config --root /mnt
```


### Generate a password hash for the admin user

When adding users declaratively, we must provide a [hashed password](https://nixos.org/manual/nixos/stable/options.html#opt-users.users._name_.initialHashedPassword), which can be generated using `mkpasswd`:

```text
[root@nixos:~]# mkpasswd -m sha-512
Password: 
$6$E9KhMqnbJgiE$lohYFW7ckILTrYGi59tbRtRQO31Jka73N/1vNnHgAbFWMubawPjFAimB.FkfWuCfODwDuuruKXR6SDQucsDuF.
```

### Generate flake.nix

`flake.nix` looks very similar to `configuration.nix`, and much of it works in the same way. We could do everything inside `flake.nix`, but instead we're going to simply point it to `configuration.nix`.

**Notes**:
- We must specifically enable flakes in our new configuration because they are still an experimental feature.
- Choose your own hostname for `myhostname`

```text
cat << 'EOF' > /mnt/etc/nixos/flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-21.05";
  outputs = { self, nixpkgs }:
    # We define a hostname here because it's used multiple times
    let hostname = "myhostname";
    in {
      nixosConfigurations.${hostname} = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";
        modules =
          [ ({ pkgs, ... }: {
              # Let 'nixos-version --json' know about the git revision of this flake.
              system.configurationRevision = nixpkgs.lib.mkIf (self ? rev) self.rev;

              # Enable flakes experimental feature
              nix = {
                package = pkgs.nixFlakes;
                extraOptions = ''
                  experimental-features = nix-command flakes
                '';
                registry.nixpkgs.flake = nixpkgs;
              };

              # Set the hostname
              networking.hostName = "${hostname}";

              # Import our system configuration
              imports = [
                ./hardware-configuration.nix
                ./configuration.nix
              ];
            })
          ];
      };
    };
}
EOF
```


### A simple base configuration.nix

The default `configuration.nix` has a lot of commented out options that can be useful to enable. Take a look at the default `/mnt/etc/nixos/configuration.nix` first before overwriting it. The following is what I would call a bare minimum config for a VM:

Before pasting the following snippet into your installer shell, you'll want to:
- Pick your own username and home dir
- Generate your own hashed password (`mkpasswd -m sha-512`)
- Set your time zone


```text
cat << 'EOF' > /mnt/etc/nixos/configuration.nix
{ config, pkgs, ... }:

{
  boot.loader = {
    systemd-boot.enable = true;
    efi.canTouchEfiVariables = true;
  };

  networking = {
    useDHCP = false;
    interfaces.ens2.useDHCP = true;
    firewall.allowedTCPPorts = [ 22 ];
  };

  # We need git or else we won't be able to apply flakes.
  environment.systemPackages = with pkgs; [
    git
  ];

  # Enable SSH so we don't have to use the console
  services.openssh.enable = true;

  # Set this to your desired time zone
  time.timeZone = "Europe/Berlin";

  # Add an admin user so we don't have to log in as root
  users.users.myuser = {
    isNormalUser = true;
    home = "/home/myuser";
    description = "My example admin user";
    # wheel allows sudo, networkmanager allows network modifications
    extraGroups = [ "wheel" "networkmanager" ];
    # For password login (works with console and SSH):
    hashedPassword = "$6$E9KhMqnbJgiE$lohYFW7ckILTrYGi59tbRtRQO31Jka73N/1vNnHgAbFWMubawPjFAimB.FkfWuCfODwDuuruKXR6SDQucsDuF.";
    # For SSH key login (works with SSH only):
    #openssh.authorizedKeys.keys = [ "ssh-dss AAAAB3Nza... myuser@foobar" ];
  };
}
EOF
```

### Set up a git repository

All files in the flake must be part of a source code repository. Any files not committed to the repo will be ignored by nix flake commands, so we must create a repository and commit our files to it:

```text
cd /mnt/etc/nixos
git init
git add --all
git commit -m 'Initial version'
```



Run the installer
-----------------

When installing, we simply point `nixos-install` to our `myhostname` configuration in the flake:

```text
nixos-install --no-root-passwd --flake /mnt/etc/nixos/#myhostname
```

**Note**: We've specified `--no-root-passwd`, which means that you cannot log in as root (you must use your admin account instead). If you omit this option, the installer will ask you to provide a root password at the end of the install process.

As a side-effect, nixos-install will generate `flake.lock`, which locks to the current versions of the software being installed. You should also add this to the source code repository.

If you've made any mistakes, it will print out error messages detailing what you need to fix in your `configuration.nix`.

**Note**: If you've set the `networking.hostName` value and your local DNS server support it, you can now use the hostname rather than the IP address to conect to this machine.

[Back to the VM installer instructions](installing-vm.md#configure-and-install)
