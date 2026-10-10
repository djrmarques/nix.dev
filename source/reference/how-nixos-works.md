# How Nix Os works

This article contains all information for users to understand their NixOs systems.
To keep it consice, some familiarity with linux is assumed.
If you already have NixOS installed, we encourage you to follow along with the commands presented here.

## Acronyms

- (FHS) Filesystem Hierarchy Standard [^fhs]

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

As an example, here I am listing all versions that I have on my system from the [mmb](https://codeberg.org/djrm/mmb) package (installed via `nix-shell`):

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

All paths in there start with an hash, which is computed out of all the dependencies that are using to build it.
I can then run my package

```sh
❯ /nix/store/5dmgcvl5075yi87xv64zamjiprq1fjnp-mmb-0.6.4.0/bin/mmb --help
mmb - a static blog generator
```

## Which packages is my current system using

The latest section may bring up a question: "So how do I know which version is my current system using"

Let's see where our `ls` comes from

```sh
> which ls
/run/current-system/sw/bin/ls
```

This does not point to the `nix/store`, however `/run/current-system/sw/bin/ls` is just a symlink that point to the store

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

## Profiles, Users and Generations

Anytime you run

## Deleting unused packages

## What does `nixos-rebuild` do?

## How does `shell.nix` work?

## References

- https://nix.dev/tutorials/module-system/
- https://nixos.org/manual/nixos/stable/
- https://nixos.org/guides/how-nix-works/
- https://nix.dev/manual/nix/2.24/package-management/

[^fhs]: https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard
[^hash]: https://nixos.org/guides/how-nix-works/
