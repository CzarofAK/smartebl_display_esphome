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
OTA updates, and remote logging. The board's onboard Ethernet (PoE) port is
a third, independent, optional wired path — see "Networking" for how these
three things relate, since two of them use "ETH"/RJ45 in their name for
unrelated reasons and are easy to conflate.

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
| Network | Wi‑Fi 6 via the onboard ESP32-C6 co-processor (`esp32_hosted`) — **active**, this display's connection to Home Assistant. Onboard Ethernet PHY (IP101GRI, RMII) with PoE — configured, optional wired path, unrelated to either Wi-Fi or the RS232 link to `smartebl`. See "Networking" below. |

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

### Networking — three independent things, none of them each other

Easy to conflate since two of them involve the letters "ETH" and an RJ45
connector for unrelated reasons. Spelled out once, here, rather than
relying on context every time it comes up elsewhere in this repo:

1. **Wi-Fi 6, via the onboard ESP32-C6 co-processor (`esp32_hosted`) —
   ACTIVE, per user decision.** This is how the display talks to Home
   Assistant (entities, OTA, logging) — configured in `.basics.yaml`
   (from `basic.yaml.example`) exactly like Wi-Fi on the user's other,
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

2. **The onboard Ethernet PHY (IP101GRI, RMII) with PoE** — the
   ESP32-P4-WIFI6-POE-ETH board's *own* wired LAN port, configured (see
   `docs/hardware.md`) but **optional and not required** now that Wi-Fi
   is the active path — plug a cable in if you want a wired fallback,
   leave it unplugged otherwise. Has nothing to do with either Wi-Fi or
   item 3 below, despite sharing an RJ45 shape and the word "Ethernet."
   ESPHome's board manifest configures both this and item 1
   simultaneously without conflict; `smart-ebl-display.yaml`'s
   `network: priority: [ethernet, wifi]` makes it deterministic —
   wired wins the default route when a cable happens to be plugged in,
   Wi-Fi otherwise.

3. **The RS232 link to `smartebl` (`link_uart`, `docs/protocol.md`) —
   NOT Ethernet, NOT a LAN protocol, despite also using an RJ45
   connector.** `smartebl`'s own hardware carries RS232 + LIN + 12V
   over that connector (see `smartebl`'s `CLAUDE.md`), and this display
   reaches it through a **separate, custom RS232-to-3.3V-TTL interface
   board** — not the onboard Ethernet PHY from item 2, which is a
   different physical port entirely. See "The RS232 link" below for the
   full picture; this item exists here mainly so nobody reads "RJ45" or
   "ETH" anywhere in this repo and assumes it's real Ethernet.

## Architecture

```
┌─────────────────────────┐  RS232 + 12V (RJ45 connector, custom
│   Smart EBL main unit   │  RS232-to-TTL interface board -
│  (smartebl, ESP32-WROVER)│  NOT Ethernet, see "Networking" #3)
│  all sensors + switches │◄───────────────────────────────────┐
└─────────────────────────┘                                    │
                                                                 ▼
┌───────────────────────────────────────────────────────────────────┐
│              Smart EBL Display (this repo, ESP32-P4)               │
│  UART link (docs/protocol.md)  →  local mirrored sensors/switches  │
│  LVGL touch UI (docs/design_rules.md)                              │
└─────────────┬───────────────────────────────────────┬─────────────┘
              │ Wi-Fi 6 (esp32_hosted) - ACTIVE        │ Ethernet PHY
              ▼                                        │ (PoE) -
      Home Assistant / OTA / logging                   ▼ optional,
                                              wired fallback if plugged in
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
- [ ] Build out the remaining pages per `docs/pages.md`'s catalog
      (Levels, Fans, Light/Features, Sensors+Levelling, and the
      Electric section's fuse-grid sub-page) — this repo's own answer
      to what `smartebl_display`'s five sections should grow into here

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

# Compile
esphome compile smart-ebl-display.yaml

# Flash via USB-C (UART0-based flashing port - see docs/hardware.md)
esphome upload smart-ebl-display.yaml

# Flash OTA afterwards, over Wi-Fi (or the wired Ethernet port, if plugged in)
esphome upload smart-ebl-display.yaml --device smart-ebl-display.local
```
