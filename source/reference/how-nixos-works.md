# How Nix Os works

This article contains all information for users to understand their NixOs systems.
To keep it consice, some familiarity with linux is assumed.
If you already have NixOS installed, we encourage you to follow along with the commands presented here.
Keep in mind the results you will get from running commands on your machine will differ from the ones on the page.

## Acronyms

- (FHS) Filesystem Hierarchy Standard [^fhs]

## Concepts
- Nix Store: Folder where all packages are installed on Nix OS, located at `nix/store`
- Derivation: Specification on how to build something
- Generation: A snapshot of your running system

## Diferences between Nix OS and a "regular" distro

The table below gives a quick overview of the differences between Nix OS and

|                          | A "regular" distribution                                              | NixOS                                                                                         |
| ------------------------ | --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **Filesystem layout**    | Follows the FHS                                                       | Does not follow the FHS. `/lib` and `/usr/lib` do not exist. Everything lives on `/nix/store` |
| **Package Location**     | Packages are unpacked into shared system directories (eg. `/usr/bin`) | Every package is placed in its own immutable path on `/nix/store`.                            |
| **Installing Packages**  | You run an imperative command such as `apt install`                   | You add a package to a configuration file and run `nixos-rebuild switch`;                     |
| **System configuration** | You edit files under `/etc` by hand                                   | A single declarative configuration describes the system;                                      |
| **Upgrades**             | An upgrade mutates files in place; old versions are overwritten       | Each rebuild creates a new _generation_. Previous generations remain available                |
| **Rollback**             | There is no general rollback; you restore from backups or reinstall   | You can return to any previous generation                                                     |

## The Nix Store
Any time you install a package (via `nixos-rebuild` or `nix-shell`), everything is added to `nix/store`.

As an example, here I am listing all versions from the [mmb](https://codeberg.org/djrm/mmb) package on my system (installed via `nix-shell`):

```sh
❯ ls /nix/store | grep -E '^[a-z0-9]{32}-mmb-[0-9]'
0jnqj8rwwdyx7vlh4gvjdisxxaq7y3vm-mmb-0.1.0.0
1cgh0581nnl5j9fglrdikkr6m9jrj6b6-mmb-0.6.4.0
3zhjvpnw09p062zv5rp86qxkqgr0zqvw-mmb-0.6.3.0
409b2q9v061h6l4kh5d2846mpjw8b3j5-mmb-0.6.3.0.drv
5dmgcvl5075yi87xv64zamjiprq1fjnp-mmb-0.6.4.0
5sclw4sk02zrzs4mcpzpiqbm47qzxghy-mmb-0.6.2.0
6lkmlyfmb1w4m8wmj9rwfzgr771s7s8c-mmb-0.6.4.0.drv
avmanvc9bg63mg771y7p7xvvsm1j8ny9-mmb-0.1.0.0.drv
clzf3yyyaf22lqb1110ns181abmd2frh-mmb-0.6.4.0.drv
ih8bds8591ia91vb767hjz6yiyb41bdh-mmb-0.5.1.0
iy8gjf9kvy2dn94qhqwisypm4imvp4w2-mmb-0.6.2.0.drv
n6s59pj8yi03is1vi6dz3az64plpn7lj-mmb-0.2.0.0
nzx3dq1zcbb0dhwv1gnbqsyfb7q01yxx-mmb-0.5.1.0.drv
p3pibv1gf4msnbph6kvj2whymyy67ghz-mmb-0.2.0.0.drv
qzv483jpxfnwhhzhsqcsfp5klr494blx-mmb-0.6.1.0.drv
```

All paths start with an hash, which is computed out of all the dependencies that are using to a particular package.[^hash].

We can see that we have files with and without the `.drv` extention.
`.drv` files contain the instructions to build a particular derivation[^drv].
Files without `.drv` are the result of building a derivation.

```sh
❯ ^ls /nix/store/n6s59pj8yi03is1vi6dz3az64plpn7lj-mmb-0.2.0.0
bin  lib  nix-support
```

We can run any package on the nix store

```sh
❯ /nix/store/5dmgcvl5075yi87xv64zamjiprq1fjnp-mmb-0.6.4.0/bin/mmb --help
mmb - a static blog generator
```

It would get very tiring having to type the full path anytime you want to run a program.
Fortunately, you don't have to, as we will see in the following section.

## Which package versions is my current system using

The latest section brings up a few questions:

- How can I run commands without typing the full store path?
- Which version of a package is my current system using?

To answer this, let's take a look at where `ls` comes from

```sh
> which ls
/run/current-system/sw/bin/ls
```

This does not point to the `nix/store`, however `/run/current-system/sw/bin/ls` is just a symlink that points to the store

```sh
❯ ls -al /run/current-system/sw/bin | grep " ls "
lrwxrwxrwx 1 root root    70  1. Jan 1970  ls -> /nix/store/lnirw7a51gnlhnrakivzg11vh4h5kszy-coreutils-full-9.11/bin/ls
```

Interestingly, the `/run/current-system` is itself a symlink to the `/nix/store`

```sh
❯ readlink -f /run/current-system/
/nix/store/lxpycjgcrzmw2k7xx46703pfbaqvqv91-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
```

The `/run` folder is a temporary filesystem written in RAM that does not survive reboots[^fhs].
While it's not a NixOS specific concept, it serves a very important purpose: It describes all the software that is running on the current system `/run/current-system` and how the system was booted `/run/booted-system`.
The current system and booted system nix store paths should match right after reboot, but become different after the first switch.

Nix Os uses the `/run/sw/`  folder to store all the packages for the current env.

```sh
❯ ^ls /run/current-system/sw/
bin  etc  lib  sbin  share
```
`/run/current-system/sw/` contains folders that you would expect on your root filesystem on a Linux Distro. But how is this set?

## Profiles, Users and Generations
From the previous section we learned that at boot the  `/run/current-system/sw/` folder is symlinked, and this defined what packages we have available in our current session.
But how does this happen?

When the system boots, the bootloader selects a generation, and that generation is symlinked into `/run/current-system/sw/`.
Now, what is a generation? Before we get there, we need to learn about profiles.

A profile is a collection of activated packages for a given user.

Let's start by taking a look at `/nix/var/nix/profiles`

```sh
❯ ls  /nix/var/nix/profiles
default -> /nix/var/nix/profiles/per-user/root/profile
per-user
system -> system-21-link
system-17-link -> /nix/store/3w1xp7387mywl1kkplcscg8bncnnxllg-nixos-system-nixos-26.11pre1075591.e554fab72f81
system-18-link -> /nix/store/lxpycjgcrzmw2k7xx46703pfbaqvqv91-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
system-19-link -> /nix/store/2cjic09qwy9zanyg2ivvqjaf4g3s15rc-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
system-20-link -> /nix/store/x9s6ghn6xcvn8v6rd45ab9djjmcjhgl2-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
system-21-link -> /nix/store/r6sshn490la8a55qyxqrfanrsmcl91jv-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
```

A few interesting things here. We have a default folder, that symlinks to `per-user/root/profile` which itself symlinks to `nix/store`

```sh
❯ ls  /nix/var/nix/profiles/per-user/root
channels -> channels-9-link
channels-8-link -> /nix/store/v1ibrr8f5z6a8wrw27a6s2g23g8n7cr0-user-environment
channels-9-link -> /nix/store/akx35ch27g21zm5k0yvghwgkz7vs7m0w-user-environment
```

The default folder

## Package deletion

## What does `nixos-rebuild` do?

## How does `shell.nix` work?

## References

- https://nix.dev/tutorials/module-system/
- https://nixos.org/manual/nixos/stable/
- https://nixos.org/guides/how-nix-works/
- https://nix.dev/manual/nix/2.24/package-management/

[^fhs]: https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard
[^hash]: https://nixos.org/guides/how-nix-works/
[^drv]: https://nixos.org/guides/nix-pills/06-our-first-derivation.html#digression-about-drv-files
