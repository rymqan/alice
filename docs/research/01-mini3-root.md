# Yandex Station Mini 3 ("Bergamot") — Root / Custom Firmware Status (Nov 2026)

Research agent output, accessed 2026-11-08. Attribution: most technical detail derives from the
*Yandex Station Extended* wiki (hackalice.me, repo `Marker284/yandex-station-extended`), CC BY 4.0.
Press coverage (Habr/Yandex, anti-malware.ru, CNews, Kod.ru, vc.ru, Shazoo) corroborates the October 2026 incident.

## 1. Stock firmware version / build scheme — VERIFIED

- Current retail build on Mini 3: **`0.369.1.50.4795902939.20261021.117`** (a second unit shows `...4794652521.20261021.117`; maintainer thinks per-unit). Shorthand: **build `.50`**. Sources: [device page](https://hackalice.me/devices/station-mini3/), [PR #76](https://github.com/Marker284/yandex-station-extended/pull/76), [Pulsar issue #50](https://github.com/Morvex885/pulsar/issues/50).
- Date field `20261021` = the **21 Oct 2026** forced OTA. Mini 2 pre-update build was `0.369.1.46.4778262943.20261013.299` ([CNews](https://www.cnews.ru/news/top/2026-10-22_rossiyane_nashli_hitryj_sposob)). Scheme: `0.<major>.1.<minor>.<serial>.<YYYYMMDD>.<n>`; Oct update moved `.46` → `.50`.
- **Factory (recovery) images** use a separate scheme: `0.6.1.4.…factory-6` (11 Nov 2024) and `0.4.1.39.2634333263.20241021.factory-4` ([Mini 3 root guide](https://hackalice.me/start/retail-ca-mini3/)). Mini 3 **Pro** uses `factory-12`.

## 2. Does root still work after the Oct 21–23 update? — VERIFIED (conditionally)

Yes on Mini 3 (Bergamot) on `.50`, but **not** via the old DNS-only method (dead after 21 Oct 2026). Current method: **"retail CA + markers from the factory image"** ([guide](https://hackalice.me/start/retail-ca-mini3/)):

- The `.50` OTA removed the **AAA Certificate Services** root (part of the compromised eMudhra chain) from the device CA bundle, so a self-made update-server certificate is no longer trusted.
- Method **re-adds the AAA root** to the retail CA bundle by booting the device's own **factory/recovery image** (still contains AAA), mounting the retail `system` partition, patching the CA file. Then markers in `/data` re-enable network ADB.
- **Hard prerequisite:** factory mode must boot and `version_full` there must show a `factory-N` build. If it shows `0.369.1.50…`, the factory partition was updated too and *the method fails* — stop. A button factory reset wipes `/data` but does **not** reflash `system`, so a patched CA survives a reset.
- **Requirements:** `adb`; a **custom HTTPS update server** on a PC whose cert chains to AAA (guide uses **eQuasar**: nginx + `resolver` line + Docker); **DNS redirection** (router or Windows ICS hotspot); `aaa-certificate-services.pem` from the `yandex-station-binaries` repo. Works from Windows/Linux/macOS (macOS needs "Local Network" permission for adb). **No UART or USB cable** — entry is wireless over ADB after booting factory mode (hold mic-off button 10–15 s at power-on).

Press: Yandex post-mortem ([Habr](https://habr.com/ru/companies/yandex/articles/1100180/): "С 21 по 23 октября мы обновили все эти устройства…", update also closes ADB); enthusiasts found a way around ([vc.ru](https://vc.ru/id867598/3213652-vzlom-yandeks-stantsii-chatgpt-zamena-alisy)).

## 3. Step-by-step procedure (Mini 3) — VERIFIED

Condensed from [hackalice.me/start/retail-ca-mini3](https://hackalice.me/start/retail-ca-mini3/):

0. **Verify factory image alive:** hold mic-off + power 10–15 s → `cat /proc/cmdline` (expect `rootfstype=ramfs`), `cat /system/vendor/quasar/version_full` (expect `factory-N`; abort if `.50`).
1. **Start update server** (eQuasar) + set DNS. Re-add the speaker in "Дом с Алисой" app ("setup failed" expected). Confirm server log shows `POST /register`, `/get_sync_info`, `/check_updates`; then `adb connect <ip>:5555` → `adb shell id` → `uid=0(root)`.
2. **Mount retail:** `mount -t ubifs -o rw ubi0:rootfs /mnt/retail` (system = mtd3; factory needs no `ubiattach` because `ubi.mtd=system` is in cmdline).
3. **Patch CA bundle** `/mnt/retail/system/vendor/quasar/certs/ca-certificates.crt` (symlink target): back up to `*.orig`, pull to PC, append `aaa-certificate-services.pem`, push back. Verify **123 certs** (was 122), file exactly **1517 bytes** larger. Hashes given per build in the guide.
4. **ADB markers:** `touch /bkupgrade/data/quasar/activate_adb_quasmodrom` and `activate_network_adb`; verify with `activate_adb.sh check`.
5. **Wi-Fi persistence:** edit `/bkupgrade/data/wpa_supplicant.conf`, set `update_config=0`, `chmod 600`.
6. **Return to normal boot:** if `upgrade_step=2`, `fw_setenv upgrade_step 1`, `fw_setenv boot_to_recovery 0`, `reboot`.
7. **Verify:** `adb connect` after reboot. ADB auto-starts *only while DNS still points to the custom server*; restoring real DNS makes firmware run `activate_adb.sh … deactivate`.
8. **Persistence:** install **Dropbear SSH** (`S99zdropbear`, port 2222) and run `yandex-noupdate.sh` *before* restoring DNS ([SSH page](https://hackalice.me/start/ssh/)).
9. **Insurance:** back up CA + dropbear to `/data/opt`; `restore-system.sh` re-applies after an OTA reflash (`/data` survives OTA).

Guide risk warnings: modifying `system` can brick; `-B` empty-password dropbear is LAN-exploitable; CRLF from Windows breaks busybox scripts; never publish `devices.json`/tokens.

## 4. SoC / OS / partitions / recovery — VERIFIED

- **SoC:** Amlogic **AXG** (device-tree `yandex,bergamot`), 4× Cortex-A53 ≤1.2 GHz. **Linux 4.9.68 aarch64 kernel; 32-bit armhf glibc 2.30 userspace, Buildroot.** ~244 MB RAM; **512 MB SLC NAND**. Linux, not Android (adbd present as diagnostic bridge).
- **Partition table** ([device page](https://hackalice.me/devices/station-mini3/)): mtd0 bootloader (4 MB), mtd1 tpl (6 MB), mtd2 boot (20 MB, encrypted, Amlogic Secure Boot), mtd3 **system (176 MB, UBI ubi0 vol `rootfs`, ro)**, mtd4 nv (2 MB jffs2 → /tee), mtd5 **bkupgrade (232.5 MB, UBI ubi1 → /data)**, mtd6 recovery (8 MB), mtd7 recovery_fs (51.5 MB). **No dm-verity on system** → `remount,rw` edits persist. OTA rewrites mtd0–mtd4; `/data` (mtd5) survives.
- **Backups:** wiki gives **no clean dd/nanddump workflow** for Mini 3. General `contrib/rollback.sh` writes NAND over root ADB, untested on Mini-class, **cannot revive a dead device**. USB Burning Tool support **not implemented**.
- **Unbrick:** relies on the **factory image in mtd6/7**. UART 115200 8N1 3.3 V on `ttyS0` but **login blocked** (`getty_rabbit_hole`, `androidboot.rh_unlock=0`); U-Boot var `rabbit_hole_debug=1` may unlock (untested). U-Boot has `usb_burning`/`update` env vars but **no verified USB-burning procedure** on wiki. **No SD-card recovery.** Weakest-documented area — treat a dead boot/system as unrecoverable.

## 5. Blocking updates after root — VERIFIED

`yandex-noupdate.sh` ([SSH page](https://hackalice.me/start/ssh/)): (1) replaces `ota_script.pub` with a foreign key so OTA packages fail signature check (main protection), (2) stubs `device_ota_update.sh`, (3) removes `maind --updatesd` from `launch.json`. Revert: `yandex-noupdate.sh restore`. ADB markers protected with `chattr +i`. No hosts/iptables approach. Caveat: online `account_config`/`system_config` sync can re-push config. Yandex: OTA "installs automatically", owners "cannot opt out"; blocked (subscription) stations are checked server-side. No wiki evidence of Yandex fingerprinting rooted units.

## 6. Stock voice pipeline — VERIFIED ([internals/voice](https://hackalice.me/internals/voice/))

- **Wake word: on-device.** `nnet` engine in `libyandex_evil.so`, loaded by `maind`; models `activation/alisa|yandex|yasmina`, `counteractivation`, `command` (~25 no-wakeword commands), `vad`.
- **Capture/VQE: on-device**, in `maind --platformd` (**`aliced`**), Yandex VQE over 3–4 PDM mics → `vqe_0` (16 kHz mono).
- **ASR: cloud** (`quasar-general` via **uniproxy**, WebSocket `uni.ws`, Opus+protobuf+zstd to `uniProxyUrl`; server-side endpointing).
- **NLU/LLM: cloud** (VINS/Megamind, `withLocalVins:false`). **TTS: cloud** (`TtsSpeak` Opus → `audio_client` GStreamer → ALSA).
- **Daemons / local IPC (127.0.0.1):** 9875 aliced, 9900 mediad, 9906 audioclient, **9899/1961 glagol(d)** (local API, Home Assistant), 9891 external_commandsd, plus `maind` roles telemetry/fluent-bitd/updatesd. No pulseaudio — ALSA (`AML-AUGESOUND`).
- **Data to Yandex cloud:** post-wakeword speech (Opus), text queries, `MatchedUser`/`LogSpotter`, music-recognition audio, persistent cloud link, *random sampled audio clips for "beta" groups*. Telemetry ([internals/telemetry](https://hackalice.me/internals/telemetry/)): AppMetrica (`report.appmetrica.yandex.net`, incl. serial/UUID/model/kernel), clickdaemon (`clck.yandex.net`, ~185 event types), fluent-bit logs, monitord pings `quasar.yandex.ru` every 10 s.

## 7. Projects routing mic/speaker to custom logic — VERIFIED

Two styles: **uniProxy redirection** (keeps Yandex ASR/TTS) vs **full audio takeover**.

| Project | Link | Last update | Technique | Needs | External servers | Mini 3? |
|---|---|---|---|---|---|---|
| **Quasar Bridge** | [skrashevich/quasar-bridge](https://github.com/skrashevich/quasar-bridge) v0.6.2 | 2026 | Rewrites `uniProxyUrl` in `quasar.cfg`; routes to your LLM / HA / Yandex. ASR+TTS stay Yandex cloud | root+SSH | your OpenAI-compatible endpoint + Yandex | Not listed |
| **Alisa-ChatGPT voice** | [rison1337/Alisa-ChatGPT-voice-assistant](https://github.com/rison1337/Alisa-ChatGPT-voice-assistant) v0.3.3 | 2026 | Audio takeover via stock `aplay` + VQE mic | root+SSH | ChatGPT | Lite 1/2, Station 2 only |
| **station-xiaozhi** | [kotekompotek/station-xiaozhi](https://github.com/kotekompotek/station-xiaozhi/releases) v1.0.0 | Oct 2026 | XiaoZhi alongside Alice; audio to XiaoZhi during talk | root via net-ADB | xiaozhi.me (China) | tested Lite 2 |
| **YAM** | [nyakokitsu/yam](https://github.com/nyakokitsu/yam) v0.1.19 (EOL) | Oct 2026 | `yamd` + `.yap` Lua plugins; Quasar IPC incl. `ipc.speech` | root+ADB; aarch64 | GitHub + yapkg | not confirmed |
| **BetterYAM** | [BetterYAM](https://github.com/Yandex-Station-Universal-Repo/BetterYAM) (MIT) | README "abandoned" | per-model `betteryamd`, `byam` CLI | ADB | raw.githubusercontent.com | **Yes — `yandexmini3` daemon** |
| **Pulsar** | [MyrikLD/pulsar](https://github.com/MyrikLD/pulsar) | active | Speaker becomes **HA voice satellite via ESPHome Native API** + Music Assistant, no Yandex cloud | root | your HA server | **Mini 3 support on `.50`** ([PR #30](https://github.com/Morvex885/pulsar/pull/30)) |
| **node-red-contrib-alice-self-hosting** | [twocolors/…](https://github.com/twocolors/node-red-contrib-alice-self-hosting) v0.2.6 | 2026 | Self-hosted Alice-skill backend (cloud-side) | public URL + TLS | Yandex + Node-RED | N/A |
| **quasard / v3rt3p** | wiki Backend recipes | — | self-hosted uniproxy: own ASR+LLM+TTS | root | your server | not listed |

Wiki's HA voice satellite recipe is documented only for Lite 2/Plum; Pulsar independently claims Mini 3. Nov 6 Midi demo ([Shazoo](https://shazoo.ru/2026/11/06/193638/)) ran local ASR (Parakeet) + LLM + TTS on home hardware.

## 8. Return-to-stock / kill switch — VERIFIED

Per [recovery](https://hackalice.me/start/recovery/) + per-project rollbacks: `yandex-noupdate.sh restore`, remove `S99zdropbear`, restore CA from `*.orig`, remove router DNS records; Quasar Bridge `control.sh rollback` + `uninstall.sh`. A **hardware factory reset** wipes `/data` (markers/SSH/Wi-Fi) but **not** `system` edits — stock restore requires both reset *and* rollback. Yandex advises against reset ("не рекомендуем сбрасывать Станцию без острой причины").

## 9. License — VERIFIED

Wiki is **CC BY 4.0** (attribution + link to repo; mark changes). BetterYAM, node-red-contrib, YAM are MIT.

## Sources (accessed 2026-11-08)

Wiki: [README](https://github.com/Marker284/yandex-station-extended), [home](https://hackalice.me/), [start](https://hackalice.me/start/), [Mini 3 root guide](https://hackalice.me/start/retail-ca-mini3/), [SSH/no-update](https://hackalice.me/start/ssh/), [devices](https://hackalice.me/devices/), [Mini 3](https://hackalice.me/devices/station-mini3/), [Mini 3 Pro](https://hackalice.me/devices/station-mini3-pro/), [architecture](https://hackalice.me/internals/architecture/), [voice](https://hackalice.me/internals/voice/), [telemetry](https://hackalice.me/internals/telemetry/), [recovery](https://hackalice.me/start/recovery/), [uart-shell](https://hackalice.me/start/uart-shell/), [recipes](https://hackalice.me/recipes/), [packagemanagers](https://hackalice.me/recipes/packagemanagers/), [llm](https://hackalice.me/recipes/llm/).
GitHub: PR #76, PR #31 (wiki), BetterYAM, YAM, quasar-bridge, Alisa-ChatGPT, station-xiaozhi, pulsar + PR #30 / issue #50, node-red-contrib-alice-self-hosting.
Press: [Habr/Yandex post-mortem](https://habr.com/ru/companies/yandex/articles/1100180/), [anti-malware.ru](https://www.anti-malware.ru/news/2026-10-22-111332/51812), [CNews](https://www.cnews.ru/news/top/2026-10-22_rossiyane_nashli_hitryj_sposob), [vc.ru](https://vc.ru/id867598/3213652-vzlom-yandeks-stantsii-chatgpt-zamena-alisy), [Shazoo](https://shazoo.ru/2026/11/06/193638/).

Unreachable: 4PDA (HTTP 403), web.archive.org.

## ASSUMED / UNKNOWN

- Mini 3 YNDX model number: not found.
- Whether `chattr +i` keeps ADB markers alive after DNS restore on Mini 3: wiki says untested.
- Partition dump / unbrick via USB burning: not documented for Bergamot — treat bricking as unrecoverable.
- Build-number duplication (`4794652521` vs `4795902939`): likely per-unit.
