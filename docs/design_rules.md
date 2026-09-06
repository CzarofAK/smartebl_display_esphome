# Smart EBL Display (ESP32-P4) — Design and Build Specification

As of: 2026-09-05. Applies to all pages of this display. Adapted from
[`m5dial_fram/design_rules.md`](https://github.com/CzarofAK/m5dial_fram/blob/main/design_rules.md)
— read there first if a rule here seems to assume context; only the
parts that carried over to a rectangular touch panel are repeated below,
and this document says explicitly where it stops following that one. If
a page deviates from this document, either the page is wrong or this
document is outdated — either way it must be corrected, not ignored
(the same house rule, restated for this repo).

## 1. Target device

| | |
|---|---|
| Board | Waveshare ESP32-P4-WIFI6-POE-ETH (ESP32-P4, 32 MB PSRAM) |
| Display | 10.1-DSI-TOUCH-A, 800×1280, MIPI-DSI, `mipi_dsi` platform |
| Touch | GT9271 via `gt911` platform, I²C |
| Mounting orientation | **Landscape** — `lvgl: rotation: 90` |

## 2. What carries over from `m5dial_fram`, unchanged

These are the parts of that document that are properties of the *fleet*
(every Smart EBL display, regardless of screen shape), not of the round
M5Dial specifically. Reuse them verbatim rather than re-deriving a
second, possibly-drifting copy:

- **§6's color palette**, byte for byte — background `0x000000`, good
  `0x2ECC71`, warning `0xF39C12`, alarm/error `0xE74C3C`, invalid
  `0x555555`, secondary line `0xBDC1C6`, footer `0x707070`, neutral
  active `0x4A9EFF`. A page on this display and a page on the M5Dial
  showing the same quantity must agree on what color it gets.
- **§6's thresholds** — battery SOC, starter battery voltage, grid
  load, fresh/grey water, gas per bottle. These are business-logic
  constants about the vehicle, not display geometry; there's no reason
  for this display to disagree with the M5Dial about when the leisure
  battery counts as low.
  For the fuse-derived values that only exist here (`smartebl` exposes
  16 individual fuses this display can show that the M5Dial doesn't),
  reuse the same rule `smartebl` already applies to itself:
  `fuse_fN_ok = fuse_fN_voltage > 1.0 V`.
- **§7's invalid-value rule**, adapted for the link instead of the API:
  a value fed by the RS232 link (`docs/protocol.md`) that hasn't been
  refreshed within that document's staleness window (§4 there) must
  render as `---` in `0x555555`, unit kept (`--- %`, not `---`) exactly
  like the M5Dial's `#ifdef USE_API` pattern — same reasoning, different
  transport. A button driven by that link follows the same rule: `PUMP
  ---`, not `PUMP AUS`, while the link is down.
- **§9's binding pattern** — one drawing script per page, called both
  from the value's own update path and from `esphome: on_boot:
  priority: -100` so the page doesn't sit on its placeholder text until
  the first real value arrives. "Several interdependent values → one
  script that redraws the page" applies here for exactly the reason it
  applies on the M5Dial: without it, a footer or label computed from
  two related values sits wrong between the two updates.
- **§10's naming scheme**, with one addition: `link_` for anything
  belonging to the RS232 reader/writer itself (the buffer global, the
  `send_link_command` script) rather than to any one page — it's shared
  infrastructure, the same reason `m5dial_fram`'s design_rules.md keeps
  the rotary encoder and the day/night globals in the shared header
  file instead of on a page.

| Prefix | For |
|---|---|
| `page_` | LVGL page |
| `lbl_` | label |
| `btn_` | button |
| `s_` | sensor / text sensor fed by the link (data source) |
| `draw_` | a page's drawing script |
| `link_` | RS232 link infrastructure (buffer, reader, writer) — not page-specific |

## 3. What does *not* carry over

Everything in that document tied to a **round 240×240** display: §1's
r=120 existence check, §4's arc geometry (135°→45°, the double-ring
layout, the per-radius text-width tables), §3's exact pixel constants
(`y_main: -20` etc. — those were tuned by looking at a physical M5Dial,
not derived from a formula, and reapplying them to an 800×1280 panel
would be exactly the "invented equivalent" this document should not do).
A rectangular 10.1" panel has no corners to clear and roughly 25× the
pixel area — page layout here gets its own grid, decided by looking at
the actual device once real pages are built, not guessed in advance.

## 4. Open items

- ~~**Mounting orientation.**~~ Decided: landscape, `lvgl: rotation: 90`
  in `smart-ebl-display.yaml`. The panel is natively 800×1280
  (portrait) - anything laid out against this orientation is laid out
  against a 1280×800 canvas, not the panel's native dimensions.
- ~~**Rectangular layout grid.**~~ First pass established by the first
  three real pages (`page_home`, `page_electric`, `page_climate`):
  `smart-ebl-display.yaml`'s `substitutions:` block carries `screen_w`/
  `screen_h` (1280x800, the landscape canvas after `rotation: 90`),
  `bar_h` (64, top status bar), `nav_w` (130, left nav rail), and
  `content_x`/`content_y` (where every page's own content starts) —
  same reasoning as `m5dial_fram`'s §3 ("identical values in every page
  file, so merging collapses to one block"). Still first-pass, not
  final: per this section's own reasoning, refine the numbers once
  seen on the real device, but reuse these names rather than inventing
  a second grid.
- **Performance at 800×1280.** LVGL software rendering at this
  resolution is a tracked concern upstream —
  [esphome/esphome#16873](https://github.com/esphome/esphome/issues/16873)
  is specifically about making an even lower-resolution (800×800) P4
  MIPI-DSI UI usable. Keep an eye on redraw cost (avoid a
  whole-screen `draw_` script running every 200 ms if it turns out to
  cost real frame time) once more than the one bring-up page exists.
  **Confirmed on real hardware (2026-09-06):** intermittent
  `E lcd.dsi: can't fetch data from external memory fast enough,
  underrun happens` in the boot log - almost certainly this same
  upstream concern (the main LVGL display buffer at this resolution is
  large enough that it has to live in PSRAM regardless of anything this
  repo's own config does, and DSI's own DMA fetch competing with
  anything else on the PSRAM bus can starve it). The Mainscreen clock's
  own canvas used to add to that PSRAM pressure until it was shrunk to
  fit internal RAM instead (see `smart-ebl-display.yaml`'s
  `clock_face` widget and `m5dial_clock_sbb`'s README) - that removes
  one contributor, not the underlying upstream issue itself. Not
  something this repo can fix on its own; watch esphome#16873.
  **Still recurring (confirmed against a fuller real boot log, roughly
  every 30-90s during normal WiFi+HA operation, not just at boot) -
  non-fatal, the panel keeps working.** One mitigation was tried and
  **reverted**: `lvgl: buffer_size: 25%` (shrinking LVGL's own staging
  draw buffer, separate from the DSI peripheral's continuous full-frame
  hardware buffer which stays full-size regardless, so LVGL's rendering
  work would contend less with that continuous DMA read on the shared
  PSRAM bus) - the commonly-recommended starting point for this symptom
  on ESP32-P4 MIPI-DSI boards, not a finding of this repo's own. Pulled
  after the repo owner reported visible flicker on real hardware right
  after it landed (same flash as `page_levels`) - consistent with a
  smaller buffer meaning more, smaller flushes, one of which landing
  mid-scan would read exactly like a faint tear. Back to ESPHome's own
  default (unset, full-frame buffer); the underrun log line itself is
  still open, unresolved upstream (esphome#16873) - a confirmed new
  visual artifact was worse than an already-tracked non-fatal log line,
  so this wasn't a net improvement to keep.
- ~~**Multi-page navigation model.**~~ Decided at the concept level:
  [`docs/pages.md`](pages.md) — a persistent left nav rail (not a
  swipe/tab-bar), a bottom-left quick-switch popup reachable from any
  page, and a persistent top status bar. See that document; it also
  carries its own open items (alarm ranking, a backlight pin, the
  floor-plan page's source image) that belong there, not here.
- ~~**Alarm/status-line precedence.**~~ Decided at the concept level:
  per user decision, *every* alarm (fuses, tanks, gas, sensors,
  levelling) centralizes into the one top status bar
  (`smartebl_display`'s Master Warning/Caution design, adopted as-is) —
  not a per-page footer like `m5dial_fram`. `m5dial_fram/design_rules.md`
  §6's within-category precedence rules (damage before inconvenience,
  the gas combined-supply logic) still apply to the *values* shown here,
  but now feed one cross-category ranking instead of several independent
  per-page ones. That ranking is itself still a draft — see
  `docs/pages.md` §7.
