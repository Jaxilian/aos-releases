# Secure Boot

Decided 2026-10-14. How AOS starts on a laptop with Secure Boot on --
which is how most laptops are sold, and switching it off is beyond most
people -- without a Microsoft signature of our own and without asking
anyone to manage keys. docs/layout.md is the disk and boot design this
builds on.

## The chain

```
firmware (Microsoft UEFI CA 2011 in db, as shipped)
  -> EFI/BOOT/BOOTX64.EFI  shim 16.1, Fedora's, signed by Microsoft     [firmware checks]
  -> EFI/BOOT/grubx64.efi  systemd-boot, signed with the AOS key         [shim checks, MOK]
                           (the name is shim's default loader; the file is systemd-boot since 2026-10-20)
  -> EFI/Linux/aos-<version>-<slot>[+N[-M]].efi  the slot's UKI, signed  [shim's loader protocol, MOK]
       systemd-stub + kernel + microcode + initramfs
       + the command line: aos.core=auto aos.hash=<root hash> ...
       + .sbat: aos.uki,<generation> (revocable by a shim of our own)
  -> initramfs: the slot whose dm-verity matches aos.hash             [verity, every block]
  -> the core, read-only, verified block by block -> systemd
```

Each arrow is a signature check except the last two, which are a hash
the signed UKI carries. Nothing outside the signed files decides what
runs: `loader/loader.conf` and the entries' names are on the unsigned
ESP, but with Secure Boot on they only choose *which* signed UKI
starts, and the UKI's stub ignores any command line a loader passes
(systemd-stub does so whenever Secure Boot is on). A UKI changed by one
byte, a bare kernel, an unsigned initramfs -- all refused
(secureboot-test.py tamper). With Secure Boot off the same loader boots
the same UKIs; nothing else is on the ESP to boot.

### Why this chain

- **A distribution's shim, not our own.** Getting shim signed for AOS
  means a reviewed shim build (github.com/rhboot/shim-review), an EV
  code-signing certificate and a Microsoft partner account, months and
  money. Fedora's signed shim is BSD-licensed and redistributable as
  it is; it trusts Fedora's own certificate (built in) and whatever the
  machine's owner put in the MOK list. That second door is what MOK is
  for, and the user opens it once (below). Fedora's RPM is GPG-checked
  and pinned by hash (`package/aos-shim`).
- **systemd-boot, since 2026-10-20 (GRUB before).** GRUB was kept at
  first because the A/B slots lived in grubenv and per-slot menus; it
  was the largest parser of unsigned input in the chain, GPLv3, and the
  one open CVE in the triage. systemd-boot's boot counting carries the
  slot choice in the entries' names (docs/layout.md "Which slot
  boots"), it is built by the systemd the image already has (with the
  SBAT section external.mk's meson options give it), and it loads only
  the UKIs under EFI/Linux. It is installed under the name shim's
  default loader has, grubx64.efi, so Fedora's signed shim needs no
  change. Legacy BIOS boot went with GRUB (the ISO keeps GRUB's El
  Torito image for BIOS firmware alone; an installed disk is UEFI only).
- **A UKI per slot, not a signed kernel.** With GRUB + a signed kernel
  the initramfs and the command line would be unsigned: `init=`,
  `rdinit=` or another initramfs would boot with the same PCR 7 and
  unseal a TPM-bound disk. The UKI makes kernel, initramfs and the
  verity root hash one signed thing. Its command line cannot name the
  disk's partitions (they are made at install time), so it says
  `aos.core=auto aos.data=auto`: the initramfs takes slots 3 and 4 of
  the disk whose ESP the stub was loaded from (systemd-stub's
  `LoaderDevicePartUUID`), opens the one whose verity matches the hash,
  and partition 5 is the aos partition; it writes what it found to
  `/run/aos/boot-args` for slot.sh and aos-install. GRUB saves
  `booted=<slot>` in grubenv before it chainloads, and the initramfs
  tries that slot first: only an order (two slots holding the very same
  core, as update-test.py makes, would both match), the hash decides. The other menu
  entries (serial console, safe graphics, no kiosk) are profiles of the
  same UKI, `@1`..`@3`, which the stub accepts with Secure Boot on.
- **osslsigncode** signs (a host package; the build host has no
  sbsign or pesign, and systemd's own systemd-sbsign would need a second
  host build of systemd with OpenSSL). ukify (host-systemd) builds the
  UKIs from the stub that `BR2_PACKAGE_SYSTEMD_BOOT` builds; external.mk
  gives that stub its SBAT section (meson turns the autodetection off
  when cross-building).

## What is signed, with what

| File | Where | Signed by | Made by |
|---|---|---|---|
| shimx64.efi, mmx64.efi | core `/usr/lib/aos/efi`, ESP `EFI/BOOT/{BOOTX64,mmx64}.efi` | Microsoft (for Fedora) | package/aos-shim |
| grubx64.efi (systemd-boot) | core `/usr/lib/aos/efi`, ESP `EFI/BOOT/` | AOS key | post-build.sh |
| aos-sb.cer | core `/usr/lib/aos/efi`, ESP `ENROLL_THIS_KEY_IN_MOKMANAGER.cer` | -- (the key's certificate) | post-build.sh |
| core.efi (disk UKI) | release asset `aos-X-x86_64-core.efi`, ISO `/aos/core.efi`, ESP `aos/<slot>/aos.efi` | AOS key | post-image.sh |
| live.efi (live UKI) | ISO `/boot/live.efi` | AOS key | post-image.sh |

The UKI is not in the core: it holds the core's root hash, which exists
only once the core is made. It travels beside it -- on the ISO, as a
release asset in SHA256SUMS (signed with the apm key), and aos-update
fetches it when the release lists it and checks it names the core's
hash before it stages it.

## The user's one enrollment

A stock laptop, Secure Boot on, the AOS stick in:

1. Firmware starts shim (Microsoft trusts it); shim does not know the
   AOS key: **"Verification failed: (0x1A) Security Violation"**, OK.
2. MokManager, the blue screen: any key, then **Enroll key from disk**,
   the volume **AOS_ESP**, the file **ENROLL_THIS_KEY_IN_MOKMANAGER.cer**
   (first in the list), Continue, **Yes**, Reboot.
3. From then on the machine starts anything signed with the AOS key:
   the live medium, the installed system, every update. No password,
   no setting changed; the key sits in shim's MOK list in the
   firmware's variables.

That is the whole cost, once per machine; install.md should show the
six screens (the screendumps of secureboot-test.py enroll). A machine
installed with Secure Boot off gets the key queued by the installer
instead: `/usr/lib/aos/sb-enroll` runs `mokutil --import` with an
eight-digit one-time password and prints it (the installer window's
Done page shows it, since 0.3.22: before, it went by in a log nobody
saw); at the next start
MokManager asks "Enroll MOK", Continue, Yes, the password, Reboot.
Afterwards Secure Boot can be turned on at any time. The same command
by hand (`/usr/lib/aos/sb-enroll` as root, on a development build) on a machine updated from an
older release. With Secure Boot on it says the key is enrolled and does
nothing -- it must be, or nothing would have started.

Some machines (Secured-core PCs: Surface, some ThinkPads) ship with the
Microsoft *third-party* UEFI CA turned off; there shim itself is refused
and the firmware's "Allow Microsoft 3rd Party UEFI CA" has to be on.
The G14 test machine is in Setup Mode (no platform key), which means
Secure Boot cannot be on until its firmware's factory keys are restored.

## Updates and machines installed before this

- aos-install puts the core's own loader on the ESP (`slot_install_efi`
  in slot.sh: shim + signed GRUB + MokManager + the certificate, or the
  unsigned GRUB alone on a build without the key) instead of running
  grub-install for UEFI; the BIOS GRUB is as before.
- **From 0.3.10**: its aos-update writes the new core and stages the
  kernel/initramfs as always (it does not know core.efi); the new
  release boots with Secure Boot off through 0.3.10's GRUB, and at its
  confirm boot-sync replaces grub-install's loader with the release's
  (only files that differ; BOOTX64.EFI last, so a cut-off write leaves
  the old loader whole). The next update, by the new aos-update, also
  stages core.efi. Until then that slot has no UKI: with Secure Boot on
  it does not start, and GRUB says so (turn it off, or update once).
- `grub.version` is no longer written by boot-sync: an older release
  rolled back to compares its grub-install against it, finds it equal,
  and leaves the signed loader in place.

## The TPM key: PCR 7 and a signed PCR 11 policy

Until round 136 the installer, aos-encrypt and aos-tpm ran
`systemd-cryptenroll --tpm2-device=auto` with no PCR option. Since
systemd 258, the version AOS ships, that binds to no PCR at all
(its NEWS: "The new default PCR mask for new TPM2 enrollments is thus
empty"), not to PCR 7 as these pages said: any system on the same machine, a live stick or
another distribution, could unseal the key. Now every enrolment
(aos-install, by default or with `--encrypt`; aos-encrypt; `aos-tpm
enroll` and `migrate`) goes through `tpm_seal` in
`/usr/lib/aos/tpm.sh` (by `tpm_enroll`, or `tpm_migrate` for
`migrate`):

    systemd-cryptenroll --tpm2-device=/dev/tpmrm0 --tpm2-pcrs=7 \
        --tpm2-public-key=/usr/lib/systemd/tpm2-pcr-public-key.pem \
        --tpm2-public-key-pcrs=11 DEV

(`--tpm2-pcrs=7` alone on a build whose core has no PCR public key),
and then reads the token back (`tpm_bound`: every `systemd-tpm2`
token object in `cryptsetup luksDump --dump-json-metadata`, read with
jq, must have `tpm2-pcrs` [7] and, with the public key,
`tpm2_pubkey_pcrs` [11] and a `tpm2_pubkey` that is the shipped PEM;
nothing else in the header counts); a token that does not is wiped
again and the enrolment counts as failed. No enrolment is made after
a rescue shell in the boot (`/run/aos/pcr-capped`, below): its PCRs
are not the machine's. That read-back is also the
installer's whole test of its encryption default (layout.md,
Encryption): the live medium cannot unseal the key it just made, by
design. The key opens only when both hold:

- **PCR 7** is what it was at enrolment: Secure Boot's state and the
  certificate that verified the chain (shim, the AOS MOK key).
- **PCR 11** has a value signed with the AOS PCR key. systemd-stub
  measures the UKI's sections (kernel, initramfs, command line,
  os-release, the chosen profile) into PCR 11; the initramfs then adds
  the word `enter-initrd` (systemd-pcrextend) just before it asks the
  TPM. post-image.sh signs, for each profile of core.efi, the PCR 11
  value of exactly that state (`ukify --pcr-private-key
  --pcr-public-key --phases=enter-initrd`), into the UKI's `.pcrsig`;
  the stub hands it to the initramfs as
  `/.extra/tpm2-pcr-signature.json`, which init passes to
  systemd-cryptsetup (`tpm2-signature=`).

Right after the TPM attempts (the unseal and its debug retry), before
the ESP is mounted, the keymap loaded, the passphrase asked, any fsck
or any filesystem mounted, init extends PCR 11 with `leave-initrd`:
from then on no signature matches, so nothing parsed after it (a
passphrase-opened volume, btrfs, vfat), neither the running system's
root nor a shell in the initramfs, can unseal the key again. If the
TPM refuses that word, init extends PCR 11 with
`aos-leave-initrd-failed` instead, which moves it off the signed value
as well; if it refuses both, init says so and goes on to the prompt
(never stranding the person; TODO G19 with the final extend). When
both `enter-initrd` and `leave-initrd` went in, init writes
`/run/aos/pcr-phases-ok`, which `aos-tpm migrate` requires.

`fail` (a reason and a shell) never restarts the machine: an open
encrypted partition is first `cryptsetup luksSuspend`ed (its key leaves
the kernel; `cryptsetup luksResume aos` asks for the passphrase or the
recovery key before 'exit' carries on), PCR 11 is moved off as above,
PCR 7 gets `aos-rescue-shell` (no PCR-7 token can unseal in that shell),
`/run/aos/pcr-capped` is written and `pcr-phases-ok` removed, and then
the shell comes. Before round 136 an open disk or a refused extend
restarted the machine every 30 s, a loop with no way in. A new release is a new kernel and initramfs, a
new PCR 11 value and a new signature by the same key: updates need no
re-enrolment.

live.efi is not PCR-signed: the live medium carries no signature, so
its root cannot unseal an installed disk even with PCR 7 the same
(tpm-test.py tries it with the installed UKI's signature and
`enter-initrd` extended by hand). A Fedora kernel through our shim
fails PCR 7 and PCR 11. With Secure Boot off the stub accepts a
command line from the loader (`rdinit=/bin/sh`), and a shell could
then extend `enter-initrd` itself: PCR 7 is what stops that, since
Secure Boot off is a different PCR 7 than the one enrolled with it on.
A disk enrolled with Secure Boot off has only the "taken out of its
machine" protection, as before.
PCR 7 also moves with a firmware or dbx update (systemd's reason for
dropping it as a default): the next boot asks for the passphrase once,
and `aos-tpm enroll` (Settings) seals the key again.

**The prompt's keyboard.** `/etc/vconsole.conf` is on the encrypted
partition, so the initramfs cannot read it before the prompt. The name
of its keymap goes on the ESP instead, as `aos/keymap` (one line,
`[A-Za-z0-9_-]`, 32 characters at most): aos-install writes it with the
layout it was given, boot-sync (at every confirm) copies it from
vconsole.conf, so a layout changed in Settings reaches the prompt from
the boot after the next. The initramfs carries kbd's `loadkeys`, gzip
and the i386 keymaps; it mounts the ESP only after the TPM attempts and
`leave-initrd`, and loads the named map before asking. The ESP is not
authenticated: someone who can write it can only pick another shipped
keymap. The recovery key is modhex letters (`cbdefghijklnrtuv`) only,
the same keys on every Latin layout, so it types right even when the
keymap is wrong; it is shown in groups of eight and typed without the
spaces.

**The PCR key.** `aos-pcr.key` (RSA 2048, mode 600) and `aos-pcr.pem`
(its public half) beside the Secure Boot key in `~/AppData/aos.release/sb`
(`$AOS_SB_KEYS`), never in git or the image. post-build.sh copies the
`.pem` into the core as `/usr/lib/systemd/tpm2-pcr-public-key.pem`;
post-image.sh signs with the key and refuses a target whose public key
is not that one; release.sh refuses a build when the key is on the
host but the core does not ship its `.pem`, or core.efi lacks one
`.pcrsig` per profile or carries another `.pcrpkey`. Once the key
exists, its fingerprint is committed as
`br2ext/board/aos/keys/aos-pcr.fingerprint` (`sha256sum < aos-pcr.pem`,
the hash alone): from then on release.sh demands the `.pem`, the
`.pcrpkey` with that hash and a `.pcrsig` per profile whether or not
the private key is on the host, so a build from a host without it can
never ship and leave every sealed disk at its passphrase. Without the
key the build binds to PCR 7 alone. Made once, on the build host:

```sh
cd ~/AppData/aos.release/sb && umask 077
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out aos-pcr.key
openssl pkey -in aos-pcr.key -pubout -out aos-pcr.pem
```

(The build's `output3/host/bin/openssl` when the host has none.) A
lost key means the next release cannot sign: every TPM key asks for the
passphrase once and `aos-tpm enroll` binds it to the new key. A leaked
key lets its holder sign their own UKI's PCR 11, but PCR 7 still needs
their UKI to boot through shim, i.e. signed with the Secure Boot key.

**Disks enrolled before.** `aos-tpm migrate` rebinds a TPM token that
`tpm_bound` does not accept. It is not a boot service: `aos-update
--confirm` runs it once the running slot is confirmed, so a release
that fails to boot never changes the key and the slot before it keeps
opening. It changes nothing, exit 0 and one line saying why, unless:
Secure Boot is on (with it off any system started on the machine sees
the same PCR 7, so binding to it adds no protection and only a
recovery-key prompt at every firmware or dbx update); init wrote
`/run/aos/pcr-phases-ok` this boot and no rescue shell capped the
PCRs; and, on a build with the PCR key, this boot's UKI carried a
signature (`/run/aos/tpm2-pcr-signature.json`) and every UKI on the
ESP, the other slot's too, carries the AOS PCR key, so a rollback still
opens without the recovery key. Then it seals the new token unlocking
with the old one (`--unlock-tpm2-device`: a pre-136 token has no PCRs
and a PCR-7-only token keeps opening after the initramfs), checks the
new token with `tpm_bound`, and only then wipes the old ones by slot
number; a new token that is not right is wiped and the old stays
(exit 1). Nothing is asked of the person.

A disk sealed with Secure Boot off (aos-install, aos-encrypt, `aos-tpm
enroll`) is still bound to PCR 7 (and the signed PCR 11), which adds
the firmware-update prompt without adding protection while Secure Boot
stays off; whether such an enrolment should bind no PCR until Secure
Boot is turned on (and `migrate` then binds it) is the owner's call
(round 136).

## What remains open (the threat model's gap)

What this stops: an attacker with the disk but not the AOS key cannot
make the machine run their kernel, initramfs, command line or core
while Secure Boot is on; a TPM-sealed aos partition (PCR 7 and a
signed PCR 11, above) unseals only in the initramfs of a UKI signed
with the AOS keys. What it does not:

- **Downgrade.** An older signed UKI and its core are still signed.
  Someone who can write the disk can put a vulnerable older release
  back. Fix later: an SBAT-like generation in the UKI (`.sbat`
  `aos.uki,N`) bumped when a release fixes something serious, or the
  TPM policy on PCR 11 with systemd-measure's signed policies.
- **Fedora's door.** shim also trusts Fedora's certificate: anything
  Fedora signed boots on a machine with our shim, as it would with
  Fedora's own. PCR 7 records which certificate verified the image, so
  a Fedora kernel does not unseal an AOS disk's TPM key; it can still
  boot and read an unencrypted aos partition (as any live medium can
  with Secure Boot off).
- **Downgrade, the fix's half.** Every UKI carries `aos.uki,N` in its
  .sbat section (post-image.sh) since 2026-10-20; Fedora's shim carries
  Fedora's SbatLevel, so a lower N is not refused until a shim of our
  own ships (docs/product-impl.md R4).
- **Modules: signatures enforced.** Every module in the image is
  signed with the kernel's build key, the NVIDIA ones included (the
  kernel's build signs external modules at install), and the UKI's
  command line carries `module.sig_enforce=1`.
- **Lockdown, integrity mode, under Secure Boot** (round 136). The
  kernel carries `patches/linux/0002-lockdown-...`: with
  `CONFIG_LOCK_DOWN_IN_EFI_SECURE_BOOT` the lockdown LSM raises itself
  to integrity when the EFI stub saw Secure Boot on
  (`boot_params.secure_boot`, which also honours shim's MokSBState), the
  way Fedora's and Ubuntu's kernels do. The same signed UKI boots with
  Secure Boot off, and then lockdown is `[none]`: a machine whose owner
  turned Secure Boot off has given up the chain anyway, and a
  `lockdown=` on the command line would lock that owner out of what
  they chose. Under it even root cannot change the running kernel:
  unsigned modules, /dev/mem and /dev/port, MSR writes, ioperm/iopl,
  kexec_load, raw PCI BAR and config writes through sysfs, ACPI table
  and custom-method overrides, writable debugfs files,
  bpf_probe_write_user, unsafe module parameters. Integrity mode leaves
  perf, tracing, kprobes and BPF reads alone (those are confidentiality
  mode's). Hibernation is refused too, and AOS dropped it (power.md).
  `/sys/kernel/security/lockdown` reads `none [integrity]
  confidentiality`; `lockdown136-test.py` checks it.
- **The firmware's own CA.** Fedora's shim 16.1 is signed by the
  Microsoft UEFI CA 2011. Microsoft signs new shims with its 2023 CA;
  a machine whose db has only the 2023 CA (new ones, later) needs a
  shim signed with it -- a newer Fedora shim RPM, a version bump in
  package/aos-shim.
- **Legacy BIOS boot** and Secure Boot off are outside this entirely.
- **The key.** Everything rests on `aos-sb.key`. Whoever has it can
  sign code every enrolled machine runs.

## The company's keys

Three files sign everything, one certificate is built into the shim,
and none of them is in git. They live on the build host in
`~/AppData/aos.release/sb/` (or `$AOS_SB_KEYS`), directory 700; until
0.4.77 it was `~/.config/aos-sb`, where sb.sh still finds the keys and
says to move them:

| File | What | Where it goes |
|---|---|---|
| `aos-root.key` | the root CA's key, offline (passphrase; a token later) | nowhere: signs `aos-sb.crt` once every three years |
| `aos-root.crt` / `.cer` | `CN=Adelon AB Secure Boot Root CA`, CA:TRUE, ten years | `.cer` is the shim's vendor certificate (`package/aos-shim` OWN, `br2ext/shim/`); `.crt` lets `sb_sign` check the chain |
| `aos-sb.key` | the signing key (file, or on a PKCS#11 token: below) | signs systemd-boot and every UKI |
| `aos-sb.crt` / `.cer` | `CN=Adelon AB Secure Boot Signing 2026`, codeSigning, three years, issued by the root | `.crt` goes into each signature; `.cer` into the image for MokManager and mokutil, as today |

A shim built with the root as its vendor certificate trusts anything
signed under that root: the signing certificate can be renewed (a new
`aos-sb.*` under the same root, every three years, or at once if the
signing key leaks) without a new shim or a new review. Only the root
is forever: its key stays off the build host.

### Making them

The root, on an offline machine (a live medium with no network is
enough), the key passphrase-protected:

```sh
mkdir -m 700 aos-sb && cd aos-sb && umask 077
openssl req -new -x509 -newkey rsa:2048 -sha256 -days 3650 \
  -subj "/CN=Adelon AB Secure Boot Root CA/O=Adelon AB/C=SE" \
  -addext basicConstraints=critical,CA:TRUE \
  -addext keyUsage=critical,keyCertSign,cRLSign \
  -keyout aos-root.key -out aos-root.crt
openssl x509 -in aos-root.crt -outform DER -out aos-root.cer
```

The signing certificate under it (the same machine, the same session;
the CSR can come from a token instead, below):

```sh
openssl req -new -newkey rsa:2048 -sha256 -nodes \
  -subj "/CN=Adelon AB Secure Boot Signing 2026/O=Adelon AB/C=SE" \
  -keyout aos-sb.key -out aos-sb.csr
cat > aos-sb.ext <<'EOF'
basicConstraints=critical,CA:FALSE
keyUsage=critical,digitalSignature
extendedKeyUsage=codeSigning
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid
EOF
openssl x509 -req -in aos-sb.csr -CA aos-root.crt -CAkey aos-root.key \
  -CAcreateserial -sha256 -days 1095 -extfile aos-sb.ext -out aos-sb.crt
openssl x509 -in aos-sb.crt -outform DER -out aos-sb.cer
openssl verify -CAfile aos-root.crt aos-sb.crt      # must say OK
chmod 644 aos-root.crt aos-root.cer aos-sb.crt aos-sb.cer
```

To the build host go `aos-root.crt`, `aos-root.cer`, `aos-sb.key`,
`aos-sb.crt`, `aos-sb.cer` (into `~/AppData/aos.release/sb/`); `aos-root.key`
stays offline, two copies (an encrypted stick in a drawer, and
another drawer). RSA 2048 is what shim, MokManager and every
distribution's MOK use; shim does not check validity dates.

**Replacing the self-signed key** (`aos-sb.*` made with the recipe
of 0.3.10-0.3.3x, `CN=AOS Secure Boot 2026`): the file names stay, so
`post-build.sh` and `post-image.sh` change nothing; put the new five
files in place of the old three and `make`. Machines that enrolled the
old certificate in their MOK list need the new one too: ship one
release whose `sb-enroll` queues `aos-sb.cer` while everything is still
signed with the old key, then sign with the new (the rotation below).
With our own shim the MOK step disappears: the root is in the shim.
`sb_sign` (board/aos/sb.sh) refuses to sign as soon as `aos-root.crt`
is beside the key and `aos-sb.crt` does not chain to it -- the loader
it signed would be refused by the shim shipped next to it.

For QEMU rounds without the company's keys, the old self-signed recipe
still works and nothing checks a chain (no `aos-root.crt`):

```sh
openssl req -new -x509 -newkey rsa:2048 -nodes -sha256 -days 3650 \
  -subj "/CN=AOS Secure Boot 2026/O=AOS" -addext extendedKeyUsage=codeSigning \
  -addext keyUsage=digitalSignature -keyout aos-sb.key -out aos-sb.crt
openssl x509 -in aos-sb.crt -outform DER -out aos-sb.cer
```

### The signing key on a token (`AOS_SB_PKCS11`)

The signing key never has to be a file. With it on a PKCS#11 token (a
YubiKey's PIV slot, any HSM OpenSC drives), `sb_sign` signs through
osslsigncode's PKCS#11 path when `AOS_SB_PKCS11` is set to the key's
PKCS#11 URI:

```sh
export AOS_SB_PKCS11='pkcs11:token=YubiKey;object=aos-sb'
export AOS_SB_PKCS11_MODULE=/usr/lib64/pkcs11/opensc-pkcs11.so   # the default
export AOS_SB_PKCS11_PROVIDER=/usr/lib64/ossl-modules/pkcs11.so  # the default: the host's pkcs11-provider
make
```

`aos-sb.crt` and `aos-sb.cer` stay files (the certificate is public);
`aos-sb.key` need not exist. `sb_have_key` counts the token as the
key. The command behind it is

```sh
osslsigncode sign -h sha256 -provider $AOS_SB_PKCS11_PROVIDER \
  -pkcs11module $AOS_SB_PKCS11_MODULE -key "$AOS_SB_PKCS11" \
  -certs aos-sb.crt -in FILE -out FILE.signed
```

(the URI goes to `-key`; osslsigncode's `-pkcs11cert` is for a
certificate read from the token, which ours is not). The host's
osslsigncode is built against Buildroot's own OpenSSL, which has no
PKCS#11 provider of its own: the build host's `pkcs11-provider`
package supplies `/usr/lib64/ossl-modules/pkcs11.so`, loaded into it
through `-provider`. Making the key on a YubiKey: `ykman piv keys
generate --algorithm RSA2048 9c aos-sb.pub`, the CSR with `ykman piv
certificates request 9c aos-sb.pub aos-sb.csr`, signed by the root as
above, then `ykman piv certificates import 9c aos-sb.crt`; the PIN is
asked at each signature (two UKIs and one loader per build) unless the
token caches it.

### Backup and rotation

- **Lose the signing key**: issue a new one under the root; nothing
  else changes (with Fedora's shim: the MOK rotation below).
- **Lose the root key**: a new root means a new shim and a new
  review. Two offline copies.
- **Rotation with Fedora's shim** (the MOK path): make the new pair,
  ship one release whose `sb-enroll` queues the new certificate
  (`mokutil --import`, password shown) while everything is still
  signed with the old key; once machines have both, sign with the new
  one. A *compromised* key cannot be rotated quietly: every machine
  must `mokutil --delete` it (a release can queue that).
- **Revocation with our own shim**: a compromised signing certificate
  is refused by a shim rebuilt with it in its vendor dbx, or by a
  `sbat-level.csv` generation when the signed thing carries SBAT
  (every UKI does: `aos.uki,N`).
- A build without the key is unsigned and post-build/post-image say
  so; release.sh refuses a build that is unsigned, signed by another
  key, or whose core.efi is not the UKI of its core.img, whenever the
  key is on the host.

## The 2023 CA

Microsoft signs new shims with its **Microsoft UEFI CA 2023**
(third-party CA, sha256 of the DER
`f6124e34125bee3fe6d79a574eaa7b91c0e7bd9d929c1a321178efd611dad901`,
https://go.microsoft.com/fwlink/?linkid=2239872); Fedora's shim 16.1
is signed by the **Microsoft Corporation UEFI CA 2011**
(`48e99b991f57fc52f76149599bff0a58c47154229b9f8d603ac40d3500248507`,
https://go.microsoft.com/fwlink/p/?linkid=321194; it expires on
2026-06-27, which firmware ignores). A machine sold before 2024 has
only the 2011 CA in db unless a firmware or Windows update added the
2023 one; a 2023-signed shim does not start there. (The link
https://go.microsoft.com/fwlink/?linkid=2239776 is the *Windows* UEFI
CA 2023, which signs Windows' own loader, not shims; db usually holds
both.)

`/usr/lib/aos/sb-check-ca` reads the `db` variable from efivars and
prints `2023`, `2011-only`, `none` or `no-efi` (its `--help`). The
installer will choose by it: `2023` -- our shim; `2011-only` --
fwupd's db update from the LVFS first (`fwupdmgr` with the
`KEK`/`db` capsule Microsoft publishes), else Fedora's shim and the
MOK path, saying so; `none` -- a machine with the third-party CA
switched off (see "The user's one enrollment").

### Two ISO flavours (design note)

Until no supported machine lacks the 2023 CA, a release carries two
ISOs from the same core and the same signed UKIs, differing only in
the loader files on the ESP: `aos-X-x86_64.iso` with our shim (no
enrollment) and `aos-X-x86_64-mok.iso` with Fedora's (the enrollment
of "The user's one enrollment"). `AOS_SHIM=fedora|own` in the
environment of `make` would select which `package/aos-shim` mode the
image gets -- today that is the Config.in choice, so a release chain
builds the image twice, or post-image.sh takes both shims from a
second `aos-shim` package and makes both ISOs from one core
(preferred: one core, one set of hashes, two 1 MB ESP images). The
installed disk gets the shim the ISO started with; `boot-sync` may
later swap in ours when `sb-check-ca` says `2023`. Nothing of this is
built yet.

## Testing

```sh
AOS_OUT=$PWD/output2 python3 br2ext/board/aos/secureboot-test.py enroll   # fresh SB vars, MokManager drives
V=$PWD/output2/images/OVMF_VARS.sb-enrolled.fd
AOS_OUT=$PWD/output2 python3 br2ext/board/aos/secureboot-test.py live
AOS_SB=1 AOS_SB_VARS=$V AOS_OUT=$PWD/output2 python3 br2ext/board/aos/boot-test.py install
AOS_OUT=$PWD/output2 python3 br2ext/board/aos/secureboot-test.py disk
AOS_OUT=$PWD/output2 python3 br2ext/board/aos/secureboot-test.py tamper
AOS_SB=1 AOS_SB_VARS=$V AOS_OUT=$PWD/output2 python3 br2ext/board/aos/update-test.py   # A/B under Secure Boot
# From a published 0.3.10 install (boot-test.py install with AOS_OUT at a
# directory holding its ISO as images/rootfs.iso9660), Secure Boot off,
# to this build, then MokManager's password, then Secure Boot on:
VIRT_FW_VARS=/path/to/virt-fw-vars AOS_OUT=$PWD/output2 \
  python3 br2ext/board/aos/secureboot-test.py upgrade /path/to/0.3.10/aos-disk.img
```

virt-fw-vars (python-virt-firmware) is not on the build host; a venv
with `pip install --no-deps virt-firmware cryptography pefile` serves.
The drivers read the serial line from a thread while the firmware draws
(Reader in secureboot-test.py): unread, the socket fills, QEMU stalls
the guest's serial output, and MokManager's 10-second countdown runs out
before its first line arrives.

`AOS_SB=1` (with `AOS_SB_VARS`) puts every boot-test.py-based driver on
the Secure Boot firmware; without it they run as before, Secure Boot
off.
