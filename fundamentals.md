Fundamentals
============

Where are the configuration files?
----------------------------------

The current configuration of your machine is stored in `/etc/nixos/configuration.nix` and any other `*.nix` files it imports. The configuration files are all you'll need to back up in order to be able to rebuild this machine.



How do I apply my configuration changes?
----------------------------------------

After changing your configuration, you must tell NixOS what to do with it:

| Command                | Build | Run | Boot | Meaning                                                   |
| ---------------------- | ----- | --- | ---- | --------------------------------------------------------- |
| `nixos-rebuild build`  |   Y   |     |      | Build, but don't run it (to test that it at least builds) |
| `nixos-rebuild test`   |   Y   |  Y  |      | Build and run it, but don't persist across reboots        |
| `nixos-rebuild boot`   |   Y   |     |  Y   | Build, but don't run it until next reboot                 |
| `nixos-rebuild switch` |   Y   |  Y  |  Y   | Build and run it now, and also persist across reboots     |

There are many other useful rebuild commands that you can read about by typing `nixos-rebuild --help`.



I'm worried that my machine won't boot after my changes!
--------------------------------------------------------

Specify a boot profile like so: 

```
nixos-rebuild switch -p myprofile
```

This will instruct NixOS to store this configuration under a new GRUB boot menu `NixOS - Profile 'myprofile'`.

If the changes are so dangerous that you don't even want them in your `/etc/nixos/configuration.nix`, you could specify a different configuration file:

```
nixos-rebuild switch -p myprofile -I nixos-config=./my-dangerous-config.nix
```


I broke something and want to revert my changes!
------------------------------------------------

Use `nixos-rebuild --revert` to roll back to the previous configuration.
