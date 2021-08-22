NixOS Fundamentals
==================

Where are the configuration files?
----------------------------------

Your global configuration files will be in `/etc/nixos`. Either there will be a `configuration.nix` file (the older way), or a `flake.nix` file (the newer way). The configuration files are all you need to back up in order to be able to rebuild this machine.



How do I apply my configuration changes?
----------------------------------------

After changing your configuration, you must tell NixOS what to do with it:

| Command                | Build | Run | Boot | Meaning                                                   |
| ---------------------- | ----- | --- | ---- | --------------------------------------------------------- |
| `nixos-rebuild build`  |   Y   |     |      | Build, but don't run it (to test that it at least builds) |
| `nixos-rebuild test`   |   Y   |  Y  |      | Build and run it, but don't persist across reboots        |
| `nixos-rebuild boot`   |   Y   |     |  Y   | Build, but don't run it until next reboot                 |
| `nixos-rebuild switch` |   Y   |  Y  |  Y   | Build and run it now, and also persist across reboots     |

If you're applying configuration from flakes, you must specify the flake to configure from (e.g. `nixos-rebuild switch --flake /mnt/etc/nixos/#system`)

There are many other useful rebuild commands that you can read about by typing `nixos-rebuild --help`.



I'm worried that my machine won't boot after my changes!
--------------------------------------------------------

Specify a boot profile like so: 

```
nixos-rebuild switch -p myprofile
```

This will instruct NixOS to store this configuration under a new GRUB boot menu `NixOS - Profile 'myprofile'`.



I broke something and want to revert my changes!
------------------------------------------------

Use `nixos-rebuild --revert` to roll back to the previous configuration.



What does an empty configuration look like?
-------------------------------------------

As a flake (`/etc/nixos/flake.nix`):

```text
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-21.05";
  outputs = { self, nixpkgs }: {
    nixosConfigurations.system = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules =
        [ ({ pkgs, ... }: {

            # You'll always need to import at least a hardware configuration
            imports = [ ./hardware-configuration.nix ];

            # Put your options here
          })
        ];
    };
  };
}
```

As a configuration (`/etc/nixos/configuration.nix`):

```text
{ config, pkgs, ... }:

{
  system.stateVersion = "21.05";

  # You'll always need to import at least a hardware configuration
  imports = [ ./hardware-configuration.nix ];

  # Put your options here
}

```

A hardware configuration (`/etc/nixos/hardware-configuration.nix`) should be generated using `nixos-generate-config`, and will look something like this:

```text
{ config, lib, pkgs, modulesPath, ... }:

{
  imports = [ (modulesPath + "/profiles/qemu-guest.nix") ];

  boot.initrd.availableKernelModules = [ "ata_piix" "uhci_hcd" "ehci_pci" "virtio_pci" "sr_mod" "virtio_blk" ];
  boot.initrd.kernelModules = [ ];
  boot.kernelModules = [ "kvm-intel" ];
  boot.extraModulePackages = [ ];

  fileSystems."/" = {
    device = "/dev/disk/by-uuid/60c2db68-f368-45b2-b6f5-fb3fcf23e4f7";
    fsType = "ext4";
  };

  fileSystems."/boot" = {
    device = "/dev/disk/by-uuid/8C31-366F";
    fsType = "vfat";
  };

  swapDevices = [ { device = "/dev/disk/by-uuid/bb553d7d-d52d-42e8-b38a-882f2040436f"; } ];
}
```
