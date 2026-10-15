# How Nix Os works

This article contains all information for users to understand their NixOs systems.
To keep it consice, some familiarity with linux is assumed.
If you already have NixOS installed, we encourage you to follow along with the commands presented here.
Keep in mind the results you will get from running commands on your machine will differ from the ones on the page.

## Acronyms

- (FHS) Filesystem Hierarchy Standard [^fhs]

## Concepts
- Nix Store: Folder where all packages are installed on Nix OS, located at `nix/store`
- Derivation: Specification on how to build a package
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

`/run/current-system` is itself a symlink to the `/nix/store`

```sh
❯ readlink -f /run/current-system/
/nix/store/svjpwfiqik42g7r14aiwk0ja8q16r3ch-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
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
From the previous section we learned that at boot the  `/run/current-system/sw/` folder is symlinked to the nix store, and this defines what packages we have "activated" in our current session.
But how does this happen?

When the system boots, the bootloader selects a generation, and that generation is symlinked into `/run/current-system/sw/`.
Now, what is a generation? Before we get there, we need to learn about profiles.

A profile is a collection of activated packages for a given user.

Let's take a look at `/nix/var/nix/profiles`

```sh
❯ ls /nix/var/nix/profiles
system -> system-22-link
system-17-link -> /nix/store/3w1xp7387mywl1kkplcscg8bncnnxllg-nixos-system-nixos-26.11pre1075591.e554fab72f81
system-18-link -> /nix/store/lxpycjgcrzmw2k7xx46703pfbaqvqv91-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
system-19-link -> /nix/store/2cjic09qwy9zanyg2ivvqjaf4g3s15rc-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
system-20-link -> /nix/store/x9s6ghn6xcvn8v6rd45ab9djjmcjhgl2-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
system-21-link -> /nix/store/r6sshn490la8a55qyxqrfanrsmcl91jv-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
system-22-link -> /nix/store/svjpwfiqik42g7r14aiwk0ja8q16r3ch-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
```

A few interesting things here.
We have a `system` folder that symlinks to `/nix/var/nix/profiles/system-22-link`.
`system-22-link` is generation 22, which in my case is the one that I selected at my bootloader.

We can can also see that our `/run/current-system` point to the same place as out generation.

```sh
❯ readlink -f /run/current-system
readlink -f /nix/var/nix/profiles/system
readlink -f /run/booted-system

/nix/store/svjpwfiqik42g7r14aiwk0ja8q16r3ch-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
/nix/store/svjpwfiqik42g7r14aiwk0ja8q16r3ch-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
/nix/store/svjpwfiqik42g7r14aiwk0ja8q16r3ch-nixos-system-nixos-26.11pre1087755.e7439b6b14ad
```

But what is inside out generation?

```sh
❯ ls /nix/store/svjpwfiqik42g7r14aiwk0ja8q16r3ch-nixos-system-nixos-26.11pre1087755.e7439b6b14ad

activate
bin
boot.json
dry-activate
etc -> /nix/store/n7kvw2x5vzl88k9ahc488vwgzw0alivw-etc/etc
extra-dependencies
firmware -> /nix/store/ava9558fa42h4gj5i8i2y150zcql0nz0-firmware/lib/firmware
init
init-interface-version
initrd -> /nix/store/8rhswp998yq8li7jqwslblpanmd91lh4-initrd-linux-6.18.55/initrd
kernel -> /nix/store/l55iycmwfsvc9nrb97z9f1cr3dykjqvx-linux-6.18.55/bzImage
kernel-modules -> /nix/store/fm0fkr4pjyp2ravap5mlj5r1lfvcfc9p-linux-6.18.55-modules
kernel-params
nixos-version
prepare-root
specialisation
sw -> /nix/store/klzpfsz9ab64j7cmzbn5mwvgzxmnm6ni-system-path
switch-inhibitors -> /nix/store/dhs8pdmdkymcakndj12k2gbhcv08camg-switch-inhibitors
system
systemd -> /nix/store/l5hiab3932k2n4d6nirh09mk6xgxp61w-systemd-261.3
```
It's pretty much our full system, including the kernel, systemd and the firmaware.

One final thing to point out, there is an activate shell script.

```sh
❯ cat /nix/store/svjpwfiqik42g7r14aiwk0ja8q16r3ch-nixos-system-nixos-26.11pre1087755.e7439b6b14ad/activate | head
#!/nix/store/10dxp0qxqxxsyiljrh2kp0xqhz6arhcx-bash-5.3p15/bin/bash

export PATH="$PATH"

source /nix/store/mhxn5kwnri3z9hdzi3x0980id65p0icn-lib.sh

systemConfig='/nix/store/svjpwfiqik42g7r14aiwk0ja8q16r3ch-nixos-system-nixos-26.11pre1087755.e7439b6b14ad'

export PATH=/empty
```

We will get back to this script when we talk about `nixos-rebuild`

## Package deletion and garbage collection
We saw that we can have multiple versions of the same package, and even multiple versions of our entire system on the nix store.
Does anything ever get deleted? Will I run out of space eventually?

By default, no package will be deleted from your system. 
There is however, a usefull nix command to clean old packages on our system.

```sh
❯ nix-collect-garbage --dry-run
finding garbage collector roots...
removing stale link from "/nix/var/nix/gcroots/auto/iayfmjbw4xcl6m8jywg2kwasvscqiz6i" to "/tmp/nix-build-20425-1680266481/result"
removing stale link from "/nix/var/nix/gcroots/auto/xzncs5518bmmj97cdww2wvrr12273b64" to "/home/mangosteen/Projects/blog/.direnv/nix-tmp-profile.12190-1-link"
removing stale link from "/nix/var/nix/gcroots/auto/r3605aw618kz99ha2gjnvnwk83y4rma1" to "/tmp/nix-build-110713-555384285/result"
removing stale link from "/nix/var/nix/gcroots/auto/3nsqphfn61c9nbzd8ai1yhgy57b0yy41" to "/home/mangosteen/gump-result"
removing stale link from "/nix/var/nix/gcroots/auto/9g9n3rw27kvzi4wdq5m3l36v0bwfsdgj" to "/tmp/nix-build-110684-213946974/result"
removing stale link from "/nix/var/nix/gcroots/auto/g09gx196pdhanmn75ynn56yfisaxvygc" to "/home/mangosteen/Projects/blog/.direnv/nix-tmp-profile.58130-1-link"
removing stale link from "/nix/var/nix/gcroots/auto/nskhvgbh1zc0zhlnby9gv09xzbgn7ai8" to "/tmp/nix-build-113714-502215337/result"
removing stale link from "/nix/var/nix/gcroots/auto/dq0ji34wg6vp6iy46x93317xw1xm29yh" to "/tmp/nix-build-38343-3776282476/result"
removing stale link from "/nix/var/nix/gcroots/auto/b0mr1hfkrr4r07q78x2mdrqy0bnz00kp" to "/home/mangosteen/Projects/blog/.direnv/nix-tmp-profile.28232-1-link"
removing stale link from "/nix/var/nix/gcroots/auto/yc0iwzp3d06f6qnyia2ypfh4wdxhml6i" to "/tmp/nix-build-113743-2185180946/result"
removing stale link from "/nix/var/nix/gcroots/auto/1fajklyzrllq13p9jv9xhagwlh459jnq" to "/tmp/nix-build-20454-751589217/result"
removing stale link from "/nix/var/nix/gcroots/auto/3gxl5im8vggg4y0liz5lxkv7lwwg7fp1" to "/home/mangosteen/Projects/blog/.direnv/nix-tmp-profile.53191-1-link"
removing stale link from "/nix/var/nix/gcroots/auto/wy43kpgn9w4ihvsi8gma9qrs4s8sffmx" to "/home/mangosteen/Projects/blog/.direnv/nix-tmp-profile.19173-1-link"
removing stale link from "/nix/var/nix/gcroots/auto/1wcjd7mix5c73b61glghjazf320kbzbv" to "/home/mangosteen/Projects/blog/.direnv/nix-tmp-profile.24416-1-link"
removing stale link from "/nix/var/nix/gcroots/auto/ld42hnv1kp3vhk922i2kn0jywnp7qmxb" to "/tmp/nix-build-38316-1659243550/result"
determining live/dead paths...
472 store paths would be deleted
```

## What does `nixos-rebuild` do?

## How does `shell.nix` work?

## References

- https://nix.dev/tutorials/module-system/
- https://nixos.org/manual/nixos/stable/
- https://nixos.org/guides/how-nix-works/
- https://nix.dev/manual/nix/2.24/package-management/
- https://nixos.org/guides/nix-pills/11-garbage-collector.html

[^fhs]: https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard
[^hash]: https://nixos.org/guides/how-nix-works/
[^drv]: https://nixos.org/guides/nix-pills/06-our-first-derivation.html#digression-about-drv-files
