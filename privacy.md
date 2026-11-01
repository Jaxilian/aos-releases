# Privacy: what leaves an AOS machine

What the Adelon Operating System sends off the machine, to whom, and
why, written the way the GDPR asks of a Swedish company. The rule
(policies.md, "No telemetry"): AOS sends nothing about the machine on
its own. The list below is everything; a release that added to it
without a line here would be a bug.

**Controller:** Adelon AB (to be formed; until then Jax Carls, who owns
AOS). **Contact:** to be decided -- until then the security address in
security-policy.md. **Legal basis:** the legitimate interest of keeping
the system updated and secure (items 1, 2, 5), and your consent or
explicit request (items 3, 4). **Your rights:** access, correction and
erasure under the GDPR; for a report, quote the report id Settings
showed when it was sent.

The consent screen of the first-boot setup and Settings' "Send a
report" page quote this page; the two have to match.

## 1. Update checks

| | |
|---|---|
| What | Two minutes after boot and then daily (`aos-update-check.timer`), and when a person presses "Look for updates" in Software, `aos-update-check` runs `apm update` (item 2) and `aos-update --check`, which fetches `SHA256SUMS` and `SHA256SUMS.minisig` from the release channel to see whether a newer AOS exists. Nothing is installed without the person. |
| Where | `https://github.com/Jaxilian/apm-recipes/releases/download/index/` (the edge channel) or `.../stable/` (`CHANNEL=stable` in `/etc/aos/update.conf`); GitHub redirects the download to `objects.githubusercontent.com`. |
| Data sent | A plain HTTPS GET: your IP address, the URL (which names the channel) and curl's version string as the user agent. No AOS version, no machine identifier, no account. |
| Recipient | GitHub, Inc. (Microsoft), under GitHub's privacy statement. AOS runs no server of its own here and sees nothing. |
| Retention | GitHub's own request logs, on GitHub's terms. AOS keeps nothing. |

## 2. Package indexes and downloads (apm)

| | |
|---|---|
| What | `apm update` fetches each repository's `stamp.toml` and its `.minisig`, and the index archive when the stamp changed; daily with item 1, when a person presses "Look for updates", and on any `apm` command when the index is older than six hours (`index_expire` in `/opt/apm/etc/config.toml`). Installing or upgrading a package downloads the files the index names. |
| Where | The `main` repository is the same GitHub release as item 1; the third-party repository (`apm-thirdparty`, added by the person) is another GitHub release. A third-party package's files come from the URL its recipe names, often the vendor's own server (Mozilla for Firefox, Google for Chrome, Microsoft for VS Code, Valve for Steam): that server sees the download. |
| Data sent | HTTPS GETs (apm's curl allows nothing but https): your IP address, the URL, curl's user agent. No list of what is installed, no identifier. |
| Recipient | GitHub, and the vendor of each third-party package you install. |
| Retention | Theirs. AOS keeps nothing. |

## 3. Diagnostics reports (aos-report)

| | |
|---|---|
| What | Only when a person sends one: "Send a report" in Settings, the first-boot setup's consent page, or `aos-report --send` at a terminal. The archive holds the state of the machine, not the person's files: the AOS version and build, the kernel, the hardware and its drivers (`lspci`, `lsusb`, DMI), this boot's system log and the previous boot's warnings, failed services, start-up times, the programs using the most memory, the desktop's log and which programs crashed (no memory dumps), sound and Bluetooth state with the names of paired devices, disks, mounts, network status (addresses, no Wi-Fi passwords), what Software has installed and its sources. With `--hwcheck`, the hardware check's results (`aos-hwcheck`'s JSON: the DMI model, firmware, each check's pass, fail or skip). With `--steam`, Steam's own logs from the home folder -- the one thing from it, opted into separately. `aos-report --what` prints the same list on the machine. Lines that look like passwords, pre-shared keys, private keys or tokens are blanked before the archive is made; a log can still name files, networks or sites a program wrote about, so a report is personal data. |
| Sent with it | The person's note, the AOS version, and an anonymous machine number: the first 16 hex digits of sha256("aos-report:" + the machine id), so one machine's reports group without the machine id itself leaving. |
| Where | The `REPORT_URL` of the build (`/usr/lib/aos/report.conf`, or `/etc/aos/report.conf`); https only. The public builds carry no URL yet: until a receiver exists, nothing can be sent at all and Settings shows no Send button. |
| Consent | A standing consent is the file `/etc/aos/report-consent`, written by the first-boot setup's toggle (off by default) or Settings → Advanced, and removed when the toggle goes off. Without it `aos-report --send` refuses, unless the command carries `--i-asked`: the person's explicit command for this one report (Settings' "Send a report now" passes it; so can a person at a terminal). Nothing is ever sent by a timer or a service. |
| Recipient | Adelon (the receiver in `br2ext/report-server/`, run by us). The server stores the archive and its metadata under the day's date and the report id; it stores no IP address (the rate limit's counts live in memory only). |
| Retention | To be decided; the rule until then: as long as the bug it describes needs, and the directory readable by the developers only. Erasure on request by report id. |

## 4. Firmware updates (fwupd, LVFS)

| | |
|---|---|
| What | fwupd's refresh timer is not enabled. The LVFS is reached only when a person runs `fwupdmgr refresh`, `get-updates` or `update` (Settings has no firmware page yet): the firmware metadata is downloaded, and a chosen firmware file. |
| Where | `https://cdn.fwupd.org/downloads/firmware.xml.zst` (metadata) and the firmware URLs it names on the same host. |
| Data sent | An HTTPS GET with your IP address and fwupd's user agent (its version). The metadata request carries no device identifiers; which firmware file is then requested reveals the device it is for. Reports of an update's outcome (device ids, firmware versions, success or failure) go to `https://fwupd.org/lvfs/firmware/report` only with `fwupdmgr report-history`: `AutomaticReports` and `AutomaticSecurityReports` are off in `/etc/fwupd/remotes.d/lvfs.conf`. |
| Recipient | The Linux Vendor Firmware Service (the Linux Foundation); its privacy page is <https://lvfs.readthedocs.io/en/latest/privacy.html>. |
| Retention | Theirs. |

## 5. Time synchronisation (systemd-timesyncd)

| | |
|---|---|
| What | The clock is set from an NTP server at boot and about every half hour after. |
| Where | The servers the network's DHCP names, if any; otherwise the fallback list in `/etc/systemd/timesyncd.conf.d/10-aos.conf`: Netnod's Swedish servers `sth1.ntp.se`, `sth2.ntp.se`, `gbg1.ntp.se`, `gbg2.ntp.se` (run for the Swedish Post and Telecom Authority), then `0.europe.pool.ntp.org` and `1.europe.pool.ntp.org`. |
| Data sent | An NTP packet (UDP 123): your IP address and timestamps. Nothing about the machine. |
| Recipient | Netnod AB (Sweden) or the NTP pool's volunteer servers, or the network's own. |
| Retention | Theirs; Netnod publishes its policy at ntp.se. |

## 6. The weather (the Weather app, only once you choose a place)

| | |
|---|---|
| What | Nothing until you search for and choose a place in the Weather app. Then the search words go to a geocoder, and the forecast for the chosen coordinates is fetched: when you choose, when the window opens on a forecast older than half an hour, when you press Refresh, and every half hour while you are logged in (`aos-weather.timer`, which does nothing while no place is chosen). The bar shows the temperature from that forecast. |
| Where | `https://geocoding-api.open-meteo.com/v1/search` (the search) and `https://api.open-meteo.com/v1/forecast` (the forecast), plain HTTPS GETs without a key or an account. |
| Data sent | Your IP address, the words you searched for, the chosen place's latitude and longitude (two decimals of a city's centre, not your position: AOS reads no GPS and no Wi-Fi location), the language code for the place names, and curl's user agent. No machine identifier. |
| Recipient | Open-Meteo (Switzerland), under its privacy policy at open-meteo.com. |
| Retention | Theirs. AOS keeps the place in `~/.config/aos/weather` and the last forecast in `~/.cache/aos/weather`; removing the first file ends the fetching. |

## Also on the network, sending nothing about you

- **Name resolution:** every program's DNS queries go to the network's
  resolver; when the network names none, systemd-resolved's built-in
  fallbacks (Cloudflare, Google, Quad9) are used. A resolver sees the
  names you look up. European defaults are planned (white-paper.md).
- **Printers:** cups-browsed and Avahi listen for printers on the local
  network and publish nothing.
- **Third-party programs** (Firefox, Chrome, Steam, VS Code, Discord
  and the others from Software) have their own telemetry and their own
  settings for it; AOS's sandbox limits what they can read, not what
  they send. What they do is theirs, under their vendors' policies.

## What does not exist

No account, no activation, no crash reports sent by themselves, no
usage statistics, no advertising identifier. A verity-protected core
cannot be changed on the machine to add any; a release that did would
be visible in this page's history and in `policies.md`.
