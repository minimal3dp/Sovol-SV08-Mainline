# Fallback: Sovol's Latest Stock Image

Use this if mainline doesn't work out. You end up on Sovol's own Klipper build: older, but what the printer shipped with.

## Option 1: Your original eMMC (fastest)

If you never wrote to the original module:

1. Power off, unplug, lay the printer on its back, remove the 6-screw bottom plate (README Step 1).
2. Swap the original eMMC back in (mind the arrow).
3. Power on. Done.

If you never flashed the MCUs with an ST-Link, they are still stock and nothing else is needed.

## Option 2: Sovol's newest image on a spare module

Use this to get Sovol's latest release, or if the original module was overwritten.

1. Open the [Sovol download page](https://www.sovol3d.com/pages/download) → **SV08 image file (Need to use EMMC-adapter)** ([Google Drive folder](https://drive.google.com/drive/folders/10CdLCMd5jGHhtjPqmnJGEteK2nnGQku2)).
2. Download the newest image in the folder. Check whether the folder has a readme or a separate **SV08 Firmware Upgrade** folder for that release.
3. Put the spare eMMC in the USB adapter and flash the image with Raspberry Pi Imager or balenaEtcher. If verification fails, decompress the image first.
4. Install the module, power on, and wait for the stock screen.
5. Mainsail is at the printer's IP. SSH is `sovol` / `sovol`.
6. Restore your `printer.cfg` calibration values (z-offset, PID) from your backup, or re-run the calibrations.

> [!NOTE]
> The stock image is about 8 GB. On a 32 GB module the rest of the space stays unused. That's harmless.

## Rules on stock firmware

1. Don't enable Moonraker's update manager for Klipper/Moonraker.
2. Don't run KIAUH updates on the stock image.

Sovol's Klipper is modified and breaks when updated ([Sovol forum](https://forum.sovol3d.com/t/updating-klipper-mainsail-moonraker-and-system/5269/3)). Updates come only from new Sovol images.

## MCU firmware

- **Never flashed with ST-Link:** the MCUs are already stock. Nothing to do.
- **Flashed Katapult/mainline (README Steps 7–8):** restore the backups from README Step 6 with the ST-Link. This repo's `firmware-backups/` holds a fallback copy from a June 2024 EU printer.
- If Sovol's newest image ships newer MCU firmware, follow the readme in Sovol's **SV08 Firmware Upgrade** folder.
