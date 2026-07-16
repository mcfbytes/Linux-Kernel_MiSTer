# This tree is generated

It is a **build output**, not a source of truth. It was rendered from
[Buildroot_MiSTer](https://github.com/mcfbytes/Buildroot_MiSTer) by
`scripts/export-kernel-tree.sh`, which is where the kernel is actually maintained.

**Changes made directly to this tree will be erased by the next regeneration.**
To change the kernel, change the patch series in Buildroot_MiSTer
(`board/mister/de10nano/linux-patches/`) and regenerate.

## What is here

| | |
|---|---|
| Base | Pristine Linux 6.18.38 from kernel.org, hash-verified (``) |
| On top | 31 commits, one per carried MiSTer patch, original authorship preserved |
| Config | `arch/arm/configs/MiSTer_defconfig` — copied verbatim from linux.config |
| Tag | `mister-6.18.38` |

The base commit contains no MiSTer change, so

    git diff mister-6.18.38~32 mister-6.18.38

is exactly the MiSTer delta against upstream and nothing else.

## Building standalone

    make ARCH=arm MiSTer_defconfig
    make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- LOCALVERSION= zImage modules
    ./build-mister-modules.sh

Three details on that middle line, each of which will bite you otherwise.

**`LOCALVERSION=`** — empty, but set. This tree is a git repo whose HEAD is ~35 commits
past the `v6.18.38` base, so `scripts/setlocalversion` correctly calls the source
modified and appends `+`, giving `6.18.38+`. Buildroot builds the same source from a
tarball with no git around it, so its equally-patched kernel reports plain `6.18.38`.
That `+` lands in **vermagic**, and vermagic is what the kernel matches on when loading
modules:

    vermagic=6.18.38+ SMP mod_unload ARMv7 p2v8     <- without LOCALVERSION=
    vermagic=6.18.38  SMP mod_unload ARMv7 p2v8     <- Buildroot, and here WITH it

Mismatch it and modprobe rejects every module. With it, this tree's kernel and modules
are interchangeable with the shipped image's.

**`modules`** — not just `zImage`. External modules link against the kernel symbol
table in `Module.symvers`, which modpost writes during `make modules` (and which needs
`vmlinux` first). `modules_prepare` does **not** produce it, and without it modpost
calls every kernel symbol undefined
(`ERROR: modpost: "skb_pull" [8812au.ko] undefined!`) — which looks like a broken driver
and is not. `build-mister-modules.sh` checks for this and says so.

**The third line at all** — the Xbox (xone) and 11ac WiFi drivers are out-of-tree, so
`zImage` never builds them.

Building the kernel also needs `lz4` on the host, since this config sets
`CONFIG_KERNEL_LZ4`.

## Vendored out-of-tree drivers

| path | pin | build override |
|---|---|---|
| `drivers/net/wireless/realtek/rtl8812au` | 8cac6f43316a56cc89cc8cb532cd6c6ae14c4805 | CONFIG_RTL8812AU=m |
| `drivers/net/wireless/realtek/rtl8814au` | b1866ce2b857a8dfe2e147e19eb8eca0a842ce18 | CONFIG_RTL8814AU=m |
| `drivers/net/wireless/realtek/rtl8821au` | 3a7cdb591b64d99d2670e455bde67c8ab338525b | CONFIG_RTL8821AU=m |
| `drivers/hid/xone` | f2aa9fe01103d7600553b505b298ff0bd47ff280 | — |

Sources are verbatim upstream at the paths `MiSTer-v5.15` uses, so the layout matches.
They are deliberately **not** wired into Kconfig. Their own Makefiles do parse-time work
keyed off `$(shell pwd)`:

    export TopDIR ?= $(shell pwd)
    $(shell cp $(TopDIR)/autoconf_..._linux.h $(TopDIR)/include/autoconf.h)

In an in-tree build `pwd` is the kernel root rather than the module directory, so that
copy lands in the wrong place and the driver's generated `autoconf.h` never appears —
silently, because `$(shell ...)` swallows the error. These Makefiles assume they are
always built out-of-tree. `build-mister-modules.sh` therefore uses upstream's own
supported out-of-tree path, which is also exactly what Buildroot invokes to build the
shipped image — so it is a proven recipe rather than a workaround.

The pins live in `package/<name>/<name>.mk` in Buildroot_MiSTer. Bumping a driver is a
pin change there plus a re-run of the export; nothing here needs rewiring.

Unlike `MiSTer-v5.15`, which vendors these in-tree, that means a driver bump does not
touch this tree's history by hand — and the Realtek drivers here track upstreams that
build against 6.18 with **zero** compatibility patches.

## Where this branch hangs

This repo's tarball commits form a spine, and each `MiSTer-vX.Y` branch hangs off a
spine point with the MiSTer series replayed on top. This branch extends that spine the
same way, so it is the next entry rather than a foreign import:

    e12ed6c19 v5.13.12 -> 137491a75 v5.14 -> b6f2ca1c4 v5.14.5 -> aba1ef4c1 v5.15.1
                                                                       |
                                          +----------------------------+
                                          |
       [112 MiSTer commits] -> MiSTer-v5.15        (untouched)
                                          |
       v6.18.38 -> [31 MiSTer commits] -> MiSTer-v6.18

`MiSTer-v5.15` is **not modified and not an ancestor** — it is a sibling, exactly as
`MiSTer-v5.14` already is. Nothing was lost.

Two consequences worth knowing:

- The base commit's parent is itself a pristine tarball commit, so
  `git diff aba1ef4c1 v6.18.38` is the **pure upstream 5.15.1 → 6.18.38 delta**,
  with no MiSTer code on either side.
- No MiSTer-5.15 commit appears in this branch's log, which is the point: this tree
  does not contain most of them, and a log listing changes that are absent from the
  tree would be worse than no log at all.

What each 5.15 commit became — carried, superseded by an upstream commit (with the
vanilla commit cited), or deliberately dropped — is recorded per commit in
`MISTER-KERNEL-PATCH-RECON.md` in Buildroot_MiSTer. No git command can answer that:
across this much context drift `git patch-id` matches nothing, so "is this commit in
6.18.38?" is a semantic question, not a mechanical one.

## Publishing

This script never touches a remote. To publish, fetch the orphan branch into a fork
and push from there:

    git -C <your-fork> fetch <this-export-dir> MiSTer-v6.18:MiSTer-v6.18
    git -C <your-fork> push origin MiSTer-v6.18

## Fork sync point

Reconciled against `MiSTer-devel/Linux-Kernel_MiSTer` at commit `794e6f002`.

Commits added to the fork since then have **not** been triaged for backporting:

    git log --oneline 794e6f002..MiSTer-v5.15
