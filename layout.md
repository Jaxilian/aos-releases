# The cores and the aos partition

Decided 2026-10-06. What a disk holds, what boots it, and what a person
sees. Supersedes the slot description in `usr/lib/aos/slot.sh` for 0.1.x.

## Partitions

| # | Label  | What                                                       |
|---|--------|------------------------------------------------------------|
| 1 | --     | 1 MiB, empty: held the BIOS loader until systemd-boot (2026-10-20, UEFI only); kept so the slots keep their numbers |
| 2 | AOS_ESP| The EFI system partition: shim and systemd-boot (`EFI/BOOT`), the loader's configuration (`loader/loader.conf`), one unified kernel image per slot (`EFI/Linux/aos-<version>-<slot>[+N[-M]].efi`), and per slot its os-release and verity parameters as text (`/aos/a`, `/aos/b`) |
| 3 | aos-a  | Core A: a squashfs image with its dm-verity hash tree appended. Raw, no filesystem of its own. |
| 4 | aos-b  | Core B, the same                                            |
| 5 | aos    | The OS a person sees: btrfs on a fresh install since round 136, ext4 on a disk installed before (LUKS2 around either when encrypted). |

A core is immutable: every block the kernel reads from it is checked
against the hash tree, whose root hash the slot's unified kernel image
carries on its command line (`aos.hash=`; the initramfs opens the slot
whose core matches it). A core that was tampered with, or written half
way, does not mount; the loader's boot counting (below) then falls back
to the confirmed slot. Nothing writes into a core, so the kernel is no
longer an apm package: a kernel update is an OS update.

## The aos partition

```
aos/
  aos/        what the OS writes: etc/ (the overlay over the core's /etc)
              and var/ (the whole /var), and the swap file
  apm/        the package store: /opt/apm
  users/      the homes: /home
```

The initramfs mounts the partition at /aos and binds from it: `/aos/aos/var`
on /var, an overlay of `/aos/aos/etc/{upper,work}` on /etc, `/aos/apm` on
/opt/apm, `/aos/users` on /home. /tmp and /run are tmpfs. The Unix tree
is still there for every program that needs it; Files shows /aos as "AOS"
with those three folders, and a person's own files are under users/.

On the live ISO /aos is a tmpfs with the same three directories: the live
home is writable now, and the installer copies the core, not a tree.

### btrfs (round 136, 2026-11-10)

A fresh install makes the partition btrfs (`mkfs.btrfs -L aos`); a disk
installed before stays ext4 and nothing converts it (DECISIONS.md
2026-11-03): the initramfs asks blkid what the partition holds and
mounts either, ext4 exactly as before (`e2fsck -p`, `mount -t ext4`). btrfs is built into the kernel
(`CONFIG_BTRFS_FS=y`, with POSIX ACLs), since the initramfs has no
modules and no modprobe.

- **Subvolumes**: `apm` and `users` are subvolumes of the top level;
  `aos/` (etc, var, the swap file) stays a plain directory in it. The
  paths are the ones ext4 had, so the mounts and binds are unchanged
  and nothing names a subvolume (no `subvol=`). A directory cannot
  become a subvolume later without a copy, and a subvolume is what a
  snapshot takes: the store (a package update to roll back) and the
  homes (a backup) are the two worth one. Nothing snapshots yet. The
  swap file is kept out of both, since btrfs refuses to snapshot a
  subvolume holding an active swap file.
- **Mount options** (the initramfs and the installer): `noatime`, so a
  read writes no metadata (copy-on-write makes every atime update a
  new metadata block, and snapshots would pin the old ones);
  `compress=zstd:1`, the cheapest zstd level, for the store and /var,
  which compress to about half and are read far more than written;
  btrfs skips data that does not compress, and the homes' LUKS images
  are random bytes and NOCOW anyway (homed sets it), as is the swap
  file. No `discard=async` on the line: the kernel turns it on by
  itself for a device that can discard (6.2 and later); behind LUKS
  the mapping passes no discards, as with ext4.
- **No fsck at boot**: btrfs checks its own checksums on every read and
  replays its log at mount; `btrfs check` is a repair tool, not a boot
  step.
- **The swap file** is `btrfs filesystem mkswapfile` (NOCOW, not
  compressed, preallocated), not dd: a copy-on-write file is refused by
  swapon. It is extra swap behind zram on an encrypted partition, not a
  place for a hibernation image: there is none (power.md).
- **Homes** stay LUKS2 images with ext4 inside (`--storage=luks` is
  given everywhere a home is made: homed's own default on a btrfs
  /home would be a plain subvolume, in the clear).

## Homes

Every account made since 0.3 is a systemd-homed one: its home is a
LUKS2 image, `/home/<name>.home` under `users/`, ext4 inside, sparse on
the disk (discard on and off line), unlocked by the account's password
at login -- pam_systemd_home ahead of pam_unix in greetd's, login's,
aos-root's (aos-rootd's unlock), the lock screen's and system-auth's stacks, with a pam_unix
line before it (result ignored) that asks for and caches the typed
password: without one cached the module first asks homed with no
password, homed counts that as a failed attempt and is busy rewriting
the record when the real one follows at once, which is how a greeter or aos-rootd answers -- and locked again when the last session ends.
homed opens the home's LUKS device and at once marks it for deferred
removal, so it goes with the unmount; the `/dev/mapper/home-<name>`
node must stay meanwhile, since every later homework run (a password
check against the active home, the rebalance) looks for it. That is
udev's symlink, which is why libdevmapper is built with udev
synchronisation (the devmapper-udev package, 0.3.1): the copy Buildroot
builds before systemd makes and removes the node itself, and removed
it at the deferred removal. The record is in `/var/lib/systemd/home`,
not in `/etc/passwd`; the greeter and Settings list accounts through
the C library (nsswitch: `files systemd`), and aos-update's passwd
merge never sees them. The owner is made at the first boot by
`aos-firstboot` (`homectl create --storage=luks --fs-type=ext4`, the
first free uid from 1000), since homed must be running; the installer
makes no account at all. A **recovery key** (homed's, eight groups of
eight letters) is made with the home and shown once by the first-boot
setup, written nowhere; typed where the password goes, it opens the
home. A classic account (`aos-install --demo`, `--user` for the
drivers, a machine installed before 0.3) keeps working through
pam_unix; its home is a plain directory.

## Encryption

**On by default where a TPM can open it** (round 136, DECISIONS.md
2026-11-10): from the live medium onto a disk that is not removable,
on a machine with a TPM 2.0 (`/dev/tpmrm0`), `aos-install` without
`--encrypt` or `--no-encrypt` encrypts all the same, with a recovery
key it makes (64 modhex letters, enrolled without separators so it
types the same on every layout, printed once at the end in eight
groups of eight joined by spaces, `Recovery key: ...`, and typed
without the spaces). "Can open it" is
checked the way the live medium can: the installer cannot unseal the
key itself, since the policy opens it only in the initramfs of a
signed core UKI and the live medium is neither (secure-boot.md, "The
TPM key"). So the check is the enrolment: `systemd-cryptenroll` must
succeed and the token it wrote must carry `tpm2-pcrs` [7] and, on a
build with the AOS PCR key, `tpm2_pubkey_pcrs` [11] (read back with
`cryptsetup luksDump --dump-json-metadata`; `tpm_enroll` and
`tpm_bound` in `/usr/lib/aos/tpm.sh`). If either fails the header is
wiped and the partition stays in the clear, and the installer says so;
`aos-encrypt` can encrypt it later. Without a TPM the default is off,
as before. The graphical
installer's box follows the same test: ticked when `/dev/tpmrm0` is
there, and then it asks no passphrase but makes the recovery key
itself and shows it on its Done page; unticked it passes
`--no-encrypt` (through aos-rootd). Writing a stick for another
machine from an installed system never encrypts by default: the TPM
would be this machine's.

`aos-install --encrypt FILE` (the graphical installer's "Also encrypt
the whole disk", with a passphrase of its own) puts LUKS2 around the
aos partition before the filesystem: programs, settings and logs are
encrypted too, and the homes twice; the cores are not (public images,
verified). On a machine with a TPM 2.0 the installer seals a second
key to it (`tpm_enroll` in `/usr/lib/aos/tpm.sh`: `systemd-cryptenroll
--tpm2-pcrs=7 --tpm2-public-key=... --tpm2-public-key-pcrs=11`,
bound to PCR 7, the Secure Boot state, and to PCR 11 through a policy
signed with the AOS PCR key: docs/secure-boot.md, "The TPM key"; PCR 7
alone on a build without that key). aos-encrypt, `aos-tpm enroll` and
`aos-tpm migrate` (from `aos-update --confirm`, with Secure Boot on
only) seal with the same helper, and
the initramfs opens the partition with it (`systemd-cryptsetup attach
... tpm2-device=/dev/tpmrm0,tpm2-signature=...`) without a word; the
passphrase is the recovery key, asked on the console when the TPM will
not answer -- the disk in another machine, the firmware's Secure Boot
state changed, no TPM at all -- five tries, with the keymap named on
the ESP (`aos/keymap`, secure-boot.md, "The prompt's keyboard")
loaded first. With Secure Boot off PCR 7
does not change with what booted, so the TPM key protects a disk taken
out of its machine, not a machine whose boot path was swapped; with it
on, only the initramfs of a UKI signed with the AOS keys unseals it. Turning Secure Boot on, or enrolling the
AOS key, can change PCR 7: the next boot may ask for the passphrase
once (then `aos-tpm` seals it again). tpm-test.py drives both boots in QEMU
with swtpm (AOS_TPM=1). The cores carry nothing of yours; the ESP only
the name of the console keymap.
The swap file
(`aos/aos/swapfile`) is made by the installer only on an encrypted
partition: in the clear it would hold pages of the encrypted homes;
zram is the swap otherwise. There is no hibernation (power.md).

## The greeter

An owned machine (and one installed for someone else, once its owner
exists) boots to a login screen: greetd runs `/usr/lib/aos/session
--greeter` as the `greeter` account -- the compositor with ade-greeter
as its one program (the repository `greeter`), which lists the accounts,
takes a password and asks greetd over its socket to start
`/usr/lib/aos/session` as that account: the compositor and the shell as
before, with the apm environment, started again after a crash up to
four times in two minutes. The lock screen later in the session is
ade-lock, as it was. The live medium and a demo install keep
ade.service's autologin; `/etc/greetd/config.toml` in the /etc overlay
and the unit's enablement are what aos-install writes for the others,
and aos-firstboot rewrites after the first boot. greetd's PAM service
(`/etc/pam.d/greetd`) is the console's with pam_systemd, so both the
greeter and the session are logind's on seat0.

## Boot

```
firmware -> EFI/BOOT/BOOTX64.EFI: shim (signed build) or systemd-boot
         -> systemd-boot (EFI/BOOT/grubx64.efi, the name shim loads)
         -> EFI/Linux/aos-<version>-<slot>[+N[-M]].efi: the slot's UKI
            (systemd-stub + kernel + microcode + initramfs + command line)
         -> initramfs: veritysetup open the core, mount it, mount aos
            (LUKS open first when encrypted; e2fsck first on ext4), bind
            the tree, switch_root
         -> systemd
```

With Secure Boot on every arrow but the last two is a signature check
(docs/secure-boot.md). The loader shows no menu: holding Space while
the machine starts lists the entries (the UKI's profiles -- the serial
console, safe graphics, no kiosk -- and the other slot).

**Which slot boots** is the file name (systemd-boot's boot counting,
`slot.sh`): `aos-0.3.30-b+3.efi` is a slot an update staged, with three
tries; each start uses one (`+2-1`, `+1-2`, `+0-3`); a boot where ade
presented a picture and a person logged in confirms it (`aos-update
--confirm`, run by aos-update-confirm.service, upgrading.md): the name
loses its counters and the other
slot's entry is set aside as `+0-3`, listed but never the default. A
slot that fails three times ends as `+0-3` too, and the confirmed slot
boots. `aos-update --rollback` gives the other slot's entry one try and tells
the loader to take it (`bootctl set-oneshot`, by the entry's id, the
name without counters); a trial with a picture and a login confirms
itself, one without is set aside at the following reset and the
good slot boots. The initramfs tries first the slot
the loader named (`LoaderEntrySelected`), but the hash decides.

The initramfs is built by post-build.sh from the target's own binaries
(`board/aos/initramfs/`): dash, the coreutils multicall binary,
util-linux, veritysetup, cryptsetup, systemd-cryptsetup, e2fsck, and
nothing else. The command line it reads:

| Parameter        | Meaning                                                    |
|------------------|------------------------------------------------------------|
| `aos.core=`      | `PARTUUID=<uuid>` of the slot, or `live` (the medium's `/aos/core.img` through a loop device) |
| `aos.hash=`      | the verity root hash of that core                           |
| `aos.offset=`    | where the hash tree starts in the core, in bytes            |
| `aos.live=`      | on the live medium: the device (or `PARTUUID=`) holding `/aos/core.img` |
| `aos.data=`      | `PARTUUID=<uuid>` of the aos partition; absent on the live ISO |
| `aos.core=auto`, `aos.data=auto` | a signed UKI's: slots 3/4 of the disk the stub was loaded from, the one whose verity matches; partition 5 (written to `/run/aos/boot-args`) |

## Images and releases

Buildroot makes `rootfs.squashfs` (zstd) inside its fakeroot; post-image.sh
appends the hash tree (`veritysetup format`) and writes `core.img` with
`core.verity` beside it (`hash=`, `offset=`, `size=`), then assembles the ISO from
`/boot` and that image. A release ships `aos-X-x86_64.iso`,
`aos-X-x86_64-core.img` (a squashfs is compressed already) and
`aos-X-x86_64-core.verity` and the signed unified kernel image
`aos-X-x86_64-core.efi`, named in the signed SHA256SUMS. aos-update
writes the image into the idle slot with dd, checks it by opening its
verity, writes the slot's os-release and verity parameters to the ESP,
and stages the UKI (it must carry the core's hash) as the slot's entry
with three tries (upgrading.md).

A 0.1.x installation cannot update into this layout: its updater knows
tarballs and ext4 slots. It is a reinstall (`./usb.sh`), once.
