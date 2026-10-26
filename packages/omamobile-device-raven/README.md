<!-- SPDX-License-Identifier: MIT -->
# omamobile-device-raven

The Google Pixel 6 Pro's (raven, gs101) system files for Omarchy Mobile,
the counterpart of `omamobile-device-cheetah`. Bring-up and results:
[device/raven](../../device/raven/NOTES.md).

| What | Files |
| --- | --- |
| Slot B marked successful after every boot, so ABL stops falling back to slot A | `omamobile-boot-success` (+ `.service`, enabled), `omamobile-bootctl` |
| `reboot-to <reason>` (ABL reads the reason from the MAX77759's nvmem) | `reboot-to` |
| USB that survives a replug: closes the MAX77759 USB2 data switch while VBUS is present, until a Type-C driver owns the chip | `omamobile-usb-rescue` (+ `.service`, enabled), `omamobile-i2c` |
| Module policy: no MAX77759 charger, TCPC or GPIO driver; DSIM DCS allowed; btbcm streams the Bluetooth patch | `modprobe.d/omamobile-raven.conf` |
| On-demand CPU frequency (safe once the thermal zones throttle) | `tmpfiles.d/omamobile-raven-cpufreq.conf` |
| The user may switch the panel off (`bl_power`) and inhibit the touchscreen, for Omarchy's display off | `udev/rules.d/20-omamobile-display.rules` |
| Wi-Fi power save on | `NetworkManager/conf.d/omamobile-raven-wifi-powersave.conf` |
| Sleep (s2idle) once locked with the panel off, behind `omarchy-toggle omamobile-sleep on`; Wi-Fi stays connected across sleep (unloaded instead after a D3 failure, or with `/etc/omamobile/wifi-unload-for-sleep`) | `omamobile-sleepd` (+ user `.service`, enabled), `omamobile-sleep` (`system-sleep/omamobile`), `sleep.conf.d/10-omamobile.conf` |
| Modem bring-up only: its PCIe driver and CPIF never load at boot | `modprobe.d/omamobile-raven-modem.conf` |
| Bluetooth on at boot; the session waits (at most 15 s) for the adapter | `bluetooth.service` (enabled), `sddm.service.d/`, `omamobile-wait-bluetooth` |
| The ultrawide camera, loaded 40 s into the boot after a hang guard (the next boot skips it if a load never finished; `--retry` tries again); the camera modules blacklisted for autoload; the DMA-BUF heaps for the video group and the seat's user (libcamera's software ISP); the camera named for apps and the raw CSIS nodes kept out of PipeWire | `omamobile-camera-start` (+ `omamobile-camera.service`, `.timer`, enabled), `modprobe.d/omamobile-camera.conf`, `udev/rules.d/70-omamobile-dma-heap.rules`, `wireplumber/50-omamobile-cameras.conf` |
| Lens shading for the ultrawide from its own module's factory flat field (the module EEPROM, read-only through at24): checked against its checksums, fitted to the software ISP's radial model (colour fully, brightness 80 %), written to `/etc/libcamera/ipa/simple/imx386.yaml` before the camera appears; the numbers stay on the phone | `omamobile-camera-calibrate` (run by `omamobile-camera-start`) |

`reboot-to.c` and `omamobile-bootctl.c` are cheetah's, unchanged: both are
the same on every gs101/gs201 Pixel (raven's `devinfo` is `DEVI` 3.15, as
cheetah's). `omamobile-bootctl.c` is a port of Google's Apache-2.0 boot
control code and keeps that licence (`LICENSE.Apache-2.0`).

The touchscreen rule only takes effect with the Omarchy patch that makes
display off inhibit writable touchscreens (patch 0008 in
`omamobile-omarchy`); without it, it does nothing.

## Build and test

```sh
tools/build-packages packages/omamobile-device-raven   # aarch64, into out/repo
packages/omamobile-device-raven/test/all               # on any machine
```
