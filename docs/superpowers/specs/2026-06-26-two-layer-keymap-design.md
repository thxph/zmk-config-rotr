# Design: 2-layer rotr keymap with ephemeral per-layer LEDs

Date: 2026-06-26
Board: `rotr` (nRF52840, fork `refil/zmk` @ `rotrlayer`)

## Goal

Rewrite `config/boards/arm/rotr/rotr.keymap` to provide two usable content
layers with distinct underglow colors, switched via a hold-the-middle-button
navigation gesture. The LED color change must be **ephemeral** — it must not
write to NVS/flash on every switch.

## Hardware recap

- 3 keys in a 1×3 matrix → keymap positions `0` (left), `1` (middle), `2` (right).
- 1 rotary encoder ("knob"), a single sensor.
- 3 WS2812 underglow LEDs.

## Layers

Three layers total: two content layers plus one hidden navigation layer.

| # | Name   | Knob CW / CCW        | Left (0)            | Middle (1)            | Right (2)            | LED    |
|---|--------|----------------------|---------------------|-----------------------|----------------------|--------|
| 0 | base   | `RIGHT` / `LEFT`     | `DOWN`              | cmd-b (tap) / hold→nav | `UP`                | white  |
| 1 | second | `LS(RIGHT)`/`LS(LEFT)`| `DOWN`             | cmd-b (tap) / hold→nav | `UP`                | orange |
| 2 | nav    | (none)               | → Layer 0 + white   | `&trans` (held)        | → Layer 1 + orange  | —      |

- cmd-b = `LG(B)` (LGUI+B).
- "second" layer differs from base only in the knob (shifted arrows).

## Middle button: custom hold-tap

The middle key is both cmd-b (tap) and the gateway to the nav layer (hold). A
custom hold-tap is required because the stock `&lt` defaults to `tap-preferred`,
which would emit a cmd-b tap instead of activating the layer when the user
presses left/right mid-hold.

```dts
mid_lt: middle_layer_tap {
    compatible = "zmk,behavior-hold-tap";
    label = "MID_LT";
    flavor = "hold-preferred";
    tapping-term-ms = <200>;
    bindings = <&mo>, <&kp>;
    #binding-cells = <2>;
};
```

Middle key binding (both content layers): `&mid_lt 2 LG(B)`.
`hold-preferred` means any other key press during the hold immediately resolves
to the hold action (activate nav layer 2).

## Layer switching + LED color (ephemeral)

On the nav layer, left/right are macros that atomically switch the active layer
**and** set the underglow color:

```dts
to_base: to_base {
    compatible = "zmk,behavior-macro";
    label = "TO_BASE";
    #binding-cells = <0>;
    bindings = <&to 0>, <&rgb_ug RGB_COLOR_HSB(0,0,100)>;   /* white  */
};
to_second: to_second {
    compatible = "zmk,behavior-macro";
    label = "TO_SECOND";
    #binding-cells = <0>;
    bindings = <&to 1>, <&rgb_ug RGB_COLOR_HSB(30,100,100)>; /* orange */
};
```

With two layers, left always lands on Layer 0 and right on Layer 1 —
functionally the toggle requested. True N-layer wraparound cycling is not in
stock ZMK (would need a custom module); easy to revisit if more layers are added.

### Why this is ephemeral

Verified against `refil/zmk@rotrlayer`:

- `&rgb_ug RGB_COLOR_HSB(...)` dispatches to `RGB_COLOR_HSB_CMD` →
  `zmk_rgb_underglow_set_hsb()`.
- `zmk_rgb_underglow_set_hsb()` only assigns `state.color = color;` and returns.
  It does **not** call `zmk_rgb_underglow_save_state()`.
- The flash-writing paths (`change_hue/sat/brt`, `select_effect`, `on`/`off`)
  are **not** used by these macros.

Therefore each layer switch updates color in RAM only — zero NVS writes.
Trade-off (acceptable, expected for "ephemeral"): after a reboot the color
resets to the compile-time default until the next switch.

## Encoder behavior

One sensor-rotate behavior reused across layers:

```dts
enc: enc {
    compatible = "zmk,behavior-sensor-rotate-key-press";
    label = "ENC";
    #sensor-binding-cells = <2>;
    triggers-per-rotation = <30>;
};
```

- Layer 0 `sensor-bindings = <&enc RIGHT LEFT>`.
- Layer 1 `sensor-bindings = <&enc LS(RIGHT) LS(LEFT)>`.
- Layer 2 (nav): no sensor binding (transient layer).

## Config changes (`rotr_defconfig`)

| Key | From | To | Reason |
|-----|------|----|--------|
| `CONFIG_ZMK_RGB_UNDERGLOW_EFF_START` | `4` | `0` | Solid effect; required for `RGB_COLOR_HSB` color to render statically. Compile-time default, no runtime write. |
| `CONFIG_ZMK_RGB_UNDERGLOW_HUE_START` | (unset) | `0` | Boot color = white (matches Layer 0) without firing a macro. |
| `CONFIG_ZMK_RGB_UNDERGLOW_SAT_START` | (unset) | `0` | " |
| `CONFIG_ZMK_RGB_UNDERGLOW_BRT_START` | (unset) | `100` | " |

`CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_IDLE=y` stays (LEDs sleep on idle, color
restored on wake from RAM). `CONFIG_ZMK_RGB_UNDERGLOW_LAYER_ON` is left as-is.

### One-time caveat

If NVS from prior firmware has a non-solid effect saved, the solid `EFF_START`
default may be overridden at boot. A single settings/BT reset clears this; after
that, effect stays solid and only the (ephemeral) color changes per layer.

## Files touched

- `config/boards/arm/rotr/rotr.keymap` — full rewrite (behaviors, macros, 3 layers).
- `config/boards/arm/rotr/rotr_defconfig` — RGB start defaults above.

## Out of scope

- Bluetooth profile management on the keymap (kept off the content layers).
- Persisting color across reboots (explicitly not wanted).

---

## Addendum 2026-06-26: third (reset) layer + wraparound cycling

Added a third **content** layer for rebooting into the UF2 bootloader to flash
new firmware, placed into the left/right switch cycle.

**Layers (now 6: 3 content + 3 nav):**

| # | Layer        | Left         | Middle              | Right        | Knob       | LED    |
|---|--------------|--------------|---------------------|--------------|------------|--------|
| 0 | base         | `DOWN`       | `&mid_lt 3 LG(B)`   | `UP`         | →/←        | white  |
| 1 | second       | `DOWN`       | `&mid_lt 4 LG(B)`   | `UP`         | ⇧→/⇧←      | orange |
| 2 | reset        | `&bootloader`| `&mid_lt 5 LG(B)`   | `&bootloader`| (inherits) | red    |
| 3 | nav_from_base| →reset (prev)| `&trans`            | →second (next)| —         | —      |
| 4 | nav_from_second| →base (prev)| `&trans`          | →reset (next)| —          | —      |
| 5 | nav_from_reset| →second (prev)| `&trans`         | →base (next) | —          | —      |

**Wraparound cycling without a custom behavior:** stock ZMK has no
next/previous-layer behavior, so instead of one nav layer there is **one nav
layer per content layer**, each hardcoding its own prev/next `&to` targets.
Cycle order: base → second → reset → base.

**Reset layer:** both left and right tap → `&bootloader` (reboot to bootloader).
Middle stays `&mid_lt 5 LG(B)`: holding it activates the nav layer (which
shadows left/right with cycle bindings), so you can cycle away from the reset
layer without triggering a flash. New macro `to_reset` = `&to 2` +
`RGB_COLOR_HSB(0,100,100)` (red, ephemeral like the others).

6 layers is under ZMK's default 8-layer cap — no Kconfig change required.

Superseded from the original design: "True cyclic layer switching for >2 layers"
is now in scope and implemented via the per-layer nav approach above.
