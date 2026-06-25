# Two-Layer Keymap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite the `rotr` keymap to provide two content layers with distinct, ephemeral underglow colors, switched via a hold-middle + left/right gesture.

**Architecture:** A single devicetree keymap file defines a custom `hold-preferred` hold-tap on the middle key (tap = cmd-b, hold = nav layer), two color-setting macros bound to a hidden nav layer, one reused encoder behavior, and three layers (base/second/nav). A few `rotr_defconfig` RGB start values make the underglow boot solid-white so per-layer `RGB_COLOR_HSB` color writes render and stay in RAM only.

**Tech Stack:** ZMK firmware (fork `refil/zmk` @ `rotrlayer`), Zephyr devicetree, nRF52840, WS2812 underglow. Build via the existing GitHub Actions workflow (`zmkfirmware/zmk-build-arm:stable`).

## Global Constraints

- Board: `rotr`. Keymap positions: `0`=left, `1`=middle, `2`=right. One encoder sensor.
- LED color changes MUST be ephemeral — use ONLY `&rgb_ug RGB_COLOR_HSB(...)`. Never `RGB_HUI/HUD/SAI/SAD/BRI/BRD`, `RGB_EFF/EFR`, or `RGB_ON/OFF/TOG` in the switch macros (those call `save_state()` → NVS write).
- cmd-b = `LG(B)`.
- No unit-test harness exists for keymaps. "Verify" = successful firmware build. There is no local `west`; build runs in CI on push (or via the build-arm Docker image locally).
- Files touched: only `config/boards/arm/rotr/rotr.keymap` and `config/boards/arm/rotr/rotr_defconfig`.

---

### Task 1: RGB start defaults in `rotr_defconfig`

Make the underglow boot in **solid** mode at **white**, so `RGB_COLOR_HSB` colors render statically and Layer 0's color is correct on power-up without firing a macro. These are compile-time defaults — no runtime flash writes.

**Files:**
- Modify: `config/boards/arm/rotr/rotr_defconfig` (RGB underglow section, currently line 31 `CONFIG_ZMK_RGB_UNDERGLOW_EFF_START=4`).

**Interfaces:**
- Consumes: nothing.
- Produces: a solid-white underglow default that Task 2's macros recolor in RAM.

- [ ] **Step 1: Change the effect default to solid and add white start color**

Replace the existing line:

```
CONFIG_ZMK_RGB_UNDERGLOW_EFF_START=4
```

with:

```
CONFIG_ZMK_RGB_UNDERGLOW_EFF_START=0
CONFIG_ZMK_RGB_UNDERGLOW_HUE_START=0
CONFIG_ZMK_RGB_UNDERGLOW_SAT_START=0
CONFIG_ZMK_RGB_UNDERGLOW_BRT_START=100
```

Leave all other lines (including `CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_IDLE=y` and `CONFIG_ZMK_RGB_UNDERGLOW_LAYER_ON=y`) unchanged.

- [ ] **Step 2: Verify the file**

Run: `grep -n "RGB_UNDERGLOW" config/boards/arm/rotr/rotr_defconfig`
Expected output includes:
```
CONFIG_ZMK_RGB_UNDERGLOW_EFF_START=0
CONFIG_ZMK_RGB_UNDERGLOW_HUE_START=0
CONFIG_ZMK_RGB_UNDERGLOW_SAT_START=0
CONFIG_ZMK_RGB_UNDERGLOW_BRT_START=100
```
and no remaining `EFF_START=4`.

- [ ] **Step 3: Commit**

```bash
git add config/boards/arm/rotr/rotr_defconfig
git commit -m "Boot underglow solid white for per-layer colors"
```

---

### Task 2: Rewrite `rotr.keymap` — behaviors, macros, three layers

Replace the entire keymap with the two-layer design plus hidden nav layer.

**Files:**
- Modify (full rewrite): `config/boards/arm/rotr/rotr.keymap`.

**Interfaces:**
- Consumes: `RGB_COLOR_HSB` (from `dt-bindings/zmk/rgb.h`); `&to`, `&mo`, `&kp`, `&trans` (from `behaviors.dtsi`); white default from Task 1.
- Produces: layers `0`=base (white), `1`=second (orange), `2`=nav (hidden); behaviors `enc`, `mid_lt`; macros `to_base`, `to_second`.

- [ ] **Step 1: Write the full keymap file**

Replace the entire contents of `config/boards/arm/rotr/rotr.keymap` with:

```dts
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/rgb.h>

/ {
	behaviors {
		enc: enc {
			compatible = "zmk,behavior-sensor-rotate-key-press";
			label = "ENC";
			#sensor-binding-cells = <2>;
			triggers-per-rotation = <30>;
		};

		mid_lt: middle_layer_tap {
			compatible = "zmk,behavior-hold-tap";
			label = "MID_LT";
			flavor = "hold-preferred";
			tapping-term-ms = <200>;
			bindings = <&mo>, <&kp>;
			#binding-cells = <2>;
		};
	};

	macros {
		to_base: to_base {
			compatible = "zmk,behavior-macro";
			label = "TO_BASE";
			#binding-cells = <0>;
			bindings = <&to 0>, <&rgb_ug RGB_COLOR_HSB(0,0,100)>;
		};

		to_second: to_second {
			compatible = "zmk,behavior-macro";
			label = "TO_SECOND";
			#binding-cells = <0>;
			bindings = <&to 1>, <&rgb_ug RGB_COLOR_HSB(30,100,100)>;
		};
	};

	keymap {
		compatible = "zmk,keymap";

		base_layer {
			bindings = <&kp DOWN &mid_lt 2 LG(B) &kp UP>;
			sensor-bindings = <&enc RIGHT LEFT>;
		};

		second_layer {
			bindings = <&kp DOWN &mid_lt 2 LG(B) &kp UP>;
			sensor-bindings = <&enc LS(RIGHT) LS(LEFT)>;
		};

		nav_layer {
			bindings = <&to_base &trans &to_second>;
		};
	};
};
```

Notes for the implementer:
- Position order is `<left middle right>` per layer — left=`DOWN`, middle=`mid_lt`, right=`UP`.
- `mid_lt 2 LG(B)`: hold → momentary layer 2 (nav); tap → cmd-b. `hold-preferred` makes pressing left/right during the hold resolve to the layer immediately.
- Nav layer omits `sensor-bindings` intentionally — the knob falls through to the layer below; nav is transient.
- Colors: white = `RGB_COLOR_HSB(0,0,100)`, orange = `RGB_COLOR_HSB(30,100,100)`.

- [ ] **Step 2: Static review of the devicetree**

Run: `grep -nE "RGB_|rgb_ug|mid_lt|&to |&enc|layer|compatible" config/boards/arm/rotr/rotr.keymap`
Verify by eye:
- Exactly three layer nodes: `base_layer`, `second_layer`, `nav_layer`.
- The only `&rgb_ug` usages are `RGB_COLOR_HSB(0,0,100)` and `RGB_COLOR_HSB(30,100,100)` — no `RGB_TOG/ON/OFF/HUI/EFF` etc. (ephemeral constraint).
- Both content layers reference `&mid_lt 2 LG(B)` at position 1.

- [ ] **Step 3: Commit**

```bash
git add config/boards/arm/rotr/rotr.keymap
git commit -m "Rewrite keymap: 2 layers with ephemeral per-layer LEDs"
```

---

### Task 3: Build verification

Confirm the firmware compiles with the new keymap and config.

**Files:** none (CI / build only).

**Interfaces:**
- Consumes: Tasks 1 and 2.
- Produces: a green build / `zmk.uf2` artifact.

- [ ] **Step 1: Trigger the build**

Push the branch to GitHub; the `Build` workflow runs on push:

```bash
git push -u origin feat/two-layer-keymap
```

(Optional local equivalent, if Docker is available:
`docker run --rm -v "$PWD":/work -w /work zmkfirmware/zmk-build-arm:stable bash -c "west init -l config && west update && west zephyr-export && west build -s zmk/app -b rotr -- -DZMK_CONFIG=/work/config"`)

- [ ] **Step 2: Verify the build passes**

Run: `gh run watch $(gh run list --branch feat/two-layer-keymap --limit 1 --json databaseId -q '.[0].databaseId')`
Expected: workflow concludes **success**; the "West Build (rotr)" step compiles with no devicetree or Kconfig errors, and the `.config` dump shows `CONFIG_ZMK_RGB_UNDERGLOW_EFF_START=0`.

If the build fails, read the failing step's log, fix the devicetree/Kconfig issue, commit, and re-push before proceeding.

---

## Self-Review

**Spec coverage:**
- 2 content layers + hidden nav → Task 2 (three layer nodes). ✓
- Per-layer LED (white/orange) → Task 2 macros + Task 1 solid-white boot. ✓
- Hold-middle + left/right switching → Task 2 `mid_lt` + nav layer `to_base`/`to_second`. ✓
- Knob: arrows / shifted arrows → Task 2 `enc` sensor-bindings. ✓
- Buttons DOWN / cmd-b / UP on both layers → Task 2 bindings. ✓
- Ephemeral (no NVS write) → Global Constraint + Task 2 Step 2 review (only `RGB_COLOR_HSB`). ✓
- Solid effect + white boot defaults → Task 1. ✓
- Build verification → Task 3. ✓

**Placeholder scan:** No TBD/TODO; full file contents and exact commands given.

**Type/name consistency:** `enc`, `mid_lt`, `to_base`, `to_second`, layers `0/1/2` used consistently across tasks and the macro/behavior definitions. ✓
