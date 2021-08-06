Common Tasks
============

Every common task will involve modifying options in your configuration, and then rebuilding your environment.

All available options are documented [here](https://nixos.org/manual/nixos/stable/options.html)

[run `nixos-rebuild`](fundamentals.md#how-do-i-apply-my-configuration-changes):



nix-env: A word of warning
--------------------------

Many tutorials will instruct you to install software using `nix-env -i <package name>`. **Don't do this!** Always install via your configuration files if you want to maintain determinism! Save `nix-env -i` for later when you understand what's going on under the hood.



How do I install software?
--------------------------

Software packages are defined in `environment.systemPackages`:

```
  environment.systemPackages = with pkgs; [
    git
    vim
    docker
  ];
```


How do I add users?
-------------------

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






TODO

* Installing software
* User management
* Running containers
* File sharing
* Mounting filesystems
* Ensuring directory structures
* Running periodic tasks
* Setting up a network bridge
