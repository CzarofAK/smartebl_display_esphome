# Smart EBL Display — ESP32-P4 (DSI)

Firmware for the **next-generation Smart EBL display**: a
[Waveshare ESP32-P4-WIFI6-POE-ETH](https://www.waveshare.com/esp32-p4-wifi6-poe-eth.htm)
driving a [10.1-DSI-TOUCH-A](https://www.waveshare.com/10.1-dsi-touch-a.htm)
10.1" capacitive touchscreen (800×1280, MIPI-DSI, GT9271 touch), replacing the
Nextion NX8048P070-011C used by the previous
[smartebl_display](https://github.com/CzarofAK/smartebl_display) generation.

It talks to the [Smart EBL](https://github.com/CzarofAK/smartebl) main unit
(ESP32-WROVER) over the **same RS232 wiring already in the vehicle** via a
custom RS232-to-TTL interface board (not a standard Ethernet/LAN link,
despite the RJ45 connector — see "Networking" below) — the same physical
cable that today carries the Nextion's RS232 + LIN + 12V. No network hop
of any kind is on that path: reading every sensor and driving every switch
the main unit exposes works even if Wi-Fi/Home Assistant are down.

Separately, the display itself connects to **Home Assistant over Wi-Fi 6**
(via the board's onboard ESP32-C6 co-processor) for the usual HA integration,
OTA updates, and remote logging — the only network path this display uses.
The board also has an onboard Ethernet (PoE) port, but it's not configured
here (removed 2026-09-06, not needed now that Wi-Fi 6 is the always-on
path) — see "Networking" for the full picture, since it and the RS232 link
both use "ETH"/RJ45-shaped things for unrelated reasons and are easy to
conflate.

## Relationship to the other Smart EBL repositories

This repo is developed alongside three others in the same project. Only this
one is writable from here — the other three are **read-only reference
material**:

| Repository | Role | Access |
|---|---|---|
| [`smartebl`](https://github.com/CzarofAK/smartebl) | Smart EBL main unit firmware (ESP32-WROVER). The single source of truth for every sensor/switch id this display mirrors — see `smart-ebl.yaml` there. | read-only |
| [`smartebl_display`](https://github.com/CzarofAK/smartebl_display) | Previous-generation Nextion 7" UI. Its Master Warning/Caution philosophy, five sections (Electric/Levels/Climate/Status/Power) and alarm precedence rules are the UX reference this display should eventually match or improve on. | read-only |
| [`m5dial_fram`](https://github.com/CzarofAK/m5dial_fram) | FRAM cockpit M5Dial. Source of the design-language conventions adapted in [`docs/design_rules.md`](docs/design_rules.md) (palette, invalid-value handling, naming scheme, drawing-script binding pattern). | read-only |
| `smartebl_display_esphome` (this repo) | ESP32-P4 + DSI touch display firmware. | **read/write** |

If you change something in `smartebl` or `smartebl_display` that this repo
depends on (an entity id, a threshold, a page concept), it has to happen in a
session that has write access there — this repo can only consume those as
references, never edit them.

## Hardware

| | |
|---|---|
| Board | Waveshare **ESP32-P4-WIFI6-POE-ETH** — ESP32-P4 (dual-core RISC-V, no native radio) + ESP32-C6-MINI-1 Wi-Fi 6/BLE 5 co-processor over SDIO + onboard 100 Mbps Ethernet with PoE |
| Display | Waveshare **10.1-DSI-TOUCH-A** — 10.1", 800×1280, IPS, MIPI-DSI (JD9365 panel, 2 lanes), GT9271 capacitive touch (GT911-compatible) |
| PSRAM | 32 MB, in-package, hex mode |
| Flash | 32 MB NOR |
| Programming | USB-C, wired to UART0 (likely through an onboard USB-to-serial bridge, not the P4's native USB-Serial-JTAG — see `docs/hardware.md`) |
| Network | Wi‑Fi 6 via the onboard ESP32-C6 co-processor (`esp32_hosted`) — **active**, the only network path this display uses. The board also has an onboard Ethernet PHY (IP101GRI, RMII) with PoE, but it's **not configured** in `smart-ebl-display.yaml` — removed per repo-owner decision, not needed now that Wi-Fi 6 is the always-on path. See "Networking" below. |

Everything above is sourced from one of: ESPHome's own `mipi_dsi`/
`ethernet`/`esp32_hosted` component source, ESPHome's own board manifest
for this exact board (`waveshare_esp32_p4_wifi6_poe_eth` — the
authoritative source once it exists, since it's what generates the
config ESPHome's dashboard gives you for this board), or Waveshare's
public product/wiki pages read through search (this session's network
egress proxy blocks `waveshare.com`/`esphome.io` directly, so the
schematic PDF and docs site couldn't be opened here for a final visual
confirmation). See [`docs/hardware.md`](docs/hardware.md) for exactly
which pins fall in which bucket, which are carried over from a
*sibling* Waveshare P4 board and still need checking against this
board's own schematic, and which are plain placeholders you choose
yourself (the RS232 link UART pins). **Read that file's confidence
column before flashing.**

### Networking — two independent things, not each other

Easy to conflate since both involve an RJ45 connector for unrelated
reasons (a third, the onboard Ethernet PHY, used to be part of this list
too — removed 2026-09-06, see below). Spelled out once, here, rather
than relying on context every time it comes up elsewhere in this repo:

1. **Wi-Fi 6, via the onboard ESP32-C6 co-processor (`esp32_hosted`) —
   ACTIVE, per user decision, and the ONLY network path this display
   uses.** This is how the display talks to Home Assistant (entities,
   OTA, logging) — configured in `.basics.yaml` (from
   `basic.yaml.example`) exactly like Wi-Fi on the user's other,
   classic-ESP32 Smart EBL/FRAM devices, just riding over the C6's SDIO
   link instead of a native radio, since the P4 itself has none.
   ESPHome's own board manifest for this board ships this SDIO link and
   an `update:` entity that keeps the C6's own co-processor firmware
   current from Espressif/ESPHome's hosted-firmware manifest — adopted
   here as-is (confirmed pins, see `docs/hardware.md`).
   [esphome/esphome#10956](https://github.com/esphome/esphome/issues/10956)
   reported SDIO packet drops and repeated association failures on a
   sibling Waveshare ESP32-P4 + C6 board; whether the `update:` entity
   keeping firmware current avoids that is untested here — worth
   checking that issue if you see similar symptoms.

2. **The RS232 link to `smartebl` (`link_uart`, `docs/protocol.md`) —
   NOT Ethernet, NOT a LAN protocol, despite using an RJ45 connector.**
   `smartebl`'s own hardware carries RS232 + LIN + 12V over that
   connector (see `smartebl`'s `CLAUDE.md`), and this display reaches it
   through a **separate, custom RS232-to-3.3V-TTL interface board**. See
   "The RS232 link" below for the full picture; this item exists here
   mainly so nobody reads "RJ45" anywhere in this repo and assumes it's
   real Ethernet.

**The onboard Ethernet PHY (IP101GRI, RMII) with PoE** — the
ESP32-P4-WIFI6-POE-ETH board's own wired LAN port — is **REMOVED from
this config** (was item 2 in this list until 2026-09-06): per
repo-owner decision ("brauchen wir nicht"), it was only ever an
optional wired fallback and Wi-Fi 6 above is the always-on path this
display actually needs. `docs/hardware.md` keeps its pin table as
historical reference, marked removed there too.

## Architecture

```
┌─────────────────────────┐  RS232 + 12V (RJ45 connector, custom
│   Smart EBL main unit   │  RS232-to-TTL interface board -
│  (smartebl, ESP32-WROVER)│  NOT Ethernet, see "Networking" above)
│  all sensors + switches │◄───────────────────────────────────┐
└─────────────────────────┘                                    │
                                                                 ▼
┌───────────────────────────────────────────────────────────────────┐
│              Smart EBL Display (this repo, ESP32-P4)               │
│  UART link (docs/protocol.md)  →  local mirrored sensors/switches  │
│  LVGL touch UI (docs/design_rules.md)                              │
└─────────────────────────────┬───────────────────────────────────────┘
                               │ Wi-Fi 6 (esp32_hosted) - ACTIVE, only
                               ▼ network path
                     Home Assistant / OTA / logging
```

## The RS232 link — controlling everything `smartebl` controls, without a network hop

The two boards are connected point-to-point, full-duplex, over the same wire
pair that already carries the Nextion's RS232 signal in `smartebl` (its
`display_uart`, GPIO18/19, through the TRS3221 (U18) RS232 transceiver — see
`smartebl`'s `CLAUDE.md`). That transceiver converts the main unit's 3.3V
UART to true RS232 voltage levels for the cable run; the ESP32-P4's UART
pins are 3.3V logic and **cannot** be wired to that cable directly — the
display side needs its own RS232-to-3.3V-TTL interface board between the
RJ45 jack and `link_tx_pin`/`link_rx_pin` (user-built, purpose-made for
this — not the onboard Ethernet PHY, not a generic USB-RS232 dongle).

The plan is to reuse the vehicle's existing 6-pin cable between the old
CBE DS470FR electroblock and PC380 display, already routed through the
body, rather than pulling a new one — with a small 6-pin-to-RJ45 adapter
at the `smartebl` end. See `docs/hardware.md`'s "Reusing the existing
OEM 6-pin harness" section for `smartebl`'s exact RJ45 pin mapping, which
pins are safe to bridge at that adapter (and which two must never be),
and what's still open (identifying which physical wire in the old
harness is which signal, and confirming it's actually wired for both
directions).

On top of that wire, `docs/protocol.md` defines a small ASCII line protocol:
`smartebl` continuously broadcasts telemetry frames (one group of related
values at a time — fuses, tanks, electrical, switches, Truma, system), and
this display sends command frames whenever the user operates a control on
screen. No polling, no request/response round trip — RS232 is already
full-duplex point-to-point, so there's nothing to arbitrate.

**What's in this PR vs. what's follow-up work:**

- ✅ `smart-ebl-display.yaml` implements the *display* side of the link: the
  frame reader/writer, and a first working slice — `tank1_level`,
  `tank2_level`, `leisure_battery_voltage`, `starter_battery_voltage`
  mirrored read-only, plus `switch_pump` mirrored **and** controllable —
  wired into one live LVGL page, to prove the whole chain end to end.
- 📋 `docs/protocol.md` specifies the **complete** key list (every fuse,
  tank, switch, and Truma value `smartebl` exposes today) and the frame
  groups they belong to, so extending the display side is "wire one more
  `else if` following the existing pattern", not a redesign.
- ⛔ **Not done here:** the matching counterpart in `smartebl`'s own
  `smart-ebl.yaml` (broadcasting telemetry, listening for command frames).
  `smartebl` is read-only from this session — that half has to land as a
  separate change in that repo, against the spec in `docs/protocol.md`.

## Design language

`docs/design_rules.md` adapts the FRAM M5Dial's design conventions
([`m5dial_fram/design_rules.md`](https://github.com/CzarofAK/m5dial_fram/blob/main/design_rules.md))
for a much larger, rectangular touch panel: the same color palette and
alarm thresholds, the same invalid-value ("connection lost, don't show a
stale number") rule, the same `page_`/`lbl_`/`btn_`/`s_` naming scheme and
one-drawing-script-per-page binding pattern. What doesn't carry over is
everything tied to the M5Dial's round 240×240 geometry (the arc widgets, the
135°–45° gap, the two-column layout sized for an 80px inner radius) — this
display gets its own rectangular layout rules as pages are built.

[`docs/pages.md`](docs/pages.md) is the page catalog and navigation
concept: a left nav rail, a bottom-left quick-switch popup, a persistent
top status bar carrying every alarm (not per-page footers, unlike
`m5dial_fram`), and the full page list (Home, Electric, Levels, Climate,
Boiler, Fans, Light/Features, Sensors+Levelling) — read that before
`docs/design_rules.md`'s own layout-grid open item, since it answers
*what* each page needs to hold before deciding *how* to lay it out.

## Status & Roadmap

- [x] Basic ESPHome bring-up: Ethernet, PSRAM, MIPI-DSI display, GT911
      touch, RS232 link skeleton (read + write), one live home page
- [x] Adopted ESPHome's own board manifest defaults: `engineering_sample`,
      experimental IDF features, correct `phy_addr`, UART0 console, and
      the ESP32-C6 `esp32_hosted` SDIO link + firmware auto-updater
- [x] Active Wi-Fi 6 → Home Assistant (`.basics.yaml`, per user decision
      — see "Networking"), onboard Ethernet kept as an optional wired
      path with a deterministic `network: priority:` between the two
- [x] Confirmed on the real board: DSI panel reset/backlight-enable pins
      not needed (both commented out rather than guessed, see
      `docs/hardware.md`); mounting orientation decided (landscape,
      `lvgl: rotation: 90`, resolving `docs/design_rules.md` §4)
- [x] Page catalog + navigation concept decided (`docs/pages.md`):
      left nav rail, bottom-left quick-switch popup, one persistent
      status bar carrying every alarm — resolving `docs/design_rules.md`
      §4's navigation and alarm-precedence open items at the concept
      level (pixel layout is still open, per that document's own
      reasoning)
- [ ] Find the backlight PWM pin (`docs/pages.md` §7) — check the
      user-supplied [Waveshare wiki page](https://www.waveshare.com/wiki/10.1-DSI-TOUCH-A)
      for this exact panel once off this session's network block
- [ ] Confirm the cross-category alarm ranking draft in `docs/pages.md` §7
- [x] Light/Features floor-plan source photo added
      (`docs/assets/floorplan-source.jpg`, zone breakdown in
      `docs/pages.md` §7) — real entity_ids and pixel coordinates for
      the overlay are still open
- [ ] Pick+confirm free header GPIOs for the RS232 link against the
      actual board (see the TODO in `docs/hardware.md` and in
      `smart-ebl-display.yaml`) — match them to the user's own
      RS232-to-TTL interface board
- [ ] Identify which physical wire in the existing DS470FR/PC380 6-pin
      harness is RX/TX/+12V/GND, and confirm it's wired both directions
      (see `docs/hardware.md`'s "Reusing the existing OEM 6-pin harness")
- [x] **First release: three real pages built.** Mainscreen/Home (SBB
      clock — reused as-is from `m5dial_clock_sbb` — + 5 switch tiles +
      a permanently-reserved Fridge tile), Electric (Victron-style
      overview: Shore/Battery/AC-charger-voltage wired to real link
      data; AC Loads, Solar Yield and DC Loads render invalid on
      purpose — `smartebl` has no current-sense hardware to back a
      watt figure for any of them, see `docs/pages.md`'s Electric row),
      and Climate (Truma Combi 4 room-heating side via HA/
      `womolin_controller`, touch +/- buttons in place of
      `m5dial_fram`'s rotary-encoder arm/disarm step). Persistent top
      status bar (time/date + link status) and left nav rail
      implemented as LVGL `top_layer` widgets, so both survive page
      switches without being redefined per page. First-pass rectangular
      layout grid recorded in `docs/design_rules.md` §4. The
      quick-switch popup (`docs/pages.md` §3) is **not** built yet —
      this release's switches live directly on the Mainscreen instead,
      which covers the same controls without that infrastructure.
- [x] Implemented the counterpart protocol handler in `smartebl`'s own
      `smart-ebl.yaml`, for groups 1-4 (fuses, tanks, electrical,
      switches) — everything this release's pages need. Groups 5
      (Truma) and 6 (system) are not sent yet: the Climate page reads
      Truma via HA/`womolin_controller` instead of this link, so
      there's no consumer for group 5 over the wire yet.
- [ ] Extend the display-side link handler to group 1 (fuses, for a
      future fuse-grid Electric sub-page and the status bar's alarm
      ranking) and group 6 (system) — both are already broadcast by
      `smartebl` but not yet parsed here, since no page in this release
      shows either.
- [x] **Fixed a real bug found on first hardware flash:** the five
      switch mirrors (`sw_pump_mirror` etc.) used to have
      `turn_on_action`/`turn_off_action` calling `send_link_command`
      directly. ESPHome unconditionally runs those on every
      `turn_on()`/`turn_off()` call — including the one it makes at
      boot to enact the switch's initial state, and the one this repo's
      own link reader made on every incoming telemetry frame to reflect
      smartebl's real state. Net effect: every reboot silently sent
      "everything OFF" to smartebl, and every ~800ms telemetry cycle
      re-sent (and so re-pulsed) whichever bistable relay was already
      in that state, indefinitely, whether or not anything actually
      changed. Fixed: the mirrors now only hold state (the link reader
      calls `publish_state()`, not `turn_on()`/`turn_off()`), and each
      Mainscreen tile's `on_click` sends the command explicitly — the
      only place one is supposed to originate from now.
- [x] **Boiler page built** (`page_boiler`) — Truma Combi 4
      water-heating side, same `womolin_controller` integration as
      Climate, own nav rail entry. Ported from
      `m5dial_fram/page_boiler.yaml`'s three-fixed-tier stepping
      (ECO/mid/BOOST), touch-adapted the same way Climate was.
- [x] **Status bar: inside/outside temperature added**, per user
      feedback on the first hardware test. Inside reuses
      `page_climate`'s own Truma room sensor. Outside turned out to be
      the old Nextion panel's existing two-wire NTC probe, moved over
      to this board - so `s_temp_outside` is a LOCAL ADC/NTC sensor
      (`adc_temp_outside` → `r_temp_outside` → `s_temp_outside`), not
      an HA entity.
- [x] **GPIO confirmed for the outside-temp ADC: GPIO22**, not the
      original GPIO4 guess (which isn't an ADC pin on this chip at
      all). Cross-checked against ESP-IDF's own ADC channel table (repo
      owner supplied the link) — see `docs/hardware.md`'s new ADC
      section for the full ADC1/ADC2 GPIO map and which of those pins
      are already claimed by `esp32_hosted`/Ethernet/the RS232 link.
- [ ] **NTC calibration constants still unverified.** 10kΩ/B=3950 is a
      generic guess, not this specific sender's real datasheet values -
      wrong constants mean a plausible-looking but wrong reading, not
      an obvious failure. Measuring the sender's actual resistance at a
      known room temperature (repo owner has the hardware) would let
      this be calibrated properly instead of guessed.
- [x] **HA entity_ids confirmed by the repo owner** against their real
      `womolin_controller` MQTT integration - `climate.*_truma_room`/
      `_water` and `switch.*_activate_room_heater`/`_water_heater` were
      right. Also added, all real: `binary_sensor.*_room_heater_active`/
      `_water_heater_active` (is the Truma actually heating right now,
      shown next to the on/off button, not instead of it), and the
      shared (not per-room/water) `binary_sensor.*_heater_has_error`,
      `binary_sensor.*_cp_plus_alive`, `sensor.*_operating_status`,
      `sensor.*_heating_mode`, `sensor.*_heater_error_code` - all shown
      in a status/error footer on both Climate and Boiler. `select.*_
      fan_mode` and the three timer entities exist too but aren't wired
      up yet - fan mode selection and timer display are still follow-up
      work, not in this round.
- [x] **GPIO37/GPIO38 confirmed bad, moved to GPIO21/GPIO20.** The
      repo owner's own header photo settled it: GPIO37/38 are this
      board's silkscreen-labelled TXD/RXD (console UART), not a free
      pair at all. See `docs/hardware.md`'s "Not confirmed" section.
- [x] **Two real bugs from the same hardware test, fixed:** every
      page's background wasn't actually black (an LVGL page's own
      background sits on top of `lvgl: bottom_layer:` and needs its own
      `bg_color`/`bg_opa`, now set on all four pages) — and the
      `sbb_clock` widget rendered as a plain filled square, not a
      circle, because neither `transparent:` nor `show_face:` was set;
      both fixed, and the day/night colors corrected to the real SBB
      look (white face/black hands by day, inverted at night) that
      `smart-ebl-display.yaml`'s comments had backwards before.
- [x] **Manual day/night toggle** — a small `AUTO`/`DAY`/`NIGHT` button
      in the status bar (`cycle_daynight`), since the full popup
      (`pages.md` §3) isn't built yet and a manual override was
      requested on the same hardware test. No brightness control still
      (blocked on the unconfirmed backlight pin).
- [x] **Climate + Boiler merged into one page ("TRUMA"), per repo-owner
      request** — left half Heizung (room), right half Boiler (water),
      one nav entry instead of two. Both halves gained an arc (temp as
      % of a defined range - 5-30°C room, 40-80°C water - same
      "always a percentage" rule `m5dial_fram/design_rules.md` §8
      uses). Fan level added and fully working (read AND write, tap-to-
      cycle) - turned out to be `climate.womolin_controller_mqtt_truma_room`'s
      own `fan_mode` attribute (off/low/medium/high, confirmed real),
      not a separate `select.*` entity as first assumed.
- [x] **Home page tiles rearranged and centered** per repo-owner spec:
      2×3 grid (LIGHT/ICE-EX top, AUX/FRIDGE middle, POWER/PUMP bottom),
      vertically centered in the content area and horizontally centered
      between the nav rail and the clock - previously just placed at
      fixed coordinates, not actually centered or in the requested order.
- [x] **Electric page recentered, and rebuilt on real Victron/MPPT/GX
      entities - every box now has a real source.** The repo owner
      confirmed a real SmartShunt + MultiPlus behind `smartebl` (same HA
      entities `m5dial_fram/page_power_1/2/3.yaml` already use), plus an
      MPPT 190W solar charge controller and the GX device's own DC-
      consumption sensor (both new to this repo). Battery (SOC/V/A),
      Inverter/Charger mode+state (now a button opening a popup to set
      the MultiPlus mode directly - ON/CHG/INV/OFF - and adjust the
      shore/grid current limit), AC Loads (WR output power), Solar
      Yield (MPPT PV yield + status), and DC Loads (GX aggregate
      consumption) all show real data now - none of this page's boxes
      are placeholder-invalid anymore.
- [x] **Fixed the real boot crash chased through several earlier rounds
      of this list** — `assert failed:
      spi_flash_disable_interrupts_caches_and_other_cpu cache_utils.c:127
      (esp_task_stack_is_sane_cache_disabled())`, right after ESP-IDF's
      own "Disabling RNG early entropy source...", before ESPHome's own
      logger or any component setup ever runs. PSRAM speed (tried first:
      200MHz → 100MHz) was **not** the cause — that fix produced a
      byte-identical crash, which was itself the tell. Neither was any
      of this repo's own YAML content (`LV_USE_ARC`, page layout,
      sensor count - all ruled out by testing, and by `sdkconfig`/
      `partitions.csv` diffing byte-identical between a working and a
      crashing build). **Real cause**: the ESP32-C6 co-processor's own
      `esp_hosted` firmware (managed by Home Assistant's C6-firmware
      `update:` entity) had moved ahead of the HOST-side `esp_hosted`
      library this file's `esp32_hosted:` block compiles in — per that
      component's own maintainer, the C6's firmware must be >= the host
      library, never the other way around; an unpinned host library
      just takes whatever ESPHome's release bundles, with no such
      guarantee. **Fix**: pin both ends to the same known release
      (`esp32: framework: components: - espressif/esp_hosted==2.12.12`)
      plus the sdkconfig/stack-size options that came with that fix on
      the same P4+C6-hosted board family - confirmed working on real
      hardware. Keep the pinned version in sync with whatever the C6
      actually runs if this recurs. `psram: speed: 100MHz` is kept as a
      safe value, not because 200MHz was ever confirmed unsafe -
      untested since the real fix landed.
- [x] **Levels, Fans, Electric's fuse-grid sub-page, and Sensors/
      Levelling built** - see this file's changelog entry below for the
      full detail on each; the remaining catalog gap is Light/Features
      (no floor-plan entity_ids yet, `docs/pages.md` §7) and Sensors
      sub-page 1's real content (no door/window/presence entity list
      yet, same section).
- [x] **Ethernet removed** per repo-owner decision ("brauchen wir
      nicht") - the `ethernet:` block, `network: priority:`, and every
      "active wired fallback" claim in this file are gone; Wi-Fi 6 is
      now the only network path. `docs/hardware.md` keeps the PHY pin
      table as historical reference, marked removed there too.
- [x] **Real bug fixed: `[W][homeassistant.sensor:014]: 'climate...
      truma_water': Can't convert 'None' to number!` warning spam.**
      The Truma's `current_temperature`/`temperature` climate attributes
      go HA-side `None` while the unit reports nothing usable, and a
      numeric `sensor: platform: homeassistant` has no way to suppress
      the warning it logs when that happens - fixed by reading the
      attribute as a `text_sensor` instead (never tries a float parse
      itself) and parsing it by hand into a `sensor: platform: template`
      of the same id, skipping the publish (same as the old numeric
      sensor's own silent-keep-last-value behavior) when it's
      "None"/"unknown"/"unavailable". Applied to all four climate-
      attribute readers (room/water × current/target) for the same
      reason, not just the one that was actually seen warning.
- [x] **Day/night is popup-based now** (`pages.md` §4's own long-
      standing open item) - tapping the status bar's AUTO/DAY/NIGHT
      button opens a popup with the AUTO/DAY/NIGHT picker AND a
      brightness slider (`g_backlight_pct`, persisted). AUTO still
      follows `sun.sun` exactly like before. The slider has nothing real
      to dim yet - still blocked on the unconfirmed backlight PWM pin
      (`docs/hardware.md`) - `apply_backlight` is the one place a real
      `output:` write belongs once that pin is found.
- [x] **Sleep mode added.** Auto: between 22:00 and 08:00, the screen
      blanks after 10 minutes with no touch (`check_sleep_idle`,
      `touchscreen: on_touch:`); any touch wakes it immediately. Manual:
      a small SLEEP button next to the day/night one blanks on demand,
      any time. Simulated with a full-screen black overlay, not a real
      backlight-off, for the same unconfirmed-pin reason as the
      brightness slider above.
- [x] **Real bug fixed: the nav rail's own divider line ran the full
      screen height** even though only 3 of ~8 planned nav entries
      exist yet, leaving a stray-looking grey line hanging below the
      last button (TRUMA) in otherwise-empty space. Now stops at the
      bottom of the last real button; extend it when the next nav entry
      is added (comment left in place at the widget itself).
- [x] **Real bug fixed: German umlauts and the Victron-flow arrows
      (→/↓) rendered as blank boxes.** ESPHome/LVGL's built-in
      `montserrat_NN` fonts only rasterize plain ASCII - declared real
      `font:` entries instead (`font_ui_16/20/24/40`, Montserrat via
      `gfonts:`) with an explicit glyph list covering everything this
      file's labels actually use, umlauts/ß/arrows included, and swapped
      every `text_font: montserrat_NN` reference to the matching one.
- [x] **Truma page: naming and layout cleanup.** "HEIZUNG (RAUM)" /
      "BOILER (WASSER)" shortened to just "HEIZUNG" / "BOILER" per
      repo-owner spec. The "SOLL nn°C" target line moved from a single
      line above the +/- buttons to two lines ("SOLL" / "nn°C",
      "ECO"/"BOOST" gets its own third line on the Boiler side) centered
      in the gap BETWEEN them instead.
- [x] **Electric page: taller boxes, real spacing, several new real
      values.** Both rows grew (130px/170px → 210px/290px) with matched
      top/bottom margins - this page used to leave most of its content
      area empty. New per repo-owner spec: Shore's main value is now the
      MultiPlus's real input power (sub-left/right voltage/frequency,
      replacing the old W+A sub-line); AC Loads mirrors that shape with
      output power/voltage/frequency (replacing the old %-of-1600W-
      nominal figure); Solar Yield is now a button opening its own
      ON/OFF (`switch.mppt_190w`) + charge-limit
      (`number.mppt_190w_stromstarke`) popup, sub-left/right now the
      MPPT's own DC battery-bus voltage/current; Battery gained an
      IDLE/CHARGING/DISCHARGING state line (interpreted from the
      SmartShunt's net current) and a remaining-time line between the
      SOC and a proper 3-across V/A/W bottom row (fixing a real overlap
      bug between the old mid-line and the starter-voltage line), plus
      its own popup for `switch.fram_parkmodus`. Also fixed: the
      Charger popup's mode buttons were left-to-right ON/CHG/INV/OFF,
      now OFF/CHARGER/INVERTER/ON; its own OFF button's "active"
      highlight used to be styled identically to "inactive" (never
      visibly highlighted); and the LIMIT label drifted off-center as
      its digit count changed for want of `text_align: CENTER` - all
      three fixed. DC Loads' main value is still a placeholder
      (repo owner: "tbd", no entity picked yet).
- [ ] **Victron GUIv2's Inverter/Charger detail screen has a vertical
      bar this page doesn't have an equivalent for yet** - repo owner
      asked what it represents. Best guess from this session (NOT
      confirmed against Victron's own source or a live GX device): the
      AC input current as a percentage of the configured input current
      limit, i.e. `sensor.multiplus_eingangsstromstarke_l1` relative to
      `number.multiplus_strombegrenzung` (already wired into this page's
      Charger popup). Confirm on a real GX Touch/VRM before wiring an
      indicator to it - this is a guess, not a verified finding.
- [ ] **Clock jitter root-caused, fixed upstream in `m5dial_clock_sbb`,
      not in this repo.** The second hand's uneven "hopping" motion
      traced to `transparent: true` (needed so the clock's square canvas
      corners show the page's black background through) forcing a full
      60-tick redraw on an ARGB8888 PSRAM-backed canvas every single
      `render_interval`, exactly the risk that component's own docs
      already flagged. Fixed there: a full redraw is now only needed
      once (plus after a night-mode flip) when `show_face: true`, same
      as this repo's own usage - see that repo's changelog. Nothing to
      change here beyond picking up the updated component on the next
      `external_components:` fetch (git `ref: main`, no pin to bump).
- [x] **Levels, Fans, Electric's fuse-grid sub-page, and Sensors/
      Levelling built** (`docs/pages.md`'s catalog, minus Light/Features
      and Sensors sub-page 1's content - both still open, no entity
      lists yet). Also: the Electric section's "1/2" sub-page indicator
      (`pages.md` §2) is implemented for real for the first time, and
      reused for the new Sensors section too.
    - **Levels** (`page_levels`) - built, but corrected against what
      `smartebl`'s protocol/hardware actually expose rather than
      matching `docs/pages.md`'s original Fresh/Waste/Gas A/Gas I/Diesel
      catalog description verbatim: that hardware has exactly 3 generic
      tank-sensor inputs (`tank1_level`/`tank2_level`/`tank3_level`), no
      dual-bottle gas sensor and no diesel tank of its own. FRISCH/GRAU
      are `tank1`/`tank2` over the link (already wired before this page
      existed); TANK 3 is `tank3_level`, newly wired (protocol.md called
      it "planned" - `smartebl` already broadcasts it, this display just
      hadn't parsed it), physical identity unconfirmed, shown generically
      rather than guessed. GAS A/GAS I are the same real
      `sensor.gas_1_stand`/`sensor.gas_2_stand` HA entities
      `m5dial_fram/page_gas.yaml` already reads for this vehicle's actual
      (separate) gas monitoring - same precedent as this repo's own
      Electric page reusing `m5dial_fram`'s real Victron entities.
      Diesel has no known real entity anywhere in this project (checked
      `m5dial_fram` too) - not built, `docs/pages.md` corrected to match
      rather than left describing a column with nothing behind it.
      Rendered as vertical LVGL `bar` widgets (fills bottom-up at
      width<height, no hand-rolled segments) - same "Victron-style
      vertical tank column" idea as the original Nextion mockup, LVGL-
      native implementation. Thresholds reused byte-for-byte from
      `m5dial_fram/page_water.yaml`/`page_gas.yaml` (design_rules.md §2).
    - **Electric fuse grid** (`page_electric_fuses`) - all 16 fuses,
      protocol group 1, now real end to end (`smartebl` already
      broadcast it; this display hadn't parsed it until now). Named for
      their real function (`smartebl`'s own CLAUDE.md hardware table),
      not the old Nextion mockup's generic placeholders. One real
      refinement over the naive `fuse_fN_ok = voltage > 1.0V` rule:
      these ADCs read the LOAD side, so a switched group (Light/12V/
      AUX/Pump) reads near 0V any time its own group switch is simply
      off - the same reading a blown fuse would give. Cross-referenced
      each switched fuse against that group's own switch mirror (already
      on this display via the link) so "AUS" (switch off, expected) and
      "AUSGEFALLEN?" (should be live per its own switch, isn't) are two
      different things instead of one ambiguous red dot. F2/F3 (vehicle/
      D+-controlled) and F10 (fridge D+/programmable) have no switch
      mirror here to check against - shown read-only, never alarmed,
      same "don't claim precision this display doesn't have" reasoning
      as the Home page's Fridge tile.
    - **Fans** (`page_fans`) - same two entities `m5dial_fram/
      page_fans.yaml` uses (FANBOARD, HVAC), touch-adapted the same way
      Climate/Boiler's +/- buttons already were (no rotary-encoder "arm"
      step - each half's own +/- buttons write directly, debounced).
      Two-halves-with-a-divider layout, same as the Truma page, rather
      than that M5Dial file's concentric double ring (round-panel-
      specific, design_rules.md §3).
    - **Sensors / Levelling** (`page_sensors_levelling`) - ported from
      `m5dial_fram/page_levelling.yaml` verbatim (bubble-in-target-rings
      metaphor, worst-corner-wins bubble color, same still-open item
      there: the four corner cm sensors read 0 until an HA-side template
      sensor does the real degrees-to-cm math). Sensors sub-page 1
      (`page_sensors_overview`) is an honest placeholder, not a guess at
      content - `docs/pages.md` §7's own open item (no door/window/
      presence entity list yet) is unchanged, but the SENSORS nav entry
      and its own "1/2" indicator exist now, so Levelling is reachable
      without waiting on page 1.
    - **`esphome`'s own `merge_warnings` noise, silenced.** The
      page_electric row-template pattern (`&box_frame`/`&row_main`/etc.,
      merged into each box via `<<:`) is, by inspection of ESPHome's own
      `yaml_util.py` merge-key handling (confirmed against ESPHome
      2026.8.2, matching a real build log) and by re-running `esphome
      config` on this exact file, working exactly as intended - every
      box's own `id:`/`text:`/`x:` override DOES win over the anchor's
      placeholder. The ~19 "Key '...' was dropped while processing a
      '<<' merge" warnings on every single compile all just point at the
      anchor's OWN definition line (not the overriding box) because
      that's where ESPHome's warning renderer sources the reported
      position from - not a sign anything was ever broken. Silenced via
      `esphome: { merge_warnings: false }`, the warning's own suggested
      remedy, rather than left to reappear on every future compile.
    - **DSI buffer-underrun mitigation attempt, UNTESTED on real
      hardware.** `E lcd.dsi: can't fetch data from external memory fast
      enough, underrun happens` (design_rules.md §4's existing tracked
      item, esphome#16873) is still recurring per a fuller real boot log
      (roughly every 30-90s during normal operation, non-fatal). Added
      `lvgl: buffer_size: 25%` - shrinks LVGL's own staging draw buffer
      (separate from the DSI peripheral's own continuous full-frame
      hardware buffer, which stays full-size regardless) to reduce how
      much PSRAM bandwidth LVGL's rendering work competes for against
      that continuous DMA read. The commonly-recommended starting point
      for this exact symptom on ESP32-P4 MIPI-DSI boards - not this
      repo's own finding, and not confirmed to help here; if it doesn't
      measurably improve the log or introduces a visible redraw seam,
      reverting is deleting that one line. See design_rules.md §4 for
      the full write-up.

## Building & Flashing

```bash
# Install ESPHome (2025.8+ — the mipi_dsi component this config uses
# landed in ESPHome 2025.8.0)
pip install esphome

# Copy the templates and fill in your values
cp secrets.yaml.example secrets.yaml
cp basic.yaml.example .basics.yaml
# edit both — Wi-Fi/OTA/API credentials in secrets.yaml, the fallback
# AP password directly in .basics.yaml, and smart-ebl-display.yaml's
# RS232 link GPIOs once you've confirmed them against docs/hardware.md

# Validate before flashing (per m5dial_fram's own house rule — cheap and
# catches duplicate ids / dangling references as a hard error)
esphome config smart-ebl-display.yaml

# ⚠ Whenever you touch `psram:` (mode/speed) or anything else that maps to
# an ESP-IDF sdkconfig option, `esphome clean` first - see the note right
# below this block for why this isn't optional on this board.
esphome clean smart-ebl-display.yaml

# Compile
esphome compile smart-ebl-display.yaml

# Flash via USB-C (UART0-based flashing port - see docs/hardware.md)
esphome upload smart-ebl-display.yaml

# Flash OTA afterwards, over Wi-Fi (the only network path - Ethernet is
# not configured, see "Networking" above)
esphome upload smart-ebl-display.yaml --device smart-ebl-display.local
```

### ⚠ `esphome clean` after any PSRAM/sdkconfig change - not optional here

Confirmed on this exact board (chip revision v1.3, the ES silicon `engineering_sample: true` above already flags): changing `psram:` (e.g. `speed: 200MHz` → `100MHz`, done in an earlier round to try to fix a boot crash) reflashed and produced the **byte-identical crash** - same PC, same register dump, same stack contents down to the last hex digit - as before the change. That's not "the fix didn't work", that's ESPHome's incremental build reusing stale compiled objects that don't match the new sdkconfig, so the flashed binary never actually reflected the new PSRAM setting at all. This is a known ESPHome bug on ESP32-P4 specifically:
[esphome/esphome#15336](https://github.com/esphome/esphome/issues/15336) (same v1.3 ES silicon, same crash family) → fixed by
[esphome/esphome#15439](https://github.com/esphome/esphome/pull/15439) ("force a full rebuild if any sdkconfig options are changed" - the specific culprit named there is `execute_from_psram`), later relaxed in
[esphome/esphome#18876](https://github.com/esphome/esphome/pull/18876).

Whether or not your installed ESPHome version already includes that fix, `esphome clean smart-ebl-display.yaml` before `compile` is the safe manual equivalent - cheap, and removes any doubt about whether a config change actually reached the flashed binary. Do this any time `psram:`, `esp32: framework:`, or anything else that isn't plain application logic changes - not just once.
