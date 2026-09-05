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

## Not confirmed — placeholders, must be verified before flashing

**DSI panel reset / backlight-enable pins.** `mipi_dsi`'s
`WAVESHARE-10.1-DSI-TOUCH-A` model (built into ESPHome ≥2025.8, so no
custom `init_sequence` is needed — see below) only supplies the panel's
timing/init data, not which GPIOs drive its reset and backlight on
*this* board. The only concrete numbers found anywhere (GPIO27 reset /
GPIO26 backlight PWM) are documented for a different board —
`ESP32-P4-WIFI6-Touch-LCD-XC` — and must **not** be assumed to carry
over to the POE-ETH board's separate DSI connector. `smart-ebl-display.yaml`
leaves `reset_pin:`/`enable_pin:` as an obvious placeholder
(`GPIO_TODO_RESET` / `GPIO_TODO_BACKLIGHT`) that will fail config
validation on purpose, forcing you to fill in the real pins from the
POE-ETH schematic PDF before it can compile:
`files.waveshare.com/wiki/ESP32-P4-WIFI6-POE-ETH/ESP32-P4-WIFI6-POE-ETH-Schematic.pdf`
(also blocked from this session — open it from a normal connection).

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
