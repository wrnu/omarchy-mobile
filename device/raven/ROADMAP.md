<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 Pro (raven): the overall plan

Status (2026-11-03, `7.3.0-rc6-omamobile-push1-gcc0d345e31f0`, branches
`omamobile-gs101-camera-v2` (kernel) and `omamobile-raven-device-v2` (this
repository), both on the wrnu forks): what works, what is open, and the
order to take it in. Each item links the plan that holds its facts and
runbook; this file only orders them. Update it when an item moves.

Two people work on raven upstream: robotosaith (the display series,
Bluetooth, suspend, the modem runtime, the microphones and stage media) and
this branch (the cameras, the camera suspend fix, the louder speaker level,
the fingerprint reader). Before a pull request from here, check
robotosaith's open ones (section 3) so the two don't collide.

## 1. Where things stand

| Area | State here | Upstream (`main` / `omamobile`) | Plan |
| --- | --- | --- | --- |
| Boot, storage, display, touch, buttons, USB, Wi-Fi | works | display in review (kernel #116), Wi-Fi unpublished | [BRINGUP-M1](BRINGUP-M1.md), [DISPLAY-PLAN](DISPLAY-PLAN.md), [WIFI-PLAN](WIFI-PLAN.md) |
| GPU | Panfrost at the bootloader's clock | DVFS and cooling in review (kernel #189) | [GPU-PLAN](GPU-PLAN.md) |
| Battery, charging | works (MAX77759 at stock's limits) | merged | [POWER-PLAN](POWER-PLAN.md) |
| Haptics, torch, GNSS, sensors | works | haptics and torch merged; GNSS and the AoC unpublished | [HAPTICS-TORCH-PLAN](HAPTICS-TORCH-PLAN.md), [GNSS-PLAN](GNSS-PLAN.md) |
| Bluetooth | pairs and plays A2DP (robotosaith); crackles with busy Wi-Fi; one silent start in seven reboots (#77) | results in review (#395) | [BLUETOOTH-PLAN](BLUETOOTH-PLAN.md) |
| Speakers, microphones | both; the louder c9 level (817, top 11, bottom 14) verified 2026-11-02 | microphones (#400) and stage media under protection (#401) in review | [AUDIO-PLAN](AUDIO-PLAN.md) "C9 on raven" |
| Cameras | ultrawide 4032x3016 and main 2024x1536 at boot, lens shading, autofocus; capture after suspend fixed (RC4.10) | unpublished | [CAMERA-PLAN](CAMERA-PLAN.md) |
| Suspend | s2idle with RTC and power-key wakes | "testing" in review (#395); UFS level 5 in review (kernel #188) | [SUSPEND-PLAN](SUSPEND-PLAN.md) |
| Modem | calls, texts, data on robotosaith's kernel; not enabled by the package | runtime for raven in review (#389); CPIF for gs101 unpublished | [MODEM-PLAN](MODEM-PLAN.md) |
| NFC | not started | | |
| Fingerprint | offline research | | [FINGERPRINT-PLAN](FINGERPRINT-PLAN.md) |

## 2. Open items, in order

### Now (safety and correctness first)

1. **The amplifiers' boost peak current (kernel `0a1a3d838`, built, not yet
   on the phone).** raven's DT had `cirrus,boost-peak-milliamp = <3500>`, the
   value stock's audio HAL sets at init; for media its `speaker` path lowers
   it to 1.70 A (top) and 1.90 A (bottom), keeping 3.50 A for voice calls
   (`mixer_paths.xml` in the owner's vendor image). The mainline driver sets
   the limit once at probe, so the DT value is what every stream runs at,
   and at the c9 level the boost converter is on: about twice stock's media
   peak from the battery (a droop and brown-out risk at low charge, not a
   coil one; stock's own 3.50 A shows the hardware takes it). The commit sets
   1700 and 1900. Next: push it (one boot), re-run AUDIO-PLAN "C9 on raven"'s
   3.5-minute music step, and listen at full volume for clipping. Speaker
   calls then run below stock's 3.50 A: note it for the modem's call audio.
2. **Kernel #188** (UFS `spm_lvl` 5 on gs101) is already in the camera
   branch in another form (`gs101_ufs_drv_init()` sets level 5); when #188
   merges, drop ours for it at the next rebase. The two UFS `dme-set
   0x4021` lines per s2idle are known noise (`tools/phone-smoke`).
3. **Coordinate the audio work with #400 and #401.** Both change the
   speaker service's AoC condition and take device-raven 0.1.31/0.1.32 and
   omamobile-audio 0.5.14/0.5.15, as this branch now does (rebased onto
   main's 0.1.30 and 0.5.13 on 2026-11-03: cameras 0.1.31, speakers 0.1.32
   with audio 0.5.14; the phone still runs the pre-rebase 0.1.31 and 0.5.11).
   Once #400/#401 merge, this branch's c9 work (`verified-firmware`,
   `speaker_protection=1`, the boost peak) goes as one pull request on top,
   renumbered; #401 also measured raven's stage media under protection
   (AUDIO-PLAN R6), which this branch's C9 run builds on. Main's raven image
   (`ROOTFS_PACKAGES`) still lacks omamobile-audio; #400 adds it.
4. **Fold the NOT FOR MERGE DT commit** (`1adc782c9`, the rebase repairs of
   `gs101-raven.dts`) into the commits it repairs, before any kernel PR.

### Next (features)

5. **Fingerprint reader** (Goodix optical, Trusty app `g6.app`):
   [FINGERPRINT-PLAN](FINGERPRINT-PLAN.md), FB-F10. Our `goodix-fp` driver
   already serves it; new are the DT node (IRQ gpa9-2, reset gpp4-3,
   LDO27M), raven's case in the extract tool and the chroot runner, and the
   panel's LHBM (the biggest kernel piece: our S6E3HC3 driver has none).
   Before F1: Google's **factory zip** for this build (only the OTA zip is on
   disk) and a read-only `mfg_data` copy (the 2026-10-23 backup lacks it).
   F4, the first RPMB write, needs the owner's recorded choice (plan
   section 5).
6. **Cameras:** full resolution on the main camera (CAMERA-PLAN options 2
   and 3); `omamobile-camera-calibrate` generating the main camera's tuning
   from a stored fit (today a phone-local `s5kgn1.yaml`); the ultrawide's
   33 WDMA errors at stop.
7. **GPU DVFS:** take kernel #189 for a daily test once it settles.
8. **Modem by default:** after #389 and the gs101 CPIF kernel are
   published, the device package's modem target as on cheetah
   (`omamobile-modem-boot.target`), with MODEM-PLAN's suspend section.
9. **NFC:** not started; a plan first (stock DT, the controller, neard), and
   the 9a's NFC-in-s2idle lesson (AGENTS.md) from the start.

### Later (publishing and polish)

10. **Upstream pull requests** from this branch, when the owner says go
    (none until then), each with the owner's `Signed-off-by` on kernel
    commits: the camera series (CSIS, the DPHY, the sensors, CMU_CSIS, the
    DTZPC), `GENPD_FLAG_RPM_RESUME` (a genpd core change: to the genpd
    maintainers with RC4.8-RC4.10 as the case), the 10 ms gs101 domain
    wait, and device-raven's camera and speaker packaging.
11. **`device/raven/kernel.config` pins ACPM, its mailbox and `sec_acpm` as
    modules.** Main now skips built-in `INITRAMFS_MODULES` (#393), so the
    pin is no longer needed to build; it stays until a boot with them built
    in has been checked against the regulator core's unused-rail cleanup.
12. **Small things:** NTP failed to sync on one boot (the clock a day
    behind, Chromium's certificate errors); no IPv6 default route; the
    `DEVICES.md` row after each change; this branch's `tools/lib/alarm.sh`
    Docker fix upstream.

## 3. Open upstream work on raven (2026-11-03)

| PR | What | Touches this branch |
| --- | --- | --- |
| omarchy-mobile #400 | microphones without speaker protection (audio 0.5.14, device-raven 0.1.31) | yes: versions, the speaker service |
| omarchy-mobile #401 | speakers under protection at stage media (audio 0.5.15, device-raven 0.1.32) | yes: the same, and the boost peak note |
| omarchy-mobile #395 | Bluetooth audio, clean reboots, suspend to testing (docs) | `DEVICES.md` |
| omarchy-mobile #389 | the modem runtime on raven (ext4 NV, props check) | no |
| linux-omarchy-mobile #116 | the display (gs101 DECON, DSIM, S6E3HC3) | the base of the camera branch |
| linux-omarchy-mobile #188 | UFS power-down in suspend on gs101 | ours already has it (item 2) |
| linux-omarchy-mobile #189 | GPU DVFS and cooling | item 7 |
| omarchy-mobile #2, #77 | the oriole/raven tracking issue; Bluetooth start-up after a warm reboot | |
