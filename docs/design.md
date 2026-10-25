# Garden telematics software design

Status: draft, October 2026.

## Summary

Zephyr LoRaWAN cluster nodes report to a Raspberry Pi 5 hub on site. The hub runs a minimal Yocto image with ChirpStack, one first-party service (`gardend`), and a static web UI served over the Pi's own Wi-Fi access point. Gardeners scan a QR code to see sensor data. Admins use the same UI behind a login, and updates are signed RAUC bundles uploaded from a phone.

Design principles, in priority order:

1. **All data stays on-site.** Nothing depends on the internet, and nothing phones home.
2. **Minimal exposed surface.** One listener on the Wi-Fi (nginx), and everything else on localhost or unix sockets. The fundamentals stay sound if the hub ever goes online.
3. **Easy maintenance.** A read-only root, atomic A/B updates with automatic rollback, and no hand configuration per box.
4. **Nodes stay generic.** Plugging in a probe of a known type needs no reflash. Meaning, units and location live on the hub.

## System overview

```
 node-1 ────────┐            ┌──────────────────────── Pi 5 hub ────────────────────────┐
 node-2 ────────┼─ LoRaWAN ──┤ Concentratord ──> Mosquitto <──> ChirpStack v4           │
 node-n ────────┘            │      │                │               │                  │
                             │ gpsd + chrony      gardend ────> SQLite, Redis           │
                             │                       ^   └──> RAUC (D-Bus)              │
 Phones <────── Wi-Fi ───────┤ Wi-Fi AP ──────> nginx + web UI                          │
                             └──────────────────────────────────────────────────────────┘
```

Uplinks flow from the nodes through Concentratord, Mosquitto and ChirpStack, then into `gardend`, which owns storage and the API. Phones reach only nginx, through the AP. Everything else talks over localhost, unix sockets or D-Bus.

## Repositories

| Repository | Contents |
| --- | --- |
| garden-node-board | Node PCB hardware |
| garden-node-firmware | Zephyr node firmware, plus `proto/`: the wire format, test vectors and sensor profiles |
| garden-hub | This document, `meta-garden`, `gardend` and the web UI. Pulls in garden-node-firmware as a submodule for `proto/`. |

## Node firmware

Nodes run Zephyr on the STM32WLE5: a Wio-E5 or RAK3172 module on the custom board, or a RAK3372 WisBlock for the prototype, with one firmware image for both.

- **Radio:** Zephyr's LoRaWAN subsystem, LoRaWAN 1.0.4, Class A, OTAA, US915 sub-band 2 to match the RAK5146's 8 channels.
- **Duty cycle:** wake, switch on the probe rail through a load switch, sample, send one uplink, deep sleep. The interval is set at runtime and defaults to 30 minutes.
- **Buses:**
    - 1-Wire: the in-tree `w1` subsystem, with DS18B20 on a UART-based w1 master.
    - RS-485: the in-tree Modbus RTU client. EC is deferred, but the port and 4-conductor cable go in now.
    - SDI-12: a new out-of-tree bus driver with its own devicetree binding, running 1200 baud 7E1 on an inverted UART with a GPIO-driven break and direction pin. Per-probe handling sits on top of it, the way sensor drivers sit on i2c.
- **Payload:** see [`proto/wire-format.md`](https://github.com/TDuke94/garden-node-firmware/blob/main/proto/wire-format.md). Readings carry their age, and the hub timestamps them from GNSS-disciplined gateway time, so nodes need no wall clock.
- **Buffering:** a flash ring log (Kconfig, default 8 KB) holds unsent sets across connectivity lapses and reboots.
- **NFC:** an ST25DV dynamic tag on the board, used for tap-to-commission with the same command codec as LoRaWAN downlinks.
- **Images:** MCUboot. Bench updates go through mcumgr, and FUOTA over LoRaWAN comes later.
- **Bench access:** Zephyr shell over the debug UART.

## Node configuration and sensor discovery

A new probe of a known kind needs no reflash.

| Bus | Discovery | Identification | What still needs config |
| --- | --- | --- | --- |
| 1-Wire | Full: ROM search finds every 64-bit ID | Family code (0x28 = DS18B20) | Which bed each probe is in |
| SDI-12 | Good: `a!` across 62 addresses, about 2 s | `aI!` returns vendor, model, version and serial | Readdressing with `aAb!`, since most sensors ship at `0` |
| RS-485 Modbus | None in practice | None | Address and profile per device |

- **Devicetree** describes fixed hardware only: ports, UARTs, probe-rail load switches.
- **Probe table** in Zephyr `settings` on flash: port, bus address or ROM ID, and profile per probe. A bus scan fills it at boot, daily, or on command.
- **Sensor profile packs** describe how to read each sensor model, as data rather than code. They come from one source in `proto/profiles/`, which builds both the node pack and the hub's decode metadata.
- **Manifest uplink** after any table change. Data uplinks carry the manifest generation, so the hub always decodes against the right layout.
- **The hub owns meaning.** Units, scaling and location live on the hub, edited through the admin UI.

## Hub OS and packaging

The hub runs a Yocto image built from poky, meta-raspberrypi (machine `raspberrypi5`), meta-openembedded (oe, networking, webserver), meta-rauc, and one first-party layer, `meta-garden`. Pi OS Lite with docker compose was the alternative. Yocto won on read-only root, atomic updates, and installing only what we ask for.

- **Languages:** Rust, C or C++ for every service and tool. Python is only for build-time glue, and the runtime image ships no Python interpreter.
- **No containers.** Each service is a systemd unit, sandboxed with `DynamicUser`, `ProtectSystem=strict`, `PrivateTmp`, `RestrictAddressFamilies` and a minimal `CapabilityBoundingSet`.
- **Partitions:** boot, rootfs A and rootfs B (read-only), and `/data` (ext4, unencrypted) for databases, config and uploads.
- **Image variants:** `prod` without debug tools, and `dev` adding a debug shell and gdbserver.

| Service | Role | Listens on |
| --- | --- | --- |
| nginx | TLS, static UI, reverse proxy to `gardend` | Wi-Fi and Ethernet, 80/443 |
| gardend | Ingest, public API, admin API, update orchestration | unix socket |
| chirpstack | LoRaWAN network server | localhost (API and UI) |
| chirpstack-concentratord | RAK5146 driver, reset and GNSS | IPC socket |
| chirpstack-mqtt-forwarder | Concentratord to MQTT bridge | none |
| mosquitto | Message bus | localhost |
| redis | ChirpStack device sessions and queues | unix socket |
| gpsd, chrony | GNSS, PPS and system time | localhost |
| hostapd, systemd-networkd, dnsmasq | Wi-Fi AP, DHCP, DNS redirect | Wi-Fi |
| rauc | A/B updates | D-Bus |
| sshd | Admin shell, keys only | Ethernet only |

### Reduced-surface image option

There are two image recipes from one distro config. `garden-image` is the normal build. `garden-image-tiny` removes anything that adds attack surface: anything that listens, escalates privilege, interprets code, or loads code at runtime. The goal is surface reduction, not size, so ordinary tools like coreutils stay. Both images are delivered as RAUC bundles, so a site can switch between them with an ordinary update.

| Area | `garden-image` | `garden-image-tiny` |
| --- | --- | --- |
| Kernel | meta-raspberrypi's `bcm2712` defconfig plus our fragments | Our own defconfig with only the hardware we use (SPI, RP1 GPIO and UART, NVMe, mt7921u, nftables, dm-verity). No Bluetooth, sound, DRM, NFS or unused USB classes. |
| Kernel hardening | Distro defaults | No module loading after boot (or signed modules only), lockdown mode, unprivileged BPF and user namespaces off, no `/dev/mem`, kexec off |
| Listeners | As in the service table | The same, checked in CI by a test that fails on any unexpected open socket |
| Privilege | Distro defaults | No setuid or setcap binaries except audited ones, no `sudo`, root login only on the Ethernet console with keys |
| Runtime code | No package manager | Also no interpreters, compilers or debuggers |
| Init | systemd | systemd with unused components off (logind, resolved, timesyncd, homed, machined) |
| Debug | shell, `strace` and `gdbserver` in the `dev` variant | none |

Surface reduction gets checked rather than assumed. Yocto's `buildhistory` tracks the package list per build. The SPDX SBOM and `cve-check` output list what's exposed in each release. A CI test boots the image and lists open sockets and setuid files. Optional features, such as a camera, come back as their own kernel fragment and package group.

## Gateway, network server and time

ChirpStack v4 runs the whole LoRaWAN side locally. The open-source edition of The Things Stack was the alternative, but it's heavier and built for enterprise deployments.

- **Concentrator:** ChirpStack Concentratord drives the RAK5146 (SX1303) over SPI. It toggles the reset and power-enable pins itself through the gpiod chardev, with a configurable chip and pin (`sx1302_reset_chip`, `sx1302_reset_pin`). That should replace the Pi 5 reset scripts by pointing it at the RP1 gpiochip, but it needs a bench check. ChirpStack Gateway OS doesn't list the Pi 5 as supported.
- **Building it:** chirpstack, concentratord and mqtt-forwarder are Rust, built with Poky's cargo classes. ChirpStack embeds a React UI at build time, which is awkward under bitbake. Plan A builds the UI assets outside bitbake and feeds them in. Plan B wraps upstream's arm64 release binary in a recipe.
- **Storage:** ChirpStack's SQLite build (available since v4.10.1; the backend is chosen at compile time), plus Redis, which ChirpStack still needs. v4.16.1 had a SQLite migration bug that deleted devices, fixed in v4.16.2, so we pin versions and test every upgrade's migrations.
- **Exposure:** ChirpStack's API and UI bind to localhost. Admin tasks go through `gardend`, which calls ChirpStack's gRPC API. The ChirpStack UI itself is reachable only through an SSH tunnel.
- **Time:** the RAK5146 GNSS feeds gpsd, Concentratord reads GNSS through gpsd, and chrony disciplines the clock from PPS, making the hub a stratum-1 source. The Pi 5 RTC, with its battery fitted, covers cold boots before a GNSS fix.
- **RTK base (later):** a separate multiband receiver (F9P or UM980) on a roof antenna, with an NTRIP caster on the hub. The RAK5146 GPS isn't RTK-grade.

## Ingest, storage and API

`gardend` is the one first-party hub service, written in Rust. It decodes uplinks, stores readings, and serves every API the UI and any future phone app use.

- **Ingest:** subscribes to ChirpStack uplink events on local MQTT and decodes them against the node's current manifest, using the `garden-proto` crate. ChirpStack's JS codecs are stateless and can't look up manifests, so decoding happens here.
- **Storage:** SQLite in WAL mode, with tables for raw uplinks, nodes and manifests, probes (profile, location) and readings. Every reading references the raw uplink it came from, for provenance. Even a few dozen measurement points every 30 minutes come to well under a million rows a year, so no time-series database is needed.
- **Public API (read-only, no auth):** current values, history for a probe or area, and node health.
- **Admin API (authenticated):** probe registry edits, node provisioning through ChirpStack gRPC, downlink commands, updates, backups, and Wi-Fi settings.
- **API shape:** versioned JSON over HTTP with an OpenAPI spec, served on a unix socket behind nginx.
- **Alerts:** a rules table (frost, high temperature, dry soil) evaluated on ingest. For now alerts show as banners in the UI. The notifier interface stays a stub until the hub has internet.
- **Backups:** nightly SQLite online backups plus a config export to a USB drive, and a download button in the admin UI.

## User interfaces

There's one web app. The admin console is a login-gated section of the gardener UI, not a separate product, and both run on `gardend`'s API.

- **Gardener view (no login):** scan the Wi-Fi QR code, and the captive-portal redirect opens the app. It shows current values by area and 7-day trends. It's static files served by nginx, phone-first.
- **Admin console (login):**
    - Probe registry: name, profile, location, and approving probes found in a new manifest.
    - Nodes: provisioning in ChirpStack, last seen, battery, and config commands.
    - Hub: service health, disk, GNSS fix and time sync, recent journal errors.
    - Updates: upload a bundle, watch progress, see slot status.
    - Backups, plus the Wi-Fi SSID and password, which also regenerates the QR code.
- **Not shipped:** Grafana, which would be the largest attack surface on the box and is hard to build from source, and Cockpit, which is a privileged shell in a browser. Grafana can still run on a laptop against a database export.

## Phone integration

| Option | What it gives | Cost and surface | Verdict |
| --- | --- | --- | --- |
| QR code at the hub | Joins the Wi-Fi and opens the app | None | Baseline |
| NFC sticker or QR on each node and bed | Tap or scan to open that bed's page; phones read NDEF URLs without an app | About $1 a tag, passive, no hub change | Do it |
| ST25DV dynamic NFC tag on the node PCB | Tap to commission: read DevEUI and status, write commands | About $1 a part plus I2C, reachable only physically | On the board |
| BLE on nodes or hub | Live readings and config from a phone | Second radio on nodes (the STM32WLE5 has no BLE), another stack listening on the hub, and a native app (no Web Bluetooth on iOS) | No |

Writing to a node tag from a web page only works in Chrome on Android (Web NFC). iOS would need a native app, and a native app is the point at which BLE or NFC writes from iOS are worth revisiting.

## Updates

Hub updates are signed RAUC bundles with A/B slots on the Pi 5's native `tryboot`. SWUpdate was a close second, with a built-in upload server but more surface and configuration. Mender was ruled out because it's centred on its server, which cuts against keeping everything on-site.

1. **Bench basics:** A/B slots, a custom RAUC bootloader backend for `tryboot` and `autoboot.txt`, and `rauc install` over SSH or USB. A health-check unit marks the slot good once Concentratord is receiving and ChirpStack, `gardend` and nginx are up. `tryboot` is one-shot, so a failed boot falls back on its own.
2. **Phone upload:** download the `.raucb` from a release onto a phone, open the admin console near the hub, pick the file and tap Install. `gardend` streams it to `/data`, calls RAUC's D-Bus `InstallBundle`, and reports progress over server-sent events. RAUC's signature check is the security boundary; the admin login guards against nuisance and filling the disk.
3. **Online (with a modem):** a "Check for updates" button. The hub fetches a release manifest, and RAUC streams verity bundles over HTTPS.
4. **Node FUOTA (optional per release):** the bundle carries node images, and a post-install hook schedules a deployment through ChirpStack's built-in FUOTA (since v4.12). Nodes receive it with Zephyr's LoRaWAN FUOTA services.

Tryboot backend status (checked October 2026): upstream RAUC has an open PR ([rauc#1599](https://github.com/rauc/rauc/pull/1599)) adding a Raspberry Pi firmware bootchooser that manages `autoboot.txt`. Bootlin runs RAUC on the Pi 5 with firmware-only boot, because U-Boot lacks PCIe support for the BCM2712 and NVMe needs it, but they haven't published their backend. The plan is to write a small custom backend now and switch to upstream once #1599 lands.

Before every update, the installer snapshots the SQLite databases, and it restores them if the new slot isn't marked good, because ChirpStack's schema migrations only run forward.

## Security, build and signing

The gardener Wi-Fi is treated as untrusted. Only nginx is reachable from it, and every change to the hub needs either a signed bundle or an admin login.

- **Network:** nftables drops by default. Wi-Fi allows DHCP, DNS, and ports 80/443 to nginx. Ethernet adds SSH with keys only.
- **TLS offline:** a local CA, installed once on the admin's phone. The gardener view also works over plain HTTP, since it's public and read-only.
- **Secrets:** LoRaWAN keys and admin credentials live on `/data`, unencrypted, protected by file ownership and systemd credentials. Without a TPM or secure boot, an on-board unlock key would add little. Revisit this if secure boot is adopted.
- **Later, online:** outbound-only WireGuard for remote admin, still with no inbound ports.
- **CI:** runs only on a self-hosted runner on the maintainer's build server, never on cloud runners. The repositories are public, so workflows that use the runner trigger only on pushes to protected branches, on tags, and on manual dispatch, never on `pull_request` from forks. The runner sits in a group restricted to these repositories, and each job runs in a fresh, ephemeral environment.
- **Signing:** CI produces unsigned bundles. Signing is a separate step on the build server through PKCS#11, using a YubiKey PIV key that requires a touch.
- **Secure boot (later):** Raspberry Pi's `rpi-sb-provisioner` supports the Pi 5. It burns a public-key hash into OTP, after which the board boots only bootloader images signed with that key. The step can't be undone, and Yocto images aren't documented as supported, so it stays a later, deliberate decision.

## Decisions

| Decision | Notes |
| --- | --- |
| Yocto hub image with systemd sandboxing, no containers | Pi OS plus docker compose rejected |
| ChirpStack v4 with Concentratord on the RAK5146 | |
| SQLite everywhere; Redis is the only database server | ChirpStack SQLite build |
| No disk encryption on `/data` | Revisit together with secure boot |
| Compiled languages only; `gardend` in Rust | Python only for build glue |
| Custom web UI with a login-gated admin section | No Grafana or Cockpit in production |
| RAUC with Pi 5 `tryboot`, no U-Boot | Custom backend until rauc#1599 lands |
| Reduced-surface `garden-image-tiny` alongside the normal image | |
| Pi is the Wi-Fi AP, through a Netgear A8000 on 2.4 GHz | Coverage near the hub only |
| Alerts are a stub until the hub has internet; sample interval 30 minutes or slower | |
| ST25DV dynamic NFC tag on the node PCB | |
| Three repositories: board, node firmware, hub | |

## Open items

- Bench-check Concentratord's reset-pin config against the Pi 5's RP1 gpiochip.
- Bench-run ChirpStack's SQLite build, including an upgrade migration.
- Decide whether to adopt secure boot.
- Wire-format open questions (see `proto/wire-format.md`).
- Who maintains the system day to day.

## Sources

- [ChirpStack changelog](https://www.chirpstack.io/docs/chirpstack/changelog.html): SQLite build (v4.10.1), SQLite migration fix (v4.16.2), built-in FUOTA (v4.12)
- [ChirpStack Gateway OS hardware support](https://chirpstack.io/docs/chirpstack-gateway-os/hardware-support.html)
- [Bootlin: Safe updates using RAUC on Raspberry Pi 5](https://bootlin.com/?p=19769)
- [rauc#1599: Raspberry Pi firmware bootchooser](https://github.com/rauc/rauc/pull/1599)
- [Raspberry Pi Secure Boot Provisioner](https://www.raspberrypi.com/documentation/security/secure-boot-provisioner.html)
