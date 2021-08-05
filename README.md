NixOS Beginner's Handbook
=========================

[NixOS](https://nixos.org/) is a declarative, reproducible approach to system building. The documentation is great if you're an expert, but for beginners it can be very confusing. This guide is intended as a gentle, opinionated, hands-on introduction to NixOS.



Contributing
------------

No one is an island. The more people contribute, the better this guide will become! Please, if you see any place where you can help, open a pull request!



Installing
----------

The first question beginners will have is "How do I install this thing?"

As a beginner, you'll want some kind of sandbox to play around in and break stuff before you're ready to entrust your important things, so we'll focus on [installing NixOS in a VM](installing.md).



Fundamentals
------------

### Where are the configuration files?

The current configuration of your machine is stored in `/etc/nixos/configuration.nix` and any other `*.nix` files it imports. The configuration files are all you'll need to back up in order to be able to rebuild this machine.

### How do I apply my configuration changes?

After changing your configuration, you must tell NixOS what to do with it:

| Command                | Build | Run | Boot | Meaning                                                   |
| ---------------------- | ----- | --- | ---- | --------------------------------------------------------- |
| `nixos-rebuild build`  |   Y   |     |      | Build, but don't run it (to test that it at least builds) |
| `nixos-rebuild test`   |   Y   |  Y  |      | Build and run it, but don't persist across reboots        |
| `nixos-rebuild boot`   |   Y   |     |  Y   | Build, but don't run it until next reboot                 |
| `nixos-rebuild switch` |   Y   |  Y  |  Y   | Build and run it now, and also persist across reboots     |

There are many other useful rebuild commands that you can read about by typing `nixos-rebuild --help`.

### What if I'm worried that my machine won't reboot after the changes?

Specify a boot profile like so: 

```
nixos-rebuild switch -p myprofile
```

This will instruct NixOS to store this configuration under a new GRUB boot menu `NixOS - Profile 'myprofile'`.

If you don't even want this in your `/etc/nixos/configuration.nix`, you could specify a configuration file:

```
nixos-rebuild switch -p myprofile -I nixos-config=./my-dangerous-config.nix
```

### What if I broke something and want to revert my changes?

Use `nixos-rebuild --revert` to roll back to the previous configuration.



Common Tasks
------------

### How do I install software?

#### nix-env: A word of warning

Many tutorials will instruct you to install software using `nix-env -i <package name>`. **Don't do this!** Install via your configuration files only if you want to maintain determinism! Save `nix-env -i` for later when you understand what's going on under the hood.


TODO

* Installing software
* User management
* Running containers
* File sharing
* Mounting filesystems
* Ensuring directory structures
* Running periodic tasks
* Setting up a network bridge
