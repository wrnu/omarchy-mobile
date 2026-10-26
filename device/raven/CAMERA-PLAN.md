<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 Pro (raven): cameras

Status (2026-10-24): **RC0 done (read-only, §5); RC1.1 and RC1.2 pass on the phone (§5); RC1.3 passes: the ultrawide's module EEPROM answers (§5); RC1.4 passes: the ultrawide's IMX386 answers with model `0x0386` (§5); RC2.0 (power `pd_csis`, read one CSIS register) offline checks pass, RC2.0a passes; RC2.0b stopped at the power-off (the PD driver's 1 ms wait); RC2.0e passes, RC2.0f shows the power-off is stuck, not slow; RC2.0h finds `pd_csis` off at hand-off; RC2.0m passes, RC2.0n shows the power-off sticks without the bus clock too; RC2.0o-p (stock's DTZPC save and restore through EL3) offline checks pass, RC2.0o passes; RC2.0p stops: both DTZPC calls return 0, and the power-off still sticks; stock never powers CSIS without PDP: RC2.0q-r (PDP around the CSIS cycle) offline checks pass, RC2.0q passes; RC2.0r stops: PDP's power-off sticks too, so every power-off after our own power-on sticks; our PD driver wrote the whole CONFIGURATION register where stock changes only bit 0: RC2.0s-t (bit 0 only, plus logging) offline checks pass; RC2.0s shows CONFIGURATION holds only bit 0, so that wasn't it; stock restores the CMU state it saved from the bootloader's on state after every power-on, which we never do; RC2.0u-x (read that state) offline checks pass; RC2.0u stops: genpd's sync_state powers boot-on domains off despite `pd_ignore_unused` (so RC2.0h's "off at hand-off" was wrong); RC2.0y (a bring-up sync_state that keeps `pd_csis` on) passes; RC2.0v-x pass: the bootloader leaves every CSIS Q-channel disabled with its clock request forced (`QCH_CON` `0x003f0002`), the user mux on CMU_TOP; next, the same registers after our own power-on: RC2.0z passes; RC2.0za hangs the SoC: CMU_CSIS, readable in the bootloader's state, hangs after our own power-on; the DTZPC was never saved from the bootloader's state, so its restore had nothing to put back [I]; RC2.0zc-ze (stock's save-then-restore order, attended): RC2.0zc and RC2.0zd pass: **with stock's DTZPC save (from the bootloader's state) and restore, CMU_CSIS is reachable after our own power-on** (the user mux reads its reset value 0x0); RC2.0ze reads all 56 registers (three are at reset values: the user mux, the controller option and the BUSP divider), but the power-off after our power-on still sticks; RC2.0zf-zg (write those three back after the DTZPC restore) offline checks pass; RC2.0zf0 and RC2.0zf pass: the three CMU_CSIS registers take the bootloader's values back after the DTZPC restore; RC2.0zg passes: **with the DTZPC save/restore and the CMU restore, pd_csis powers off after our own power-on**, a full cycle from Linux; RC2.0zh0 and RC2.0zh pass: **CMU_CSIS in the gs101 clock driver (runtime-PM save and restore) cycles pd_csis twice, all 56 registers equal the bootloader's each time**; RC2.0zi0 and RC2.0zi pass: SYSREG_CSIS's DRCG and MEMCLK through the CMU, both cycles power off; RC2.0zj (the DTZPC save and restore always on, the first save at boot) offline checks pass; RC2.0zj0 and RC2.0zj pass: **pd_csis cycles with no bring-up switch**, the first DTZPC save at boot; RC2.0zk0 and RC2.0zk pass: the bring-up switches and logging dropped, pd_csis still cycles; RC2.0zl0 and RC2.0zl pass: **the DTZPC as upstream's `samsung,dtzpc`** (André Draszik's gs101 power domain series, v8), pd_csis still cycles (§4, §5).** No camera driver binds on
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

What stock does around `pd_csis`, from Google's GPL sources [V]:

- **Power-on is the same two steps as ours.** `csis_on` writes
  `CSIS_CONFIGURATION` (PMU `0x2400`) bit 0 and waits for `CSIS_STATUS`
  (`0x2404`) bit 0 (`G/drivers/soc/google/cal-if/gs101/flexpmu_cal_local_gs101.h:922-925`),
  which is what `exynos-pm-domains.c` does for `google,gs101-pd`.
- **Around it, stock runs two things we don't.** It runs the DTZPC SMC
  (`0x82000410`, `RUNTIME_PM_TZPC_GROUP` 2, address `need_smc = 0x1a410204`):
  save, mode 0, before power-off; restore, mode 1, after power-on
  (`gs101-pm-domains.dtsi:143-151`, `pmucal_local.c:52-61, 122-130`,
  `exynos-pd_el3.c:16-38`).
  It also restores CMU_CSIS's saved registers after power-on (`csis_save`,
  `:927-…`: `DIV_CLK_CSIS_BUSP`, `PLL_CON0_MUX_CLKCMU_CSIS_BUS_USER`).
  Without that restore, CMU_CSIS is at its reset values after our power-on,
  and its user mux selects the oscillator [I]. That is enough for a
  register read, but streaming will need a CMU_CSIS driver or those two
  writes.
- **Stock gs101 claims `pd_csis` from the CSIS SysMMUs**
  (`gs101-sysmmu.dtsi:93, 123`) and the camera's top device (with
  `pd_pdp`, `pd_gdc`, `pd_itp`, `pd_mcsc`, `pd_tnr`; `gs101-isp.dtsi:932`).
  `lwis_csi` (`gs101-isp.dtsi:137-381`) holds **`pd_pdp`** (line 378;
  corrected 2026-10-24 after RC2.0p, this said it named none), and it enables `UMUX_CLKCMU_CSIS_BUS`. On our
  phone CMU_TOP's `gout_cmu_csis_bus` is gated (hardware `N`, 2026-10-24),
  so RC2.0 enables it.
- **Raven evidence on the DTZPC** [I]: BLK_AOC was off at hand-off and
  powered on by our code, which made no SMC. Its S2MPU then read `0x5`
  with no abort (AUDIO-PLAN R2.1g), so D_TZPC_AOC's reset state let the
  non-secure side in. The SMC itself hung EL3 when BLK_AOC was off
  (R2.1c, R2.1d). So RC2.0 makes no SMC. If the read aborts, the next
  test leaves `pd_csis` as ABL left it (no genpd for it), which is how
  cheetah read `0x05040000`.

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

- **RC2.0** power `pd_csis` and read one register, link 0's version at
  `0x1a440000`: **the first CSIS access on gs101, with no DTZPC SMC.**
  No kernel change. The `camera-csis-test` overlay adds a `simple-pm-bus`
  (`/soc@0/bus@1a440000`) that holds `pd_csis` and `CLK_DOUT_CMU_CSIS_BUS`.
  Runtime PM's `power/control` = `on` powers the block, and `auto` powers it
  off. `pd_pdp` stays off. (§1.4 then said stock's CSIS node names no
  domain; it holds `pd_pdp`, see RC2.0p.)
  `tools/camera-csis-probe` checks that genpd, runtime PM and the clock
  all read on, and only then reads the register once, through a
  read-only `/dev/mem` mapping (raven's root has no `devmem`).

  The overlay changes boot too: genpd powers `pd_csis` on once while it
  attaches the bus, then off when the bus stays suspended. That is the
  first gs101 power-on through `google,gs101-pd` on this phone, unattended
  at boot; RC2.0a reads its result. No driver touches the block's
  registers at boot.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0a | build `images` with `camera-csis-test` (same kernel, `ga01f3357a`), **push (human's go)**; then `tools/phone-ssh 'grep -E "^csis|bus@1a440000" /sys/kernel/debug/pm_genpd/pm_genpd_summary; dmesg \| grep -iE "power domain\|csis\|simple-pm-bus"'` | boots; `csis off-0` with the bus listed `suspended`; no "Power domain csis enable/disable failed" | no boot (a hang at the boot-time power-on); an enable or disable failure; `csis` not `off-0` | push without `camera-csis-test` (drop it from `DTB_OVERLAYS`); a failed boot falls back to slot A | medium: an unattended first power-on at boot |
  | RC2.0b | `tools/camera-stage --watchdog x --cmd 'READ=0 bash -s' --script tools/camera-csis-probe` | `csis on`, `runtime_status: active`, `gout_cmu_csis_bus enable=1 … hw=Y`; then after `auto`: `off-0`, `suspended`, `enable=0 … hw=N`; no kernel messages but the markers | the domain doesn't come on (timeout) or doesn't go off; the clock doesn't follow; any abort or SError; a watchdog reset | the EXIT trap (`auto`); a reboot | low: power and clock only, no register |
  | RC2.0c | `tools/camera-stage --watchdog x --cmd 'READ=1 bash -s' --script tools/camera-csis-probe` | "csis link 0 version: 0x05040000" (cheetah's, the same IP [I]); off as in RC2.0b; no abort | `python3` killed by an external abort (`SIGBUS`, "synchronous external abort" in dmesg: the DTZPC or CMU state after our power cycle) or a hang (the watchdog resets the phone; save pstore with `--pstore`); `0x00000000` or `0xffffffff`. Record and stop; don't repeat the read | the EXIT trap; a reboot | medium: the first CSIS register read on gs101 |
  | RC2.0d | `tools/phone-smoke` | as before | a regression | reboot | none |

  After RC2.0b's stop (§5), the PD driver waits 10 ms for gs101 domains
  and prints the status word on a timeout (`b7be5bb94`). RC2.0e-g repeat
  the power test and then the read on that kernel:

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0e | build `out-raven-camera` (`b7be5bb94`), `images` as before, **push (human's go)**; then RC2.0a's checks | `uname -r` has `gb7be5bb94`; `csis off-0`, the bus `suspended`; no "Power domain … failed" for any domain | no boot; any domain's enable or disable failure | push `a01f3357a` images (`out-raven-aclang` if that fails) | medium: new kernel, every gs101 domain's timing |
  | RC2.0f | RC2.0b again (`READ=0`) | on as before; after `auto`: `csis off-0`, `suspended`, no failure | "disable failed (status 0x…)": record the word (bit 0 still 1: stuck, not slow); stop, no read | the EXIT trap; a reboot | low |
  | RC2.0g | RC2.0c (`READ=1`), only after RC2.0f passed on the same boot | as RC2.0c | as RC2.0c | as RC2.0c | medium |

  RC2.0f showed that `pd_csis` can't be power-cycled yet: after our
  power-on, its power-off sticks (§5). So RC2.0h-l read the block as ABL
  hands it over. `camera-csis-abl-test` replaces `camera-csis-test` and
  sets `pd_ignore_unused`, so genpd keeps every domain in the state it read
  from the PMU at probe. Its bus holds only the CSIS bus clock, no power
  domain. `tools/camera-csis-abl-probe` refuses to read unless genpd reads
  `csis on`, nothing has switched it this boot, and the clock is ungated.
  The 56 CMU_CSIS and SYSREG_CSIS registers in `READ=cmu` are stock's
  `csis_save` list: what a restore after power-on needs. Cost while the
  overlay is in: every unused domain stays on, as before `dcadd701b`.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0h | `images` with `camera-csis-abl-test` (same kernel `gb7be5bb94`), **push (human's go)**; then `grep pd_ignore_unused /proc/cmdline`, the genpd summary, `dmesg` for "Power domain" | boots; `pd_ignore_unused` on the command line; **`csis on`** (ABL leaves it powered, as `dcadd701b`'s measurement implies); the bus `suspended` and under no domain; no "Power domain … failed" | no boot; `csis off-0`: ABL leaves it off, so stop: no read is safe, and the plan needs a power-on path that works | push with `camera-csis-test` (or without either) | low: the pre-`dcadd701b` domain state plus a clock-only bus |
  | RC2.0i | `tools/camera-stage --watchdog x --cmd 'READ=version bash -s' --script tools/camera-csis-abl-probe` | `csis on` before and after; `gout_cmu_csis_bus` `enable=1 … hw=Y`; "CSIS_LINK0_VERSION 0x1a440000 = 0x05040000" | as RC2.0c (an external abort, a hang, 0 or all ones): record, stop | the EXIT trap; a reboot | medium: the first CSIS read on gs101 |
  | RC2.0j | as RC2.0i with `READ=cmu1` | one value for `PLL_CON0_MUX_CLKCMU_CSIS_BUS_USER` (`0x1a400600`); no abort | as RC2.0i | as RC2.0i | low after RC2.0i |
  | RC2.0k | as RC2.0i with `READ=cmu` | 56 values, recorded in §5 (ABL's configuration; not unique to the phone) | as RC2.0i | as RC2.0i | low |
  | RC2.0l | `tools/phone-smoke` | as before | a regression | reboot | none |

  RC2.0h found `pd_csis` off at hand-off (§5), so there is no ABL state
  to read. RC2.0m-n test the remaining suspect we control: simple-pm-bus
  gates the CSIS bus clock just before genpd's power-off.
  `camera-csis-pd-test` (replacing `camera-csis-abl-test`, default
  bootargs) is `camera-csis-test` without `clocks`. The probe sees the
  missing clock, never touches the clock, and refuses `READ=1`.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0m | `images` with `camera-csis-pd-test` (same kernel `gb7be5bb94`), **push (human's go)**; then the genpd summary and `dmesg` for "Power domain" | boots; `csis off-0`, the bus `suspended` under `csis`; no "Power domain … failed" (genpd's attach at boot powers on and off) | no boot; a failure at boot (record the status word: then the attach cycle sticks even without the clock) | push with `camera-csis-test` | low |
  | RC2.0n | `tools/camera-stage --watchdog x --cmd 'READ=0 bash -s' --script tools/camera-csis-probe` | `csis on`, `active`; after `auto`: **`csis off-0`**, no failure; the clock untouched (`enable=0`) | "disable failed (status 0x…)": the clock wasn't the cause; record, stop | the EXIT trap; a reboot | low: power only |

  If RC2.0n passes, the CSIS driver must keep its bus clock until the
  domain is off. Whether the block can be read without that clock (CMU_CSIS's
  user mux at reset may select the oscillator [I]) is a separate step. If
  it fails, the DTZPC save SMC before the power-off is the remaining
  difference from stock.

  RC2.0n failed (§5), so RC2.0o-p add stock's DTZPC calls. Kernel
  `6d15de5fd`: for a gs101 domain with `google,tzpc-smc`, the PD driver
  makes EL3's `SMC_CMD_PREPARE_PD_ONOFF` (`0x82000410`, group 2) restore
  (mode 1) after each power-on and save (mode 0) before each power-off;
  a failed save leaves the domain on. It makes them only while
  `exynos_pm_domains.tzpc_smc` is `Y`, which defaults to `N`. That matters
  because **boot does cycle `pd_csis`**: `platform_probe()` attaches the bus
  with `PD_FLAG_ATTACH_POWER_ON`, then genpd powers it off once the bus is
  idle. That cycle's power-off finished in RC2.0a and RC2.0m without an
  error, so **the first cycle after boot works and the attended second
  one sticks** [I, from the code and the missing failure line; RC2.0o
  logs it]. With `tzpc_smc=N` the driver logs "DTZPC … skipped" at each
  of those boot calls instead. `camera-csis-smc-test` is
  `camera-csis-pd-test` plus `&pd_csis { google,tzpc-smc = <0x1a410204>; }`
  (D_TZPC_CSIS, stock's `need_smc`). `camera-csis-probe SMC=1` sets the
  parameter just before the power-on and clears it after the power-off.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0o | `images` with `camera-csis-smc-test` (kernel `g6d15de5fd`), **push (human's go)**; then `tools/phone-ssh 'cat /sys/module/exynos_pm_domains/parameters/tzpc_smc; grep -E "^csis" /sys/kernel/debug/pm_genpd/pm_genpd_summary; dmesg \| grep -E "Power domain csis"'` | boots; `N`; `csis off-0`; one "DTZPC restore skipped" and one "DTZPC save skipped" (the boot cycle, shown); no "smc returned", no failure | no boot; any "smc returned" (a call at boot); a failure at boot | push with `camera-csis-pd-test` | low: no SMC runs |
  | RC2.0p | `tools/camera-stage --watchdog x --cmd 'SMC=1 READ=0 bash -s' --script tools/camera-csis-probe` | `csis on`, `active`, "DTZPC restore 0x1a410204: smc returned 0"; after `auto`: "DTZPC save … returned 0", then **`csis off-0`**, no failure; `tzpc_smc: N` | a hang or watchdog reset (EL3 hung: as the AoC); a non-zero return (record it); "disable failed (status 0x…)" again: the SMC isn't the difference; stop | the EXIT trap; a reboot (`pstore` after a reset) | medium: the first DTZPC call to EL3 on gs101 |

  If RC2.0p passes, the read comes back as a separate step with the clock
  (the `camera-csis-test` bus plus the property). If it fails without a
  hang, what's left is the time the domain stays on: try a cycle with the
  shortest gap between on and off, to match the working boot cycle.

  RC2.0p failed with both calls returning 0 (§5). Stock never runs CSIS
  with PDP off (§1.4: `lwis_csi` holds `pd_pdp`; the top device holds both),
  and CSIS's OTF, VO and AXI bridges have PDP as their partner, so RC2.0q-r
  power PDP around the CSIS cycle. No kernel change (`g6d15de5fd`).
  `camera-csis-pdp-test` is `camera-csis-smc-test` plus a second
  simple-pm-bus (`bus@1aa40000`, PDP's first page, never read) that holds
  `pd_pdp`. `camera-csis-probe PDP=1` powers PDP on, then CSIS on, then
  CSIS off, then PDP off. One change: the DTZPC calls stay off (`SMC=0`).

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0q | `images` with `camera-csis-pdp-test`, **push (human's go)**; then `tools/phone-ssh 'grep -E "^(csis\|pdp\|ipp) " -A2 /sys/kernel/debug/pm_genpd/pm_genpd_summary; dmesg \| grep -E "Power domain"'` | boots; `csis` and `pdp` `off-0`, each bus `suspended` under its domain; no "failed" | no boot; a failure at boot (record the status word) | push with `camera-csis-smc-test` | low |
  | RC2.0r | `tools/camera-stage --watchdog x --cmd 'PDP=1 READ=0 bash -s' --script tools/camera-csis-probe` | `pdp on`, then `csis on`, `active`; after `auto`: **`csis off-0`**, no failure; then `pdp off-0` | "csis disable failed": PDP isn't the difference; "pdp disable failed": record both status words; any abort; stop | the EXIT trap; a reboot | low: power only, two domains |

  If RC2.0r passes, the CSIS read comes back with PDP held (the clock bus
  plus the PDP bus). If CSIS still sticks with PDP on, the next single
  change is stock's whole order: PDP on and the DTZPC calls together
  (`PDP=1 SMC=1`).

  RC2.0r failed for PDP too (§5): every power-off after our own power-on
  sticks. Stock's CAL changes only bit 0 of a domain's CONFIGURATION
  (read-modify-write); our gs101 support wrote the whole register. Kernel
  `611093c5b` uses `regmap_update_bits()` (on gs101, the PMU's
  set/clear-bit aliases), and the bring-up commit `afcea63ba` logs
  CONFIGURATION and STATUS of each gs101 domain at probe and CONFIGURATION
  before and after every switch, through the kernel's PMU regmap. Same
  overlay (`camera-csis-pdp-test`); the kernel is the one change.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0s | `images` (kernel `gafcea63ba`, `camera-csis-pdp-test`), **push (human's go)**; then `tools/phone-ssh 'dmesg \| grep -E "Power domain"; grep -E "^(csis\|pdp) " /sys/kernel/debug/pm_genpd/pm_genpd_summary'` | boots; one "CONFIGURATION … STATUS … at probe" per gs101 domain (record the values: which bits besides bit 0); `csis` and `pdp` `off-0`; no failure | no boot; a failure at boot; a domain whose CONFIGURATION changes more than bit 0 in an "on/off: CONFIGURATION a -> b" line | push kernel `g6d15de5fd` | low |
  | RC2.0t | `tools/camera-stage --watchdog x --cmd 'READ=0 bash -s' --script tools/camera-csis-probe` (CSIS alone, as RC2.0n) | `csis on`; after `auto`: **`csis off-0`**, no failure; both CONFIGURATION lines change bit 0 only | "disable failed (status 0x…)": the full write wasn't the cause; record the CONFIGURATION lines; stop | the EXIT trap; a reboot | low: power only |

  If RC2.0t passes, RC2.0u repeats it with `PDP=1` (PDP's own power-off),
  and the CSIS read comes back after that with the clock.

  RC2.0s ruled the CONFIGURATION write out and showed every domain on at
  hand-off (§5), so RC2.0u-x repeat RC2.0h-k's read of the bootloader's
  CMU_CSIS state, now that there may be one to read. They read the
  registers stock saves before its first power-off and restores after
  every power-on (`csis_save`), the values we never restore. Same kernel
  (`gafcea63ba`, which logs the probe state); `camera-csis-abl-test`
  (`pd_ignore_unused`, a bus with the CSIS bus clock and no domain)
  replaces `camera-csis-pdp-test`. No CSIS link register is read
  (`READ=version` is skipped).

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0u | `images` with `camera-csis-abl-test`, **push (human's go)**; then `tools/phone-ssh 'grep pd_ignore_unused /proc/cmdline; dmesg \| grep -E "Power domain"; grep -E "^(csis\|pdp) " /sys/kernel/debug/pm_genpd/pm_genpd_summary'` | boots; "csis: CONFIGURATION 0x1 STATUS 0x1 at probe"; **`csis on`**; no "on:"/"off:" line for csis (nothing switched it) | no boot; `csis off-0` (as RC2.0h: nothing to read; record the probe line, stop) | push with `camera-csis-pdp-test` | low |
  | RC2.0v | `tools/camera-stage --watchdog x --cmd 'READ=none bash -s' --script tools/camera-csis-abl-probe` | `csis on` before and after; `gout_cmu_csis_bus` `enable=1 … hw=Y` | the clock doesn't come on; `csis` changes state | the EXIT trap | low: the clock only |
  | RC2.0w | as RC2.0v with `READ=cmu1` | one value for `PLL_CON0_MUX_CLKCMU_CSIS_BUS_USER` (`0x1a400600`); no abort | an external abort, a hang; stop | the EXIT trap; a reboot | medium: the first CMU_CSIS read on gs101 |
  | RC2.0x | as RC2.0v with `READ=cmu` | 56 values, recorded in §5 (the bootloader's configuration; not unique to the phone) | as RC2.0w | as RC2.0w | low after RC2.0w |

  The values then go into the save and restore design: compare them with
  CMU_CSIS's reset values (Google's `cmucal-sfr.c` [I]), and restore the
  ones that differ after each power-on.

  RC2.0u stopped (§5): genpd's `sync_state` powers boot-on domains off
  whatever `pd_ignore_unused` says. RC2.0y replaces RC2.0u: kernel
  `2572f3c60` gives the PD driver a bring-up `.sync_state` that leaves a
  domain with `google,keep-boot-on` on (genpd's `stay_on` stays set, so
  nothing powers it off until a reboot), and `camera-csis-abl-test` now sets
  that property on `pd_csis` and no longer overrides the bootargs.
  `camera-csis-abl-probe` checks for the property and the driver's line
  instead of `pd_ignore_unused`. RC2.0v-x then run as written.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0y | `images` (kernel `g2572f3c60`, the new `camera-csis-abl-test`), **push (human's go)**; then `tools/phone-ssh 'dmesg \| grep -E "Power domain (csis\|pdp)\|17462400.power-domain"; grep -E "^(csis\|pdp) " /sys/kernel/debug/pm_genpd/pm_genpd_summary'` | boots; "csis: CONFIGURATION 0x1 STATUS 0x1 at probe"; "17462400.power-domain: left as the bootloader left it"; **`csis on`**, no "csis off" line; `pdp off-0` (powered off as before) | no boot; `csis off-0` or a "csis off" line; the line missing | push with `camera-csis-pdp-test` (kernel `gafcea63ba`) | low |

  RC2.0x read the bootloader's CMU_CSIS (§5). RC2.0z-zb read the same 56
  registers after our own power-on, to get the reset values a block we
  power runs with. Same kernel (`g2572f3c60`; without `google,keep-boot-on`
  the domain is powered off at `sync_state` as usual). `camera-csis-test`
  (the bus holds `pd_csis` and the CSIS bus clock) replaces
  `camera-csis-abl-test`. `camera-csis-probe READ=cmu1|cmu` reads the
  abl-probe's list (generated, checked equal), one 32-bit load each;
  `KEEP=1` leaves the bus held on, and `HELD=1` reads a bus left on, so the
  single-register read and the full read share one power-on and need no
  power-off in between (it would stick).

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0z | `images` with `camera-csis-test`, **push (human's go)**; then `tools/phone-ssh 'dmesg \| grep -E "Power domain csis\|17462400.power-domain"; grep -E "^csis" -A2 /sys/kernel/debug/pm_genpd/pm_genpd_summary'` | boots; "csis … at probe" then "csis off: CONFIGURATION 0x1 -> 0x0" (no "left as the bootloader left it"); `csis off-0` with the bus `suspended`; no failure | no boot; a failure; `csis on` | push with `camera-csis-abl-test` | low |
  | RC2.0za | `tools/camera-stage --watchdog x --cmd 'READ=cmu1 KEEP=1 bash -s' --script tools/camera-csis-probe` | `csis on`, `active`, `gout_cmu_csis_bus enable=1 … hw=Y`; "0x1a400600 = 0x…" (the user mux after our power-on); the bus stays on | an external abort, a hang; stop | a reboot | medium: the first read of a block we powered |
  | RC2.0zb | `tools/camera-stage --watchdog x --cmd 'READ=cmu HELD=1 bash -s' --script tools/camera-csis-probe` | 56 values, recorded in §5 against RC2.0x's; the trap's power-off then probably sticks (record its status, as RC2.0n) | as RC2.0za | a reboot | low after RC2.0za |

  The differences are what a restore after power-on must write. If the
  reset `QCH_CON` has `ENABLE` 1, that is the lead for the stuck power-off.

  RC2.0za hung the SoC (§5): after our own power-on CMU_CSIS isn't
  reachable. **Never read the CSIS block after a power-on that had no
  DTZPC restore from a real save.** RC2.0zc-ze do stock's order, attended.
  Kernel `90a130fd6` remembers the domains `google,keep-boot-on` kept and
  adds a `release_kept` parameter that runs genpd's `sync_state` power-off
  for them on demand. `camera-csis-tzpc-test` sets `google,keep-boot-on` and
  `google,tzpc-smc` on `pd_csis`, with the `camera-csis-test` bus (domain
  and clock). `tools/camera-csis-tzpc-probe` (first run) turns
  `tzpc_smc` on, releases the domain (the save, then its first power-off,
  from the bootloader's state), powers it on (the restore), and reads one
  register only after a clean save and restore; `HELD=1` reads all 56,
  then makes the power-off with the save.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0zc | `images` (kernel `g90a130fd6`, `camera-csis-tzpc-test`), **push (human's go)**; then `tools/phone-ssh 'dmesg \| grep -E "Power domain csis\|17462400.power-domain"; grep -E "^csis" -A2 /sys/kernel/debug/pm_genpd/pm_genpd_summary; cat /sys/module/exynos_pm_domains/parameters/tzpc_smc'` | boots; "csis … at probe"; "left as the bootloader left it"; `csis on` with the bus `suspended`; `N`; no "smc returned", no "csis off" | no boot; `csis off-0`; any DTZPC call at boot | push with `camera-csis-test` (kernel `g2572f3c60`) | low |
  | RC2.0zd | `tools/camera-stage --watchdog x --cmd 'READ=cmu1 bash -s' --script tools/camera-csis-tzpc-probe` | "released (release_kept)"; "DTZPC save 0x1a410204: smc returned 0x0"; `csis off-0` (the power-off from the bootloader's state); then "csis on", "DTZPC restore … returned 0x0", `active`, the clock on; **"0x1a400600 = 0x…"** (RC2.0w read `0x10`); the bus stays on | a hang (the restore didn't make the block reachable, or EL3 hung); a non-zero return; the release power-off fails: record, stop | a reboot (pstore after a reset) | medium: the first read after a restore; the first save from the bootloader's state |
  | RC2.0ze | `tools/camera-stage --watchdog x --cmd 'READ=cmu HELD=1 bash -s' --script tools/camera-csis-tzpc-probe` | 56 values (compare with RC2.0x); then "DTZPC save … 0x0" and **"pd_csis powered off"** (`csis off-0`) | a hang; "disable failed (status 0x…)" (record: the save and restore don't fix the power-off) | a reboot | low after RC2.0zd |

  RC2.0ze (§5): the save and restore make the block reachable, but the
  power-off after our power-on still sticks, and a power cycle resets three
  CMU_CSIS registers: the user mux, `CONTROLLER_OPTION` and the BUSP
  divider. RC2.0zf-zg write those back, as stock writes back what it saved
  after every power-on. Kernel `3b9217136` reads a `google,cmu-restore`
  list of address/value pairs and, with its `cmu_restore` parameter set,
  writes them after a power-on **only when the DTZPC restore just ran and
  returned 0** (otherwise it logs "CMU restore skipped"); each write logs
  the value it reads back. A bad property leaves the domain without the
  restore, never without the domain. `camera-csis-tzpc-test` adds the three
  pairs with the bootloader's values (RC2.0x) in stock's save order.
  `tools/camera-csis-tzpc-probe` `CMU=1` turns `cmu_restore` on with
  `tzpc_smc` and requires three restore lines that read back their values
  before any read; the EXIT trap turns both off.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0zf0 | `images` (kernel `g3b9217136`, `camera-csis-tzpc-test` with `google,cmu-restore`), **push (human's go)**: also clears RC2.0ze's stuck domain; then `tools/phone-ssh 'dmesg \| grep -E "Power domain csis\|17462400.power-domain\|cmu-restore"; grep -E "^csis" -A2 /sys/kernel/debug/pm_genpd/pm_genpd_summary; grep . /sys/module/exynos_pm_domains/parameters/{tzpc_smc,cmu_restore}'` | boots; "csis … at probe"; "left as the bootloader left it"; `csis on`; `N` and `N`; no "smc returned", no "CMU restore", no "cmu-restore ignored" | no boot; any DTZPC call or CMU write at boot; "ignored" | push kernel `g90a130fd6` with the RC2.0zc overlay | low |
  | RC2.0zf | `tools/camera-stage --watchdog x --cmd 'CMU=1 READ=cmu1 bash -s' --script tools/camera-csis-tzpc-probe` | as RC2.0zd, then three "CMU restore 0x…1a400600 = 0x10, reads 0x10" (`0x1a400800` `0xf1000000`, `0x1a401800` `0x1`) lines that read back their values; **"0x1a400600 = 0x00000010"**; the bus stays on | a hang (a CMU write to a live clock tree); a value that doesn't read back (the probe stops before reading); "restore skipped" | a reboot | medium: the first writes to CMU_CSIS from Linux; the values are the bootloader's, which RC2.0x read in a working block |
  | RC2.0zg | `tools/camera-stage --watchdog x --cmd 'CMU=1 READ=cmu HELD=1 bash -s' --script tools/camera-csis-tzpc-probe` | 56 values, **all equal to RC2.0x's**; "DTZPC save … 0x0", then **"pd_csis powered off"** (`csis off-0`) | a hang; "disable failed (status 0x1)" (record: the CMU state isn't it either; next, the 51 Q-channels and the clock order at power-off) | a reboot | low after RC2.0zf |

  RC2.0zg passed (§5). RC2.0zh does the same through the clock driver, the
  upstream way, instead of the PD driver's raw writes. Kernel `b8cf607d1`
  (four commits on `3b9217136`):

  - `dt-bindings: clock: google,gs101-clock: add CMU_CSIS`: the compatible
    and 73 clock IDs.
  - `clk: samsung: gs101: add CMU_CSIS`: the user mux, the BUSP divider
    and 71 gates from Google's CAL (`cmucal-sfr.c`, `cmucal-node.c`),
    registered with `exynos_arm64_register_cmu_pm()` (as exynos5433's
    CMUs): at probe it writes the global auto-gating option (`0xf1000000`,
    the bootloader's value), saves all 126 registers on runtime suspend
    and writes them back on runtime resume.
  - `arm64: dts: exynos: gs101: add the CSIS clock controller`: `cmu_csis`
    in `pd_csis`, **disabled**.
  - `pmdomain: … never power off a tzpc-smc domain without the save`
    (bring-up): with `tzpc_smc` clear a `google,tzpc-smc` domain stays on
    ("kept on: no DTZPC save", `-EBUSY`) instead of powering off with no
    save, and after a successful save every power-on restores whatever
    `tzpc_smc` says. So the CMU's runtime resume can't reach an
    unrestored block (the RC2.0za hang).

  No `samsung,sysreg` yet (dt-validate's one new message): with it, the
  driver also writes SYSREG_CSIS's DRCG_EN (all ones) and MEMCLK (clear),
  away from the bootloader's 0 and 1; that is its own later step.
  `camera-csis-cmu-test` (replaces `camera-csis-tzpc-test` in
  `DTB_OVERLAYS`) keeps `pd_csis` on with `google,tzpc-smc`, enables
  `cmu_csis`, and gives the test bus CMU_CSIS's own PCLK, so that powering
  the bus runtime-resumes the CMU. `tools/camera-csis-cmu-probe` releases
  the domain with the save, then makes two on/off cycles, each reading the
  56 registers and comparing them with RC2.0x's; it reads clock counters,
  never `clk_summary` (which would resume the CMU).

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0zh0 | `images` (kernel `gb8cf607d1`, `camera-csis-cmu-test`), **push (human's go)**; then `tools/phone-ssh 'dmesg \| grep -E "Power domain csis\|17462400.power-domain\|1a400000\|cmu_csis"; grep -E "^csis" -A3 /sys/kernel/debug/pm_genpd/pm_genpd_summary; readlink /sys/bus/platform/devices/1a400000.clock-controller/driver; cat /sys/bus/platform/devices/1a400000.clock-controller/power/runtime_status /sys/module/exynos_pm_domains/parameters/tzpc_smc'` | boots; "left as the bootloader left it"; `csis on` with the CMU and the bus `suspended`; the CMU bound to `gs101-cmu`; "Unable to get CMU sysreg" (expected); `N`; no "smc returned", no "kept on" | no boot (the CMU's probe-time access); the CMU not bound; `csis off-0` | push kernel `g3b9217136` with `camera-csis-tzpc-test` | medium: the CMU now probes at boot, in the bootloader's state (RC2.0x read that state safely) |
  | RC2.0zh | `tools/camera-stage --watchdog x --cmd 'bash -s' --script tools/camera-csis-cmu-probe` | the save and `off-0`; then twice: "csis on", one more clean restore, the bus and CMU `active`, **"56 of 56 registers equal the bootloader's"**, then "pd_csis powered off" with one more clean save | a hang; a mismatch (recorded, the run goes on to the power-off); "did not power off"; "kept on" | a reboot | medium: the clock driver's first restore of 126 registers (RC2.0zf wrote three) |

  RC2.0zh passed (§5). RC2.0zi adds SYSREG_CSIS (kernel `802343823`,
  three commits on `b8cf607d1`): a `google,gs101-csis-sysreg` syscon at
  `0x1a420000`, `samsung,sysreg` on `cmu_csis`, and the DRCG_EN and MEMCLK
  offsets in `csis_cmu_info`, as CMU_DPU has them. At probe the CMU then
  writes DRCG_EN all ones and clears MEMCLK bit 0 (the bootloader leaves
  0 and 1), and saves and restores both across runtime PM. **The syscon
  has no clock**, unlike gs101's other sysregs: a syscon keeps its clock
  prepared for good (`regmap_mmio_attach_clk()`), and a prepared CMU_CSIS
  clock would hold the CMU runtime-active and `pd_csis` on forever. Only
  the CMU touches it, from its probe and runtime PM, with the block
  clocked. Never read its debugfs regmap while `pd_csis` is off.
  `tools/camera-csis-cmu-probe SYSREG=drcg` prints the two SYSREG values
  instead of comparing them.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0zi0 | `images` (kernel `g802343823`, same overlays), push; then RC2.0zh0's check | as RC2.0zh0, but **no** "clock-controller@1a400000: Unable to get CMU sysreg" | no boot (the probe's SYSREG write); the CMU not bound | push kernel `gb8cf607d1` | medium: the first write to SYSREG_CSIS |
  | RC2.0zi | `tools/camera-stage --watchdog x --cmd 'SYSREG=drcg bash -s' --script tools/camera-csis-cmu-probe` | as RC2.0zh, with "54 of 54 registers equal the bootloader's", DRCG_EN non-zero and MEMCLK 0 in both cycles, and both power-offs complete | a hang; "did not power off" (record: DRCG/MEMCLK block the power-down; drop `samsung,sysreg` again) | a reboot | medium |

  RC2.0zi passed (§5). RC2.0zj drops the bring-up switches around the
  DTZPC: kernel `77c81d480` removes `tzpc_smc`, so a `google,tzpc-smc`
  domain always saves before a power-off and restores after a power-on,
  and refuses a power-on before any save ("no DTZPC save to restore,
  staying off", `-EIO`): its block can't be reached unrestored. The
  overlay drops `google,keep-boot-on`, so genpd's `sync_state` powers
  `pd_csis` off at boot and takes the first save from the bootloader's
  state, unattended (RC2.0zd took it attended). `release_kept` and
  `google,cmu-restore` stay in the kernel, unused, until the clean-up.
  `tools/camera-csis-cmu-probe` sees no `tzpc_smc` and expects the domain
  off with one clean boot-time save, then makes the two cycles.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0zj0 | `images` (kernel `g77c81d480`, `camera-csis-cmu-test` without `keep-boot-on`), push; then `tools/phone-ssh 'dmesg \| grep -E "Power domain csis\|17462400\|1a400000"; grep -E "^csis" -A3 /sys/kernel/debug/pm_genpd/pm_genpd_summary'` | boots; "csis … at probe"; one "DTZPC save 0x1a410204: smc returned 0x0" then "csis off: CONFIGURATION 0x1 -> 0x0" at `sync_state`; `csis off-0`, the CMU and bus `suspended`; no "kept on", no "no DTZPC save" | no boot (the unattended boot-time save); a non-zero return; `csis on` | push kernel `g8023438238d9` with the RC2.0zi overlay | medium: the first unattended DTZPC save at boot |
  | RC2.0zj | `tools/camera-stage --watchdog x --cmd 'SYSREG=drcg bash -s' --script tools/camera-csis-cmu-probe` | as RC2.0zi's two cycles: restore, 54 of 54, DRCG and MEMCLK kept, save, powered off | a hang; "did not power off"; "no DTZPC save" | a reboot | medium |

  RC2.0zj passed (§5). RC2.0zk is the first clean-up: kernel `5f4d1967a`
  drops `google,keep-boot-on`, `release_kept`, `google,cmu-restore` and
  the bring-up logging from the PD driver (`537fd97cb`; only DTZPC
  failures are logged now), and gives CMU_CSIS noirq system-sleep ops
  (`pm_runtime_force_suspend`/`resume`, as exynos5433's CMUs; not
  exercised until suspend works on raven). The overlay is unchanged.
  Without the "smc returned" lines `tools/camera-csis-cmu-probe` runs
  QUIET: genpd's state stands in for them (a failed restore fails the
  power-on, a failed save the power-off, and a power-on before any save
  is refused), and every DTZPC failure line stops it before a read.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0zk0 | `images` (kernel `g5f4d1967a`, same overlays), push; then `tools/phone-ssh 'dmesg \| grep -E "Power domain\|17462400\|1a400000"; grep -E "^csis" -A3 /sys/kernel/debug/pm_genpd/pm_genpd_summary'` | boots; no "Power domain" lines at all; `csis off-0`, the CMU and bus `suspended` | no boot; any "failed" or "no DTZPC save"; `csis on` | push kernel `g77c81d480b7d` | low: removes code, the active path unchanged |
  | RC2.0zk | `tools/camera-stage --watchdog x --cmd 'SYSREG=drcg bash -s' --script tools/camera-csis-cmu-probe` | "quiet=1"; as RC2.0zj's two cycles: 54 of 54, DRCG and MEMCLK kept, powered off | a hang; "did not power off"; any failure line | a reboot | medium |

  RC2.0zk passed (§5). RC2.0zl moves the DTZPC onto upstream's design.
  André Draszik's "pmdomain: samsung: add support for Google GS101" (v8,
  2026-03-18; bindings reviewed by Rob Herring, held over which tree takes
  which binding, not in our 7.3-rc6 base) describes each DTZPC as a
  `google,gs101-dtzpc` node and points the domain at it with
  `samsung,dtzpc`; the driver adds the SoC's offset (0x204 on gs101) and
  makes one save at probe, and a domain whose firmware answers `-ENOENT`
  stays always on. Kernel `37b6fd866`:
  - `962edcea2`: the driver code of v8 patch 9, adapted to our gs101 base;
  - `e6ee3d73a` (bring-up): the probe-time save counts only from a
    powered domain and only when it returns 0; a power-on with no good
    save is still refused;
  - `d3b3d5dc1`: reverts `611093c5b` (a full CONFIGURATION write again,
    as v8 does; RC2.0s showed it holds only the power bit);
  - `a0ab6a477` (André's patch 1, verbatim), `e50c4efae` (`samsung,dtzpc`
    from patch 2): the bindings;
  - `37b6fd866`: `dtzpc_csis` at 0x1a410000 in gs101.dtsi.
  The overlay sets `samsung,dtzpc = <&dtzpc_csis>` instead of
  `google,tzpc-smc`. The SMC address is the same 0x1a410204; new are the
  save at probe (0.06 s, from the bootloader's state, before the boot-time
  power-off's own save) and the full CONFIGURATION write.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.0zl0 | `images` (kernel `g37b6fd866`, the overlay with `samsung,dtzpc`), push; then RC2.0zk0's check | boots; no "Power domain" or "SMC for power domain" line; `csis off-0`, the CMU and bus `suspended` | no boot (the probe-time save); any failure line; `csis on` | push kernel `g5f4d1967ac22` with the `google,tzpc-smc` overlay | medium: a new SMC at probe |
  | RC2.0zl | `tools/camera-stage --watchdog x --cmd 'SYSREG=drcg bash -s' --script tools/camera-csis-cmu-probe` | as RC2.0zk | as RC2.0zk | a reboot | medium |

  If RC2.0c aborts, the next test is another overlay, used instead of
  `camera-csis-test` (one boot). It sets `&pd_csis` `status = "disabled"`,
  so genpd never cycles the domain and ABL's state stays. Its bus holds only
  the clock: a `power-domains` reference to a disabled provider would defer
  the bus forever, and an overlay can't delete the property. If that read
  passes, the power cycle is the cause, and the PD driver needs the SMC
  (save before every power-off, so EL3 never restores an empty slot) and
  the CMU_CSIS restore.
- **RC2.1** the ultrawide's link, link only (no DMA). Stock's DT doesn't
  name the link; cheetah's ultrawide is on link 2 with four lanes, and
  raven uses the same pads, so link 2 goes first. Kernel `e8478f15a`:
  - `c065e1fc2`: the CSIS and DC-PHY bindings take `google,gs101-csis` /
    `google,gs101-mipi-dcphy-rx` before the gs201 compatible, and the
    CSIS's `link` and `dma` clocks;
  - `6cedd38e5`: gs201-csis powers the block (pd_csis, the CMU through its
    clocks) by runtime PM, only while streaming; the IRQs and debugfs
    files touch no register while it is off (gs201's block is always on,
    gs101's is RC2.0's power-cycled domain);
  - `b7c9e0bda`: the DC-PHY enables runtime PM, so `phy_power_on` resumes
    pd_csis through the PHY core;
  - `64a6db97b`: CSIS links 0, 1, 2, 4 and 6 and the DC-PHY in gs101.dtsi
    (Google's gs101-isp.dtsi addresses: link n at 0x1a440000 + n·0x10000,
    SPI 294 + n; WDMA context 0 at 0x1a4d7000, SPI 302; PHY isolation PMU
    0x3ebc), disabled;
  - `e8478f15a`: raven's IMX386 endpoint to `csis2`, a 128 MiB camera pool
    below 4 GiB, and the DC-PHY's firmware name.
  The new overlay `camera-uw-stream` enables `csis2` and `csis_dcphy`;
  `camera-csis-cmu-test` stays (pd_csis's DTZPC, cmu_csis). Before the
  push: `phy_gs201_mipi_dcphy_rx` and `gs201_csis` are blacklisted in the
  phone's `omamobile-camera-bringup.conf`, and raven's own `gs201-dcphy.bin`
  (from raven's HAL, never committed) is in
  `/usr/lib/firmware/updates/omamobile/camera/`. On raven,
  `tools/camera-cap-test` powers the camera PMIC as `camera-uw-probe` does
  (held at most 120 s) and powers it down at exit. Cheetah's CAM DVFS
  votes don't apply: gs101's node has no `cam`/`int` clock, so the block
  runs at the bootloader's rates.
- **RC2.2** one raw frame to memory through CSIS WDMA context 0. Then
  `cam -c1 --capture=10` through libcamera's simple pipeline and soft ISP.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.1a | `images` (kernel `ge8478f15a`, `camera-uw-stream` added), push; then `tools/phone-ssh 'lsmod \| grep -E "gs201\|imx386"; ls /sys/bus/platform/devices/ \| grep -E "csis\|phy@1a4f"; grep -E "^csis" -A3 /sys/kernel/debug/pm_genpd/pm_genpd_summary'` | boots; the two devices exist, neither module loaded; `csis off-0` | no boot; a module loaded (the blacklist); `csis on` | push kernel `g37b6fd8663e4` without `camera-uw-stream` | low: disabled-by-default nodes, modules blacklisted |
  | RC2.1b | `tools/phone-ssh 'cat > /run/camera-pmic-gpio' < tools/camera-pmic-gpio`; `tools/camera-stage --watchdog x --cmd 'LINK=2 DMA=0 bash -s' --script tools/camera-cap-test` | modules bind; one sensor-to-CSIS link; colour bars stream 5 s; `stats` shows frame starts and ends rising on link 2, no CRC/ECC errors | a hang (watchdog reset); any DTZPC failure line; an abort | a reboot | medium: the first CSIS and PHY writes on gs101, under the watchdog |
  | RC2.2a | `tools/camera-stage --watchdog x --cmd 'LINK=2 DMA=1 FRAMES=4 bash -s' --script tools/camera-cap-test` | 4 frames in `/tmp/cam-2016x1508.raw` (4 × 2016×1508 packed 10-bit, stride aligned), colour bars when unpacked on the laptop | a hang; an SMMU/S2MPU fault; an all-zero file (DMA blocked) | a reboot | medium: the first DMA write |
  | RC2.2b | as RC2.2a with `PATTERN=0` | a real image (dark or lit) | as RC2.2a | a reboot | medium |

  If RC2.1b streams but link 2 sees nothing, the next boot moves the
  endpoint to another link (one per boot, link 0, 1, 4, 6 in that order),
  through an overlay that rewrites the two `remote-endpoint`s.

- **RC2.3** the result in the kernel DT, then libcamera. Kernel fixups
  `2ffe4e64b` (binding: gs101's cam clock is the CSIS bus divider),
  `47fc32224` (every gs101 CSIS node takes `CLK_DOUT_CMU_CSIS_BUS` as
  `cam`) and `cda13b14b` (raven's IMX386 on `csis1`). `camera-uw-stream`
  now enables `csis1`; the link overlays are gone.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.3a | `images`, push; `tools/camera-stage --watchdog x --cmd 'LINK=1 DMA=1 FRAMES=4 bash -s' --script tools/camera-cap-test` | as RC2.2a without an overlay's help: colour bars, INT0 bit 3 only | a hang; line errors (the cam clock missing) | push `g3f5d357b540d` with `camera-uw-link1` | medium |
  | RC2.3b | on the phone: `modprobe phy_gs201_mipi_dcphy_rx gs201_csis` with the PMIC held, then `cam -l` and `cam -c1 --capture=10 --file=/tmp/uw-#.ppm` | the IMX386 listed through the simple pipeline; 10 frames through the soft ISP | a hang; no camera listed | a reboot | medium |

- **RC2.4** the camera PMIC powered by the kernel, not `camera-pmic-gpio`.
  RC2.3's `PU=0` run (`logs/camera-20261025-171803-x.log`, colour
  bars on link 1 with S2MPG11 GPIO 4 low) showed the ultrawide doesn't
  need `pu`. Kernel `ff228c850` (binding: an optional `vdd-supply`),
  `ba663f3c2` (driver: the supply and chip select switched through runtime
  PM, autosuspend 1 s after the last LDO goes off, a register cache across
  the reset) and `5adfebfb1` (raven: `bb` and `buck` as chained fixed
  regulators, 2 ms each, into `vdd`). `tools/camera-cap-test` sees the
  `vdd-supply` property and skips its GPIO holder.

  | Step | Command | Expect / Pass | Stop if | Rollback | Risk |
  | --- | --- | --- | --- | --- | --- |
  | RC2.4a | `images`, push; `tools/phone-ssh 'ls /sys/class/regulator/*/name \| xargs grep -l cam-pmic; grep -E "gpio-(0\|1\|3)" /sys/kernel/debug/gpio'` | boots, taint 0; `cam-pmic-bb` and `cam-pmic-buck` registered; S2MPG11 GPIOs 1 and 3 requested and low | no boot; either GPIO high | push `ga3ad00e61091` | low: two off-by-default regulators |
  | RC2.4b | `tools/camera-stage --watchdog x --cmd 'LINK=1 DMA=1 FRAMES=4 bash -s' --script tools/camera-cap-test` | the PMIC binds (I2C reads at probe), then `runtime_status` `suspended` within 2 s; IMX386 ID read, colour bars as RC2.3a; after the run GPIOs 0, 1, 3 low | probe fails (an I2C NACK: the chip didn't power up); the chip stays `active` with every LDO off; a hang | a reboot | medium: a new power path for the camera PMIC |
  | RC2.4c | as RC2.4b with `CAM="-c1 -s width=2016,height=1508 --capture=30"` | libcamera captures as RC2.3c | as RC2.4b | a reboot | medium |

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

- **RC1.3d** (`logs/camera-20261023-220801-x.log`, watchdog armed): the
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

### RC1.4a-b on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-ga01f3357ac7e`): pass

- **RC1.4a** `blacklist imx386` added (three lines in
  `omamobile-camera-bringup.conf`); the mode file installed as
  `/usr/lib/firmware/updates/omamobile/camera/imx386-uw.bin`, checksum as
  on the laptop.
- **RC1.4b** pushed and booted. `camera-sensor@1a` `okay`, `0-001a`
  (`imx386`) present and unbound, `imx386` not loaded,
  `camera-eeprom-ldo1` gone; `98-9e` as before. `tools/phone-smoke` as
  before (141 dmesg error lines, 70 of them the AoC mailbox at boot). The
  battery read "Discharging" at 94 % on the cable right after this boot,
  as after RC1.3b, and "Charging" again later that time: the charger
  settling after boot, not the camera work [I].

### RC1.4c-d on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-ga01f3357ac7e`): pass

`logs/camera-20261023-230229-x.log` (kmsg streamed, watchdog armed).

- The PMIC came up as in RC1.3 (`99`: `0a`, `0b`, `1b`); LDO1 and LDO4
  off (3.05 V, 2.90 V), `dout_cmu_cis_clk0` at 24576000 Hz, enable 0.
- `modprobe imx386`: "mode 4032x2272: link frequency 794624000 not in DT,
  dropped" (expected: the DT lists 774.144 MHz only), then **"imx386
  0-001a: model 0x0386"** 44 ms after the probe began (15 ms of stock's
  waits plus the I2C). Bound, `runtime_status=suspended`; LDO1, LDO4 and
  the MCLK back off. Power-down to `99: 00`.
- **The ultrawide's whole power path is stock's and works:** PMIC, LDO1
  and LDO4, MCLK from the oscillator on gpp3-0, reset on gpp6-3, and the
  sensor at 0x1a on `hsi2c_3`. The DFTMUX gate stock names wasn't needed.
- No kernel messages after the step but the expected ones (driver lines,
  SSH audit records). **Correction:** the "no errors after the step"
  checks for RC1.3c and RC1.3d used an `awk` time filter that never
  matched (`[` before the number), so they checked nothing; their laptop
  logs, checked again from each run's start marker, show only the
  expected driver lines, so the results stand.
- **RC1.4d** `tools/phone-smoke` as before.

### RC2.0 offline checks (2026-10-24, `7.3.0-rc6-omamobile-push1-ga01f3357ac7e`): pass

- No kernel change: `camera-csis-test` on the running kernel's DTB. It
  compiles; `fdtoverlay` applies it after the other eight overlays. The
  node resolves to `pd_csis` (`power-domain@17462400`) and `cmu_top` clock
  95 (`CLK_DOUT_CMU_CSIS_BUS`), and dtc gives no warning on the applied
  tree. `dt-validate`: 928 lines before and after, the same baseline;
  nothing about `bus@1a440000`.
- On the phone (read-only): `csis off-0`, `pdp off-0`;
  `gout_cmu_csis_bus` enable 0, hardware `N` (399.36 MHz parent,
  `dout_cmu_csis_bus` 66.56 MHz); `simple-pm-bus` is built in (it binds
  `soc@0`); no `devmem` or busybox on the root, hence the Python reader.
- `tools/camera-csis-probe`: `bash -n` clean, its reader compiles. The
  reader maps one page `PROT_READ` from `/dev/mem` (`O_SYNC`) and loads
  one aligned 32-bit word.


### RC2.0a on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-ga01f3357ac7e`): pass

`logs/push-raven-camera-5.log`. Same kernel, DTB with `camera-csis-test`.
Booted; `systemctl is-system-running` `running`, taint 0. genpd: `csis
off-0`, `pdp off-0`, and `soc@0:bus@1a440000` listed `suspended`, bound to
`simple-pm-bus`. No "Power domain csis enable/disable failed", no abort or
SError in the boot log, so the boot-time attach and power-off went through.
`gout_cmu_csis_bus` enable 0, hardware `N`, as before.

### RC2.0b on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-ga01f3357ac7e`): stop at power-off

`logs/camera-20261023-233319-x.log` (kmsg streamed, watchdog armed).

- **Power-on passed:** `power/control` `on` gave `csis on`,
  `runtime_status: active`, `gout_cmu_csis_bus enable=1 prepare=1 hw=Y`.
- **Power-off failed:** `auto` gave "Power domain csis disable failed" 2.5
  ms after the marker. genpd keeps `csis on` with the bus `suspended`, and
  the clock reads enable 0 but hardware `Y`. No abort, taint 0, the phone
  is fine (checked again 30 s later).
- **The likely cause** [I]: `exynos_pd_power()` gives a domain about 1 ms
  (10 polls, 80-100 µs apart) to reach the new status. Stock's `csis_off`
  is the same write and wait, but `pmucal_rae` waits up to 5000 polls of
  1 µs (`G/drivers/soc/google/cal-if/pmucal_rae.c:108-119`,
  `flexpmu_cal_local_gs101.h:1039-1042`). Another candidate: simple-pm-bus
  gates the CSIS bus clock just before genpd powers off. The boot-time
  power-off at 0.33 s (from ABL's state, clock untouched) worked.
- **So genpd and the hardware may disagree**: CONFIGURATION was written 0,
  and the block may have gone off after the timeout. **Nothing reads the
  block on this boot**; RC2.0c waits for a reboot (the next push does it).
- **Next (offline):** in the PD driver, wait as long as stock (5 ms), and on
  a timeout log the STATUS word read through the PMU regmap. That tells a
  slow power-off from a stuck one. Then repeat RC2.0b.

### RC2.0e offline checks (2026-10-24, `7.3.0-rc6-omamobile-push1-gb7be5bb94247`): pass

- `b7be5bb94` "pmdomain: samsung: exynos-pm-domains: wait longer for gs101
  domains": gs101 domains get 10 ms (Google's CAL: up to 5000 polls of 1
  µs), other Exynos domains keep 1 ms, and a timeout prints the last
  status word. `checkpatch --strict` clean on the diff (one older CHECK at
  the subdomain code); `W=1` with the Android clang: no warnings.
- Kernel, modules, initramfs and images with all nine overlays, including
  `camera-csis-test`.

### RC2.0e on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-gb7be5bb94247`): pass

`logs/push-raven-camera-6.log` (modules pushed too). Booted, `running`,
taint 0. `csis off-0`, `pdp off-0`, the bus `suspended`. No "Power domain
… failed" for any domain, no abort or SError. The AoC came up as usual
(sound card, `wlp1s0`, `usb0` present).

### RC2.0f on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-gb7be5bb94247`): stop, the power-off is stuck

`logs/camera-20261024-004159-x.log` (kmsg streamed, watchdog armed).

- Power-on as in RC2.0b (`csis on`, `active`, clock `enable=1 hw=Y`).
- Power-off: **"Power domain csis disable failed (status 0x1)"** after the
  full 10 ms. STATUS bit 0 is still 1: the domain is stuck, not slow, so
  the timeout was not the cause. CONFIGURATION is 0, so the request is
  pending. No abort, taint 0, `running`. No read on this boot.
- **The likely cause** [I]: after `csis_on`, stock restores CMU_CSIS's saved
  registers. These are about 40 `QCH_CON_*` (the Q-channels of the
  SysMMUs, SSMTs, QEs, PPMUs, the eight link wrappers and the sysreg),
  plus `DIV_CLK_CSIS_BUSP` and `PLL_CON0_MUX_CLKCMU_CSIS_BUS_USER`
  (`flexpmu_cal_local_gs101.h:927-…`). The PMU's power-down asks these
  Q-channels to quiesce. At reset values they may never answer, so the
  power-off hangs. The boot-time power-off works because it starts from
  ABL's configured CMU_CSIS.
- Stock saves these values at each power-off, from the running CMU. We
  never have them, because genpd powers the domain off at boot without a
  save. So the next step is to read what ABL leaves in CMU_CSIS, while
  the domain is still in ABL's state.

### RC2.0h offline checks (2026-10-24, `7.3.0-rc6-omamobile-push1-gb7be5bb94247`): pass

- `camera-csis-abl-test` applies after the other eight overlays, in place
  of `camera-csis-test`. `/chosen/bootargs` is the DT's own plus
  `pd_ignore_unused`; raven's ABL merges the DT bootargs with the vendor
  command line (the live `/proc/cmdline` has both). `bus@1a440000` holds
  only `cmu_top` clock 95. dtc gives no warnings on the applied tree;
  `dt-validate` gives 928 lines (the baseline), nothing about `/chosen` or
  the bus.
- `tools/camera-csis-abl-probe`: `bash -n` clean; the reader compiles; 56
  registers, as in stock's `csis_save` (54 in CMU_CSIS, 2 in SYSREG_CSIS).
- Images built.


### RC2.0h on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-gb7be5bb94247`): stop, ABL leaves `pd_csis` off

`logs/push-raven-camera-7.log`. Booted, `running`, taint 0;
`pd_ignore_unused` on the command line; "PM: genpd: Not disabling unused
power domains"; the bus `suspended` and under no domain; no "Power domain"
failure.

- **Every gs101 domain the PD driver manages reads `off-0`** with nothing
  switching them: `csis`, `pdp`, `dns`, `tpu`, `bo`, `tnr`, `gdc`, `mcsc`,
  `itp`, `ipp`, `g3aa`, `mfc`, `g2d`, `aoc`. That is STATUS as the driver
  read it from the PMU at probe. These reads are real: the same path read
  1 after RC2.0b's power-on and `0x1` in RC2.0f's failure. **So ABL hands
  over BLK_CSIS powered off** (as BLK_AOC, AUDIO-PLAN R2.1f), unlike
  cheetah. `dcadd701b`'s message ("blocks the bootloader leaves powered")
  doesn't hold for these domains on this phone; its measured saving needs
  another explanation.
- **Stop:** no CSIS read is safe in this state. RC2.0i-k don't run.
- **Corrections to RC2.0f's analysis:** there was never a working
  boot-time power-off of CSIS from ABL's state; the domain was never on.
  And stock's first camera power-on also starts from CMU_CSIS's reset
  values: it has saved nothing yet, so it restores nothing. So the missing
  restore can't be the whole difference. What stock does that we don't,
  around its first cycle: the DTZPC save SMC before the power-off, and its
  clock handling. Ours gates `gout_cmu_csis_bus` (simple-pm-bus's runtime
  suspend) just before genpd's power-off, and that gate still read
  hardware `Y` after the disable, in both RC2.0b and RC2.0f.
- **Next, one change:** the genpd power cycle without the clock: a bus
  with `pd_csis` only, no `clocks`, and no read. If the power-off then
  completes, gating the bus clock before the power-off is what sticks.

### RC2.0m offline checks (2026-10-24, `7.3.0-rc6-omamobile-push1-gb7be5bb94247`): pass

`camera-csis-pd-test` applies in place of `camera-csis-abl-test`: the bus
holds `pd_csis` only, and the bootargs are the DT's own. dtc gives no
warnings; `dt-validate` gives 928 lines (the baseline). `camera-csis-probe`
skips the clock and refuses `READ=1` when the bus has no `clocks`;
`bash -n` clean. Images built.


### RC2.0m on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-gb7be5bb94247`): pass

`logs/push-raven-camera-8.log`. Booted, `running`, taint 0, no
`pd_ignore_unused`. `csis off-0` with the bus `suspended` under it; no
"Power domain" failure, abort or SError. Whether genpd's attach really
cycled the domain at boot isn't logged, so this doesn't yet show a clean
power-off; RC2.0n does.

### RC2.0n on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-gb7be5bb94247`): stop, still stuck without the clock

`logs/camera-20261024-010136-x.log` (kmsg streamed, watchdog armed).

- Power-on: `csis on`, `active`. The clock was never enabled
  (`enable=0`), yet **`gout_cmu_csis_bus` reads hardware `Y` as soon as the
  domain is on**. The gate follows the block's own clock request
  (hardware-controlled), which explains RC2.0b and RC2.0f's `enable=0
  hw=Y`.
- Power-off: **"Power domain csis disable failed (status 0x1)"** again. So
  gating the bus clock was not the cause. No abort, taint 0.
- **What's left that stock does and we don't:** stock's `exynos-pd.c`
  `__exynos_pd_power_off()` adds only idle-IP bookkeeping and BTS
  scitokens around `cal_pd_control()`, and `cmu_id` serves only
  `cmucal-debug.c`. Inside `cal_pd_control()` (`pmucal_local.c`), stock
  runs the DTZPC SMC on every cycle. Power-off: save the CMU registers
  (reads), then the save SMC (`0x82000410`, mode 0, `0x1a410204`, group 2),
  then `csis_off`. Power-on: `csis_on`, then the restore SMC (mode 1), even
  on the first power-on from ABL's off state, then the register restore,
  which has nothing to restore yet. **The SMC is the one difference left**
  [I]: EL3 may do part of the power-down handshake for a TrustZone-guarded
  block.
- **Risk:** the AoC's save SMC hung EL3 (AUDIO-PLAN R2.1c-d), but BLK_AOC
  was powered off then. Here CSIS reads on (STATUS 1) when the save would
  run.
- The domain is again stuck mid-power-off on this boot: no read; the next
  push reboots it.

### RC2.0o-p offline checks (2026-10-24, kernel `6d15de5fd`): pass

checkpatch `--strict` clean; the driver builds with `W=1` (Android clang)
with no warnings; the object carries `exynos_pm_domains.tzpc_smc`.
`camera-csis-smc-test` applies in place of `camera-csis-pd-test`;
`dt-validate` gives 932 lines against the 928 baseline, the four new ones
being `google,tzpc-smc` on `power-domain@17462400`, which has no binding
(the AoC's has the same warning). `camera-csis-probe` with `SMC=1`
refuses to run without the property, the parameter, or with a DTZPC call
already made this boot, and refuses `READ=1`; `bash -n` clean,
shellcheck as before.

### RC2.0o on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g6d15de5fd7f1`): pass, with a correction

`logs/push-raven-camera-9.log`. Booted, `running`, taint 0, `tzpc_smc`
`N`, `csis off-0` with the bus `suspended` under it, no "smc returned", no
failure, no abort or SError.

- **The boot log differs from the prediction:** one line, at 0.334 s right
  after "PM: genpd: Disabling unused power domains": "Power domain csis:
  DTZPC save skipped (tzpc_smc=0)". There was no "restore skipped", so our
  power-on callback never ran at boot. **The PD driver read csis's STATUS
  as on at probe on this boot**, and genpd powered it off as unused. The
  bus's attach found the domain already on and called nothing. That
  power-off completed (no "disable failed").
- RC2.0h read `off-0` at probe (with `pd_ignore_unused`), so the hand-off
  state isn't the same every boot. This boot followed RC2.0n's stuck
  power-off (STATUS 0x1) by a warm reboot; the PMU may keep that state
  across it [I]. RC2.0a and RC2.0m probably went the same way.
- §4's "boot cycles `pd_csis`" is wrong as stated: boot powers it off
  only when it reads on at probe. What stands: **a power-off from the
  state found at boot works, and the power-off after our own power-on
  sticks.** RC2.0p's preconditions hold.

### RC2.0p on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g6d15de5fd7f1`): stop, the DTZPC calls aren't the difference

`logs/camera-20261024-013250-x.log` (kmsg streamed, watchdog armed).

- **The first DTZPC calls to EL3 on gs101 work:** after the power-on,
  "DTZPC restore 0x1a410204: smc returned 0x0"; before the power-off,
  "DTZPC save 0x1a410204: smc returned 0x0". No hang, no reset, taint 0,
  no abort. (On the AoC they hung only with the block powered off.)
- **The power-off still sticks:** "Power domain csis disable failed
  (status 0x1)", 10 ms after the save. `tzpc_smc` back to `N`.
- Read-only afterwards: the CSIS bus clock chain runs in hardware
  (`mout_cmu_csis_bus` 399.36 MHz from `dout_cmu_shared2_div2`,
  `gout_cmu_csis_bus` `Y`, `dout_cmu_csis_bus` 66.56 MHz `Y`), so the
  block's clock isn't missing. Every other camera domain reads `off-0`.
- **What stock does differently, from its DT** (`gs101-isp.dtsi`):
  `lwis_csi`, which owns `0x1a440000`, holds `pd_pdp`, not `pd_csis`, and
  the camera's top device holds `pd_pdp`, `pd_csis`, `pd_gdc`, `pd_itp`,
  `pd_mcsc` and `pd_tnr` together, PDP first. **Stock never runs CSIS
  with PDP off.** CSIS's OTF and VO bridges go to PDP
  (`LHS_AST_OTF0-2_CSISPDP`, `LHS_AST_VO_CSISPDP`, `LHM_AXI_D_PDPCSIS` in
  CMU_CSIS's QCH list); a low-power handshake on a bridge whose partner
  is unpowered may never finish [I]. That would also fit the boot-time
  power-off that worked (RC2.0o), if PDP was still on when genpd turned
  CSIS off as unused [I, unchecked].
- The domain is stuck mid-power-off on this boot: no read.

### RC2.0q-r offline checks (2026-10-24, kernel `6d15de5fd`): pass

`camera-csis-pdp-test` applies in place of `camera-csis-smc-test`:
`bus@1aa40000` holds phandle `0xa2`, `power-domain@17462480` (`pdp`,
`okay`). `dt-validate` gives 932 lines, as RC2.0o (no new warning).
`camera-csis-probe PDP=1` refuses to run without the PDP bus, with
`pdp` not `off-0`, or after a `pdp` failure this boot; the EXIT trap
powers CSIS off before PDP. `bash -n` clean, shellcheck as before.
Images built (`logs/images-raven-camera-12.log`).

### RC2.0q on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g6d15de5fd7f1`): pass

`logs/push-raven-camera-10.log`. Booted, `running`, taint 0, no abort or
SError. `pdp off-0` holds `soc@0:bus@1aa40000` (`simple-pm-bus`,
`suspended`) besides its `ipp` subdomain; `csis off-0` holds its bus as
before. No "Power domain … failed". As in RC2.0o, the one DTZPC line is
genpd's unused power-off of `csis` at 0.333 s ("save skipped"): csis read
on at probe again after RC2.0p's stuck power-off and a warm reboot.

### RC2.0r on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g6d15de5fd7f1`): stop, PDP sticks too

`logs/camera-20261024-014151-x.log` (kmsg streamed, watchdog armed).
`pdp on`, then `csis on`, `active`. Then "Power domain csis disable failed
(status 0x1)" with PDP still on, and "Power domain pdp disable failed
(status 0x1)" after it. No abort, taint 0. Both domains stuck on this
boot.

- **Holding PDP doesn't help, and PDP fails the same way.** So the
  failure isn't CSIS's: **every power-off that follows our own power-on
  sticks** (CSIS in RC2.0b, f, n, p, r; PDP in RC2.0r), while genpd's
  unused power-off of a domain found on at probe works (RC2.0o, q).
- **Difference found in stock's code:** stock's `csis_on`/`csis_off`
  (and PDP's) are `PMUCAL_WRITE` with mask `0x1`, and
  `pmucal_rae_write()` (`pmucal_rae.c:150-162`) read-modify-writes when
  the mask isn't `U32_MAX`: it changes bit 0 only. Our gs101 support
  (`4134b5ea0`) writes the whole `CONFIGURATION` register: `0x1` on, `0x0`
  off. **Our power-on clears every other bit of the domain's
  CONFIGURATION** [I: which bits, and what they do, is unread]. A
  domain found on at probe still has the bootloader's other bits,
  which fits its working power-off.
- Next, one change: write bit 0 only (`regmap_update_bits`), and log
  CONFIGURATION through the kernel's PMU regmap (never from userspace).

### RC2.0s-t offline checks (2026-10-24, kernel `afcea63ba`): pass

`611093c5b` and `afcea63ba`: checkpatch `--strict` clean; the driver builds
with `W=1` (Android clang) with no warnings. `regmap_update_bits()` on the
gs101 PMU regmap goes through `tensor_sec_update_bits()`, which for a
PMU_ALIVE register (offset up to 0x3fff; CONFIGURATION is 0x2400 and
0x2480) writes bit 0 through the set or clear alias and touches no other
bit. No DT or script change.

### RC2.0s on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-gafcea63ba00c`): pass, and the bit-0 theory is out

`logs/push-raven-camera-11.log`. Booted, taint 0, no abort, SError or
"SMC failed".

- **Every gs101 domain read `CONFIGURATION 0x1 STATUS 0x1` at probe:**
  aoc, g2d, mfc, csis, pdp, g3aa, ipp, itp, mcsc, gdc, tnr, bo, tpu, dns.
  CONFIGURATION holds nothing but the power bit, so the full write
  (`4134b5ea0`) and the bit-0 write (`611093c5b`) are the same here.
  RC2.0t (no other change) would repeat RC2.0n: it doesn't run.
- At 0.330 s genpd powered all 14 off as unused, each "CONFIGURATION 0x1
  -> 0x0", **with no failure**. `csis` and `pdp` `off-0`.
- So on this boot the bootloader handed every domain over on, which
  contradicts RC2.0h's `off-0` for all of them (that boot used
  `pd_ignore_unused`, which should have kept the probe state). Not
  explained; whether the hand-off state depends on the kind of reboot is
  open.
- **Where stock differs:** stock's first action on these domains is a
  power-off from the bootloader's on state, and before it
  `pmucal_rae_save_seq()` saves the block's CMU registers (`csis_save`: 56,
  the dividers, the user mux and every QCH_CON). Each later power-on
  restores them (`pmucal_rae_restore_seq()`, `need_restore`). **Stock
  never runs a block with its CMU at reset values; we always do after our
  power-on**, and that is exactly when the power-off sticks. That the
  power-off handshake runs over Q-channels whose reset configuration
  doesn't answer is a guess [I].
- `611093c5b`'s message says the full write cleared other bits; there are
  none. It still matches stock and is harmless, but its message must be
  reworded (or the commit dropped) before it goes anywhere.

### RC2.0u-x offline checks (2026-10-24, kernel `afcea63ba`): pass

`camera-csis-abl-test` applies in place of `camera-csis-pdp-test`;
`dt-validate` gives 928 lines (the baseline: `google,tzpc-smc` is gone).
`camera-csis-abl-probe`'s "switched this boot" check now matches only the
driver's on, off and failure lines, since `afcea63ba` logs "Power domain
csis: CONFIGURATION … at probe" at every boot. `bash -n` clean,
shellcheck as before. Images built (`logs/images-raven-camera-14.log`).

### RC2.0u on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-gafcea63ba00c`): stop, `pd_ignore_unused` doesn't keep the domains on

`logs/push-raven-camera-12.log`. Booted, taint 0, `pd_ignore_unused` on
the command line. "csis: CONFIGURATION 0x1 STATUS 0x1 at probe" (on at
hand-off, as in RC2.0s), then "PM: genpd: Not disabling unused power
domains" at 0.329265 s, **and then every domain off anyway**, starting
20 µs later ("aoc off: CONFIGURATION 0x1 -> 0x0", … "csis off", "pdp
off"). `csis off-0`: nothing to read; RC2.0v-x don't run.

- **Why:** `pd_ignore_unused` only stops `genpd_power_off_unused()`. A
  domain that is on at boot is held on by `genpd->stay_on` until its
  provider's `sync_state`, and `of_genpd_sync_state()` then clears
  `stay_on` and calls `genpd_power_off()` without checking
  `pd_ignore_unused` (`drivers/pmdomain/core.c:3684-3701`). Each gs101 PD is
  its own platform device with no consumer, so its `sync_state` runs as
  soon as it probes. `GENPD_FLAG_NO_SYNC_STATE` doesn't help: for a
  device-backed provider genpd sets the driver's `sync_state` anyway
  (`core.c:2696-2701`), unless the driver has its own.
- **Correction to RC2.0h:** its `off-0` for every domain was this
  power-off, not the hand-off state. The bootloader hands the camera
  domains over **on** (RC2.0s, RC2.0u), as `dcadd701b` says. RC2.0h's
  "ABL hands over BLK_CSIS powered off" and the comparison with cheetah
  are withdrawn.
- **To read the bootloader's CMU_CSIS state**, the domain must survive
  `sync_state`: a bring-up `.sync_state` in the PD driver that leaves a
  domain marked in the DT on (and `stay_on`), and calls
  `of_genpd_sync_state()` for the rest.

### RC2.0y offline checks (2026-10-24, kernel `2572f3c60`): pass

checkpatch `--strict` clean; the driver builds with `W=1` (Android clang)
with no warnings; the kernel build gives only the two usual config
overrides. genpd's own `dev_set_drv_sync_state()` returns `-EBUSY` (ignored)
once the driver has a `.sync_state`, so the driver's wins.
`camera-csis-abl-test` applies with the DT's own bootargs; `dt-validate`
gives 930 lines, the two new ones being `google,keep-boot-on`, which has
no binding (bring-up only). `camera-csis-abl-probe` refuses to run without
the property or the driver's "left as the bootloader left it" line;
`bash -n` clean, shellcheck as before. Images built
(`logs/images-raven-camera-15.log`).

### RC2.0y on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g2572f3c60926`): pass

`logs/push-raven-camera-13.log`. Booted, taint 0, no abort or SError.
"csis: CONFIGURATION 0x1 STATUS 0x1 at probe"; at 0.340643 s, right
after "genpd: Disabling unused power domains", "exynos-pd
17462400.power-domain: left as the bootloader left it
(google,keep-boot-on)"; no "csis off" line. **`csis on`**, as the
bootloader handed it over; `pdp off-0` ("pdp off: CONFIGURATION 0x1 ->
0x0"), as before. RC2.0v-x can run.

### RC2.0v on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g2572f3c60926`): pass

`logs/camera-20261024-023550-x.log`. `csis on` before, during and after;
`gout_cmu_csis_bus` `enable=1 prepare=1 hw=Y` while held, `enable=0 … hw=Y`
after `auto` (the gate follows the powered block, RC2.0n). No abort.

### RC2.0w on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g2572f3c60926`): pass, the first CMU_CSIS read

`logs/camera-20261024-023620-x.log` (kmsg streamed, watchdog armed).
**`PLL_CON0_MUX_CLKCMU_CSIS_BUS_USER 0x1a400600 = 0x00000010`**, read-exit
0, no abort, taint 0, `csis on` throughout.

- Bit 4 of an Exynos `PLL_CON0_MUX_*_USER` register is `MUX_SEL` (1: the
  clock from CMU_TOP, 0: OSCCLK) [I, from the gs101 CMU layout upstream's
  `clk-gs101.c` uses for other blocks]. So the bootloader has switched the
  CSIS bus clock to CMU_TOP's `dout_cmu_csis_bus`. If the reset value is 0,
  a block powered on by us runs its bus from the 24.576 MHz oscillator
  instead, and stock's restore after each power-on puts the 1 back.

### RC2.0x on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g2572f3c60926`): pass, CMU_CSIS as the bootloader left it

`logs/camera-20261024-023804-x.log` (kmsg streamed, watchdog armed). All
56 registers read, read-exit 0, no abort, `csis on` throughout. These are
the bootloader's configuration, not unique to the phone. Fields from
Google's GPL `cal-if/gs101/cmucal-sfr.c` (lines 3301-3302, 3789-3791,
9436-9439, 11967-11968).

| Register | Address | Value | Fields |
| --- | --- | --- | --- |
| `PLL_CON0_MUX_CLKCMU_CSIS_BUS_USER` | `0x1a400600` | `0x00000010` | `MUX_SEL` 1 (CMU_TOP's clock, not OSCCLK), `BUSY` 0 |
| `CSIS_CMU_CSIS_CONTROLLER_OPTION` | `0x1a400800` | `0xf1000000` | `ENABLE_POWER_MANAGEMENT` (29) 1, `ENABLE_AUTOMATIC_CLKGATING` (28) 1; bits 31, 30, 24 set too |
| `CLK_CON_DIV_DIV_CLK_CSIS_BUSP` | `0x1a401800` | `0x00000001` | `DIVRATIO` 1 (divide by 2), automatic clock gating (28) 0 |
| all 51 `QCH_CON_*` (`0x1a403048`-`0x1a403110`) | | **`0x003f0002`** each | **`ENABLE` 0**, **`CLOCK_REQ` 1**, `IGNORE_FORCE_PM_EN` 0, `EXPIRE_VAL` 0x3f |
| `SYSREG_CSIS_BUS_COMPONENT_DRCG_EN` | `0x1a420104` | `0x00000000` | dynamic root clock gating off |
| `SYSREG_CSIS_MEMCLK` | `0x1a420108` | `0x00000001` | |

- **The bootloader disables every CSIS Q-channel and forces its clock
  request** (`ENABLE` 0, `CLOCK_REQ` 1): each IP's clock runs under the
  CMU's manual control, and no IP's low-power handshake is used. Stock
  saves exactly these registers before its first power-off and writes them
  back after every power-on (`csis_save`).
- **What we don't know yet:** CMU_CSIS's reset values, which is what a
  block we power on runs with. If the reset `QCH_CON` has `ENABLE` 1, the
  power-down waits on Q-channel handshakes from IPs that may never answer
  (the bridges to PDP, IPP and MCSC, whose domains are off), which would
  explain a power-off that sticks only after our power-on [I].
- Next, read-only: the same 56 registers after our own power-on (the
  `camera-csis-test` bus with the clock), to get the reset values and
  compare. Then the fix: restore the bootloader's values after each
  power-on, as stock does.

### RC2.0z-zb offline checks (2026-10-24, kernel `2572f3c60`): pass

`camera-csis-test` applies in place of `camera-csis-abl-test`;
`dt-validate` gives 928 lines (the baseline). `camera-csis-probe`'s
generated list of 56 addresses equals `camera-csis-abl-probe`'s; `READ=cmu*`
needs the bus clock; `HELD=1` refuses to run unless the bus is held on and
`csis` reads on, and takes no `SMC` or `PDP`. `bash -n` clean, shellcheck
as before. Images built (`logs/images-raven-camera-16.log`).

### RC2.0z on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g2572f3c60926`): pass

`logs/push-raven-camera-14.log`. Booted, taint 0, no abort or SError.
"csis: CONFIGURATION 0x1 STATUS 0x1 at probe", then at 0.329320 s "csis
off: CONFIGURATION 0x1 -> 0x0", with no failure and no "left as the
bootloader left it". `csis off-0`, `soc@0:bus@1a440000` `suspended` under it.

### RC2.0za on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g2572f3c60926`): stop, the read hangs the SoC

`logs/camera-20261024-024808-x.log` (kmsg streamed, watchdog armed),
`logs/camera-pstore-20261024-024856` (empty).

- Our power-on: "csis on: CONFIGURATION 0x0 -> 0x1"; `csis on`, `active`,
  `gout_cmu_csis_bus enable=1 prepare=1 hw=Y`. Every check passed.
- Then the one load of `0x1a400600` (`PLL_CON0_MUX_CLKCMU_CSIS_BUS_USER`):
  the last line to reach the laptop is the probe's "READ=cmu1" mark at
  217.98 s; SSH timed out, the watchdog reset the phone, and it came back on
  slot B (same kernel, taint 0). **pstore is empty: a bus hang, not an
  abort.** RC2.0zb doesn't run.
- **The same register read `0x00000010` in the bootloader's state** (RC2.0w,
  RC2.0x). So after our power-on the block isn't reachable from the
  non-secure side, though the PMU reports it on.
- **What stock does that we don't, around exactly this:** after every
  power-on it calls EL3's DTZPC restore, then writes the saved CMU
  registers, which needs the block reachable. Its first power-off, from
  the bootloader's state, is preceded by the DTZPC save. Ours never saved:
  at boot the save was skipped (`tzpc_smc=0`), so RC2.0p's restore
  (returned 0) had nothing to put back. A D_TZPC_CSIS at its reset value
  (secure-only) would make the block unreachable to Linux [I]; the AoC
  showed the same pattern (its S2MPU read aborted until AOC_TZPC was
  restored, `c0872408a`).
- Next: let the save run at the boot-time power-off, from the bootloader's
  state (RC2.0p showed the save with the block on returns 0), then power
  on with the restore and read one register.

### RC2.0zc-ze offline checks (2026-10-24, kernel `90a130fd6`): pass

checkpatch `--strict` clean; the driver builds with `W=1` (Android clang)
with no warnings; the kernel build gives only the two usual config
overrides. `release_kept` calls the exported `of_genpd_sync_state()`, which
takes the genpd lock, clears `stay_on` and powers off only if no device
is active (the bus is suspended). `camera-csis-tzpc-test` applies in
place of `camera-csis-test`: the bus holds `pd_csis` (`0x178`) and clock
95; `dt-validate` gives 932 lines, the four new ones being the two
unbound bring-up properties on `power-domain@17462400`.
`tools/camera-csis-tzpc-probe`: `bash -n` and shellcheck clean; it reads
nothing without a clean save and a clean restore this boot, and turns
`tzpc_smc` off on any stop. Images built (`logs/images-raven-camera-17.log`).

### RC2.0zc on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g90a130fd6545`): pass

`logs/push-raven-camera-15.log`. Booted, taint 0, no abort or SError.
"csis: CONFIGURATION 0x1 STATUS 0x1 at probe"; at 0.339434 s "left as the
bootloader left it (google,keep-boot-on)"; no "csis off", no DTZPC line.
`csis on` with `soc@0:bus@1a440000` `suspended` under it; `tzpc_smc` `N`;
`release_kept` present (write-only).

### RC2.0zd on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g90a130fd6545`): pass, the restore makes the block reachable

`logs/camera-20261024-032719-x.log` (kmsg streamed, watchdog armed). No
abort, taint 0.

- 61.709044 s "released (release_kept): sync_state power-off"; 61.709122 s
  **"DTZPC save 0x1a410204: smc returned 0x0"**; 61.709149 s "csis off:
  CONFIGURATION 0x1 -> 0x0"; `csis off-0`. The first DTZPC save from the
  bootloader's state works, and the power-off after it completes.
- 61.787754 s "csis on: CONFIGURATION 0x0 -> 0x1"; 61.787961 s **"DTZPC
  restore 0x1a410204: smc returned 0x0"**; `csis on`, `active`,
  `gout_cmu_csis_bus enable=1 prepare=1 hw=Y`.
- **`0x1a400600 = 0x00000000`, no hang.** The same load hung the SoC in
  RC2.0za after a power-on whose restore had no save behind it. So **the
  block is reachable from Linux after a power-on only once EL3 restores
  D_TZPC_CSIS from a save taken in a working state** (the restore with no
  save, RC2.0p, returned 0 too but restored nothing usable) [I: the
  TZPC's reset value isn't read].
- `PLL_CON0_MUX_CLKCMU_CSIS_BUS_USER` is now its reset value: `MUX_SEL` 0,
  the oscillator. The bootloader's was `0x10` (RC2.0w). Nothing restores
  CMU_CSIS yet, so this is the state a block we power runs with.
- The bus and `tzpc_smc` stay on for RC2.0ze.

### RC2.0ze on the phone (2026-10-24, `7.3.0-rc6-omamobile-push1-g90a130fd6545`): reads pass, the power-off still sticks

`logs/camera-20261024-033302-x.log` (kmsg streamed, watchdog armed). No
abort, no SError; the phone stayed up and answered afterwards.

- **All 56 loads returned, no hang** (`read-exit=0`), about 347 s after
  the restore: the block stays reachable while the domain is held.
- Against the bootloader's values (RC2.0x):

  | Register | Bootloader (RC2.0x) | After our power-on and the restore |
  | --- | --- | --- |
  | `0x1a400600` `PLL_CON0_MUX_CLKCMU_CSIS_BUS_USER` | `0x00000010` | `0x00000000` |
  | `0x1a400800` `CSIS_CMU_CSIS_CONTROLLER_OPTION` | `0xf1000000` | `0x00000000` |
  | `0x1a401800` `CLK_CON_DIV_DIV_CLK_CSIS_BUSP` | `0x00000001` | `0x00000000` |
  | `0x1a403048`-`0x1a403110`, the 51 `QCH_CON`s | `0x003f0002` | `0x003f0002` |
  | `0x1a420104` SYSREG_CSIS `DRCG_EN` | `0x00000000` | `0x00000000` |
  | `0x1a420108` SYSREG_CSIS `MEMCLK` | `0x00000001` | `0x00000001` |

  So a power cycle resets three CMU_CSIS registers: the user mux (back to
  the oscillator), the controller option (the CMU's power-management and
  automatic clock-gating bits) and the BUSP divider. The Q-channels and
  SYSREG come back as the bootloader had them.
- `echo auto`: 409.422608 s **"DTZPC save 0x1a410204: smc returned
  0x0"**, 409.422641 s "csis off: CONFIGURATION 0x1 -> 0x0", 409.432737 s
  **"csis disable failed (status 0x1)"**; `csis on`, the bus `suspended`,
  `gout_cmu_csis_bus enable=0`. `tzpc_smc` back to N.
- So **the DTZPC save and restore make the block reachable, but don't fix
  the power-off**: it sticks after our own power-on with them as without
  them (RC2.0b-r). The power-off from the bootloader's state (RC2.0zd)
  still completes. The one difference left between the two states that
  this run can see is the three reset registers, the controller option
  above all: with its power-management bits clear the CMU may never
  answer the PMU's power-down handshake [I: stock restores all 56 before
  any use and never powers the block off with them at reset values;
  untested].
- pd_csis is stuck on until a reboot (the probe's preconditions refuse
  another run this boot).

### RC2.0zf0 on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g3b921713629f`): pass

The push wrote and verified `boot_b`, `vendor_boot_b` and `dtbo_b` and
rebooted, but no SSH came back: the phone booted (the screen showed
Omarchy running) while the laptop saw no working USB network link. Power-
cycling the laptop's USB hub brought the link back; the phone itself was
never touched, and its uptime shows the one boot. So the push's "no SSH
after reboot" was the laptop side of the link, not the kernel [I: the
hub's state isn't logged].

- Taint 0, no abort or SError.
- "csis: CONFIGURATION 0x1 STATUS 0x1 at probe"; "17462400.power-domain:
  left as the bootloader left it (google,keep-boot-on)"; `csis on`, the bus
  `suspended`.
- `tzpc_smc` N, `cmu_restore` N; no "smc returned", no "CMU restore", no
  "cmu-restore ignored". `google,cmu-restore` is in the live DT.

### RC2.0zf on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g3b921713629f`): pass, CMU_CSIS takes the bootloader's values back

`logs/camera-20261025-114519-x.log` (kmsg streamed, watchdog armed). Taint
0, no abort or SError.

- 99601.870170 s "released (release_kept)"; 99601.870244 s "DTZPC save
  0x1a410204: smc returned 0x0"; "csis off: CONFIGURATION 0x1 -> 0x0";
  `csis off-0`.
- Power-on through the bus: `csis on`, `active`, `gout_cmu_csis_bus
  enable=1`; 99601.958324 s "DTZPC restore 0x1a410204: smc returned 0x0",
  then, 16-44 us later, the three CMU writes, each reading back:
  `0x1a400600 = 0x10`, `0x1a400800 = 0xf1000000`, `0x1a401800 = 0x1`.
- **`0x1a400600 = 0x00000010`**: the user mux is back on CMU_TOP's clock,
  as the bootloader had it (RC2.0w). The first writes to CMU_CSIS from
  Linux, with no hang.
- The bus, `tzpc_smc` and `cmu_restore` stay on for RC2.0zg.

### RC2.0zg on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g3b921713629f`): pass, **pd_csis powers off after our own power-on**

`logs/camera-20261025-114609-x.log` (kmsg streamed, watchdog armed). Taint
0, no abort or SError, no "disable failed" this boot.

- **All 56 registers equal the bootloader's (RC2.0x)**: the user mux
  `0x10`, `CONTROLLER_OPTION` `0xf1000000`, the BUSP divider `0x1`, the 51
  `QCH_CON`s `0x003f0002`, `DRCG_EN` 0, `MEMCLK` 1.
- `echo auto`: 99659.646788 s "DTZPC save 0x1a410204: smc returned 0x0",
  99659.646828 s "csis off: CONFIGURATION 0x1 -> 0x0", then **"pd_csis
  powered off"**: `csis off-0`, the bus `suspended`. `tzpc_smc` and
  `cmu_restore` back to N.
- So the power-off that stuck in every run since RC2.0b (RC2.0ze, with the
  DTZPC save and restore but the CMU at reset values, included) completes
  once the three registers a power cycle resets are written back. The one
  change between RC2.0ze and RC2.0zg is those writes, so **CMU_CSIS's
  state is what the power-down handshake needs** [I: which of the three
  matters, `CONTROLLER_OPTION`'s power-management bits most likely, isn't
  separated].
- The full cycle now works from Linux, in stock's order: the DTZPC save
  from the bootloader's state, power-off, power-on, the DTZPC restore, the
  CMU restore, reads, the save, power-off.

Next: replace the bring-up mechanism (a DT list of raw addresses and
bootloader values in the PD driver) with an upstream-style design: a
CMU_CSIS clock driver that saves and restores its registers in its own
runtime PM, with pd_csis's DTZPC handling. Then the CSIS link registers.

### RC2.0zh0 on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-gb8cf607d142e`): pass

Boots on the first try (SSH back without a hub cycle). Taint 0, no abort.
"csis: CONFIGURATION 0x1 STATUS 0x1 at probe"; "/soc@0/clock-controller@1a400000:
Unable to get CMU sysreg" (expected, no `samsung,sysreg`); "left as the
bootloader left it"; `csis on` with `1a400000.clock-controller` and the bus
`suspended`; the CMU bound to `gs101-cmu`, 71 `gout_csis_*` clocks;
`tzpc_smc` N; no DTZPC call, no "kept on". The clock driver's probe-time
accesses to CMU_CSIS, in the bootloader's state, are safe.

### RC2.0zh on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-gb8cf607d142e`): pass, **pd_csis cycles with the clock driver restoring CMU_CSIS**

`logs/camera-20261025-123214-x.log` (kmsg streamed, watchdog armed). Taint
0, no abort or SError, no "failed" or "kept on" this boot.

- 30.130 s "released (release_kept)", "DTZPC save … 0x0", `off-0`.
- Cycle 1: 30.211 s "csis on", "DTZPC restore … 0x0"; the bus and the CMU
  `active`, `gout_csis_csis_pclk` and `gout_cmu_csis_bus` enabled;
  **"56 of 56 registers equal the bootloader's"**; `echo auto`: 30.603 s
  "DTZPC save … 0x0", **"pd_csis powered off"**, both devices `suspended`.
- Cycle 2: the same (30.701 s restore, 56 of 56, 31.100 s save,
  powered off).
- So the upstream-style path works: genpd's power-on, the DTZPC restore,
  then the CMU's runtime resume writing back the 126 registers it saved
  at probe (from the bootloader's state) makes the block usable and lets
  it power off again, repeatedly. The PD driver's `google,cmu-restore`
  (kernel `3b9217136`) is no longer needed.
- `tzpc_smc` back to N; pd_csis is off and stays off.

Left before this is upstreamable: `samsung,sysreg` (a `sysreg_csis` node;
the driver then sets DRCG_EN and clears MEMCLK, away from the
bootloader's 0/1: test on its own); the DTZPC save/restore as a proper PD
feature without the `tzpc_smc` parameter, `keep-boot-on` and
`release_kept` (the first power-off must still save from the
bootloader's state); dropping `google,cmu-restore`; system-sleep PM ops
for the CMU. Then the CSIS link registers.

### RC2.0zi0 and RC2.0zi on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g8023438238d9`): pass, SYSREG_CSIS through the CMU

RC2.0zi0: boots; taint 0, no abort; "Unable to get CMU sysreg" now only
for CMU_MISC (`1e080000`), not CMU_CSIS; `csis on` (kept), the CMU bound
and `suspended`, `tzpc_smc` N, no DTZPC call. The CMU's probe-time writes
to SYSREG_CSIS, in the bootloader's state, are safe.

RC2.0zi (`logs/camera-20261025-125816-x.log`): taint 0, no abort, no
"failed" or "kept on". The release save and power-off, then two cycles:
restore (26.115 s, 26.589 s), **54 of 54 registers equal the
bootloader's**, `DRCG_EN` `0x003fffff` (22 implemented bits, all on) and
`MEMCLK` `0x00000000` in both cycles, then the save (26.486 s, 26.976 s)
and **"pd_csis powered off"**. Dynamic root clock gating and the memory
clock gating don't hinder the power-down, and the CMU keeps both across
the cycle. pd_csis ends `off-0`, `tzpc_smc` N.

### RC2.0zj0 and RC2.0zj on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g77c81d480b7d`): pass, the DTZPC save and restore always on

RC2.0zj0: boots; taint 0, no abort. "csis: CONFIGURATION 0x1 STATUS 0x1
at probe", then at 0.340 s the one boot-time "DTZPC save 0x1a410204: smc
returned 0x0" and "csis off: CONFIGURATION 0x1 -> 0x0". `csis off-0`,
CMU_CSIS and the bus `suspended`, no "kept on", no "no DTZPC save". The
first save, from the bootloader's state, now runs unattended at boot
without `keep-boot-on` or `release_kept`. (It came at 0.340 s, early in
boot rather than at a late `sync_state`; which genpd path powered it off
isn't recorded.)

RC2.0zj (`logs/camera-20261025-131538-x.log`): taint 0, no abort, no
"failed", "kept on" or "no DTZPC save". Two cycles: restore (41.673 s,
42.172 s), the bus and CMU `active`, **54 of 54 registers equal the
bootloader's**, `DRCG_EN` `0x003fffff` and `MEMCLK` `0x00000000`, then
the save (42.065 s, 42.556 s) and **"pd_csis powered off"**, the clock
counts back to 0. **pd_csis now cycles with no bring-up switch**: the PD
driver saves and restores the DTZPC itself, and the clock driver restores
CMU_CSIS and its SYSREG through runtime PM.

Still to do before upstream: a binding for `google,tzpc-smc`; drop
`keep-boot-on`, `release_kept`, `google,cmu-restore` and the bring-up
`pr_info`s from the PD driver; system-sleep PM ops for CMU_CSIS. Then the
CSIS link registers.

### RC2.0zk0 and RC2.0zk on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g5f4d1967ac22`): pass, the bring-up switches gone

RC2.0zk0: boots; taint 0, no abort; no "Power domain" line at all (none
fails); `csis off-0`, CMU_CSIS and the bus `suspended`.

RC2.0zk (`logs/camera-20261025-134211-x.log`): quiet=1, no failure line.
Two cycles: genpd `on`, the bus and CMU `active`, **54 of 54 registers
equal the bootloader's**, `DRCG_EN` `0x003fffff`, `MEMCLK` 0, then
**"pd_csis powered off"**, `off-0`, the clock counts back to 0. The PD
driver without `keep-boot-on`, `release_kept`, `google,cmu-restore` or
its logging cycles pd_csis as RC2.0zj did. The CMU's system-sleep ops
are built but not exercised (suspend on raven is still to come).

Left for upstream: a binding for the DTZPC (the `google,tzpc-smc` cell
or another shape; the maintainer's call), the series curated (drop
`611093c5b`: RC2.0s showed CONFIGURATION holds only bit 0, so
`regmap_update_bits` buys nothing; reword or drop `b7be5bb94`: the
power-off was stuck, not slow), and the sleep-ops alignment CHECK.
Then the CSIS link registers.

### RC2.0zl0 and RC2.0zl on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g37b6fd8663e4`): pass, the DTZPC as upstream describes it

RC2.0zl0: boots; taint 0, no abort; no "Power domain" or "SMC for power
domain" line (the probe-time save and the boot-time power-off's save
both returned 0); `csis off-0`, CMU_CSIS and the bus `suspended`; the
other gs101 domains (`pdp`, `mfc`, `g2d`, `aoc`) `off-0` as before, with
the full CONFIGURATION write back (`611093c5b` reverted).

RC2.0zl (`logs/camera-20261025-143757-x.log`): quiet=1, no failure line.
Two cycles, each **54 of 54 registers equal the bootloader's** and
**"pd_csis powered off"**. pd_csis now runs on André Draszik's v8 design
(`samsung,dtzpc` to a `google,gs101-dtzpc` node, offset 0x204, a save at
probe) plus one bring-up guard (no power-on without a good save).

Left for upstream: move our gs101 power-domain base onto André's series
(domains as PMU children, his regmap and offset patches) once it lands,
which touches every gs101 domain user, not only the camera; then drop
the guard if a probe-time save of an off domain proves safe; post
CMU_CSIS (bindings, driver, sysreg, DT) as its own series. Then the
CSIS link registers.

### RC2.1a and RC2.1b on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-ge8478f15aaac`): RC2.1a passes, RC2.1b stops at the CSIS probe

No open pull request covers this work. linux-omarchy-mobile #9 (the gs101
power domains) is this branch's base, rebased; omarchy-mobile #22 (raven
tools), omacam and tracking issue #2 have no camera work.

Offline: checkpatch `--strict` clean on the six commits apart from the
missing Signed-off-by (the human's); W=1 clean for `gs201-csis.o` and
`phy-gs201-mipi-dcphy-rx.o`; both bindings pass dt-doc-validate; the merged
DTB has `csis2` and the DC-PHY `okay` with the endpoint, four lanes, the
pool and the firmware name, and dt-validate finds nothing camera-related.

RC2.1a: boots, taint 0; `1a460000.csis` and `1a4f0000.phy` exist, neither
module loaded. **Deviation:** `csis on`, not `off-0`: genpd's sync_state
waits for the unbound (blacklisted) CSIS consumer, so the domain stays in
the bootloader's state (as RC2.0y held it). Not a fault; binding the
drivers releases it.

RC2.1b (`logs/camera-20261025-152409-x.log`): the PMIC came up and powered
down in order; imx386 read model `0x0386` (the 4032x2272 mode dropped: its
794.6 MHz link isn't in the DT); the DC-PHY bound and runtime-suspended;
gs201-csis failed its probe:

    gs201-csis 1a460000.csis: error -ENXIO: IRQ wdma not found

gs101's GIC has `#interrupt-cells = <4>` (the PPI partition cell), and the
CSIS nodes gave three: the link interrupt parsed by luck (SPI 296, hwirq
328, with the next entry's first cell as its fourth), the WDMA entry ran
short. dt-validate didn't catch it. Fixed in the kernel with the fourth
cell on all ten entries (a fixup of `64a6db97b`); RC2.1a-b again.

### RC2.1b again to RC2.1i on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g3f5d357b540d`): stop, link 2 sees nothing

With the four-cell interrupts the drivers bind. The IMX386 streamed colour
bars for 5 s (`logs/camera-20261025-153724-x.log`), and the DC-PHY set up
link 2 for four D-PHY lanes at 1548.288 Mbps (settle 7, the PHY isolation
released). But CSIS link 2 counted nothing: FS/FE 0, `FRM_CNT` 0, no link
interrupt and no error bit. The same held with the endpoint moved, one
boot each, to links 0, 6 and 4 (overlays `camera-uw-link0`, `-link6` and
`-link4`; logs `154617`, `155115` and `155549`).

Read-only checks between those runs (one register per call, blocks known
powered and clocked):

- the CMU_CSIS link-wrap ACLK gates (0x1a402064-0x1a402080) read 0x00200000
  (on), and their QCH registers 0x003f0002;
- each DC-PHY lane's registers held what the driver wrote;
- link register 0x700 tracks the PHY's lane state: 0xe2 (or 0xf2) on a dead
  link, 0 on a link that receives (RC2.1j). It is the quickest check of
  which link the sensor's lanes reach.

Raven's HAL keeps a per-link table of DC-PHY bases (`liblyric_hwl.so`, word
8 of each 0x30-byte entry): link 0 at 0x1300, 1 at 0x1b00, 2 at 0x2300, 3 at
0x2800, 4 at 0x2b00, 5 at 0x3000, 6 at 0x3300 and 7 at 0x3800. Its PHY setup
takes (lanes, rate, C-PHY, base), and its stream start writes SYSREG_CSIS
0x408-0x410, 0x430 onwards, 0x488 and 0x500. The HAL's link-2 settings for
this sensor were misleading: they name link 2, but the lanes aren't wired
there.

### RC2.1j and RC2.1k on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g3f5d357b540d`): pass, **the ultrawide is on link 1**

RC2.1j, overlay `camera-uw-link1` (`logs/camera-20261025-163514-x.log`):
link 1, DC-PHY slave 1, receives. Over 5 s: FS and FE 147 each, all on VC
0, with per-line errors at the CSIS bus's 66.56 MHz (ERR_ID, ERR_OVER and
VRESOL: INT0 bits 0 and 4, INT1 bit 6, the same failure cheetah had at 67
MHz before its CAM vote).

gs101 has no CAM DVFS domain. The CSIS bus's rate is the divider
`dout_cmu_csis_bus` in CMU_TOP (`CLK_DOUT_CMU_CSIS_BUS`): 399.36 MHz from
its mux, divided by 6 at boot. The overlay gives it to the driver as the
optional `cam` clock, so `cam_khz=400000` sets it to 399.36 MHz while
streaming and puts the old rate back at stop.

RC2.1k (`163902`), with the `cam` clock: FS/FE 147 each, 294 interrupts,
and only one error bit: INT0 bit 3, once per frame (WRONG_CONFIG on the
embedded-data line, as on cheetah). Taint 0, and the PMIC powered down in
order.

### RC2.2a to RC2.2c on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g3f5d357b540d`): pass, **the first raw frames**

Logs `164001` to `164206`. `/tmp/cam-2016x1508.raw` on the phone is RAW10
packed, stride 2560, 1508 lines per frame. Images are unpacked on the
laptop and never committed.

| Step | Run | Result |
| --- | --- | --- |
| RC2.2a | `LINK=1 DMA=1 FRAMES=4` | 4 WDMA frames, no WDMA error, no SMMU fault; the colour bars come out right (RGGB, four bars); frame 0 is all zero |
| RC2.2b | as RC2.2a, `PATTERN=0` | real frames, all at black level (62-72) |
| RC2.2c | `FRAMES=8 PATTERN=0 CTRLS="analogue_gain=480"` | 8 frames, 0 errors; frames 1-7 range 46-97, mean 64.5: noise around black level 64, no scene |

The exposure was 3589 lines (frame length 3599, about 33 ms) with digital
gain 0x100. Noise grows with analogue gain, so the analogue chain works.

RC2.2d (`logs/camera-20261025-164506-x.log`, the run after a refused
`vblank`: v4l2-ctl calls it `vertical_blanking`):
`CTRLS="vertical_blanking=9000;exposure=10500,analogue_gain=480,digital_gain=1024"`,
frame length 10508 (about 100 ms), digital gain 0x400. 4 frames, no WDMA
error. Frames 1-3 range 33-341 with mean 74 (R 70, Gr/Gb 78, B 73), and a
contrast-stretched green plane shows a scene: a bright spot at the top
edge and smooth shading, light leaking past a lens lying face down. **The
pixel array reads out**; the earlier frames were just dark. Frame 0 is all
zero in every run (the WDMA's first buffer, before the first frame end).

Taint stayed 0 throughout, and the PMIC powered down cleanly after each
run.

Next: the kernel DT moves `uw_out` to `csis1_in` and gives `csis1` the
`cam` clock (replacing `camera-uw-stream` and the link overlays), and then
`cam -c1 --capture=10`.

### RC2.3a and RC2.3b on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-ga3ad00e61091`): pass, **libcamera captures from the ultrawide**

Kernel fixups `a3ad00e61` (binding), `47fc32224` (the `cam` clock on every
gs101 CSIS node) and `cda13b14b` (raven's IMX386 on `csis1`), with
`camera-uw-stream` enabling `csis1`. Boots, taint 0; `csis@1a450000`
has clock-names `link`, `dma`, `cam`.

RC2.3a (`logs/camera-20261025-170328-x.log`): 4 colour-bar frames, INT0
bit 3 only, as RC2.1k with the overlay.

RC2.3b: `cam` is in `libcamera-tools`, which wasn't installed; installed
from `[omamobile]` (0.7.2-4.21, the same build as libcamera). Then
`CAM="-c1 --capture=10 --file=/tmp/uw-#.ppm"` (`170430`): `cam -l` lists
"Internal back camera" (the IMX386 node) through the simple pipeline;
10 frames at 4024x3016 through the soft ISP, about 21 fps, rc 0. CSIS: 16
WDMA frames, no WDMA error, INT0 bit 3 only.

RC2.3c (`170520`), `-s width=2016,height=1508 --capture=60 --metadata`: 60 frames
at 2008x1508, 30 fps. AGC ends at its limit (AnalogueGain 16, ExposureTime
33238 µs), and frame 59 averages R 3, G 4, B 1 of 255: the phone lay face
down. A capture of a lit scene is next, and then the tuning.

### RC2.3d on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-ga3ad00e61091`): pass, **a photo from the ultrawide**

The human propped the phone up facing a lit room. Same run as RC2.3c
(`logs/camera-20261025-171109-x.log`): 60 binned frames, 30 fps, rc 0. AGC
settles at AnalogueGain 2.49 and ExposureTime 33238 µs; frame 59 averages R
106, G 94, B 85 with highlights at 255. The image is sharp and correctly
oriented, with the stand-in IMX519 colour matrix giving plausible colour, a
little warm under the room's tungsten light. Never committed: it shows the
human's home.

The ultrawide works through libcamera's simple pipeline and soft ISP. Left:
calibrate the IMX386 tuning (black level, lens shading, colour) in place of
the stand-in; make the bring-up permanent (the PMIC's bb, buck and pu lines
from a driver or package rather than `camera-pmic-gpio`'s Python holder,
the modules out of the blacklist); then omacam; then the INT bus rate for
the WDMA under load, and the curated kernel series.

### RC2.4a to RC2.4c on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g5adfebfb19c3`): pass, **the kernel powers the camera PMIC**

Offline: checkpatch `--strict` clean on the three commits (apart from the
binding's own-patch reminder, which it is, and the human's Signed-off-by);
W=1 clean for `slg51000-regulator.o`; the binding parses.

RC2.4a: boots, taint 0; `cam-pmic-bb` and `cam-pmic-buck` registered and
disabled, S2MPG11 GPIOs 1 and 3 requested by them and low.

RC2.4b (`logs/camera-20261025-174249-x.log`): no GPIO holder. The PMIC bound,
four colour-bar frames on link 1, INT0 bit 3 only, GPIOs 1 and 3 low after
the run. A closer look by hand:

| When | PMIC runtime PM | GPIO 1 (bb) / 3 (buck) |
| --- | --- | --- |
| 0.3 s after `modprobe slg51000-regulator` | active | high / high |
| 3.3 s after | suspended | low / low |
| after `modprobe imx386` (ID read through the PMIC's LDOs) | suspended | low / low |

All seven LDOs and both converters `disabled` at rest; the IMX386 read model
`0x0386` through a resume, and the chip went back to reset after it.

RC2.4c (`174356`): `cam -c1 -s width=2016,height=1508 --capture=30`, rc 0,
AGC at gain 4. Taint 0 throughout.

`tools/camera-pmic-gpio` is now needed only on a kernel without the `vdd`
supply. Next: an OF match table for `slg51000-regulator` (it has none, so it
never autoloads from the DT), then the camera modules off the blacklist and
into `omamobile-device-raven`, so the camera is there at every boot.

### RC2.5 on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g5adfebfb19c3`): pass, **the camera at every boot**

`omamobile-device-raven` 0.1.23 (built and installed on the phone): as on
cheetah, the camera modules are blacklisted for autoload, and
`omamobile-camera-start` loads them from `omamobile-camera.timer` 40 s into
the boot behind a hang marker (PMIC, then the IMX386, the DC-PHY and CSIS).
The package also has the DMA-BUF heap rule for the software ISP in a user
session, and WirePlumber rules that name the "Ultrawide camera" and hide
the raw CSIS nodes. Package tests pass on the phone apart from
"a dry run writes nothing", which fails on the unchanged package too (to
look at separately).

Bring-up files removed: `/etc/modprobe.d/omamobile-camera-bringup.conf`
(the same names as the package's file) and the user's WirePlumber drop-in
that hid the cameras.

- `systemctl start omamobile-camera`: four modules, one media device, the
  PMIC `suspended` 3 s later with GPIOs 1 and 3 low. `wpctl status`:
  "Ultrawide camera" is the default video source (libcamera device
  `imx386`); `cam -l` lists it as the user.
- omacam, started in the session: the IMX386 streams (CSIS at 399.36 MHz),
  149 frames in 5 s (30 fps), no WDMA error, 1 frame dropped at start. The
  screen was off, so no screenshot.
- Reboot: `omamobile-camera.service` active, "camera drivers loaded; 1
  media devices", no marker left, the PMIC `suspended`, the camera in
  PipeWire. Taint 0.

### RC2.6 on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g15ad21d6c8f2`): rotation

The human saw omacam's picture a quarter turn counter-clockwise: the IMX386
node had no `rotation`, so libcamera reported 0. omacam turns the image
counter-clockwise by Rotation, so upright needs 270, as on cheetah's
ultrawide (the same module). Kernel fixup `15ad21d6c`. After the push the
sensor's `camera_sensor_rotation` reads 270, and the camera loads at boot
as in RC2.5. Taint 0.

The human then took a photo in omacam: upright and good. **RC2 is done:
raven's ultrawide works in the camera app.**

### RC2.7 on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g01aee004d37f`): pass, the curated series

The 43 bring-up commits on `omamobile-gs101-camera` are curated into 26 on
`omamobile-gs101-camera-v1` (worktree `$WORK/mainline/wt-raven-camera-v1`,
output `$WORK/mainline/out-raven-camera-v1`), with no Signed-off-by yet:

- the fixups squashed; the raven commit names link 1, with the evidence
  (links 0, 2, 4 and 6 saw nothing); the sensor commit carries the rotation;
- the eleven power-domain bring-up commits are two: the 10 ms wait for gs101
  domains, and the DTZPC save and restore from `samsung,dtzpc` (André
  Draszik's design), which refuses a power-on until a save of the powered
  block has been made. That guard and the probe-time save now live in one
  place, without "bring-up" wording; the behaviour is unchanged;
- the "no clocks" comment in `gs101.dtsi` is back above `sysreg_csis`; it had
  ended up above the DTZPC node.

The tree equals `omamobile-gs101-camera` apart from those two cleanups.
`checkpatch --strict` passes on every commit; its one CHECK is the
`SET_NOIRQ_SYSTEM_SLEEP_PM_OPS` alignment that exynos5433's identical code also
gets.

On the phone: the camera loads from its timer, `cam -c1 --capture=30` gives 30
frames at 21 fps, and afterwards `pd_csis` is off, CMU_CSIS and the link are
suspended, the PMIC is suspended, there are no power-domain or SMC errors, and
taint is 0.

Still in raven's overlays, not the kernel DT: `pd_csis`'s `samsung,dtzpc` and
`cmu_csis`'s `status` (camera-csis-cmu-test). Moving them into `gs101.dtsi`
changes every gs101 board (oriole too), so it is a step of its own.

## RC3: colour and shading from the module's own factory data

No calibrated IMX386 tuning exists that we may use: libcamera has none, ours
borrows the IMX519's matrices (`imx386.yaml` in our libcamera package), and
Google's HAL tuning is Google's data (cheetah CAMERA-PLAN §1.6). The ultrawide
module's own EEPROM, an ST M24C64 at 0x50 on the sensor's bus, holds this
module's factory calibration. Reading it is read-only (repeated-start reads, and
then at24 with `read-only`); the contents are unique to the phone and stay in
`~/android/reference/raven-camera/` and on the phone, never in the repo.

### RC3.0 on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-g01aee004d37f`): the layout

With the sensor held on (`power/control` `on`, which powers LDO1), 64 reads of
128 bytes with i2c-tools gave the 8 KiB; the first 32 bytes equal RC1.3's.
Used: 0x000-0x80f, plus smaller blocks at 0x13e0-0x17bf (not decoded yet).
Decoded, all big-endian 16-bit:

| Offset | What |
| --- | --- |
| 0x032, 0x05e, 0x08a | three white-balance records of 21 words and a checksum word (the byte sum of the 42 bytes before it). Each holds three groups of seven: R, Gr, Gb, B averages, R/G and B/G × 1024, one more value. The three groups' ratios sit where daylight, cool-white fluorescent and tungsten would [I]; the ratios match the first record's own averages exactly, so that record is likely the golden module's [I] |
| 0x0b8 | four 17 × 13 grids (R, Gr, Gb, B), edge to edge of the 4:3 active area [I], each peaking at its centre cell: a factory flat field. The corners get about a fifth of the centre's light, red a little less than green and blue, so uncorrected corners are dark and cyan |
| 0x7a0 | the grids' checksum (byte sum of the 1768 bytes) |

Both checksum kinds verify on this module.

### RC3.1 on the phone (2026-10-25, `7.3.0-rc6-omamobile-push1-gbdb2225f3931`): pass, the EEPROM through at24 and lens shading at every boot

- Kernel `bdb2225f3` describes the EEPROM (`st,24c64`, `read-only`,
  `vcc-supply = <&cam_ldo1>`, disabled); overlay `camera-uw-eeprom` enables it.
  at24 was already loaded for the board's other EEPROM. At boot its probe
  defers until LDO1 exists, then "8192 byte 24c64 EEPROM, read-only"
  (`nvmem` mode 0400); runtime-suspended afterwards, with the PMIC. The sysfs
  read is byte for byte RC3.0's dump.
- `omamobile-device-raven` 0.1.24: `omamobile-camera-calibrate` checks both
  checksum kinds and the grid's shape, fits the software ISP's radial model
  (`1 + k1 r^2 + k2 r^4` per colour) to the grid, correcting colour fully and
  brightness by 0.8 (corner gain about 3.6 instead of 5, so the corners keep a
  little falloff and less noise), and writes the package's tuning with that
  `LensShading` to `/etc/libcamera/ipa/simple/imx386.yaml`, which libcamera
  reads before `/usr/share`. `omamobile-camera-start` runs it after the PMIC
  and before the camera drivers, as libcamera (and PipeWire) read the tuning
  when the camera appears. The model fits the grid within about 12 % at its
  worst cell (an off-centre optical axis [I]).
- Reboot with the file removed: at24 at 40.16 s, the file written, the CSIS
  link at 40.56 s; `cam` logs "Using tuning file
  /etc/libcamera/ipa/simple/imx386.yaml". Corners in a room scene went from
  about 0.8-0.9 of the centre to 0.9-1.0. Taint 0.

Left: the human's judgement of a photo; using the white-balance records (the
module's R/G and B/G under three lights could anchor the AWB colour
temperature curve, once the lights' temperatures are known); the colour
matrices stay the IMX519's.

## RC4: the main camera (S5KGN1)

### RC4.0 offline (2026-10-25): the mode file, the node, the blacklist

- **Mode file.** `tools/extract-camera-data --apex … --sensor s5kgn1` failed
  on raven's HAL (`42ab61e0…`): its GN1 virtual table has the init list
  selector two slots before the model-ID function, where cheetah's has it one
  before (the mode selector is three before the init selector in both). The
  tool now tries both and keeps the first per table. Raven's HAL has
  cheetah's init list size (1565 writes, `0x0114` = `0x0201`: three C-PHY
  trios) and the same four modes: 4080x3072 at 30 fps, 4080x2288 at 60,
  2016x1136 at 120 and 2032x1536 at 60. The file stays off the repository
  (`~/android/reference/raven-camera/s5kgn1-main.bin`) and is installed as
  `/usr/lib/firmware/updates/omamobile/camera/s5kgn1-main.bin`.
- **The link.** Raven's `gs201-dcphy.bin` has a C-PHY section (3 rates,
  1-2147 Msps), and the GN1's 1862 Msps is inside it. Three trios need a
  four-lane slave (links 0 and 1); the ultrawide holds link 1, so the
  endpoint goes to link 0 [I]. RC4.3 checks it with link register 0x700.
- **Kernel `dee78cf3f`** (on `omamobile-gs101-camera-v1`): `hsi2c_1` with
  stock's SoC pull-ups (400 kHz for bring-up; stock runs 1 MHz), the GN1 at
  0x3d with `vio` = LDO1, `vaux` = LDO2, `vdig` = `vana` = LDO3 (3.05, 3.05
  and 2.85 V, read on the phone: the module makes its own rails, so the names
  follow the driver's power order), `samsung,power-delays-us = <1000 1000
  1000 9000>` for stock's waits, reset gpp2-2, MCLK `CIS_CLK3` on gpp9-0 at
  24.576 MHz, `rotation` 90 as cheetah's (not seen on raven yet); the C-PHY
  endpoint to `csis0`. Disabled. `checkpatch --strict`: only the missing
  Signed-off-by; `dt-validate` (dtschema on the laptop) has nothing on the
  new nodes, also with them enabled.
- **`omamobile-device-raven` 0.1.25** blacklists `s5kgn1`; installed on the
  phone before any DTB enables the node.
- Overlay `camera-main-test` enables USI1, `hsi2c_1` and the sensor; the
  CSIS link stays off.

| Step | Command | Expect / pass | Stop | Rollback | Risk |
| --- | --- | --- | --- | --- | --- |
| RC4.1 | `images` with `camera-main-test` (kernel `dee78cf3f`), push; then `tools/phone-ssh 'uname -r; lsmod \| grep -c s5kgn1; ls /sys/bus/i2c/devices/ \| grep 003d; journalctl -b -u omamobile-camera \| tail -3'` | boots; `s5kgn1` not loaded (0); an `N-003d` device with no driver; the ultrawide's camera start as before | no boot; `s5kgn1` loaded (the blacklist missed) | push without `camera-main-test` | low: a disabled-driver node and an idle I2C bus |
| RC4.2 | `tools/camera-stage --watchdog x --cmd 'bash -s' --script tools/camera-main-probe` | "S5KGN1, model 0x08e1"; bound, `runtime_status: suspended`; LDO2 and LDO3 off afterwards (LDO1 as before), `dout_cmu_cis_clk3` disabled at 24576000 | a chip ID read failure (`-ENXIO`/timeout: suspect S2MPG11 GPIO 4, the PMIC's pull-up line, which the kernel doesn't raise; record and stop); a wrong ID; an LDO left on; any oops | reboot (the driver doesn't load at boot) | medium: the first power-up of the main module |
| RC4.3 | the CSIS link 0 overlay and a 5 s colour-bar stream under `camera-cap-test`, link register 0x700 read once | FS/FE counting on link 0; 0x700 reads 0 | 0x700 0xe2/0xf2 and no frames: the link guess is wrong; try link 2's C-PHY (two trios) next, one boot each | as RC4.2 | medium |

Later: the camera at boot (`omamobile-camera-start`), a tuning file, the focus actuator (`ak7377`), and the OIS controller's EEPROM.
