NixOS Beginner's Handbook
=========================

[NixOS](https://nixos.org/) is a declarative, reproducible approach to system building. The documentation is great if you're an expert, but for beginners it can be very confusing. This guide is intended as a gentle, opinionated, hands-on introduction to NixOS.


Work-in-progress
----------------

This is a work-in-progress! Some parts are missing information, and other parts I haven't figured out yet! If you see anything amiss or feel like helping, open an issue or a PR!


Installing
----------

The first question beginners will have is "How do I install this thing?"

As a beginner, you'll want some kind of sandbox to play around in and break stuff before you're ready to entrust your important things, so we'll focus on [installing NixOS in a VM](installing-vm.md).


OS Fundamentals
---------------

Before getting into the most common tasks, take a moment to review the [fundamentals of running NixOS](os-fundamentals.md).


Common Tasks
------------

How do I do XYZ? Find out in the [common tasks section](common-tasks.md)!


Nix Expression Language
-----------------------

A quick [introduction to the Nix expression language](nix-language.md), from which everything Nix is built.


Configuration Language Basics
-----------------------------

This section covers the [most common configuration structures and idioms](config-basics.md).


Common Problems
---------------

There will be some sharp edges and counterintuitiveness in any sizeable system. If you get stuck, have a look at the [common problems section](common-problems.md).
