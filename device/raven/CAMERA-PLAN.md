<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 Pro (raven): cameras

Status (2026-10-23): **RC0 done (read-only, §5); RC1.1 built and gated, not run.** No camera driver binds on
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
- **RC1.2** SLG51000: bb, buck, cs; read the chip ID and **every LDO's
  selector and enable register, read-only**; all outputs stay off. Expect the
  chip ID to answer within 10 ms of cs. Record the default voltages here.
  Stop if a selector is outside its DT range.
- **RC1.3** LDO1 only, 1 ms; `hsi2c_3` enabled; read the ultrawide EEPROM
  `0x50` (first 32 bytes, never written, dump kept off-repo). Stop on NACK.
- **RC1.4** the ultrawide's full sequence (LDO1, LDO4, MCLK, reset); read the
  IMX386 model ID at `0x0016`. Expect `0x0386`. Then power down in reverse.

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
