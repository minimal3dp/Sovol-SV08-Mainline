# Mainline Klipper on the SV08 with Armbian (Debian 13)

This replaces Steps 2–4 of the main [README](../README.md). It installs the official **Armbian "Sovol SV08"** image (Debian 13 Trixie), then current mainline Klipper, Moonraker and Mainsail via KIAUH.

> [!CAUTION]
> Work in progress. As of 2026-10, Armbian lists the SV08 image as *community supported*, and its maintainer has not verified it on SV08 hardware. Use a **spare eMMC module** and keep the stock module untouched so you can roll back.

> [!NOTE]
> **Test status (2026-10-04):** Steps A–F have been completed on one SV08 using **Track 1** (stock MCU firmware): Ethernet, Makerbase MKS 32 GB eMMC, Armbian SV08 image with kernel 6.18.54. Klipper connected to both stock MCUs, Mainsail showed the expected "deprecated MCU code" warning, and `Z_OFFSET_CALIBRATION` ran without errors with the #245 fix. A full test print is still pending. Please report your results in an issue.

## Two ways to finish

| | **Track 1: host only (no ST-Link)** | **Track 2: full mainline** |
|---|---|---|
| OS + Klipper/Moonraker/Mainsail | Mainline, latest | Mainline, latest |
| MCU firmware (mainboard + toolhead) | Sovol stock (Klipper v0.12.0-40) | Mainline Klipper via Katapult |
| ST-Link needed | No | Yes (README Steps 6–8) |
| Rollback | Swap the eMMC back. MCUs untouched. | Swap the eMMC back **and** reflash the MCU backups |
| Limits | "Deprecated MCU code" warnings. No accelerometer until reflash. Klipper may drop compatibility with old MCU code in a future update. | None known |

Track 1 is a fully reversible way to run mainline. Do Track 2 later when you have an ST-Link.

> [!NOTE]
> **Track 1 evidence.** Klipper master (`v0.13.0-786`, 2026-10-01) has fallback code for older MCU firmware. It was run in batch mode against the data dictionaries extracted from the stock SV08 mainboard and toolhead firmware (`firmware-backups/`), using this repo's `files-used/config`. Config load, `G28`, `QUAD_GANTRY_LEVEL`, `Z_OFFSET_CALIBRATION` and `BED_MESH_CALIBRATE` all completed. It has since been confirmed on real hardware: see the test status above.

---

## Prerequisites

1. A spare eMMC module + USB eMMC adapter (e.g. Makerbase MKS EMMC-ADAPTER V2 + 32 GB module).
2. An **Ethernet** connection for first boot. Wi-Fi works too but needs HDMI + keyboard for first login.
3. The config backup from your stock printer (README Prerequisites).
4. A computer that can open **two SSH sessions** at once. The second one watches the kernel log during big installs.

---

## Step A: Collect info from the stock printer

1. Boot the printer on the **stock** eMMC.
2. Back up your config: Mainsail → Machine → select all → Download.
3. SSH in (`sovol` / `sovol`) and record the MCU serials:
   ```bash
   ls -l /dev/serial/by-id/
   ```
4. Record the MCU firmware versions:
   ```bash
   grep -m2 "Loaded MCU" ~/printer_data/logs/klippy.log
   ```
   Expected: `v0.12.0-40-g77619e91-dirty-...`. If yours differs, note it in your issue/PR.
5. `sudo poweroff`, unplug, remove the stock eMMC (README Step 1). Label it **STOCK** and set it aside.

## Step B: Flash Armbian to the spare eMMC

1. Install [Armbian Imager](https://imager.armbian.com/).
2. Put the spare eMMC in the USB adapter (mind the arrow) and plug it into your computer.
3. In Armbian Imager pick **Sovol → Sovol SV08 → Minimal → Debian 13 (Trixie)**.
   - If Sovol isn't listed, download the image from <https://www.armbian.com/boards/sovol-sv08> and flash it with Raspberry Pi Imager or balenaEtcher.
   - If Etcher fails verification, decompress `.img.xz` to `.img` first.
4. Select the eMMC drive → flash → eject.

> [!IMPORTANT]
> Do **not** copy `sun50i-h616-sovol-emmc.dtb`, and do **not** edit `BoardEnv.txt` or `system.cfg`. Those are for the old CB1 2.3.4 image only. The Armbian SV08 image already has the SV08 device tree.

## Step C: First boot and login

1. Install the eMMC in the printer, connect Ethernet, power on.
2. Find the printer's IP in your router (look for a new device, likely named `sovol-sv08`).
3. SSH in as root:
   ```bash
   ssh root@<printer-ip>     # password: 1234
   ```
4. Complete the first-run wizard: new root password, **create your user**, locale/timezone. Skip Wi-Fi if you're on Ethernet.
5. Log out and SSH back in as your new user.

## Step D: Prepare the OS

> [!NOTE]
> The SV08 image uses **Armbian's nightly repository** (`beta.armbian.com`), so Armbian packages show versions like `26.11.0-trunk.72`. Debian packages come from the normal Debian mirrors. This is how Armbian ships community-supported boards. It's why Step D.2 holds the kernel.

1. Check Ethernet is up:
   ```bash
   ip -br a        # end0 should be UP with an IP
   ```
2. Hold the kernel so an update can't break networking. Kernel 6.18.33 broke H616 Ethernet earlier in 2026 (since fixed). Without HDMI you would be locked out.
   ```bash
   dpkg -l 'linux-image-*' 'linux-dtb-*' | grep ^ii      # note exact package names
   sudo apt-mark hold linux-image-current-sunxi64 linux-dtb-current-sunxi64
   apt-mark showhold                                     # both should be listed
   ```
3. Open a **second** SSH session and leave the live kernel log running in it during the whole update:
   ```bash
   sudo dmesg -w
   ```
4. In the first session, update everything else:
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   ```
   This takes 10–20 minutes. Lines like `Not upgrading: linux-dtb-current-sunxi64 linux-image-current-sunxi64` are expected; that's the hold working.

> [!WARNING]
> **Don't power off during an update** unless it's truly hung. A hard power-off mid-update can leave files with the right size but the wrong contents. The root filesystem uses ext4 writeback mode. To tell slow from hung, see [If an update stalls or is interrupted](#if-an-update-stalls-or-is-interrupted).

5. Confirm the package system is clean. This should print **nothing**:
   ```bash
   sudo dpkg --audit
   ```
6. Stop the boot delay from network wait, then reboot:
   ```bash
   sudo systemctl mask systemd-networkd-wait-online.service
   sudo reboot
   ```
7. SSH back in and confirm Ethernet survived a warm reboot and the root filesystem filled the eMMC:
   ```bash
   ip -br a; df -h /        # end0 UP; / about 28G on a 32 GB module
   ```

## Step E: Install Klipper, Moonraker, Mainsail

1. Install git and KIAUH:
   ```bash
   sudo apt install git -y
   cd ~ && git clone https://github.com/dw-0/kiauh.git
   ./kiauh/kiauh.sh
   ```
2. In KIAUH → **Install**, in this order: **Klipper**, **Moonraker**, **Mainsail** (defaults; download recommended macros = yes). Add **Crowsnest** if you use the camera (see [Webcam](#optional-webcam)).
3. Input-shaper dependencies: KIAUH → **Advanced** → **Extra Dependencies** → **Input Shaper**. Don't use the README's manual `apt install ... libatlas-base-dev` line; that package isn't expected on Debian 13.
4. Quit KIAUH and `sudo reboot`.
5. Open `http://<printer-ip>` in a browser. Mainsail should load. Klipper errors are expected until Step F.

> [!NOTE]
> Skip the README's Step 4 `sources.list` block and the 2025-04-28 pip workaround. Both are for Debian 11 only.

## Step F: Configure the printer (README Step 5, with changes)

1. Follow README **Step 5**, with these differences:
   - The username is the one you created, **not** `biqu`.
   - Copy `files-used/sovol-addons/*.py` to `~/klipper/klippy/extras/`. Use this repo's current `z_offset_calibration.py`, which has the fix for #245.
2. In `printer.cfg`, set the serials under `[mcu]` and `[mcu extra_mcu]` from `ls /dev/serial/by-id/`. With stock firmware they start `usb-Klipper_stm32f103xe_` and end `-if00`.

> [!WARNING]
> Do the **toolhead-unplug test** from README Step 8 to confirm which serial is `mcu` and which is `extra_mcu`. If they're swapped, **the heater can turn on full blast.**

3. `FIRMWARE_RESTART` in Mainsail.
4. Expected on **Track 1**: a Mainsail warning that each MCU "has deprecated code… recompiling and flashing is recommended." This is normal with stock MCU firmware.
5. Test in this order, ready to hit Emergency Stop:
   1. Temperatures read sensibly (room temp)
   2. `G28 X`, then `G28 Y`
   3. `G28 Z`
   4. `QUAD_GANTRY_LEVEL`
   5. `Z_OFFSET_CALIBRATION`
   6. `BED_MESH_CALIBRATE`
   7. PID tune extruder and bed
   8. A small test print

## Optional: Webcam

Do this **after Step F**. Step F copies this repo's `crowsnest.conf`, which overwrites any camera settings made earlier.

1. Check the camera is detected and copy its stable path:
   ```bash
   lsusb | grep -i cam; ls -l /dev/v4l/by-id/
   ```
   Use the entry ending in `-video-index0`.
2. If Crowsnest isn't installed yet: `./kiauh/kiauh.sh` → **Install** → **Crowsnest** → reboot. Crowsnest v5 needs Python 3.10+, so it works here (Debian 13 has 3.13), unlike on the old image (#251).
3. Edit `~/printer_data/config/crowsnest.conf`. In `[cam 1]`, replace `device: /dev/video0` with the `/dev/v4l/by-id/...-video-index0` path. Then:
   ```bash
   sudo systemctl restart crowsnest
   ```
4. In Mainsail: **Settings (gear)** → **Webcams** → **Add webcam**:
   - **Stream URL:** `/webcam/?action=stream`
   - **Snapshot URL:** `/webcam/?action=snapshot`
   - **Service:** MJPEG-Streamer (adaptive)
5. No picture? Run `sudo journalctl -u crowsnest -n 30 --no-pager` and check the log for the detected devices.

## Step G: Finish

- **Track 1:** you're done. Before updating Klipper later, read the Mainsail warnings. If a future Klipper drops support for the old MCU code, Klipper will refuse to connect. Either roll Klipper back (`cd ~/klipper && git checkout <previous-commit>`) or do Track 2.
- **Track 2:** README **Steps 6–8** (ST-Link backup → Katapult → Klipper firmware). Use the fixed `update_klipper_mcus_sv08.sh` from #262 if the stock script fails with "No Serial Device found".

---

## If an update stalls or is interrupted

These steps come from a real stall during Step D.4 (2026-10-04). Do them in order.

### 1. Slow or hung?

Leave the stuck window alone. In a new terminal:

| Check | Result | Meaning |
|---|---|---|
| `ssh <user>@<ip>` | Connects | Still alive. In the new session, check the `dmesg -w` window and run `vmstat 2 5`. Non-zero `bo` means it's still writing, so wait. |
| `ssh <user>@<ip>` | `kex_exchange_identification: Connection reset by peer` | Badly stalled. Go to the ping check. |
| `ping -c 5 <ip>` | Replies, but SSH keeps resetting for 5+ min | The kernel is alive but storage is hung. It won't recover on its own; power-cycle. |
| `ping -c 5 <ip>` | No replies | Fully frozen; power-cycle. |

Before power-cycling, screenshot the `dmesg -w` window. Lines with `mmc2`, `timeout`, `I/O error` or `EXT4-fs error` point to the eMMC module or its link to the board.

### 2. Repair the package system after a power-cycle

1. Check what was left half-done (read-only):
   ```bash
   sudo dmesg | grep -iE 'mmc|i/o error|ext4|blk_update'; echo ---; sudo dpkg --audit
   ```
2. With `dmesg -w` running in a second session:
   ```bash
   sudo dpkg --configure -a
   sudo apt --fix-broken install -y
   ```
3. If apt reports `libasound2t64 : Depends: libasound2-data (>= ...) but ... is to be installed`, upgrade the ALSA packages together:
   ```bash
   sudo apt install -y libasound2-data libasound2t64 libatopology2t64
   ```
4. If `dpkg-deb` reports `lzma error: compressed data is corrupt` for a package, the downloaded `.deb` was damaged by the power-off. Confirm it, delete it, and re-download:
   ```bash
   cd /var/cache/apt/archives
   apt-cache show <package>=<version> | grep ^SHA256      # expected checksum
   sha256sum <package>_<version>_*.deb                     # actual checksum
   sudo rm <package>_<version>_*.deb                       # if they differ
   sudo apt --fix-broken install -y
   ```
5. Finish and confirm:
   ```bash
   sudo apt full-upgrade -y
   sudo dpkg --audit        # must print nothing
   ```

### 3. Check for other damaged files

```bash
sudo dpkg --verify 2>&1 | grep -v ' c /' > ~/verify.txt; wc -l < ~/verify.txt
awk '{print $NF}' ~/verify.txt | xargs -r dpkg -S 2>/dev/null | cut -d: -f1 | sort | uniq -c | sort -rn | head
```

Expected and harmless on this image:

- **~200 lines from `linux-dtb-current-sunxi64`.** `/boot` is FAT and can't hold symlinks, so Armbian renames `/boot/dtb-<version>` to `/boot/dtb`. dpkg then reports the files as "missing" from the original path.
- **A few lines from `base-files`** (Armbian edits its OS identity files) and **one from `armbian-bsp-cli-sovol-sv08-current`**.

Anything else, spread across many unrelated packages, means real damage. Reflash the module and start again at Step C.

---

## Rollback

1. Power off, unplug.
2. Swap the **STOCK** eMMC back in.
3. **Track 1:** done; the printer is fully stock.
4. **Track 2:** also reflash `mainboard_original_firmware.bin` / `toolhead_original_firmware.bin` with the ST-Link.

## Troubleshooting

| Symptom | Fix |
|---|---|
| No IP in router after 5 min | Reflash. Try Raspberry Pi Imager instead of Etcher. Check the Ethernet cable/port. |
| Ethernet gone after an update | You updated the kernel. Reflash, and do Step D.2 before upgrading. |
| Update stuck on one package for 10+ min; new SSH sessions reset | See [If an update stalls or is interrupted](#if-an-update-stalls-or-is-interrupted). |
| `lzma error: compressed data is corrupt` | Damaged download from an interrupted update. Delete that `.deb` and re-run (section above, item 4). |
| `ln: failed to create symbolic link '/boot/uInitrd': Operation not permitted` | Normal. `/boot` is FAT and Armbian falls back to renaming the file. |
| Wi-Fi drops / won't reconnect | Switch to NetworkManager: `sudo apt install network-manager`, set `renderer: NetworkManager` in `/etc/netplan/*.yaml`, `sudo netplan apply` ([Armbian forum](https://forum.armbian.com/topic/55108-troubleshooting-wifi-failures/)) |
| `/dev/serial/by-id` empty | Check the USB cables to both boards. On Track 2, recheck Katapult menuconfig (#226, #260). |
| `Unknown config object 'gcode_move'` | Old `z_offset_calibration.py`. Use this repo's current version (#245). |
| Webcam: no picture | Set `device:` to the `/dev/v4l/by-id/...-video-index0` path, restart Crowsnest, check `journalctl -u crowsnest`. |

## Sources

- Armbian SV08 board page: <https://www.armbian.com/boards/sovol-sv08>
- SV08 board support: [armbian/build#10255](https://github.com/armbian/build/pull/10255)
- H616 Ethernet fixes: [#10430](https://github.com/armbian/build/pull/10430), [#10492](https://github.com/armbian/build/pull/10492), [forum thread](https://forum.armbian.com/topic/59962-btt-pi-ethernet-not-working-after-upgrade-from-61268-to-61833/)
- Community reports: [#229](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/229), [PR #123](https://github.com/Rappetor/Sovol-SV08-Mainline/pull/123), [ljg-dev notes](https://github.com/ljg-dev/sovol-sv08-mainline)
- Crowsnest on Python 3.9: [#251](https://github.com/Rappetor/Sovol-SV08-Mainline/issues/251)
