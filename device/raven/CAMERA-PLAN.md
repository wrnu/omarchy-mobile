<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 Pro (raven): cameras

Status (2026-10-23): **RC0 done (read-only, §5); RC1.1 and RC1.2 pass on the phone (§5); RC1.3 passes: the ultrawide's module EEPROM answers (§5); RC1.4 (the IMX386's ID) built and checked offline, waiting for the push.** No camera driver binds on
raven, so there's no `/dev/video*` or `/dev/media*`, and omacam now says "No camera
found" after 20 s (omacam `4920def`, local). Kernel branch
`omamobile-gs101-camera` (from `omamobile-gs101-int` `0422abf64`), worktree
`$WORK/mainline/wt-raven-camera`, output `$WORK/mainline/out-raven-camera`.

Cheetah's [CAMERA-PLAN](../cheetah/CAMERA-PLAN.md) is the reference: all
four of its cameras stream through drivers that are already in raven's tree.
This plan records what is the same on gs101, what differs, and the order to
bring raven up. Path (a) as on cheetah: raw capture through CSIS WDMA,
libcamera's simple pipeline with the software ISP, no Google ISP and no LWIS.

## 1. Facts

`G/` is Google's GPL raviole device tree (`kernel/google-modules/raviole-device`,
branch `android17-6.18-gs101`, `17dda29`), `GD/` its `arch/arm64/boot/dts/google/`.
`C/` is cheetah's CAMERA-PLAN, `L/` our kernel (`omamobile-gs101-int`).
[V] verified in the source, [I] inferred.

### 1.1 The four cameras

| Camera | LWIS node | Sensor | I2C bus (base), addr | Reg width | Reset | MCLK (clock, pad, rate) |
|---|---|---|---|---|---|---|
| Main | `sensor-gn1` | Samsung GN1 [V name; S5KGN1 part [I]] | `hsi2c_1` (`0x10900000`), `0x3d` | 16/16 | gpp2-2 | `CIS_CLK3`, gpp9-0, 24 MHz |
| Front | `sensor-imx663` | Sony IMX663 [V] | `hsi2c_2` (`0x10910000`), `0x1a` | 16/8 | gpp4-2 | `CIS_CLK1`, gpp5-0, 24 MHz |
| Ultrawide | `sensor-imx386` | Sony IMX386 [V] | `hsi2c_3` (`0x10920000`), `0x1a` | 16/8 | gpp6-3 | `CIS_CLK0`, gpp3-0, 24 MHz |
| Tele | `sensor-imx586` | Sony IMX586 [V] | `hsi2c_4` (`0x10930000`), `0x1a` | 16/8 | gpp6-2 | `CIS_CLK2`, gpp7-0, 24 MHz |

Evidence: `GD/gs101-raven-camera.dtsi:139-454` (node names at 143, 224, 297,
370); bus bases `GD/gs101-usi.dtsi`. Stock runs buses 1, 2, 4 at 1 MHz and
bus 3 at 950 kHz (`gs101-raven-camera.dtsi:14-119`).

Other parts on the buses [V] (`gs101-raven-camera.dtsi:456-862`,
`gs101-raviole-ldaf.dtsi:18-41`):

| Bus | Parts besides the sensor |
|---|---|
| `hsi2c_1` | `lc898128` at `0x24` (OIS controller holding the module EEPROM, 32-bit values), AF `ak7377` at `0x0c`, laser AF `stmvl53l1` at `0x29` |
| `hsi2c_2` | EEPROM `m24c64x` at `0x51` (front: fixed focus) |
| `hsi2c_3` | EEPROM `m24c64x` at `0x50`; **no AF actuator** (cheetah's ultrawide has one) |
| `hsi2c_4` | `sem1215sa` at `0x34`: EEPROM, OIS and AF in one, as on cheetah's tele |
| `hsi2c_7` (`0x10960000`) | the torch `lm3644` at `0x63` and the camera PMIC at `0x75` |

### 1.2 Power: an SLG51000, not cheetah's SLG51002

- **Camera PMIC: Dialog SLG51000** at `0x75` on `hsi2c_7` [V]
  (`GD/gs101-raviole-camera-pmic.dtsi:7-13`), driven by four S2MPG11 GPIOs:
  GPIO 1 `bb`, GPIO 3 `buck`, GPIO 0 `cs`, GPIO 4 `pu`. Google's order
  (`G/drivers/mfd/slg51000-core.c:298-339`): bb high, 2 ms; buck high, 2 ms;
  cs high, 10 ms; pu high, 1 ms. Off is the reverse. No `dlg,reg-init`, so
  stock writes no trim values.
- **Seven LDOs; the DT gives ranges, not voltages** [V, `:29-69`]. LWIS only
  enables them, so the voltages are the chip's defaults [I]. RC1.2 reads the
  selectors before anything is enabled.

  | LDO | Used by |
  |---|---|
  | LDO1 | every part (sensors, EEPROMs, AF, OIS, laser AF): the I/O rail [I] |
  | LDO2, LDO3 | main sensor, `lc898128`, `ak7377` |
  | LDO4 | ultrawide |
  | LDO5, LDO6 | tele sensor and `sem1215sa` |
  | LDO7 | front |

- **S2MPG11 GPIO 2** is the tele's `enable-gpios` (`gs101-raven-camera.dtsi:389-391`),
  so S2MPG11 GPIOs 0-4 are all camera. Their bank is PM `0x98-0x9e`, one bit
  per GPIO (`G/include/linux/mfd/samsung/s2mpg11-register.h:105-111`): **not**
  S2MPG13's per-GPIO `SET` registers, so `gpio-s2mpg13` doesn't apply as is.
- **Power-up sequences** [V, `gs101-raven-camera.dtsi`]:
  - Ultrawide: LDO1, 3 ms; LDO4, 1 ms; MCLK, 1 ms; reset high, 10 ms.
  - Front: LDO1, LDO7, MCLK, reset high, 8 ms.
  - Main: LDO1, LDO2, LDO3 (1 ms each); reset high, 1 ms; MCLK, 9 ms (reset before MCLK, as on cheetah).
  - Tele: LDO1, LDO5 (3 ms), enable GPIO, LDO6, MCLK, reset high, 8 ms.
  - EEPROMs `0x50` and `0x51`: LDO1 only, 1 ms. **The first safe targets.**

### 1.3 The capture path: the same IP as gs201

[V] `GD/gs101-isp.dtsi:19-170`, `gs101-sysmmu.dtsi:83-127`, `gs101-s2mpu.dtsi:62-74`,
`gs101-pm-domains.dtsi:143-161`, `G/include/dt-bindings/interrupt-controller/gs101.h:311-350`.

- CSIS links 0-7 at `0x1a440000 + n * 0x10000`, CSIS DMA `0x1a4d0000`, DC-PHY
  `samsung,mipi-phy-m0s4s4s4s4s4` at `0x1a4f0000` with the same per-link
  windows, isolation offset `0x3ebc` and reset register `0x1a420500` as
  cheetah (`C/` 1.3). **Same addresses and compatibles: the same blocks.**
- Different: every SPI (links 294-301, CSIS WDMA 302-305), the PMU base
  (`pd_csis` at `0x17462400`, `pd_pdp` at `0x17462480`).
- SysMMU v8 `sysmmu_csis0/1` (`0x1a510000`, `0x1a540000`) and S2MPU
  (`0x1a520000`, `0x1a550000`): bypass, as on cheetah.
- **Not known yet:** which link and lane count each sensor uses, and D-PHY or
  C-PHY. Cheetah learned this from the HAL and the phone (`C/` 4.x).

### 1.4 What our kernel already has

All built as modules in `out-raven-aclang` (from `omamobile-gs101-int`):
`gs201-csis` (`eee796ad2`), `phy-gs201-mipi-dcphy-rx` (`d99c133b7`),
`imx386` (`1a1d5f22e`), `s5kgn1` with GN1, 3J1 and GM5 (`a127e5d36`),
`sem1215` (`9e7b07793`), `slg51002-regulator` (`eb36e03b0`),
`gpio-s2mpg13` (`9744465e6`). `clk-gs101.c` has `CIS_CLK0-7`, but its divider
lacks the `CLK_DIVIDER_ROUND_CLOSEST` cheetah added for gs201.

**Different from cheetah, and the first risk:** raven's DT lets genpd turn
`pd_csis` and `pd_pdp` off at boot (`dcadd701b`), and our gs101 PD driver
doesn't make the DTZPC restore SMC (`0x1a410204`). Cheetah's driver relies
on ABL leaving the domains on. **A CSIS read with the domain off aborts the
SoC**, and whether the registers are reachable after genpd powers the domain
back on without that SMC is unknown [I]. RC2.0 settles it.

## 2. Same and different, in short

| | Raven against cheetah | Work |
|---|---|---|
| Ultrawide IMX386 | same sensor, bus, address, reset and MCLK pads; two supplies, no AF | DT, supply names |
| Main GN1 | same part [I], bus, address, widths, pads; 24 MHz MCLK (cheetah 26), three LDOs | DT, a power-sequence variant in `s5kgn1` |
| Tele IMX586 | new sensor; same `sem1215sa` lens controller | new sensor driver |
| Front IMX663 | new sensor (cheetah: S5K3J1) | new sensor driver |
| Main OIS/AF | `lc898128` + `ak7377` at cheetah's addresses (probably the same parts [I]) | AF driver (mainline `ak7375` has no ak7377) |
| Camera PMIC | SLG51000 on `hsi2c_7` behind S2MPG11 GPIOs | S2MPG11 GPIO driver; mainline `slg51000-regulator` handles only `cs`: bb, buck and pu as GPIO-fixed regulators |
| CSIS, DMA, DC-PHY | same IP | gs101 compatibles, IRQs, PMU values; runtime PM for `pd_csis` |

## 3. Design

- Kernel (`omamobile-gs101-camera`), one commit per piece:
  1. `gpio-s2mpg11`: the S2MPG11 GPIO bank over the existing ACPM regmap.
     `sec-common.c` already creates the `s2mpg11-gpio` cell (`samsung,s2mpg11-gpio`)
     and nothing binds it (RC0, §5), so this is a platform driver shaped like
     `gpio-s2mpg13` (`dev_get_regmap(parent, "gpio")`) with S2MPG11's
     bit-per-GPIO registers. Probe logs the bank as ABL left it and changes nothing.
  2. `slg51000-regulator` (mainline): `CONFIG_REGULATOR_SLG51000=m`; bb, buck and
     pu as `regulator-fixed` supplies in the order above, cs as the driver's own GPIO.
  3. DT: `hsi2c_1-4` and `hsi2c_7`'s camera PMIC, the SLG51000 LDOs, the sensors,
     all `disabled`, enabled by overlays in `device/raven/overlays/` (one per stage).
  4. gs101 compatibles in `gs201-csis` and the DC-PHY driver, with runtime PM
     so the driver holds `pd_csis` and `pd_pdp` on while it runs.
  5. `imx386` supply names; `s5kgn1` raven variant; then `imx663`, `imx586`.
- Every new module is blacklisted in `omamobile-device-raven` before a DTB
  enables its node.
- Sensor mode tables come from the user's own vendor image through
  `tools/extract-camera-data` (as cheetah) and are never committed. EEPROM
  dumps (factory calibration, unique to the phone) stay in `~/android/reference/`.
- Userspace: a libcamera tuning file and sensor helper for IMX386 first; omacam's
  lens table for raven's four cameras last.

## 4. Runbooks (gated)

The human runs or approves each step. Arm a kmsg stream to the laptop and
`--watchdog` before any step that loads a driver. **Read only blocks known to
be powered and clocked, one register per call on a first probe.**

### RC0: read-only, the current kernel. Risk: none

```sh
# RC0.1 camera power domains as genpd sees them (software state only)
tools/phone-ssh 'grep -E "csis|pdp|g3aa|ipp|itp|mcsc|tnr|gdc|dns" /sys/kernel/debug/pm_genpd/pm_genpd_summary'
#   expect: pd_csis and pd_pdp listed; record on/off (dcadd701b: expect off)
# RC0.2 CIS clocks as left at boot
tools/phone-ssh 'grep -iE "cis_clk|csis" /sys/kernel/debug/clk/clk_summary'
#   expect: CIS_CLK0-3 present; record enable count and rate
# RC0.3 reset and MCLK pads (pinctrl's software view)
tools/phone-ssh 'grep -E "gpp(2|4|6)-[23]|gpp(3|5|7|9)-0" /sys/kernel/debug/pinctrl/*/pins'
#   expect: resets low or input, MCLK pads not in function 2
# RC0.4 the bus with the torch and the camera PMIC
tools/phone-ssh 'ls /sys/bus/i2c/devices/; cat /sys/bus/i2c/devices/*/name'
#   expect: the lm3644 at 0x63 bound; nothing at 0x75 yet
```

Stop condition: none (read-only). No probing of addresses on any bus.

### RC1: the camera power path, one change per boot. Risk: low to medium

Each stage is a kernel push with its overlay in the DTB (ASK FIRST). The
overlay is part of slot B's DTB, so rollback means pushing the previous
build's images back to slot B (`KERNEL_OUT=…/out-raven-aclang bin/omamobile push`,
which runs today's `0422abf64` kernel), or using slot A.

**RC1.1** needs kernel `omamobile-gs101-camera` (`0422abf64` + the binding, `gpio-s2mpg11`
and the disabled DT node) built in `out-raven-camera`, and
`camera-gpio-test` added to `DTB_OVERLAYS` in `device/raven/device.conf`
for that push only. The driver only reads: registering the chip reads each
pin's direction, and probe reads `0x98`, `0x99`, `0x9a` and `0x9e` once,
through the ACPM regmap. Nothing requests a pin.

| Step | Command | Expect / Pass | Stop if | Rollback | Risk |
| --- | --- | --- | --- | --- | --- |
| RC1.1a | `KERNEL_DIR=…/wt-raven-camera KERNEL_OUT=…/out-raven-camera bin/omamobile images`, then **push (human's go)** | boots; slot B marked good; `uname -r` has `g23f2d45a3` | no boot, no SSH | slot A falls back; push `out-raven-aclang` | medium: new kernel |
| RC1.1b | `tools/phone-ssh 'dmesg \| grep -i s2mpg11-gpio; grep -A7 s2mpg11 /sys/kernel/debug/gpio'` | `in … out … out-en … remote …` logged; a 6-line gpiochip, nothing requested | probe error; ACPM errors or timeouts in dmesg | push `out-raven-aclang` | read-only |
| RC1.1c | record the four values here | bits 0-4 of `out & out-en` all 0 (camera PMIC and tele supply off) | any of bits 0-4 driven high: ABL left part of the camera power on. Record it and stop before RC1.2 | none needed | read-only |
| RC1.1d | `tools/phone-smoke` | as before the push | a regression | push `out-raven-aclang` | read-only |

Bits 5 and up aren't camera pins; record them without judging them.
Leave the overlay in `DTB_OVERLAYS` once RC1.1 passes; RC1.2 builds on it.

**RC1.2: the SLG51000's first power-up, read-only.** Needs kernel
`a937977fc` or later (the disabled `pmic@75` node on `hsi2c_7`) and
`camera-pmic-test` added to `DTB_OVERLAYS` after `camera-gpio-test`.
Mainline's binding has only `dlg,cs-gpios`, so `tools/camera-pmic-gpio`
raises bb (S2MPG11 GPIO 1) and then buck (GPIO 3) through the GPIO
driver's character device. The kernel does the PMIC writes; the tool has a
60 s dead-man and never requests GPIO 2 (tele) or 4 (pu). Then
`slg51000-regulator` loads by hand. It raises cs, waits 10 ms, and reads:
each LDO's MINV/MAXV range, LDO1/2's voltage range bit, LDO5/6's bypass bit,
each LDO's selector and the enable matrix (through `regulator` sysfs), and
the fault log. It writes nothing. The DT's LDOs carry names only, so the
regulator core sets no voltage. The driver has no OF match table, so the
node can't autoload it; it's blacklisted anyway. Mainline reads no chip ID:
the range reads succeeding are the proof that the chip answers.

DT ranges to judge the read voltages by (Google's
`gs101-raviole-camera-pmic.dtsi`): LDO1 and LDO2 2.25-3.3 V; LDO3, LDO4 and LDO7
1.2-3.75 V; LDO5 and LDO6 0.5-1.2 V.

| Step | Command | Expect / Pass | Stop if | Rollback | Risk |
| --- | --- | --- | --- | --- | --- |
| RC1.2a | `tools/phone-ssh 'echo "blacklist slg51000_regulator" > /etc/modprobe.d/omamobile-camera-bringup.conf'` | the file is there | | `rm` it | none |
| RC1.2b | build `out-raven-camera`, `images` with `camera-pmic-test`, **push (human's go)** | boots, `uname -r` has `ga937977fc`; `pmic@75` `okay`; an unbound `*-0075` I2C device; `slg51000_regulator` not loaded; `98-9e` as in RC1.1 | no boot; the driver loaded by itself | push `out-raven-aclang` | medium: new kernel |
| RC1.2c | `tools/phone-ssh 'cat > /run/camera-pmic-gpio' < tools/camera-pmic-gpio`, then `tools/camera-stage --watchdog x --cmd 'bash -s' --script tools/camera-pmic-probe` | `99: 0a` after hold, `0b` after modprobe; driver bound, "Found chip selector property"; seven `cam_ldoN` with `state=disabled` and voltages inside the ranges above; no I2C errors; after power-down `99: 00` | modprobe fails (NACK: the chip didn't power up, maybe the remote bit, §5): the script powers down; record, stop. Any LDO `enabled`: record, stop (it powers down). A voltage outside its range: record, stop. I2C timeouts, or the torch losing its bus. A watchdog reset. | the script's EXIT trap powers down (rmmod, cs, buck, bb); the hold's 60 s dead-man; a reboot | low-medium: the first change to hardware state (camera PMIC inputs on, LDOs off) |
| RC1.2d | `tools/phone-ssh 'grep -E "^9[89a-e]:" /sys/kernel/debug/regmap/*pmic-2-pmic/registers'`, `tools/phone-smoke` | `99: 00`; smoke as before | anything still high | reboot | read-only |

Record each `cam_ldoN` voltage here: RC1.3 and RC1.4 set them against
these defaults.
**RC1.3: the ultrawide's module EEPROM, read-only.** Needs kernel
`b9aba3226` or later: `usi3` in I2C mode, `hsi2c_3` at 400 kHz with stock's
SoC pull-ups on gpp6-0/1, still disabled, and labels `cam_ldo1-7` on the
SLG51000's LDOs. The `camera-eeprom-test` overlay enables the bus and adds a
`regulator-output` node on `cam_ldo1` (`CONFIG_REGULATOR_USERSPACE_CONSUMER=m`,
blacklisted), the sysfs switch for LDO1. Nothing on the bus is described,
so no driver talks to it.

Stock powers this EEPROM with LDO1 alone, 1 ms (`gs101-raven-camera.dtsi`,
`eeprom-m24c64x-imx386`, `lwis,read-only`), and its PMIC driver raises `pu`
(S2MPG11 GPIO 4) after chip select and the PMIC's setup, 1 ms, and keeps it
high while the PMIC is on (`G/drivers/mfd/slg51000-core.c:333-336, 695-709`).
What `pu` feeds isn't known (the camera buses' pull-ups is a guess [I]), so
`tools/camera-eeprom-probe` follows stock: bb, buck, chip select, `pu`,
then LDO1. Stock runs the SLG51000 in control-register mode, where its
`set_voltage_sel` does nothing, so stock's LDO1 is the 3.05 V default that
RC1.2 read; ours is too (no constraints). Stock leaves the bus pins as
plain inputs while the camera is off; ours stay in I2C function with the
pull-ups on while the overlay is in the DTB (cheetah does the same, CAM2.3).

The read: one byte at the current address (no address written), then 32
bytes from offset 0 as one transfer (two address bytes, a repeated start,
the read; no stop condition between them, so the EEPROM can't start a write
cycle). The bytes are factory calibration: they stay on the phone in
`/run/camera-eeprom-uw.bin` and then `~/android/reference/raven-camera/`,
never in a log or the repo. The tool prints only a summary.

| Step | Command | Expect / Pass | Stop if | Rollback | Risk |
| --- | --- | --- | --- | --- | --- |
| RC1.3a | `tools/phone-ssh 'echo "blacklist userspace_consumer" >> /etc/modprobe.d/omamobile-camera-bringup.conf'` | both lines in the file | | remove the line | none |
| RC1.3b | build `out-raven-camera`, `images` with `camera-eeprom-test`, **push (human's go)** | boots, `uname -r` has `gb9aba3226`; `hsi2c_3` probed as an `i2c-N` with nothing on it; `slg51000_regulator` and `userspace_consumer` not loaded; `98-9e` as in RC1.1 | no boot; either module loaded by itself; I2C errors on the new bus | push `out-raven-aclang` (the RC1.2 build is overwritten by this one) | medium: new kernel |
| RC1.3c | copy `tools/camera-pmic-gpio` to `/run`, then `tools/camera-stage --watchdog x --cmd 'READ=0 bash -s' --script tools/camera-eeprom-probe` | `99: 0a`, then `0b` (driver bound), then `1b` (`pu`); the switch binds with LDO1 off; after "LDO1 on" `cam_ldo1 state=enabled microvolts=3050000`; no I2C or regulator errors; after power-down `99: 00` and LDO1 `disabled` | `0b` or `1b` not reached; the switch doesn't bind; the enable fails or LDO1 reads anything but 3.05 V; the torch loses `hsi2c_7`; a watchdog reset | the EXIT trap (LDO1 off, rmmod, `pu`, cs, buck, bb); the hold's 60 s dead-man; a reboot | low-medium: the first LDO output on |
| RC1.3d | the same with `READ=1` | "eeprom: ack"; "read 32 bytes: N distinct byte values" (not all `0xff`, not all `0x00`); then the power-down as in c | NACK or a timeout (power is already going down: record the bus error, don't power anything else); a stuck bus (`i2c-exynos5` timeouts) | as in c | low: reads only |
| RC1.3e | `tools/phone-ssh 'cat /run/camera-eeprom-uw.bin' > ~/android/reference/raven-camera/eeprom-uw-head.bin`, then `tools/phone-ssh 'rm /run/camera-eeprom-uw.bin'`; `tools/phone-smoke` | 32 bytes kept off-repo; smoke as before | a regression | reboot | none |

On a NACK, check the cheetah lesson first (CAM2.3): try stock's idle state
for the bus pins before blaming the rail or the address.
**RC1.4: the IMX386's model ID.** Needs kernel `a01f3357a` or later: the
sensor node `camera-sensor@1a` on `hsi2c_3`, disabled, with stock's reset
(gpp6-3), MCLK pad (gpp3-0, function 2, 5 mA) and clock (CIS_CLK0), and a
3 ms enable ramp on LDO1. The `camera-uw-test` overlay enables the bus and
the node and replaces `camera-eeprom-test` in `DTB_OVERLAYS` (that
overlay's switch takes LDO1 exclusively).

The two open questions from RC0 and RC1.2 are settled from stock's files:

- **Supplies.** Stock's `sensor-imx386` lists only `ldo1` and `ldo4`
  (`gs101-raven-camera.dtsi:292-360`): LDO1, 3 ms; LDO4, 1 ms; MCLK, 1 ms;
  reset, 10 ms; off in reverse (reset, 1 ms, MCLK, 1 ms, LDO4, 3 ms, LDO1).
  No 1.8 V rail is described, so the module makes its interface and core
  supplies from LDO1 (3.05 V) [I]. Our node names LDO1 as both `vif` and
  `vdig` and LDO4 as `vana`; the imx386 driver enables them in that order
  and LDO1's ramp delay gives the 3 ms. Our power-down has no 3 ms between
  LDO4 and LDO1.
- **MCLK.** Raven's own IMX386 init list writes `EXCK_FREQ` (`0x0136`) =
  `0x1893`, 24.574 MHz: stock tells the sensor it runs from the 24.576 MHz
  oscillator, which is what `dout_cmu_cis_clk0` gives (RC0.2). CIS_CLK0's
  other parent (shared0 / 3) can't make 24 MHz either. Stock also names a
  `GATE_DFTMUX_CMU_CIS_CLK0`; in mainline that's only a Q-channel register,
  not a clock, and cheetah streams without it [I].

The mode data comes from raven's own camera HAL
(`tools/extract-camera-data --apex` on `/apex/com.google.pixel.camera.hal.apex`
from the vendor image; HAL sha256 `42ab61e0…`): 139 init writes, four modes
(4032x3016 30 fps, 4032x2272 60, 2016x1508 60, 2016x1136 60), link 774.144
MHz for all but 4032x2272 (794.624 MHz); 774.144 MHz is also the rate in
cheetah's DT. The file stays off-repo.

The driver's probe: parse the data and the endpoint, power on as above,
read `0x0016`, register controls (no register writes: runtime PM isn't
enabled yet), then runtime-suspend, which powers the sensor down again.

| Step | Command | Expect / Pass | Stop if | Rollback | Risk |
| --- | --- | --- | --- | --- | --- |
| RC1.4a | `tools/phone-ssh 'echo "blacklist imx386" >> /etc/modprobe.d/omamobile-camera-bringup.conf; mkdir -p /usr/lib/firmware/updates/omamobile/camera'`, then `tools/phone-ssh 'cat > /usr/lib/firmware/updates/omamobile/camera/imx386-uw.bin' < …/imx386-uw.bin` | three blacklist lines; the 1864-byte file, checksum as on the laptop | | remove the line and the file | none |
| RC1.4b | build `out-raven-camera`, `images` with `camera-uw-test`, **push (human's go)** | boots, `uname -r` has `ga01f3357a`; `camera-sensor@1a` `okay`; an unbound `*-001a` on the `hsi2c_3` bus; `imx386` not loaded; `98-9e` as before | no boot; `imx386` loaded by itself | push `out-raven-aclang` | medium: new kernel |
| RC1.4c | copy `tools/camera-pmic-gpio` to `/run`, then `tools/camera-stage --watchdog x --cmd 'bash -s' --script tools/camera-uw-probe` | `99: 1b` before the driver; "imx386 …: model 0x0386"; bound, `runtime_status=suspended`; LDO1 and LDO4 `disabled` and `dout_cmu_cis_clk0` enable 0 afterwards; power-down to `99: 00` | "chip ID read failed" (a NACK: MCLK or reset, or the DFTMUX gate) or another ID: the driver has powered down already; record, stop. Any LDO still enabled after probe; I2C errors; a watchdog reset | the EXIT trap (rmmod, `pu`, cs, buck, bb); the hold's dead-man; a reboot | medium: the sensor's first power-up |
| RC1.4d | `tools/phone-smoke` | as before | a regression | reboot | none |

### RC2: the first frames from the ultrawide. Risk: medium

- **RC2.0** the CSIS driver with runtime PM: power `pd_csis` and `pd_pdp` on, then read one
  register, link 0's version at `0x1a440000`. **The first CSIS access on gs101,
  with no DTZPC SMC:** one register, kmsg streamed, `--watchdog`. Stop on an
  abort. Next, add the SMC to the PD driver and retry.
- **RC2.1** find the ultrawide's link and lanes. Arm all eight links' error
  interrupts, start the sensor streaming, and see which link reports SOT
  activity (cheetah's method, `C/` 4.4).
- **RC2.2** one raw frame to memory through CSIS WDMA (`v4l2-ctl --stream-mmap
  --stream-count=1`). Then `cam -c1 --capture=10` through libcamera.

### Later

RC3: the main GN1 (C-PHY if cheetah's main is any guide: the schedule risk).
RC4: IMX586 and IMX663 drivers. RC5: focus (`ak7377`, `sem1215sa`), OIS last.

## 5. Results

### RC0 (2026-10-23, `7.3.0-rc6-omamobile-push1-g0422abf64c05`): pass

- **RC0.1** `pm_genpd_summary`: `csis`, `pdp`, `g3aa`, `ipp`, `itp`, `dns`,
  `mcsc`, `tnr`, `gdc` all `off-0`; `pdp` is the parent of `ipp`, and `itp` of
  `dns`. As `dcadd701b` intends. **CSIS registers are unreachable until a
  driver powers `csis` on** (RC2.0).
- **RC0.2** `CIS_CLK0-7`: enable and prepare counts 0, every
  `dout_cmu_cis_clkN` at **24.576 MHz** (the gs101 oscillator, undivided).
  Stock's DT asks for 24 MHz. Whether stock's MCLK really runs at 24 MHz
  (another parent) or 24.576 MHz is open: the sensor PLL tables depend on it.
  Check what the extracted mode tables assume before RC1.4. `gout_cmu_csis_bus`
  is gated (399.36 MHz parent, 66.56 MHz `dout`).
- **RC0.3** none of the camera reset or MCLK pins (gpp2-2, gpp3-0, gpp4-2,
  gpp5-0, gpp6-2, gpp6-3, gpp7-0, gpp9-0) is claimed. Samsung pinctrl's debugfs
  shows no function or level, and reading the GPIO registers was left out
  (RC0 reads software state only). The level is unknown.
- **RC0.4** I2C: only `hsi2c_7` (`10960000`, i2c-0: `lm3644` at `0x63` bound,
  nothing at `0x75`), `10970000` (i2c-1: an `at24` at `0x50`, not a camera
  EEPROM) and `10d50000` (i2c-2: MAX77759) are enabled. `hsi2c_1-4` are not
  in the DT yet.
- **Found:** platform devices `s2mpg10-gpio` and `s2mpg11-gpio` exist, unbound,
  and no S2MPG GPIO chip is registered. `slg51002-regulator.ko` and
  `leds-lm3644.ko` are in the module tree; there's no `slg51000` module (not
  in the config).

### RC1.1 offline checks (2026-10-23, `7.3.0-rc6-omamobile-push1-g23f2d45a3298`): pass

- `out-raven-camera` built from `omamobile-gs101-camera` `23f2d45a3`:
  exit 0; `gpio-s2mpg11.ko` and `slg51000-regulator.ko` built. The only
  compiler warnings are the two existing `-Wformat` ones in `aoc_alsa_compr.c`.
- `checkpatch --strict`: 0 errors, 0 warnings on all three commits.
- `W=1` on `gpio-s2mpg11.o`: no warnings.
- `dt_binding_check DT_SCHEMA_FILES=samsung,s2mpg11-pmic.yaml`: the example validates.
- `CHECK_DTBS=y` on `gs101-raven.dtb` and `gs101-oriole.dtb`: nothing on
  `pmic-2` or its `gpio` node; the findings that remain (Trusty, `soc_id`)
  were there before.
- `camera-gpio-test` applies to the new `gs101-raven.dtb` with `fdtoverlay`:
  the node goes from `disabled` to `okay`.

### RC1.1 on the phone (2026-10-23, `7.3.0-rc6-omamobile-push1-g23f2d45a3298`): pass

- **RC1.1a** push from `out-raven-camera` with `camera-gpio-test`: boot_b,
  vendor_boot_b and dtbo_b written and verified; the phone booted the new kernel.
- **RC1.1b** `s2mpg11-gpio s2mpg11-gpio: in 0x00 out 0x00 out-en 0x3f remote 0x3f`
  at 3.8 s; gpiochip45, 6 GPIOs, none requested; no ACPM errors.
- **RC1.1c** all six pins are outputs driven low, so the camera PMIC's bb,
  buck, cs and pu lines and the tele enable are off. **Open for RC1.2:**
  every pin has its remote-control bit (`0x9e`) set. Google's driver never
  reads or writes that register and drives the pins through `0x99` alone,
  so stock presumably runs the same way, but it isn't verified. RC1.2's
  first write should be checked by its effect (the SLG51000 answering on
  `hsi2c_7` after bb, buck and cs), not only by reading `0x99` back.
- **RC1.1d** `tools/phone-smoke`: all ok, no failed units; dmesg error lines
  133 (138 before the push).
- Raven's LWIS nodes give power-up sequences only, no `regulator-voltages`,
  so the SLG51000 defaults are still RC1.2's to read.

### RC1.2 on the phone (2026-10-23, `7.3.0-rc6-omamobile-push1-ga937977fcd51`): pass

- **RC1.2a** blacklist in `/etc/modprobe.d/omamobile-camera-bringup.conf`.
- **RC1.2b** booted; `pmic@75` `okay`; `0-0075` (`slg51000`) present and
  unbound; the module didn't load by itself; `98-9e` as in RC1.1.
- **RC1.2c** (`logs/camera-20261023-210643-x*`, watchdog armed):

  | GPIO registers | `98` (in) | `99` (out) | `9a` (out-en) | `9e` (remote) |
  |---|---|---|---|---|
  | before | 00 | 00 | 3f | 3f |
  | bb and buck high | 0a | 0a | 3f | 3f |
  | cs too (driver bound) | 0b | 0b | 3f | 3f |
  | after power-down | 00 | 00 | 3f | 3f |

  The input register follows the output, so the pins really switch, and
  **the remote-control bit doesn't get in the way** (RC1.1's open question).
  The SLG51000 answered: "Found chip selector property", every range read
  succeeded, bound as `0-0075`, no I2C errors, and the torch kept its bus.
  The chip's defaults, all LDOs **off**:

  | LDO | Default | DT range | Used by |
  |---|---|---|---|
  | 1 | 3.05 V | 2.25-3.3 V | every part (§1.2) |
  | 2 | 3.05 V | 2.25-3.3 V | main |
  | 3 | 2.85 V | 1.2-3.75 V | main |
  | 4 | 2.90 V | 1.2-3.75 V | ultrawide |
  | 5 | 1.15 V | 0.5-1.2 V | tele |
  | 6 | 1.15 V | 0.5-1.2 V | tele |
  | 7 | 1.80 V | 1.2-3.75 V | front |

  All inside their ranges. LDO1 at 3.05 V is not a 1.8 V I/O rail, so §1.2's
  "LDO1 is the I/O rail" [I] is probably wrong: it's more likely the
  analog/EEPROM/AF supply, with the sensors' 1.8 V I/O from elsewhere (as
  cheetah's `vddio_cam`). Check stock's LWIS power-up lists for an S2MPG
  LDO before RC1.4. RC1.3 (EEPROM on LDO1) is unaffected: EEPROMs run at 3 V.
- **The safety net had a gap:** on the Mac, `tools/camera-stage` couldn't
  start its kmsg stream (`setsid: command not found`) and went ahead anyway,
  so the laptop has no kernel log of RC1.2c; the phone's own `dmesg` was
  saved afterwards (nothing hung). Fixed: a Perl `setsid` fallback, and the
  tool now refuses to touch anything until this run's own start marker has
  reached the laptop's log.
- **RC1.2d** `99: 00` again; `tools/phone-smoke` all ok. dmesg error lines
  went from 133 to 214, all of it 139 `wc-mbox … Try increasing
  MBOX_TX_QUEUE_LEN` from the AoC mailbox within the first 30 s of boot,
  before any camera step. Not from this work, but worth watching
  (AUDIO-PLAN).

### RC1.3 offline checks (2026-10-24, `7.3.0-rc6-omamobile-push1-gb9aba32265ec`): pass

- `out-raven-camera` built from `omamobile-gs101-camera` `b9aba3226` (the
  SLG51000 commit is now `1225e6733`, with the `cam_ldoN` labels): exit 0,
  no new compiler warnings; `userspace-consumer.ko` built.
- `checkpatch --strict` on `b9aba3226`: only the missing Signed-off-by
  (the human signs off).
- `dt-validate` on `gs101-raven.dtb`, and on it with all eight overlays
  applied (`fdtoverlay`): nothing on `usi@109200c0`, `i2c@10920000`,
  `pmic@75`, its LDOs or `camera-eeprom-ldo1`; what remains (`soc_id`,
  Trusty, the USB PHY's `orientation-switch`) was there before.
- With the overlays, `hsi2c_3` and `usi3` are `okay` and
  `camera-eeprom-ldo1` points at `cam_ldo1`'s phandle. `images` built
  boot, vendor_boot and dtbo with the AVB footer.

### RC1.3a-b on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-gb9aba32265ec`): pass

- **RC1.3a** `blacklist userspace_consumer` added to
  `/etc/modprobe.d/omamobile-camera-bringup.conf`.
- **RC1.3b** pushed (boot_b, vendor_boot_b and dtbo_b written and verified)
  and booted. `usi@109200c0`, `i2c@10920000` and `pmic@75` `okay`;
  `camera-eeprom-ldo1` present, unbound; neither `slg51000_regulator` nor
  `userspace_consumer` loaded; `98-9e` `00 00 3f 00 00 00 3f` as in RC1.1.
  No new I2C messages. `tools/phone-smoke` as after RC1.2, but for four
  more dmesg error lines (218), one of them one more AoC mailbox line.
- **The I2C bus numbers moved.** `hsi2c_3` probes first and takes `i2c-0`,
  so `hsi2c_7` is now `i2c-1` (the camera PMIC is `1-0075`, the torch
  `1-0063`) and the MAX77759's bus is `i2c-3`. `omamobile-usb-rescue` finds
  its bus by address and found `i2c-3`; `camera-eeprom-probe` now finds the
  PMIC by glob. Never write a bus number into a tool: find it from
  `of_node`.

### RC1.3c on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-gb9aba32265ec`): pass

`logs/camera-20261023-215005-x.log` (laptop clock; kmsg streamed, watchdog
armed). `hsi2c_3` was `i2c-0`.

| Step | `98` (in) | `99` (out) | LDO1 |
|---|---|---|---|
| before | 00 | 00 | |
| bb and buck high | 0a | 0a | |
| slg51000-regulator bound as `1-0075` (cs) | 0b | 0b | |
| `pu` high | 1b | 1b | |
| switch bound (`camera-eeprom-ldo1`) | 1b | 1b | `disabled`, 3050000 µV, 0 users |
| LDO1 on | 1b | 1b | `enabled`, 3050000 µV, 1 user |
| after power-down | 00 | 00 | `disabled` |

`9a` and `9e` stayed `3f`. **The first SLG51000 output switched on and off
as asked**: the `enabled` and `disabled` states are read back from the
chip's enable register, not from the regulator core. No I2C, regulator or
other errors after the step began; both modules unloaded and the holder
exited. `pu` (GPIO 4) switches like the other pins.

### RC1.3d-e on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-gb9aba32265ec`): pass

- **RC1.3d** (`logs/camera-20261023-220807-x.log`, watchdog armed): the
  power-up as in RC1.3c, then on `i2c-0` (`hsi2c_3`) "eeprom: ack" for the
  current-address byte, and the 32 bytes from offset 0 read in one
  transfer: **20 distinct byte values**, so not blank and not a stuck bus.
  Power-down as in RC1.3c (`99: 00`, LDO1 `disabled`); no I2C or other
  errors after the step; both modules unloaded. **The ultrawide's bus, its
  pull-ups, LDO1 and the EEPROM's address are all as stock describes.**
  Whether `pu` is needed for the bus isn't known: it was high, as on stock.
- **RC1.3e** the 32 bytes are in `~/android/reference/raven-camera/`
  (checksums matched) and gone from the phone. `tools/phone-smoke` as
  before (216 dmesg error lines; the battery reads "Charging" again, so
  the "Discharging" after RC1.3b was momentary).

### RC1.4 offline checks (2026-10-24, `7.3.0-rc6-omamobile-push1-ga01f3357ac7e`): pass

- `out-raven-camera` built from `a01f3357a` (a transient `cp` race in the
  config merge on the shared volume stopped the first try; the rerun was
  clean): no new warnings. `checkpatch --strict`: only the missing
  Signed-off-by.
- `dt-validate` on `gs101-raven.dtb` and on it with the eight RC1.4
  overlays (`camera-uw-test` in place of `camera-eeprom-test`): nothing on
  `camera-sensor@1a`, `cam-uw-mclk-on-pins`, `hsi2c_3` or the PMIC; the
  same pre-existing findings as for RC1.3.
- `images` built with `camera-uw-test` and the AVB footer.
- Raven's mode file (1864 bytes) and DC-PHY file are in
  `~/android/reference/raven-camera/`.

