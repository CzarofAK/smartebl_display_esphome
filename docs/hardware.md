# Hardware reference — ESP32-P4-WIFI6-POE-ETH + 10.1-DSI-TOUCH-A

As of: 2026-09-05. Applies to `smart-ebl-display.yaml`.

This session's network egress proxy blocks `waveshare.com` (all
subdomains, including the file host carrying the schematic PDF) and
`esphome.io`, so nothing here could be confirmed by opening the primary
source directly from this session. Every row below says exactly how it
was sourced and how confident that makes it — **check the "Confidence"
column before trusting a pin number enough to wire it.**

## Confirmed — cross-checked against ESPHome's own source

ESPHome's `ethernet` component hard-codes a fixed set of RMII pins for
every ESP32-P4 board (`ESP32P4_RMII_DEFAULT_PINS` in
`esphome/components/ethernet/__init__.py`), because RMII TX/RX/CRS_DV/
TX_EN aren't freely reassignable on this chip the way they are on
classic ESP32. Those hard-coded numbers match what Waveshare's own wiki
publishes for this board's Ethernet section — good, independent
confirmation from two different sources:

| Signal | GPIO | Notes |
|---|---|---|
| TXD0 | GPIO34 | |
| TXD1 | GPIO35 | |
| RXD0 | GPIO29 | |
| RXD1 | GPIO30 | |
| CRS_DV | GPIO28 | |
| TX_EN | GPIO49 | |

**Confidence: high.**

## Confirmed — from ESPHome's own board manifest

MDIO/MDC, the RMII reference clock, and the PHY address *are*
board-specific wiring choices (unlike the fixed RMII pins above), so
there's no ESPHome-side default to cross-check them against on general
principle - but ESPHome ships an actual board manifest for this board
(`definitions/boards/waveshare_esp32_p4_wifi6_poe_eth`), and picking
this board in the ESPHome dashboard generates exactly these values.
That's a materially better source than the search-indexed Waveshare
wiki page this table originally cited, and it agrees with that page -
upgraded from medium to high confidence:

| Signal | GPIO | Notes |
|---|---|---|
| MDIO | GPIO52 | Ethernet management interface |
| MDC | GPIO31 | Ethernet management interface |
| REF_CLK | GPIO50 | **Input** to the ESP32-P4 — the IP101GRI PHY generates 50 MHz from its own 25 MHz crystal and feeds it in. In ESPHome terms: `clk: {pin: GPIO50, mode: CLK_EXT_IN}`, not `CLK_OUT`. |
| PHY reset/power | GPIO51 | Wired as `ethernet: power_pin:` |
| PHY address | 1 | `ethernet: phy_addr: 1` — ESPHome's own default is 0; this board straps the IP101GRI to address 1. Easy to miss since a wrong `phy_addr` doesn't fail loudly, it just never links. |
| PHY | IP101GRI | `ethernet: type: IP101` |

Touch I²C bus, still only sourced from a search summary of Waveshare's
wiki (not independently confirmed the way the table above now is), but
consistent with the same two pins being used across several other
Waveshare ESP32-P4-WIFI6 boards (a good sign it's a deliberate, repeated
header convention rather than a one-off):

| Signal | GPIO |
|---|---|
| I2C SDA (touch) | GPIO7 |
| I2C SCL (touch) | GPIO8 |

**Confidence: medium.** Plausible and internally consistent, but came
back through a search summary rather than the primary page — glance at
the board's silkscreen or the schematic PDF once you have normal
internet access, before final wiring.

## Confirmed — Wi-Fi 6 / BLE via the ESP32-C6 co-processor (esp32_hosted)

Same board manifest, same confidence upgrade: the SDIO link to the
onboard ESP32-C6-MINI-1 is standard infrastructure ESPHome's own
board definition wires up, not something guessed for this repo.
`smart-ebl-display.yaml` configures it, plus the `update:` entity that
keeps the C6's own firmware current from Espressif/ESPHome's hosted
firmware manifest; the active `wifi:` block that actually rides on this
transport lives in `.basics.yaml` (per user decision - see the README's
"Networking" section for how this, the onboard Ethernet PHY below, and
the RS232 link to `smartebl` are three unrelated things).

| Signal | GPIO |
|---|---|
| SDIO CLK | GPIO18 |
| SDIO CMD | GPIO19 |
| SDIO D0 | GPIO14 |
| SDIO D1 | GPIO15 |
| SDIO D2 | GPIO16 |
| SDIO D3 | GPIO17 |
| C6 reset (`reset_pin`, `active_high: true`) | GPIO54 |

**Confidence: high.**

## Touch controller

The 10.1-DSI-TOUCH-A's touch chip is the GT9271, which the GT911
ESPHome component (`touchscreen: platform: gt911`) already handles —
same protocol family, default I²C address `0x5D`. `smart-ebl-display.yaml`
deliberately configures no `interrupt_pin`/`reset_pin`: on other
Waveshare ESP32-P4 DSI boards, the touch controller's interrupt line
isn't routed to the MCU at all, and reset is left unconfigured on
purpose to avoid disturbing the controller's address-select strapping.
Both pins are optional in the ESPHome component and it falls back to
polling — same pattern m5dial_fram uses for its own touch controller
(`design_rules.md`'s ft5x06 note: "With interrupt_pin, the first flash
produced: touchscreen is marked FAILED"). **Confidence: medium** — the
polling behavior is confirmed from the ESPHome component itself, but
whether *this* board also leaves the interrupt line unrouted is
inferred from sibling boards, not confirmed for the POE-ETH variant
specifically.

## Confirmed on the real board — DSI panel reset / backlight-enable pins not needed

**Superseded** — this document's earlier versions treated
`reset_pin:`/`enable_pin:` as blocking placeholders (`GPIO_TODO_RESET`/
`GPIO_TODO_BACKLIGHT`) to be filled in from the schematic before the
config could even compile. Both are optional in `mipi_dsi`'s schema,
and confirmed on the real board: the panel comes up over the DSI init
sequence alone, no host-driven reset pulse needed, and the backlight is
apparently either always-on or controlled some other way outside this
config. `smart-ebl-display.yaml` now leaves both commented out rather
than guessing GPIOs. **Confidence: high** — confirmed working on the
actual hardware, not inferred from a sibling board. If software
backlight control is ever wanted, that's a real GPIO to find on the
POE-ETH schematic PDF
(`files.waveshare.com/wiki/ESP32-P4-WIFI6-POE-ETH/ESP32-P4-WIFI6-POE-ETH-Schematic.pdf`,
blocked from this session — open it from a normal connection), not a
placeholder to guess at.

## Not confirmed — placeholders, must be verified before flashing

**RS232 link UART pins.** Not a hardware fact to look up at all — these
are two free GPIOs *you* choose from the 28 broken out on the board's
2×20 header, wired to whichever pins your RS232-to-TTL transceiver
breakout lands on (see the README's "RS232 link" section for why that
extra transceiver chip is needed). `smart-ebl-display.yaml`'s
`link_tx_pin`/`link_rx_pin` substitutions are placeholders — just avoid
the GPIOs already spoken for above (7, 8, 14–19, 28–31, 34, 35, 49–52,
54) and whatever else you use on the header. Most of those are almost
certainly point-to-point traces to the onboard PHY/co-processor rather
than pins ever routed to the header in the first place, so this is
unlikely to actually crowd the 28 header GPIOs the way the raw count
suggests — but it's not confirmed either way, so treat the list as
"definitely don't use these," not as "only 7 GPIOs remain."

## Reusing the existing OEM 6-pin harness (DS470FR ↔ PC380)

The vehicle already has a 6-pin cable run between the old CBE DS470FR
electroblock and the old PC380 display panel — already routed through
the vehicle body, which is worth reusing rather than pulling a new
cable. Whether that works comes down entirely to `smartebl`'s own RJ45
pinout (its `J9` connector, from that repo's
`information/PCB connectors.md` — read-only reference, not something
this repo can edit):

| RJ45 pin | Signal | Direction (from `smartebl`) |
|---|---|---|
| 1 | RXD_PANEL | **input** (display → `smartebl`) |
| 2 | TXD_PANEL | **output** (`smartebl` → display) |
| 3 | LIN | bidirectional — **not needed** by this display (the link protocol, `docs/protocol.md`, is pure RS232) |
| 4 | +BATT (fused F21) | output, display power |
| 5 | +BATT (fused F21) | output, display power — **same node as pin 4** on `smartebl`'s own board |
| 6 | NC | — |
| 7 | GND | — |
| 8 | GND | — **same node as pin 7** on `smartebl`'s own board |

Only 4 wires are actually needed: RX, TX, +BATT, GND. Pins 4/5 and 7/8
are safe to bridge at an adapter (a single incoming +BATT wire feeding
both pin 4 and 5, a single GND wire feeding both 7 and 8) precisely
*because* they're already tied to the same net on `smartebl`'s PCB —
bridging them there doesn't create a second, competing source. **Pins 1
and 2 must never be bridged to each other** — those are two distinct,
independently-active signals on the *same* end of the cable (RX in,
TX out); shorting them together ties `smartebl`'s own UART transmit and
receive lines to each other, guaranteed non-functional and needlessly
stresses the driver. A 6-wire cable comfortably covers RX + TX + +BATT
+ GND with two wires to spare (plausibly the old LIN wire, unused here,
plus one more).

**Open items — not something this session can resolve, no documentation
of the DS470FR/PC380 harness itself exists in any repo this session can
reach:**
- Which of the 6 physical wires is RX vs. TX vs. +12V vs. GND is a
  CBE-specific assignment, unrelated to `smartebl`'s own J9 numbering.
  Identify it before wiring the adapter — GND via continuity to
  chassis/battery negative, +12V via a voltage check with the panel
  powered, RX vs. TX from old CBE service documentation if available,
  or empirically (power up the old system, scope/logic-probe which
  wire toggles when the DS470FR boots — that one is its TX).
- Confirm the old PC380 actually used **both** directions of that
  cable (display → EBL, not just EBL → display). A display-only
  protocol with no return channel would leave this display's touch
  input with no way back to `smartebl` over the reused cable, even
  though the wire is physically present.

## Display model string

```yaml
display:
  - platform: mipi_dsi
    model: WAVESHARE-10.1-DSI-TOUCH-A
```

This model already exists in ESPHome's `mipi_dsi` component
(`esphome/components/mipi_dsi/models/waveshare.py`), inheriting its
full init sequence, timing, 800×1280 dimensions and 2-lane/1.5 Gbps
DSI config from the identical panel used on Waveshare's
`ESP32-P4-Nano-10.1` kit. **Confidence: high** — read directly from
ESPHome's own source, not from a Waveshare page.

## Required companion component: esp_ldo

`display.mipi_dsi` lists `esp_ldo` as a hard `DEPENDENCIES` entry -
`esphome config` fails outright ("Component display.mipi_dsi requires
component esp_ldo") without one somewhere in the file. The ESP32-P4's
DSI/CSI PHY is powered from the chip's internal adjustable LDO, not the
3.3V rail directly, so this isn't optional. `smart-ebl-display.yaml`'s
`esp_ldo:` block (`channel: 3`, `voltage: 2.5V`) matches ESPHome's own
test fixture for this exact component
(`tests/components/mipi_dsi/test.esp32-p4-idf.yaml`) exactly - **not**
a value guessed for this board. No `ldo_id:` reference exists anywhere
else in the config; ESPHome wires the LDO to the DSI PHY internally
once the component is present, whatever channel/id you give it.
**Confidence: high** — read directly from ESPHome's own test fixture.

## Engineering-sample silicon

ESPHome's board manifest sets `esp32: engineering_sample: true` for
this board. That flag exists because early ESP32-P4 chips shipped as
"ES" (engineering sample) silicon with a few ESP-IDF workarounds needed
around it; the manifest defaulting to `true` implies current-production
units of this board still ship that silicon. **Confidence: high** as
"this is what ESPHome assumes for this board" — not independently
verified against a die marking on any specific unit. If your board
turns out to have production (non-ES) silicon, flipping this to
`false` is the only thing to change.

## Programming / console UART

**Corrected from this document's first version**, which assumed the
P4's native USB-Serial-JTAG (no separate bridge chip) - wrong. The
board manifest sets `logger: hardware_uart: UART0` as standard, which
only makes sense if this board's USB-C port is wired to UART0 (almost
certainly through an onboard USB-to-serial bridge chip, not the P4's
native USB peripheral) rather than USB-Serial-JTAG. `smart-ebl-display.yaml`
now sets this explicitly. **Confidence: high** (board manifest), though
which physical bridge chip is on the board specifically is still
unconfirmed - doesn't change how you flash it either way.

## PSRAM / Flash

32 MB in-package PSRAM (`psram: mode: hex`, ESP32-P4 only supports hex
mode — quad/octal aren't valid on this variant per ESPHome's own
`psram` component schema) and 32 MB NOR flash, per Waveshare's product
listing. `speed: 200MHz` is the fastest mode ESPHome allows for
`esp32p4` (20/100/200 MHz); drop it to `100MHz` if boot is unstable on
your specific unit.
