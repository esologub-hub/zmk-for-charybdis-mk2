# Charybdis: ZMK / Zephyr 4.1 migration

## Hardware and dependencies

Both halves use nice!nano V2, explicitly selected as
`nice_nano@2.0.0//zmk`. The right half remains the central and carries the PMW3610;
the left half remains a peripheral. Neither matrix wiring nor SPI wiring changes.

| Component | Revision |
| --- | --- |
| ZMK | `4dd95d7840eb3d5a3f59bb29fbf1ab054715ef85` |
| Zephyr (`v4.1.0+zmk-fixes` snapshot) | `10ba6d0cb38bc3d258775d27982f707599320085` |
| badjeff PMW3610 driver | `44b4a76b74d293a93cec4ccb7e04cb8d29c10f93` |

Zephyr is explicitly overridden in the west manifest because the fixes branch in
ZMK's imported manifest can move. Its module imports retain ZMK's exclusions.
The driver remains external: its ZMK activity-aware wake modes are used by this
keyboard and are not interchangeable with the stock driver configuration.

## Preserved settings

- All 56 bindings on each of the three active layers and the JSON editor layout.
- Layer-tap: 200 ms, balanced, quick tap 150 ms; mod-tap: tap-preferred.
- Matrix transform, diode direction, GPIO assignments and sensor bus pins.
- PMW3610: 600 CPI, swapped axes, both axes inverted, force-awake and 4 ms mode.
- Layer 1 snipe scaling 1/3; layer 2 scrolling scaling 1/5 with Y inversion.
- Bluetooth name SoloCharybdis, TX power +8 dBm, preferred maximum interval 9,
  latency 16, ACL TX buffers 32, event RX buffers 64, L2CAP TX buffers 32.
- Battery reporting every 60 seconds; RGB remains disabled.

The Win+Space macro now explicitly releases Space and Left Win after pressing
them. `macro_release` only changes the interpretation of subsequent bindings;
ending the macro immediately after that marker left both keys pressed. The
combination and existing timing controls remain unchanged.

## Building

Use the GitHub Actions build for this branch, or a ZMK/Zephyr 4.1 build environment:

```sh
west init -l config
west update --fetch-opt=--filter=tree:0
west zephyr-export
west manifest --freeze > west-frozen.yml
west build -s zmk/app -d build-right -b 'nice_nano@2.0.0//zmk' -- \
  -DZMK_CONFIG="$(pwd)/config" -DSHIELD=charybdis_right
west build -s zmk/app -d build-left -b 'nice_nano@2.0.0//zmk' -- \
  -DZMK_CONFIG="$(pwd)/config" -DSHIELD=charybdis_left
west build -s zmk/app -d build-reset -b 'nice_nano@2.0.0//zmk' -- \
  -DZMK_CONFIG="$(pwd)/config" -DSHIELD=settings_reset
```

Keep the firmware, frozen west manifest, final `zephyr/.config`, and
`zephyr/zephyr.dts` together when saving a validated build.

## Flashing and rollback

1. Save the known-working UF2 files for both halves before flashing. The latest
   source commit before migration is `ffcef1ccc3102d1dfe61bb766cf5069ffb4fa076`;
   it is not a substitute for a saved, tested binary. Rebuilding the old source
   can produce different results because its driver and workflow were unpinned.
2. Flash only the matching `charybdis_left` and `charybdis_right` artifacts to the
   respective controllers through their UF2 bootloaders. Update both halves.
3. Do not flash `settings_reset` as part of the normal upgrade: it erases stored
   settings and can require pairing the halves and hosts again. That artifact is
   built for recovery only.
4. If hardware checks fail, restore both saved working binaries. If stored
   settings prove incompatible, investigate pairing separately; do not assume a
   reset is necessary or that it preserves existing bonds.

## Hardware acceptance (owner)

- [ ] All 56 positions on all three layers produce their previous bindings.
- [ ] Tap/hold and quick-tap behavior matches the previous firmware.
- [ ] Win+Space switches language and releases both keys; subsequent typing works.
- [ ] Cursor direction, 600 CPI sensitivity, snipe and both scroll axes match.
- [ ] Three mouse buttons work, including clicks while moving the trackball.
- [ ] USB and Bluetooth output work; both halves reconnect after a power cycle.
- [ ] Sleep/wake behavior, trackball wake latency and battery reporting match.
- [ ] Existing host and inter-half Bluetooth bonds survive the upgrade, or any
      necessary re-pairing and the reason are documented.

Compilation and source comparisons do not complete these hardware checks.
