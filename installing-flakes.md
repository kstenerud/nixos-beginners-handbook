Installing using Flakes
=======================

Flakes are a new experimental feature (in mid-2021) that will soon enter mainstream. [This blog post](https://www.tweag.io/blog/2020-05-25-flakes/) gives a good overview of the problems flakes solve.



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

A flake package is based around the `flake.nix` file, which will behave similarly to the old `configuration.nix` file. We'll create a basic flake that has SSH enabled and an admin user (because logging in as root all the time is dangerous).

We'll be putting the flake directly in the usual `/etc/nixos` dir (basically, `flake.nix` will replace `configuration.nix`). The cool thing is that if you already have a flake set up in a repository for your machine, you could just boot the installer image, git clone your flake repo into `/mnt/etc/nixos`, and run `nixos-install`.

### Generate hardware-configuration.nix

We need a valid `hardware-configuration.nix`, so we'll generate one using `nixos-generate-config` and then throw out the `configuration.nix` file.

```text
nixos-generate-config --root /mnt
rm /mnt/etc/nixos/configuration.nix
```

### Generate a password hash for the admin user

When adding users declaratively, we must provide a [hashed password](https://nixos.org/manual/nixos/stable/options.html#opt-users.users._name_.initialHashedPassword), which can be generated using `mkpasswd`:

```text
[root@nixos:~]# mkpasswd -m sha-512
Password: 
$6$TxcQlxAlAM$Kz8iWHm..ZkLuLGq1oLshWVemgc1PIJihKPROQGZBvvSnE85pdMT7Wr7J4f50Qbq2dsUitoT0GPQ8yxhKyddM1
```

### Generate flake.nix

Now on to our actual machine configuration! `flake.nix` looks very similar to `configuration.nix`, and much of it works in the same way.

Before pasting the following snippet into installer shell, you'll want to:
- Pick your own username and home dir
- Generate your own hashed password
- Set your time zone

**Notes**:
- The `system` in `nixosConfigurations.system` is an arbitrarily chosen name that will be used in the call to `nixos-install`. You could actually name it anything you want, or have multiple configurations that you can choose from when calling `nixos-install` such as `desktop`, `laptop`, etc.
- We must specifically enable flakes in our new configuration because they are still an experimental feature (in mid-2021).

```text
cat << 'EOF' > /mnt/etc/nixos/flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-21.05";
  outputs = { self, nixpkgs }: {
    nixosConfigurations.system = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules =
        [ ({ pkgs, ... }: {
            system.configurationRevision = nixpkgs.lib.mkIf (self ? rev) self.rev;

            imports = [ ./hardware-configuration.nix ];

            boot.loader.systemd-boot.enable = true;
            boot.loader.efi.canTouchEfiVariables = true;

            networking.useDHCP = false;
            networking.interfaces.ens2.useDHCP = true;

            # Enable flakes experimental feature
            nix = {
              package = pkgs.nixFlakes;
              extraOptions = ''
                experimental-features = nix-command flakes
              '';
              registry.nixpkgs.flake = nixpkgs;
            };

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
              hashedPassword = "$6$TxcQlxAlAM$Kz8iWHm..ZkLuLGq1oLshWVemgc1PIJihKPROQGZBvvSnE85pdMT7Wr7J4f50Qbq2dsUitoT0GPQ8yxhKyddM1";
              # For SSH key login (works with SSH only):
              #openssh.authorizedKeys.keys = [ "ssh-dss AAAAB3Nza... myuser@foobar" ];
            };
          })
        ];
    };
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

When installing, we simply point `nixos-install` to our `system` configuration in the flake:

```text
nixos-install --flake /mnt/etc/nixos/#system
```

As a side-effect, nixos-install will generate `flake.lock`, which locks to the current versions of the software being installed. You should also add this to the source code repository.

If you've made any mistakes, it will print out error messages detailing what you need to fix in your `configuration.nix`.

The last installer step will ask you to set the root password (you can use `nixos-install --no-root-passwd --flake /mnt/etc/nixos/#system` to disable this and leave it blank):

```text
setting root password...
Enter new UNIX password: ***
Retype new UNIX password: ***
```
