# SV08 → Modern OS: Research Notes & Reference Map

Working notes for the `sv08-cb1-v3` branch. Compiled 2026-10-04. Not part of the user-facing guide. The guide itself will go in `docs/install-armbian.md`.

---

## 1. Recommendation

**Target the official Armbian "Sovol SV08" image** (Debian 13 Trixie minimal, kernel 6.18.x).

- Armbian merged native board support for the SV08 on 2026-07-25 ([armbian/build#10255](https://github.com/armbian/build/pull/10255)). Board page: <https://www.armbian.com/boards/sovol-sv08>. The latest image as of today was built 2026-09-29 with kernel 6.18.54.
- It is the CB1 kernel and u-boot plus the two SV08 hardware differences: the eMMC is capped at 45 MHz (Sovol's validated clock) and the `uart3-pi` overlay is on by default. That means no custom `.dtb` copying and no `fdtfile=` hacks.
- It updates through `apt` like any Armbian board, so it doesn't go end-of-life the way the CB1 2.3.4 / Bullseye image did.
- **Risk:** support level is "Community". The maintainer (lexfrei) owns a Sovol Zero, not an SV08. The SV08 device tree has had no hardware verification from the maintainer. Your install would be one of the first documented ones, so record everything.

**Fallback:** the Armbian **BigTreeTech CB1** image (Trixie minimal) with `fdtfile=sun50i-h616-bigtreetech-cb1-emmc.dtb`. Several SV08 owners have run it for months (refs #229, ljg-dev). It has no eMMC clock cap, which is fine on your MKS 32 GB module and **not** fine on the stock 8 GB module.

**Don't use:** BTT CB1 V3.0.0 or V3.1.0. On V3.0 there are repeated reports of freezes, segfaults, and Wi-Fi drops (PR #123). On V3.1 the only report so far is "nothing but problems" (wwebers, PR #123, 2026-05-31).

### Options compared

| Option | OS / kernel | SV08-specific DT | SV08 field reports | Verdict |
|---|---|---|---|---|
| Armbian **Sovol SV08** | Trixie / 6.18.54 (2026-09-29) | Yes (45 MHz eMMC, uart3) | Few; new since Jul 2026 | **Primary** |
| Armbian **BigTreeTech CB1** | Trixie / 6.18.43 (26.8.1, 2026-08-08) | No (CB1 eMMC dtb) | Several, months stable (mostly on 6.12.68) | Fallback |
| BTT CB1 **V3.1.0** | Trixie / 7.0 (2026-04-30) | No | 1 negative | Skip |
| BTT CB1 **V3.0.0** | Bookworm / 6.6 | Needs community dtb | Multiple unstable | Skip |
| Zeron prebuilt image (`zqr.pw` link) | Armbian 25.11 / 6.18.4 | Patched | Few positive | Skip: opaque, build scripts never published |
| Sovobian (lexfrei) | Trixie / 6.12.68 | Yes | n/a | Retired 2026-08-05 → superseded by Armbian SV08 |
| BTT CB1 **V2.3.4** (current guide) | Bullseye (EOL 2026-08-31) / 5.16 | Sovol dtb | Lots, historically | Legacy only |

---

## 2. Why the current guide breaks (your KIAUH failure)

On CB1 2.3.4 the problems stack up:

1. **Debian 11 is EOL** (LTS ended 2026-08-31). The mirrors moved to `archive.debian.org` (PR #264). As of 2026-10-03, `bullseye-security` on the archive has **no Release file**, so `apt update` fails, and KIAUH aborts its dependency step (issue #265; workaround is a `snapshot.debian.org` URL).
2. **Python 3.9.** Current Crowsnest v5 needs 3.10+ (issue #251). PyPI packages keep raising their minimum Python too (KIAUH #776).
3. The Klipper pip dependency errors already noted in the README (2025-04-28 workaround).

Moving to Trixie (Python 3.13, supported apt) removes all three.

---

## 3. Reference map

Duplicates removed: your list had #229 comment 3765616568 four times and the ljg-dev README twice.

### Core: build the new guide from these

| Ref | What it gives us |
|---|---|
| [Armbian: Sovol SV08 board page](https://www.armbian.com/boards/sovol-sv08) **(new)** | The image to use. Trixie minimal, kernel 6.18.54. |
| [armbian/build#10255](https://github.com/armbian/build/pull/10255) **(new)** | Explains the SV08 DT: CB1 dtsi + 45 MHz eMMC cap + uart3. Hardware-verification status. |
| [SV08 DTS in armbian/build](https://github.com/armbian/build/blob/main/patch/kernel/archive/sunxi-6.18/dt_64/sun50i-h616-sovol-sv08.dts) **(new)** | Exact pin/eMMC config. Useful when debugging. |
| [Rappetor README](https://github.com/Rappetor/Sovol-SV08-Mainline#method-1-write-image-directly-to-emmc) (your fork) | Steps 1, 5–8 stay mostly the same. Steps 2–4 get replaced. |
| [ljg-dev: install latest Armbian](https://github.com/ljg-dev/sovol-sv08-mainline/tree/main/docs/2-install-latest-armbian) | Closest existing walkthrough (Armbian Imager → armbianEnv → first login → KIAUH). Written for the CB1 image; see the gotchas in §4. |
| [ljg-dev: back up stock eMMC](https://github.com/ljg-dev/sovol-sv08-mainline/tree/main/docs/1-backup-emmc) | `dd` + `sha256sum` + `zstd` backup procedure. Worth adopting as a guide step. |
| [#229 comment (AndrewLinden)](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/229#issuecomment-3765616568) | Minimal proven recipe on the Armbian CB1 image: correct single-line `overlays=`, mask `networkd-wait-online`. |
| [#229 thread](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/229) | Field reports: Btrfs-scan boot hang, HDMI `disp_mode`, Etcher checksum fail (decompress `.xz` first), Wi-Fi dongle. |
| [PR #123 thread](https://github.com/Rappetor/Sovol-SV08-Mainline/pull/123) | Why CB1 3.x was abandoned. **Kernel hold** for Ethernet (wwebers, 2026-05-30). Host-MCU flash retry. Config-persistence complaint. |
| [Armbian forum: BTT Pi Ethernet broken 6.12.68→6.18.33](https://forum.armbian.com/topic/59962-btt-pi-ethernet-not-working-after-upgrade-from-61268-to-61833/) **(new)** | **Matters to you (Ethernet).** H616 `sunxi_gmac` broke in 6.18.33. Reported fixed mid-July. |
| [armbian/build#10430](https://github.com/armbian/build/pull/10430) + [#10492](https://github.com/armbian/build/pull/10492) **(new)** | The Ethernet fixes: EMAC driver restored (2026-08-15), warm-reboot Ethernet loss fixed (2026-09-07). Both predate the 2026-09-29 SV08 image. |
| [#262 MCU flash script fix](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/262) **(new)** | `update_klipper_mcus_sv08.sh` fails on current udev/katapult (missing `-if00`, `validate_device`, `sleep 3` race). Probably needed on Trixie. |

### Useful: specific problems

| Ref | Use |
|---|---|
| [#265 bullseye-security archive broken](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/265) **(new)** | Evidence for "why move off 2.3.4" (your exact failure class). |
| [#251 Crowsnest vs Python 3.9](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/251) | Same; the new OS resolves it. |
| [Armbian forum: CB1 Wi-Fi failures](https://forum.armbian.com/topic/55108-troubleshooting-wifi-failures/) | Wi-Fi users only. Best fix is switching netplan to NetworkManager. Source of the `wait-online` mask. Not needed on Ethernet. |
| [BTT CB1 releases](https://github.com/bigtreetech/CB1/releases) (V3.1.0, V2.3.4) | V3.1.0 = Armbian 26.05 build, kernel 7.0, Trixie. Comparison only. |
| [KIAUH issues](https://github.com/dw-0/kiauh/issues?q=clone%20klipper) | #803: on Armbian Trixie KIAUH re-prompts for `libusb` (harmless). #776: Python min-version drift. Nothing blocking on Trixie. |
| [#245](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/245) / [#241](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/241) **(new)** | Sovol addons (`z_offset_calibration`) and `PROBE_CALIBRATE` rename break on current Klipper. Step 5 problem, separate from the OS. |
| [#260](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/260), [#226](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/226) **(new)** | MCUs not enumerating after Katapult / missing `/dev/serial`. Usually wrong Katapult menuconfig. Troubleshooting section. |
| [Sovol downloads](https://www.sovol3d.com/pages/download) → [Drive "SV08 image file"](https://drive.google.com/drive/folders/10CdLCMd5jGHhtjPqmnJGEteK2nnGQku2) | Stock eMMC image for **rollback**. Link it in the guide's recovery section. |
| [lexfrei/sovobian](https://github.com/lexfrei/sovobian) **(new)** | Archived. Explains lineage; no longer an install path. |

### Background only

| Ref | Note |
|---|---|
| [PR #123 comment (glebchek dtb)](https://github.com/Rappetor/Sovol-SV08-Mainline/pull/123#issuecomment-2913783730) | Sovol-vs-CB1 DT diff (UART3, 45 MHz mmc2, regulators). Now built into the Armbian SV08 DT. |
| [PR #123 comment (ss1gohan13)](https://github.com/Rappetor/Sovol-SV08-Mainline/pull/123#issuecomment-3712260225) | Anecdotal "some run 3.0 fine". |
| [Sovol3d/SV08](https://github.com/Sovol3d/SV08) | Sovol's open-source hardware/firmware. Last push Nov 2024. Not needed for the OS. |
| [Sovol forum thread 5269](https://forum.sovol3d.com/t/updating-klipper-mainsail-moonraker-and-system/5269/3) | July 2024, stock-image update warnings. Obsolete for mainline. |
| [Reddit r/Sovol "SV08 updating"](https://www.reddit.com/r/Sovol/comments/1jebef8/sv08_updating/) | Couldn't load (site blocked here). March 2025, so predates every relevant change. Skip. |
| AliExpress listing | Makerbase MKS eMMC-Adapter V2 + 32 GB module. Matches the guide's prerequisite. |
| Google search link | Not a reference. Drop. |

---

## 4. Gotchas the new guide must handle

Items marked **[verify]** need checking on your printer before they go in the guide.

1. **Image source.** Use Armbian Imager → vendor Sovol → SV08 → Minimal Trixie, or download from the board page. **[verify]** that Imager lists Sovol boards.
2. **Flashing.** If Etcher/Rufus fails verification, decompress `.img.xz` → `.img` first (#229).
3. **No `.dtb` copy, no `BoardEnv.txt`, no `system.cfg`.** Those are CB1 2.x-only. Armbian uses `/boot/armbianEnv.txt`.
4. **`overlays=` must be ONE line, space-separated** (e.g. `overlays=uart3-pi ws2812 spidev1_1`). The ljg-dev guide copies the BTT multi-line style. With u-boot env, the last line wins, so only `spidev1_1` loads. **[verify]** whether `ws2812`/`spidev1_1` are needed at all. The stock printer.cfg drives everything over the USB MCUs.
5. **First login over Ethernet:** SSH `root@<ip>`, password `1234`. The first-run wizard creates your user and needs no HDMI. **[verify]**
6. **Filesystem resize.** Armbian expands root on first boot. ljg-dev's manual `fdisk` resize is probably unnecessary. **[verify]** with `df -h /`.
7. **Ethernet after kernel updates.** Check `ip a` (interface `end0`) after first boot **and** after a warm `sudo reboot`. If it ever breaks after `apt upgrade`: `sudo apt-mark hold linux-image-current-sunxi64 linux-dtb-current-sunxi64` and report upstream.
8. **Boot delay.** `sudo systemctl mask systemd-networkd-wait-online.service`. **[verify]** whether it's still needed on the SV08 image.
9. **User name.** It's your own user, not `biqu`. README has 6 `biqu` references; `usbmount/README.md` also has some.
10. **Step 4 changes:** drop the Bullseye `sources.list` block and the 2025-04-28 pip workaround **[verify]**. The numpy line includes `libatlas-base-dev`, which is likely gone in Trixie **[verify]**. Use KIAUH → Advanced → Extra Dependencies → Input Shaper instead.
11. **MCU flash script:** apply the #262 fixes before Step 8, or document a manual `flashtool.py` path.
12. **Stock 8 GB eMMC:** only with the SV08 image (45 MHz cap). Recommend a 32 GB module regardless.

---

## 4b. Round 2 findings (2026-10-04)

1. **Stock MCU firmware has no bootloader.** Both `firmware-backups/*.bin` images have their vector table at 0x08000000 and no second app image. The embedded data dictionary reports Klipper `v0.12.0-40-g77619e91-dirty` (built 2024-02-29 / 03-01). Any MCU reflash therefore needs an ST-Link. There is no USB path.
2. **Current Klipper host still talks to the stock MCU firmware.** `klippy/mcu.py` and `extras/bus.py` keep fallbacks for the old `analog_in_state`, `spi_set_software_bus` and `STEPPER_BOTH_EDGE` formats and flag them via `deprecate_mcu_code`. Batch-mode test: Klipper `v0.13.0-786` + stock dictionaries + `files-used/config`. `G28`, `QUAD_GANTRY_LEVEL`, `Z_OFFSET_CALIBRATION`, `BED_MESH_CALIBRATE` and `PROBE_ACCURACY` exited 0. This enables a no-ST-Link "Track 1".
3. **#245 root cause and fix.** `z_offset_calibration.py` calls `lookup_object('gcode_move')` during config load. Current Klipper hasn't created that object yet at that point. Fix: `load_object(config, 'gcode_move')`. Applied in the working tree.
4. **Why 2.3.4 can't be fixed cleanly in place.** Moonraker raised its build minimum to Python 3.10 on 2026-08-23 (`245f718`, v0.11.0). Crowsnest v5 needs 3.10+ (#251). Bullseye ships 3.9. Klipper's pins (greenlet 2.0.2 / cffi 1.14.6) still allow 3.9. The apt archive's `bullseye-security` is broken (#265). A bullseye→bookworm dist-upgrade on BTT's 5.16 kernel has no documented reports. Verdict: technically possible with pinned Moonraker v0.10.0 + snapshot apt + no Crowsnest, but a dead end. Don't document it.

## 4c. Round 3: field test on Mike's SV08 (2026-10-04)

Setup: Armbian SV08 image (Trixie minimal, kernel 6.18.54) on the MKS 32 GB eMMC, Ethernet. Steps A–E completed.

1. **The image uses Armbian's nightly repository** (`beta.armbian.com`, versions `26.11.0-trunk.NN`). The first update moved `armbian-firmware`, `armbian-bsp-cli-sovol-sv08-current` and `linux-u-boot-sovol-sv08-current` from trunk.62 to trunk.72. The kernel hold (`linux-image/dtb-current-sunxi64`) worked as intended.
2. **The first `apt full-upgrade` stalled** about 56% in, on `libasound2t64`, for 10+ minutes. Ping kept replying, but new SSH sessions failed with `kex_exchange_identification: Connection reset by peer`. A power-cycle recovered it. dmesg after reboot showed no mmc/ext4 errors, and the log from the hung boot was lost. **Root cause unknown.**
3. **The power-cycle damaged a downloaded file.** The `armbian-firmware` .deb had the correct size but the wrong SHA256, and it read back the same way twice after a cache drop, so the eMMC reads are stable. Consistent with ext4 `data=writeback` losing unflushed data. Fixed by deleting the .deb and re-downloading.
4. **Normal `dpkg --verify` noise on this image:** ~201 "missing" entries for `linux-dtb-current-sunxi64`, because the FAT `/boot` can't hold symlinks and Armbian renames `dtb-<ver>` to `dtb`. Plus 4 from `base-files` and 1 from the bsp package. Not corruption.
5. **The repair also needed** the ALSA trio (`libasound2-data libasound2t64 libatopology2t64`) upgraded together.
6. **During the repair run, with `dmesg -w` watching, the system stayed quiet.** No eMMC errors were observed. Next time, keep `dmesg -w` running from the start of any large install.
7. **Step F / Track 1 confirmed on hardware.** Klipper master connected to the stock v0.12.0-40 MCUs. Mainsail showed the "deprecated MCU code" warning, as predicted. `Z_OFFSET_CALIBRATION` with the `load_object` fix (#245) ran without errors. Test print pending (planned 2026-10-05).

## 5. Proposed repo changes (branch `sv08-cb1-v3`)

1. `docs/install-armbian.md`: new Steps 2–4 for Armbian SV08, with Track 1 (no ST-Link) and Track 2 (full). Drafted. `docs/restore-sovol-stock.md`: fallback. Drafted. `files-used/sovol-addons/z_offset_calibration.py`: #245 fix applied.
2. `README.md`: replace the "DO NOT USE V3.0.0" caution with a pointer to the new doc. Keep 2.3.4 as "Legacy". Done.
4. `docs/install-armbian.md` updated with the round-3 field results: nightly-repo note, `dmesg -w`, stall/interrupted-update recovery, `dpkg --verify` interpretation, webcam section. Done.
3. Later, separate PRs: the #262 script fix, and the Step 5 addon/config updates (#245/#241).
