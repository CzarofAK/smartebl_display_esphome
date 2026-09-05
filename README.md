# Smart EBL Display — ESP32-P4 (DSI)

Firmware for the **next-generation Smart EBL display**: a
[Waveshare ESP32-P4-WIFI6-POE-ETH](https://www.waveshare.com/esp32-p4-wifi6-poe-eth.htm)
driving a [10.1-DSI-TOUCH-A](https://www.waveshare.com/10.1-dsi-touch-a.htm)
10.1" capacitive touchscreen (800×1280, MIPI-DSI, GT9271 touch), replacing the
Nextion NX8048P070-011C used by the previous
[smartebl_display](https://github.com/CzarofAK/smartebl_display) generation.

It talks to the [Smart EBL](https://github.com/CzarofAK/smartebl) main unit
(ESP32-WROVER) over the **same RS232 wiring already in the vehicle** — the
RJ45 cable that today carries the Nextion's RS232 + LIN + 12V. No Wi‑Fi, no
Home Assistant, no router required for the display to read every sensor and
drive every switch the main unit exposes. Ethernet (via the board's onboard
PoE port) is only there for OTA updates and an optional Home Assistant API
connection — it is not on the path between the display and the vehicle
electrics.

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
| Network | Onboard Ethernet PHY (IP101GRI, RMII) with PoE — primary/active path. Wi‑Fi 6 + BLE via the onboard ESP32-C6 co-processor (`esp32_hosted`) are wired up as infrastructure but **not activated** (no `wifi:` block yet) — see below. |

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

### Why Ethernet, not Wi-Fi 6

ESPHome's own board manifest for this exact board configures both
Ethernet **and** the ESP32-C6 Wi-Fi 6/BLE co-processor (`esp32_hosted`,
over SDIO) as standard, plus an `update:` entity that keeps the C6's
own firmware current from Espressif/ESPHome's hosted-firmware manifest.
`smart-ebl-display.yaml` adopts that SDIO plumbing (confirmed pins, see
`docs/hardware.md`) so Wi-Fi/BLE are a `wifi:` block away whenever
wanted — but doesn't add that block, and Ethernet stays the primary,
active path. Two reasons:

- The display's actual job (control via RS232, see below) needs no
  network at all — Ethernet only serves OTA/HA, and PoE means one less
  cable to route.
- [esphome/esphome#10956](https://github.com/esphome/esphome/issues/10956)
  reported SDIO packet drops and repeated Wi-Fi association failures on
  a sibling Waveshare ESP32-P4 + C6 board. Whether that was really a
  stale-C6-firmware problem (which the `update:` entity now guards
  against) or something deeper is untested here. If you do add a
  `wifi:` block, check that issue first and make sure the "C6 Firmware"
  update entity reports current before relying on it.

## Architecture

```
┌─────────────────────────┐        RS232 (existing RJ45 cable)
│   Smart EBL main unit   │◄──────────────────────────────────┐
│  (smartebl, ESP32-WROVER)│    fuses · tanks · battery ·       │
│  all sensors + switches │    Truma · relay switches           │
└─────────────────────────┘                                    │
                                                                 ▼
┌───────────────────────────────────────────────────────────────────┐
│              Smart EBL Display (this repo, ESP32-P4)               │
│  UART link (docs/protocol.md)  →  local mirrored sensors/switches  │
│  LVGL touch UI (docs/design_rules.md)                              │
└───────────────────────────────────────────────────────────────────┘
                 │ Ethernet (PoE) — optional
                 ▼
        Home Assistant / OTA (not required for vehicle control)
```

## The RS232 link — controlling everything `smartebl` controls, without Wi-Fi

The two boards are connected point-to-point, full-duplex, over the same wire
pair that already carries the Nextion's RS232 signal in `smartebl` (its
`display_uart`, GPIO18/19, through the TRS3221 (U18) RS232 transceiver — see
`smartebl`'s `CLAUDE.md`). That transceiver converts the main unit's 3.3V
UART to true RS232 voltage levels for the cable run; the ESP32-P4's UART
pins are 3.3V logic and **cannot** be wired to that cable directly — the
display side of the harness needs its own RS232-to-3.3V-TTL transceiver
(e.g. a MAX3232/SP3232 breakout) between the RJ45 jack and `link_tx_pin`/
`link_rx_pin`. This is a hardware change to make once, independent of
firmware.

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

## Status & Roadmap

- [x] Basic ESPHome bring-up: Ethernet, PSRAM, MIPI-DSI display, GT911
      touch, RS232 link skeleton (read + write), one live home page
- [x] Adopted ESPHome's own board manifest defaults: `engineering_sample`,
      experimental IDF features, correct `phy_addr`, UART0 console, and
      the ESP32-C6 `esp32_hosted` SDIO link + firmware auto-updater as
      inactive infrastructure (see "Why Ethernet, not Wi-Fi 6")
- [ ] Verify the DSI reset/backlight-enable pins and pick+confirm free
      header GPIOs for the RS232 link against the actual board (see the
      TODOs in `docs/hardware.md` and in `smart-ebl-display.yaml`)
- [ ] Add a matching RS232-to-TTL transceiver to the display's wiring
      harness (hardware task, not firmware)
- [ ] Implement the counterpart protocol handler in `smartebl` (separate
      repo, separate PR)
- [ ] Extend the display-side link handler to the full key list in
      `docs/protocol.md` (all 16 fuses, remaining switches, Truma climate)
- [ ] Build out the remaining pages for UX parity with `smartebl_display`'s
      five sections
- [ ] Decide whether/when to add an active `wifi:` block on top of the
      already-provisioned `esp32_hosted` link, as a PoE alternative —
      check esphome/esphome#10956 first if you do

## Building & Flashing

```bash
# Install ESPHome (2025.8+ — the mipi_dsi component this config uses
# landed in ESPHome 2025.8.0)
pip install esphome

# Copy the templates and fill in your values
cp secrets.yaml.example secrets.yaml
cp basic.yaml.example .basics.yaml
# edit both — OTA/API credentials, and the RS232 link GPIOs once you've
# confirmed them against docs/hardware.md

# Validate before flashing (per m5dial_fram's own house rule — cheap and
# catches duplicate ids / dangling references as a hard error)
esphome config smart-ebl-display.yaml

# Compile
esphome compile smart-ebl-display.yaml

# Flash via USB-C (UART0-based flashing port - see docs/hardware.md)
esphome upload smart-ebl-display.yaml

# Flash OTA afterwards, over Ethernet
esphome upload smart-ebl-display.yaml --device smart-ebl-display.local
```
