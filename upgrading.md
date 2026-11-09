# Upgrading

## The kernel

The kernel version is pinned in
`br2ext/configs/aos_x86_64_defconfig`:

```
BR2_LINUX_KERNEL_CUSTOM_VERSION=y
BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE="7.2.12"
BR2_PACKAGE_HOST_LINUX_HEADERS_CUSTOM_7_1=y
```

Pinned rather than `BR2_LINUX_KERNEL_LATEST_VERSION`, which moves whenever
Buildroot is upgraded: a release has to build the same kernel next year as
it did today. To move to a newer kernel, change the value -- and the
headers series on the third line if the major.minor changed, unless the
one chosen is already the newest Buildroot knows (its prompt then says
"7.1.x or later", and a custom kernel's headers are checked loosely, so
7.2.9 builds against the 7_1 choice; 2026-10-03). That third line is not
optional. The toolchain headers follow the kernel, and with a
custom version Buildroot must be told which series; without it glibc drops
out of the configuration silently (`grep BR2_TOOLCHAIN_BUILDROOT_GLIBC
.config` comes back empty) and the build fails hours later in elfutils.

Then rebuild the kernel and the image:

```sh
make BR2_EXTERNAL=$PWD/br2ext aos_x86_64_defconfig
grep BR2_TOOLCHAIN_BUILDROOT_GLIBC .config    # must be =y
make linux-dirclean
make
```

Two things to watch. Buildroot's kernel patches and `linux-firmware` are tested
against the version it ships, so a hand-picked kernel is less well tested. And
the NVIDIA modules must support it -- 610.57.04 builds against 7.x, but a much
newer kernel may need a newer driver.

### Shipping it

The kernel is part of the core (docs/layout.md): a core is an immutable
image checked block by block, so nothing can put a kernel or a modules
directory into it afterwards. A kernel change ships as an OS update --
`release.sh --publish`, then Software's Upgrade system -- and the
previous kernel is the previous slot. (0.1.x had the kernel as an apm
package, `kernel-apkg.sh`; that road is closed.)

## Channels

`release.sh --publish` puts every release on **edge**: the `index`
release of apm-recipes, the URL in `/usr/lib/aos/update.conf`, which is
what every machine follows unless told otherwise. **stable** is the
release `stable` of the same repository, which holds what
`release.sh --promote stable` copied there from edge last -- a release
after its time on edge with no fault found, promoted without a rebuild.
A machine follows it with `CHANNEL=stable` in `/etc/aos/update.conf`
(a copy of the core's file with that line; the file replaces the
core's whole), which is what Software's Update page writes when
"Follow the stable channel" is switched on, and removes when it is
switched off. `aos-update --check` names the channel and the URL it
reads. A channel is the last part of the URL, so a local server for
the drivers works the same way (`URL=http://host/index`, `CHANNEL=x`
reads `http://host/x`).

## What an update changes, and what needs a reinstall

The rule since 0.3.3: a machine installed once is never reinstalled for
a release. What a release can change, and how it reaches the machine:

| Part | Where it lives | How a release changes it |
|---|---|---|
| Programs, libraries, the kernel, its modules, the initramfs, firmware | the slot (core + its ESP directory) | aos-update writes the other slot; the next boot runs it, a failed boot falls back |
| Kernel arguments, the boot menu | the slot's unified kernel image (its command line and profiles, post-image.sh) | with the slot: the new release boots with its own arguments the first time, a rollback with its old ones |
| The loader (shim, systemd-boot, `loader/loader.conf`) | the ESP | `boot-sync`, run by `aos-update --confirm` once the release reached its desktop: each file where it differs from the release's `/usr/lib/aos/efi`, the first stage last |
| Defaults in `/etc` the machine never touched | the core's `/etc` under the overlay | with the core, at once |
| Files in `/etc` the installer or first boot wrote (greetd's config, fstab, vconsole, drop-ins) | the overlay's upper layer, which hides the core's | a script in `/usr/lib/aos/migrate.d`, run once at the first boot of the release (`aos-migrate.service`, `/var/lib/aos/migrated`) |
| New system users and groups | `/etc/passwd` and friends in the overlay | aos-update appends the names the new core has and the machine lacks |
| The TPM key of an encrypted disk | the LUKS header | `aos-tpm enroll` on the running machine |
| Kiosk mode on an owned machine | greetd's config, the kiosk account | `aos-kiosk on "COMMAND"` / `off` (a Settings switch is to come) |
| Accounts, homes, apm packages, settings | the aos partition | never touched by an update |

A machine updated from 0.3.2 gets all of this at its first confirmed
boot of 0.3.3: the confirm runs the new release's aos-update, which runs
boot-sync.

**Still a reinstall** (each is a decision to take before the alpha, not
a hidden trap):
- The partition layout: slots of 6 GiB (the core is 0.8 GB), an ESP of
  512 MiB (two slots' kernel and initramfs are about 80 MB). Both have
  room for years; changing them means repartitioning.
- A home made by a release before homed (0.2.x): classic homes are not
  converted.

## Base OS updates

An installed disk has two root slots, each holding a core: a squashfs
with its dm-verity hash tree ([layout.md](layout.md) describes the
disk). `aos-update` moves a machine to a newer release:

```sh
apm upgrade       # every package, then the OS itself: what Software's "Upgrade system" runs (aos-rootd runs it)
apm --progress upgrade  # the same, with "::pkg", "::dl" and "::os" lines Software reads for its Update page
aos-update --check     # "AOS 0.3.0 is available (this is 0.2.0)", or that it is the newest
aos-update             # fetch, verify, write the idle slot; boots at the next restart
aos-update --rollback  # the other slot -- what ran before -- boots at the next restart
```

`apm upgrade` with no package named runs `aos-update` after the packages
(`--dry-run` runs `--check`), so one command, or one button, moves the
whole machine, the way a distribution's upgrade does; the OS part is
skipped where there is no `aos-update` (a developer's machine, the live
ISO). A package that fails -- a third-party recipe that will not build,
say -- is reported and skipped, the rest are installed, and the OS update
still runs; apm then exits 1 with "N package(s) failed".

An update interrupted at any point -- the power, the lid, Settings
closed -- leaves the machine booting what it runs now: the idle slot's
loader entry is removed before the slot is touched and a new one lands
only when the slot is complete, and a slot without the file a complete
write leaves last (`/boot/efi/aos/<slot>/verity.cfg`) has no entry and
is one `aos-update --rollback` refuses to boot. The next update simply
writes the slot again.

One update runs at a time: a second `aos-update` (or an `apm upgrade`
beside one) says "another update is already running" and leaves, and
Software's button says "Upgrading..." while one is working. Everything an update
printed is kept in `/var/log/aos-update.log`, on the data partition, so
a failed one can be read afterwards -- from another machine too, since
that partition is readable with the stick plugged in.

It fetches `SHA256SUMS` and `SHA256SUMS.minisig` from the URL in
`/usr/lib/aos/update.conf` -- `/etc/aos/update.conf` overrides it when it
exists, the /etc overlay being writable where the core is not -- (the
apm package index's release in apm-recipes,
where `release.sh --publish` puts them), checks the signature against the
OS release keys in `/usr/share/aos/os-keys` -- the core's own directory,
apart from apm's keyring (publishing.md) -- downloads
the core image the file names
(`aos-<version>-x86_64-core.img`, about 750 MB, uncompressed beyond the
squashfs's own zstd) and its verity parameters (`-core.verity`), checks
their sha256 (and the release's unified kernel image, `-core.efi`,
which must carry the core's hash), writes the image into the idle slot
and reads it back through its verity (a block that does not match its
hash is an I/O error), appends any system account the new release adds
to the overlay's `passwd`/`group`, writes the slot's os-release and
verity parameters to `/boot/efi/aos/<slot>/`, and puts the UKI on the
ESP as `EFI/Linux/aos-<version>-<slot>+3.efi`: a loader entry with
three tries, which `bootctl set-oneshot` makes the next boot's choice.
`aos-update-confirm.service`, twenty seconds after the desktop's
service is up (the login screen's, greetd, or the demo's autologin,
ade) and if it has not restarted meanwhile, confirms the running slot:
its entry loses the counters, the other slot's entry is set aside
(`+0-3`). A slot that never reaches that point uses up its tries and
is set aside by the loader itself, and the confirmed slot boots. The
hidden boot menu (hold Space) lists the other slot's entry by hand, and
`--rollback` does the same from the running system.
Nothing on the data partition -- accounts, `/home`, installed programs,
settings -- is touched by any of it. Software's Update page runs the same
command, one button for the packages and the OS, with a line per package
that says where it is (downloading with a percentage, installing, done,
failed with the reason) and a Restart button when the OS part is done.
Two minutes after a boot, and daily after that, `aos-update-check.timer`
refreshes the indexes and writes what waits to `/var/lib/aos/updates`;
the desktop shows a toast, and a click on it opens that page.

A disk installed by 0.1.x (ext4 slots, a tarball updater, the data
partition called aos-data) cannot be updated into the cores; reinstall
it. `aos-install` wipes the disk.

## Kernel options

Driver and feature choices live in
`br2ext/board/aos/linux.config.fragment`, applied on top of the mainline
`x86_64_defconfig`. Add a line, then:

```sh
make linux-reconfigure && make
```

Always confirm the option survived — the mainline defconfig or another symbol
can override it:

```sh
grep CONFIG_YOUR_OPTION output/build/linux-*/.config
```

## The NVIDIA driver

Two files, and the version must match in both:

- `br2ext/package/aos-nvidia-open/aos-nvidia-open.mk` — kernel modules
- `br2ext/package/aos-nvidia/aos-nvidia.mk` — userspace and GSP firmware

Change `_VERSION`, update the `.hash` files with the new checksums, then:

```sh
make aos-nvidia-dirclean aos-nvidia-open-dirclean && make
```

The GSP firmware must end up under `/lib/firmware/nvidia/<version>/`. The
kernel module builds that path from its own version string, so a mismatch
means the module loads and the GPU never initialises.

## Rust

`br2ext/package/aos-rust/aos-rust.mk`. Change `AOS_RUST_VERSION`, update
`aos-rust.hash` (upstream publishes a `.sha256` next to each tarball), then
`make aos-rust-dirclean && make`.

## gcc and glibc

These come from Buildroot itself, chosen in the defconfig
(`BR2_GCC_VERSION_15_X=y`). Changing gcc means rebuilding the whole toolchain
and everything above it — effectively a full rebuild. Note that `aos-gcc`
deliberately reuses whatever version Buildroot uses, so the compiler on the
target always matches its own runtime libraries.

## Buildroot itself

The root of this repository is a Buildroot release tree, imported as a single
commit with no upstream history behind it. Everything AOS adds lives under
`br2ext/`, and outside of it the tree is untouched — the diff against the
import commit is one file, `.gitignore`. That makes an upgrade a clean swap:
delete everything except `br2ext/`, and put a newer release in its place.

Buildroot versions are dates, not a running count. `2026.05.1` is the May 2026
release plus its first point update. There is a new release every three months
— `.02`, `.05`, `.08`, `.11` — and point releases on each branch for a while
after. A version that looks old usually just means this tree has not been
touched in a while, not that the project stopped.

Which version you are on:

```sh
grep '^export BR2_VERSION :=' Makefile
```

### One-time: add the upstream remote

`origin` is the AOS repository, so it has nothing to do with Buildroot. There
is nothing to pull from until you add it:

```sh
git remote add buildroot https://gitlab.com/buildroot.org/buildroot.git
git fetch buildroot --tags
```

Then see what is out, ignoring the release candidates:

```sh
git tag -l '20*' | grep -v rc | sort -V | tail
```

Two kinds of upgrade are worth telling apart. Moving along the branch you are
already on — `2026.05.1` to `2026.05.2` — is security and bugfix only: no new
kernel, no new toolchain, and only a few packages rebuild. Moving to a new
quarterly release — `2026.05` to `2026.08` — brings a new kernel, Mesa, LLVM
and usually a new gcc, which means a full rebuild.

### Doing the swap

Commit or stash anything under `br2ext/` first; this deletes files.

```sh
git ls-files -z | grep -zv '^br2ext/' | xargs -0 rm -f
git checkout 2026.08 -- .
git add -A -- . ':!br2ext'
git commit -m "buildroot 2026.08"
```

The delete is the part people skip. `git checkout <tag> -- .` restores every
file that release has, but it will not remove files that release dropped —
without the delete you keep dead packages and stale `.mk` files that the new
`Config.in` no longer mentions.

The `:!br2ext` on `git add` keeps whatever you have in progress under
`br2ext/` out of the Buildroot commit. `.gitignore` comes from upstream — it
ignores `output/`, `dl/` and `.config` like ours did, plus a few more — so
there is nothing of ours to preserve there.

`br2ext/` is the only thing in the tree that is ours. If that ever stops
being true and you patch Buildroot itself, this recipe throws the patch away
without saying so. Keep any such change as a commit you can replay on top,
or better, move it into `br2ext/`.

### Rebuilding afterwards

Reload the defconfig against the new tree and check it survived:

```sh
make BR2_EXTERNAL=$PWD/br2ext aos_x86_64_defconfig
grep BR2_LEGACY .config
```

That grep must come back empty. Options Buildroot renamed select `BR2_LEGACY`,
and `Makefile.legacy` turns that into a hard error on the next `make`, so those
you cannot miss. The kind you *can* miss is the one in
building.md: an option whose dependencies stopped being met, or
one deleted with no legacy entry, is dropped in silence. Spot-check the ones
that matter:

```sh
grep -E 'BR2_PACKAGE_MESA3D_LLVM|BR2_PACKAGE_MESA3D_VULKAN|BR2_GCC_VERSION' .config
```

Re-check our own packages against the new package infrastructure too:

```sh
./utils/check-package --br2-external br2ext/package/*/*
```

Then build. A quarterly upgrade nearly always moves gcc or glibc, which
invalidates the toolchain and everything above it, so start from clean:

```sh
make clean
make
```

`make clean` wipes `output/` but leaves `dl/` and `.config` alone, so packages
whose versions did not change are not downloaded again. Expect the same 2 to 4
hours as a first build. Avoid `make distclean` unless you mean it — it also
removes `dl/` and `output/.br2-external.mk`, so you lose the download cache and
have to pass `BR2_EXTERNAL=` again.

A point release within a branch does not need `make clean`. Read that
release's entry in `CHANGES`, dirclean the handful of packages whose versions
moved, and let the rest stand.

### What does not come along

An upgrade only moves what Buildroot ships. The NVIDIA driver and the Rust
toolchain are our packages, pinned in `br2ext/package/`, and stay exactly where
they were — use the sections above to move them. After a kernel bump, confirm
the NVIDIA modules still build against the new kernel; that is the piece most
likely to break.
