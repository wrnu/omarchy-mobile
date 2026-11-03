<!-- SPDX-License-Identifier: MIT -->
# Fingerprint plan (raven / gs101): fingerprint unlock on the Pixel 6 Pro

Status (2026-11-03): **offline research only; nothing has run on the phone
for fingerprint.** The Pixel 7 Pro stack is on by default
([cheetah FINGERPRINT-PLAN](../cheetah/FINGERPRINT-PLAN.md)) and the Pixel 9a
port passed F0-F10 ([tegu FINGERPRINT-PLAN](../tegu/FINGERPRINT-PLAN.md));
this plan says what carries over to raven and what is new. Kernel branch to
create: `omamobile-gs101-fingerprint`, off `omamobile-gs101-camera-v2`
(the kernel the phone runs, `7.3.0-rc6-omamobile-push1-gcc0d345e31f0`), in
`/Volumes/omamobile-ws/mainline/wt-raven-fingerprint`, output
`/Volumes/omamobile-ws/mainline/out-raven-fingerprint` (this Mac's kernel
tree lives on the `omamobile-ws` volume, not in `~/omamobile-kernel`).

`[V]` is verified (Google's GPL tree, the stock vendor image, our captures);
`[I]` is an inference still to test; **unknown** is unknown.

Sources (private, never committed): the stock vendor image
`~/android/reference/raven-stock/ota-vendor/vendor.img` (CP3A.261005.002.A1,
read with `debugfs`, nothing mounted), `raven-stock/getprop.txt`,
`raven-stock/vendor-modules.txt`, `raven-stock/partitions-by-name.txt`, and
the stock bugreport in `~/android/reference/raven-bugreports/`. Public:
Google's GPL device tree `kernel/google-modules/raviole-device`
(`arch/arm64/boot/dts/google/gs101-raviole-fingerprint.dtsi`,
`gs101-pmic.dtsi`, `gs101-raven.dtso`; a clone at
`/Volumes/omamobile-ws/google/raviole-device`, `17dda29efd64`), its
`drivers/input/fingerprint/` (gf_spi.c, platform.c, netlink.c), and Google's
GPL panel driver `kernel/google-modules/display`
`samsung/panel/panel-samsung-s6e3hc3.c` (branch
`android-gs-raviole-5.10-android14`).

## 1. The hardware

| Fact | Value | Source |
|---|---|---|
| Board | MP1.0 (`ro.revision`), `dtbo_idx` 6; production raven is `gs101-raven.dtso` (board_id 0x20204), which takes `gs101-raviole-fingerprint.dtsi` unchanged. Only raven EVT1 boards move the interrupt (gpa6-7) [V] | getprop; GPL dts |
| Sensor | Goodix optical under-display sensor, same HAL family as cheetah and tegu (`android.hardware.biometrics.fingerprint-service.goodix`, `libgf_hal.so`, `fp_utils.so`, `biometricsuez.so`) [V]. TA **`g6.app`** (cheetah `g7.app`, tegu `g7a.app`): an older Goodix generation [V], whose IPC ports are **unknown** (cheetah's and tegu's TAs answer on `com.goodix.fingerprint.test`) | vendor `firmware/g6.app`, `init.raviole.rc` |
| Bus | SPI owned by the secure world, as on cheetah and tegu: the node sits under `/odm`, not under an SPI controller; `spi-max-frequency = 1 MHz` is for the TA side [V] | GPL `gs101-raviole-fingerprint.dtsi` |
| AP-side driver (stock) | Google's `goodixfp.ko`, compatible `goodix,fingerprint`: gf_spi/platform/netlink, `/dev/goodix_fp`, GF_IOC_* ioctls, netlink 25 [V] | `vendor-modules.txt`, GPL `drivers/input/fingerprint/` |
| AP-side driver (ours) | `drivers/misc/goodix-fp.c` (`CONFIG_GOODIX_FP=m`, already on in the shared config and built for raven), binding `goodix,g7-udfps`: the same user interface (`/dev/goodix_fp`, the ioctls, netlink 25), with the supply switched on first open [V]. Raven needs a DT node and pins only, unless `g6.app`'s HAL uses an ioctl ours ignores (**unknown**) | kernel tree, `Documentation/devicetree/bindings/misc/goodix,g7-udfps.yaml` |
| Interrupt | **`gpa9-2`**, `interrupts = <2 0>` on `gpa9`, `fp-gpio-irq` active high; our driver takes it rising-edge [V]. gpa9 also carries the MAX77759 (gpa9-3), UWB (gpa9-4) and the bottom amplifier (gpa9-6) [V] | GPL dtsi, `gs101-raviole-*.dtsi` |
| Reset | **`gpp4-3`**, active low [V]. gpp4 is USI2's bank (uart2/spi2/hsi2c2 pin groups in `gs101-pinctrl.dtsi`); USI2 is not enabled on raven [V], so the pin is free. Its bootloader state is **unknown** (F0) | GPL dtsi, our `gs101-pinctrl.dtsi`, `gs101-raven.dts` |
| Supply | S2MPG10 (main PMIC) **LDO27M**, "L27M_UDFPS_AVDD", 2.5-3.3 V, `regulator-always-on` in Google's tree [V]. Our `gs101-pixel-common.dtsi` describes `ldo27m` ("avdd_udfps", 2.5-3.3 V) without always-on [V], so the regulator core's unused-rail cleanup turns it off ~30 s after late init unless our driver claims it (AGENTS.md) [I]. Voltage at hand-off **unknown** (F0) | GPL `gs101-pmic.dtsi`, our `gs101-pixel-common.dtsi` |
| Touch helper | `fps_touch_handler.ko` is in the stock vendor modules [V], but the vendor `build.prop` has no `fps_touch_handler_supported` property (tegu's has) [V], so raven's HAL doesn't read touches from `/dev/fth_fd` [I]; the bridge feeds touches as on cheetah | `vendor-modules.txt`, vendor `build.prop` |
| Sensor position | SystemUI's `udfpsBounds=Rect(578, 2219 - 862, 2503)` in the 1440x3120 panel's pixels: centre (720, 2361), radius 142 [V]; the HAL's own location and radius are read at F6 | stock bugreport |
| Panel | `panel-samsung-s6e3hc3` (`ro.boot.primary_panel_drv`) [V]; our `panel-samsung-s6e3hc3.c` (robotosaith's display series) has **no LHBM** [V] (section 3) | getprop, kernel tree |

## 2. The software path on stock

```
Android: FingerprintService / SystemUI (touch, LHBM request)
  -> android.hardware.biometrics.fingerprint-service.goodix (AIDL IFingerprint v3)
       libgf_hal.so, fp_utils.so, biometricsuez.so
       | /dev/trusty-ipc-dev0   /dev/goodix_fp + netlink 25
       | IDisplay (com.google.hardware.pixel.display-V15) for LHBM
  -> KeyMint (keymint-service.trusty), Gatekeeper (Trusty), storageproxyd
       | RPMB (/dev/sg1) + files under /data/vendor/ss (+ persist:/ss)
  -> Trusty: g6.app, KeyMint, Gatekeeper, storage
```

From the vendor init files [V]:

- `fingerprint-goodix.rc`: service `vendor.fingerprint-goodix`, class
  late_start, user system, groups system input, priority -20; at early-boot
  it chowns `trusty_virtio`/`trusty_core` `use_high_wq` and `/dev/fth_fd`,
  and makes `/data/vendor/fingerprint`.
- `init.raviole.rc`, post-fs-data: `chown system system /dev/goodix_fp`,
  then `trusty_apploader /vendor/firmware/g6.app`. On PROTO1.0 only, the
  global-HBM fallback (`persist.fingerprint.ghbm`); the antispoof override
  properties (`vendor.fingerprint.disable.fake`) are not needed.
- `init.gs101.rc`: `storageproxyd -d /dev/trusty-ipc-dev0 -r /dev/sg1 -p
  /data/vendor/ss -t ufs`, with `/mnt/vendor/persist/ss` linked as
  `/data/vendor/ss/persist`: **the same storage layout as cheetah** (files,
  RPMB on the UFS well-known LUN), not tegu's block devices. So the runner's
  cheetah path applies (`MAP_STORES=0`) [I, F3-F4 confirm].
- `init.gs101.rc` chowns `/sys/class/backlight/panel0-backlight/local_hbm_mode`
  to system: the panel's LHBM switch, as on cheetah [V].
- KeyMint: `keymint-service.trusty` (cheetah's, not tegu's Rust one) [V];
  `keymint-service.citadel` (Titan M2) is also present and is not used by
  the fingerprint path [I].
- Vendor `build.prop`: `persist.vendor.udfps.lhbm_controlled_in_hal_supported=true`,
  `persist.vendor.udfps.als_feed_forward_supported=true` [V]; cheetah's
  `props.conf` turns these off and the bridge does LHBM.

Partitions the runner's guards read (names only) [V]: `persist` (sda1),
`mfg_data` (sdd2). Raven has no `trusty_persist` or `trusty_userdata`.

## 3. The display: LHBM on the S6E3HC3

The optical sensor images the finger by the panel's own light, so the HAL
asks for local high brightness mode (a bright circle over the sensor) before
it captures. Our S6E3HC3 driver has none of it. What Google's GPL driver
does [V, `panel-samsung-s6e3hc3.c`]:

- `MIPI_DCS_WRITE_CONTROL_DISPLAY` (0x53) with the local-HBM bit `0x10`
  added to `BCTRL 0x20` (and dimming `0x08`, HBM `0xC0` as they are).
- Before it, from panel revision EVT1 on, "extra LHBM settings" under the
  `F0`/`F1` level-2 keys: `BD 21 81`, `B1 90` (two cycles), `60 00` (120 Hz),
  `F7 0F` (update); EVT1 panels also set the circle's position (`B0 01 E7
  1F`, `1F 2D 09 39`). Off: `BD 21 82`, `B1 00`, `60 01` or `60 00` for the
  current rate, `F7 0F`. **LHBM wants 120 Hz** ("freq set will set to
  120hz when enabling LHBM").
- A per-panel LHBM gamma: read once at probe (`B0 00 22 D8`, then read
  `D8`), written back (`B0 01 59 66`, then `66` + the bytes) after every
  enable, on EVT1-DOE and EVT1.1+ panels.
- The panel revision comes from the panel's ID registers; which revision
  this phone's panel is, is **unknown** (F0 reads what our driver logs).

Port: LHBM in our `panel-samsung-s6e3hc3.c` as on cheetah's s6e3hc4
(`local_hbm_mode`, `local_hbm_max_timeout`, `allow_lhbm` module parameter
off by default, self-off timer), with the 120 Hz switch the driver already
has (`drm_panel_set_refresh_rate()`), and the gamma read/write. This is the
largest new kernel piece.

## 4. What carries over, what is new

Unchanged from cheetah [V unless marked]:
- Trusty (`trusty_core/log/ipc/virtio`), on raven at every boot for the AoC;
  the memref fix (`trusty_transfer_mapped_memory`) is in our tree;
  `allow_cpu_offline` defaults off.
- `omamobile-fp-bridge`, libfprint `omamobile_udfps`, fprintd, the PAM
  file, the lock-screen code.
- The runner's guards (read-only copies of `mfg_data` and `persist:/ss`, the
  TDP hash check, one storage proxy per boot) with raven's partition names,
  which are the same names [V].
- The extract tool's cheetah file list, with `g6.app` for `g7.app`
  (KeyMint, Gatekeeper, storageproxyd, apploader, vndservicemanager and the
  HAL are all present by the same paths [V, vendor `files.txt`]).

New for raven:
1. `tools/extract-fingerprint-vendor -d raven`: the list above. The tool
   takes a **factory** zip; locally there is only the OTA
   (`~/android/downloads/ota-raven/`), so F1 needs the factory image of the
   phone's slot A build (CP3A.261005.002.A1) or OTA payload support (the
   tool also needs `system.img` for `servicemanager` and the runtime APEX).
2. `omamobile-fp-chroot`: a `raven` case from the board's compatible
   (`google,gs101-raven`): `TA=g6.app`, cheetah's KeyMint,
   `PANEL=1440x3120`, `PANEL_NAME` and `PANEL_STATE` as the HAL expects
   (**unknown**, F6), `MAP_STORES=0`; `props-raven.conf`
   (`ro.boot.hardware.platform=gs101`, `ro.hardware=raven`);
   `omamobile-tipc-probe` ports for `g6.app` (**unknown**).
3. Kernel, branch `omamobile-gs101-fingerprint`: a disabled
   `goodix,g7-udfps` node in `gs101-raven.dts` (gpa9-2 rising, gpp4-3 active
   low, `vdd-supply = <&ldo27m>`, pin states for both), an overlay to enable
   it, and LHBM in `panel-samsung-s6e3hc3.c` (section 3). Whether the
   binding's `g7-udfps` name fits a G6 sensor is a review question; the
   interface is the same.
4. The panel option in `omamobile-fingerprint-lhbm.conf`
   (`options panel_samsung_s6e3hc3 allow_lhbm=1`).
5. The lock UI's sensor target for raven's position (centre (720, 2361),
   radius 142 at 1440x3120).

## 5. Decision needed before F4: secure storage

**F4 is irreversible.** The first `storageproxyd` run from Linux writes the
RPMB (the write counter and Trusty's superblocks) and starts a new
tamper-detect store (TD) under `/data/vendor/ss` on userdata.

- **Effect on stock Android in slot A:** its TD lived on the old Android
  userdata, which is already gone (Linux owns userdata). Stock's own
  fingerprints, Gatekeeper and KeyMint state are lost either way; booting
  stock again needs a factory reset, as on cheetah. After F4 the RPMB's
  counter has moved on, and a stock boot would see its secure storage as
  new.
- **TDP (`persist:/ss`)**, cheetah's option A (the default, recommended):
  copy `persist:/ss` once, read-only, and serve the copy to the chroot;
  nothing is written to `persist`, so stock's TDP goes stale for stock.
  Option B (write the real `persist`) needs a storage-guard change and the
  human's explicit go; not proposed.

Record the decision, A or B, and the date here before F4.

## 6. Effort and risks

| Item | Risk | Notes |
|---|---|---|
| F4 secure storage | **Irreversible**, the human's decision | As section 5 |
| `g6.app` vs `g7.app` | Medium | Older TA: its ports, version check and the HAL's expectations of our driver are unknown until F2 and F6 |
| LHBM on S6E3HC3 | Medium | New kernel code, panel-revision dependent, wants 120 Hz; a wrong command can leave the panel bright or dark until a blank/unblank. Stop on flicker or a circle in the wrong place |
| LDO27M | Low-medium | Must be on when the HAL opens the sensor; the unused-rail cleanup can turn it off before that. Our driver enables it on open |
| HAL dependencies | Medium | As cheetah: absent services short-circuited by the shim, props from the vendor `build.prop`; expect one or two shim/prop iterations |
| Factory image | Low | F1 needs the factory zip (or OTA payload support) for the extract tool |
| Shared pins | Low | gpa9 carries three other users; the reset bank gpp4 is free |

Effort: about six phone sessions if nothing surprises (FB+F0, F2-F3, F4-F5,
F6, F7, F8-F10), each one boot at most, plus the offline LHBM port.

## 7. Runbook

Rules (cheetah section 4 applies): announce anything that affects the
screen; one change per boot; the kernel log streamed to the laptop for any
driver load; never stop and start `omamobile-fp` or storageproxyd in one
boot; never write `persist` or `mfg_data`.

### FB: backups (read-only; risk none)

`tools/nv-backup --check ~/android/reference/raven-backup-2026-10-23`.
**Pass:** every partition matches. **Stop:** any mismatch. If stock Android
has booted since 2026-10-23 (a slot A fallback counts), take a fresh backup
first (`tools/nv-backup <new dir>`), and check against that. Add
`mfg_data` to the backup set (it isn't in the 2026-10-23 set): one
read-only `dd` of the node found by `PARTNAME`, hashed on the phone and the
laptop, into the backup directory (mode 600). Rollback: none.

### F0: read-only facts on the phone (risk none)

1. Trusty: `/dev/trusty-ipc-dev0` exists, `cat
   /sys/module/trusty_core/parameters/allow_cpu_offline` is `N`, `grep -c
   trusty_transfer_mapped_memory /proc/kallsyms` > 0.
2. LDO27M: on or off and its voltage at hand-off
   (`/sys/kernel/debug/regulator/regulator_summary`, the `avdd_udfps` line).
3. gpa9-2 and gpp4-3 as the bootloader left them (one controller's pins in
   `/sys/kernel/debug/pinctrl/<controller>/pins`, never all controllers at
   once: AGENTS.md gs101 note). Expect gpp4-3 output, high (reset released)
   [I].
4. The RPMB node: `modprobe sg`, `ls -l /sys/class/scsi_generic/*/device`
   (expect one at LUN 49476), `modprobe -r sg`.
5. The panel revision our S6E3HC3 driver logs at probe (`dmesg | grep -i
   s6e3hc3`).

**Stop:** a Trusty check fails (port the fix first). Rollback: none.

### F1: offline build (laptop; no phone)

The extract tool with `-d raven` from the owner's factory image (the tree
stays in `out/`); the runner's raven case and `props-raven.conf`; the
kernel branch with the disabled node, its overlay and LHBM in the S6E3HC3
driver; checkpatch `--strict`, `W=1`, `dt-validate`;
`packages/omamobile-fingerprint/test/all` and
`packages/libfprint/test/run`. **Pass:** all clean. Rollback: none.

### F2: Trusty ports (read-only; risk low)

Connect and close with `omamobile-tipc-probe` to `apploader`, KeyMint and
Gatekeeper; never `storage.proxy`. **Expect** `open` for all three.
**Stop:** a refused or hung port. Rollback: none.

### F3: read-only copies (no writes outside userdata; risk low)

`omamobile-fp-chroot check`, then its read-only copies of `mfg_data` and
`persist:/ss/{0,nsp}` (debugfs, `persist` never mounted) into
`/var/lib/omamobile-fp/` (0700). **Stop:** a size that differs from the
partition, or `persist` mounted. Rollback: delete the copies.

### F4: secure storage, IRREVERSIBLE (the first RPMB write from Linux)

Only after FB, F3 and section 5's recorded decision. Install the package
and the vendor tree (no Trusty writes), then once:

```sh
tools/phone-ssh 'systemd-run --unit=omamobile-fp-stage -p KillMode=control-group /usr/lib/omamobile-fp/omamobile-fp-chroot start storage'
```

**Expect:** servicemanager, vndservicemanager, storageproxyd serving RPMB
requests, `fs_ready`, Trusty's log mounting its stores (TD new). **Stop:**
authentication or MAC failures, storage errors: no retry, collect the logs.
Rollback: stop the unit; the RPMB can't be rolled back. Risk:
irreversible for the RPMB, low for the running system.

### F5: the TA (risk low)

`start apploader`: "g6.app loaded" within seconds, the TA's test port
`open` (its name from F2), no oops. **Stop:** a 60 s loader timeout
(storage not up), an oops, a Trusty policy error. Rollback: stop the unit
(the app stays loaded until reboot).

### F6: the sensor and the HAL (one boot for the DT; risk low-medium)

F6a: boot with the node enabled by the overlay, `goodix_fp` blacklisted:
pins as F0, touch, the MAX77759 and the speakers still work (shared gpa9).
F6b: `modprobe goodix_fp`, one open and close: LDO27M on, an interrupt on
reset. F6c: `start hal`: "HAL/TA version is match", "Initialization
successful", and **the HAL's sensor location and radius**, recorded here
(against SystemUI's (720, 2361), 142). **Stop:** a service lookup that
waits (one shim or prop change per run), a HAL crash, KeyMint exiting.
Rollback: stop the unit, `modprobe -r goodix_fp`, the DTB without the
overlay.

### F7: LHBM on the S6E3HC3 (announce: a very bright circle and a 120 Hz switch; risk medium)

Unlocked, screen lit, panel at 120 Hz: `allow_lhbm=1`, `echo 1 >
local_hbm_mode`, 3 s, self-off. **Expect:** a circle at F6's location and
size. **Stop:** a circle elsewhere or of the wrong size, flicker that stays,
a panel that misbehaves after off, LHBM accepted at 60 Hz. Rollback: `echo
0 > local_hbm_mode`, blank and unblank, `allow_lhbm=0`, reboot.

### F8-F10: enroll, verify, fprintd, lock screen (as cheetah F8-F10)

Enroll through the bridge (writes a template to Trusty storage), a match
and a non-match, then libfprint and fprintd, then the PAM file and
`omamobile-fp.service`. **Stop:** auth-token refusals, lockouts, TOO_DARK
on every capture. Rollback: delete the enrolled id, remove the PAM file,
`systemctl mask --now omamobile-fp`.

## 8. Results

None yet.
