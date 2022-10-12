# Heroku deployment with nix
This document summarizes my learnings from various attempts to use nix with heroku.  
Please feel free to edit this if new knowledge arises.

# Integration via buildpack
Heroku uses [buildpacks](https://devcenter.heroku.com/articles/buildpacks) in order to build deployed code. A buildpack is basically just a collection of scripts, following [this API](https://devcenter.heroku.com/articles/buildpack-api), which are executed inside a sandboxed environment (stack).

There is only a [limited number of stacks](https://devcenter.heroku.com/articles/stack) that can be selected for buildpack based builds.

All stacks are ubuntu based, (basically there is one for each ubuntu release 18.xx, 20.xx, 22.xx).
The list of packages which are available in the ubuntu stacks, are [listed here](https://devcenter.heroku.com/articles/stack-packages).

It is not possible to use third party stacks (to change the base image).

Therefore it seems, the only way to integrate nix with heroku buildpacks is getting nix to work on the ubuntu based stack.

Inside the stack, priviledges are limited and the `/nix` directory canot be created.
The following workarounds have been explored:

## 1. Runnning nix inside a namespace environment, simulating the /nix directory

### 1.1 User namespaces
User namespaces could be used via tools like [bubblewrap](https://github.com/containers/bubblewrap), but this capability is missing from inside the stack sandbox. User namespaces are not available.

### 1.2 Proot
[Proot](https://github.com/proot-me/proot) is a tool that utilizes the `ptrace` system call to allow for unpriviledged `chroot`, `mount --bind` operations.

There exists a project [heroku-buildpack-nix-proot](https://elements.heroku.com/buildpacks/ocharles/heroku-buildpack-nix-proot), which is a buildpack implementing exactly that. Though it is unmaintained since years and does not work.

After fixing up the script and updating the proot binary, proot still crashes complaining about PTRACE not being available.
It seems like proot once worked inside the heroku stack, but later heroku modified the stacks environment, disabling access to the PTRACE system call. Therefore, proot is not an option as well.

## 2. Changing the nix store location
By setting `NIX_STORE_PATH`, the nix store can be moved to a different location than `/nix`. It could be moved to a location inside the `$CACHE_DIR`, for which the heroku stack allows access. Though this invalidates all caches hosting `/nix/store` based paths including `cache.nixos.org`.

To mitigate long build times, one could run their own build/caching infrastructure hosting artifacts built against the alternative nix store location.

## Integration via docker
Heroku allows [deployment via docker](https://devcenter.heroku.com/categories/deploying-with-docker) which seems like the simplest way to integrate nix.

Suitable base images:
  - [nixos/nix](https://hub.docker.com/r/nixos/nix): consists of many layers which slows down fetching during heroku build
  - [nix-docker-base](https://github.com/teamniteo/nix-docker-base): Images cotain one specific nixpkgs, nixpkgs need to pinned correctly.
  - [nixery.dev/nix/bash](https://nixery.dev/): minimal image containing just a shell and nix. Requires some small adjustments in order to make `nix build` work.
