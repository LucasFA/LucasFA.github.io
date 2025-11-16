---
title: "Packaging a Linux kernel module in Nix"
date: 2025-11-16
draft: false
---

I use a [Slimbook](https://slimbook.com) Hero laptop, based on
[Intel QC71](https://wiki.archlinux.org/title/Intel_QC71).
There is a
[kernel module](https://github.com/pobrn/qc71_laptop)
available for these, though Slimbook [develops their own patches](https://github.com/Slimbook-Team/qc71_laptop)
for their own devices.

The original kernel module [is packaged in NixOS](https://github.com/NixOS/nixpkgs/tree/master/pkgs/os-specific/linux/qc71_laptop),
and though I did try to package the fork as an essential copy-paste,
[the PR](https://github.com/NixOS/nixpkgs/pull/371848)
didn't garner any interest.

So there are two ways to package it for yourself.

## Simple

The simplest one is to create a derivation (`lib.mkDerivation`), pretty
much like the original for the other kernel module, and call it
(with `pkgs.callPackage`). You must pass the kernel version you are using
so the module is compiled against it properly. Something like this:

```nix
let
  qc71_slimbook_laptop = pkgs.callPackage ./qc71_slimbook_laptop.nix {
    # Make sure the module targets the same kernel as your system is using.
    kernel = config.boot.kernelPackages.kernel;
  };
in 
{
  # Omitted
}
```

Most of the time this is compiled
through DKMS, but on NixOS this is all just compiled at build time.

To include this in your config it's like any other module.

```nix
  boot.extraModulePackages = with config.boot.kernelPackages; [
    qc71_slimbook_laptop
  ];
  boot.kernelModules = [ "qc71_laptop" ]; # to load it at boot time
```

Note that the `kernelModules` line is not a typo.
This name is declared by the module in the [makefile](https://github.com/Slimbook-Team/qc71_laptop/blob/master/Makefile).

I haven't shown the package for a reason: it's somewhat broken.
This is (probably) me failing at this more than a limitation.
At the time I couldn't find a way to call the package with
`kernelModuleMakeFlags`, that is, the flags that the kernel has
been built with and with which the module should be built with as well.

I did have `kernel.makeFlags` available, though. Don't ask me why I
didn't use that either.

Point being, I had hardcoded the make flags.

## Rolling it into the NUR

The [Nix User Repository](https://github.com/nix-community/NUR)
is somewhat akin to Arch's AUR. Each user has
[their own repository](https://github.com/LucasFA/nur-packages),
and you can include packages, modules and overlays.

The basic structure goes something like this:

```sh
├── default.nix
├── flake.lock
├── flake.nix
├── lib
├── LICENSE
├── modules
├── overlay.nix
├── overlays
├── pkgs
├── README.md
```

As you'd expect, the package goes in `pkgs`. We'll also make use of
overlays.

I didn't have the same issues regarding make flags as I did in
the vendored configuration, and it was closer to the nixpkgs
`qc71_laptop` package.

The overlay overrides nixpkgs' `packagesFor` function, which
[builds the packages for a given kernel](https://github.com/NixOS/nixpkgs/blob/master/pkgs/top-level/linux-kernels.nix).
In nixpkgs, it is then called for each kernel.

We patch it with the same idea as
[mentioned in the NixOS wiki](https://wiki.nixos.org/wiki/Overlays#Overriding_a_package_inside_an_extensible_attribute_set),
as `packagesFor` is defined with `lib.mkExtensible`:

```nix
final: prev:
{
  linuxKernel = prev.linuxKernel // {
    packagesFor =
      kernel:
      (prev.linuxKernel.packagesFor kernel).extend (
        kFinal: kPrev:
        (final.lib.filesystem.packagesFromDirectoryRecursive {
          inherit (kFinal) callPackage;
          directory = ../pkgs/linux-packages;
        })
      );
  };
}
```

A bit verbose, and you're passing an overlay inside the overlay...
But all it's doing is patching the previous
`linuxKernel` package with the extended version of `packagesFor`.

With that, all you need to do is add the repo as an input
to your flake, add the overlay:

```nix
  nixpkgs.overlays = [
    inputs.lucasfa-nur.legacyPackages."${system}".overlays.qc71_slimbook_laptop
  ];
}
```

and add the package as if it was any other kernel module, no extra `callPackage`:

```nix
boot.extraModulePackages = with config.boot.kernelPackages; [ 
  qc71_slimbook_laptop 
];
boot.kernelModules = [ "qc71_laptop" ];
```

The nix snippets in this article are slightly modified for brevity. If you
are reading this to get your config working, I recommend you head over
to [my repo](https://github.com/LucasFA/.nixos/tree/blog/2025-11-packaging-kernel-module).
