Containers
==========

Containers live in `/var/lib/containers`

Containers can be manipulated using `nixos-container`.

Defining a container
--------------------

```
  # Define a container called "mywebserver"
  containers.mywebserver = {
    # Just like a regular nixos config
    config = { config, pkgs, ... }: {
      # Turn on the default HTTP service as an example
      services.httpd.enable = true;
      services.httpd.adminAddr = "foo@example.org";
      networking.firewall.allowedTCPPorts = [ 80 ];
    };

    # Or start manually: nixos-container start webserver
    autoStart = true;

    # Mount a host directory inside the container
    bindMounts = {
      "/home/joe" = {
        hostPath = "/mnt/backup/home/joe";
        isReadOnly = false;
      };
    };
  };
```

