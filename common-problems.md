Common Problems
===============

`nix-channel --list` does nothing!
----------------------------------

NixOS uses different profiles for a user calling nix-channel and root calling nix-channel. To work with the system profile, call it with `sudo`.
