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
| Mounting orientation | **undecided** — see §4 |

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

- **Mounting orientation.** The panel is natively 800×1280 (portrait).
  Whether it's mounted portrait or rotated 90°/270° for a landscape
  dash position is a physical install decision nobody has made yet —
  `smart-ebl-display.yaml` currently leaves `lvgl: rotation: 0`
  (portrait) as a placeholder. Pick the real mounting orientation
  before designing page layouts; changing it later reflows everything.
- **Rectangular layout grid.** `m5dial_fram`'s §3 grid (y_main/y_row/…)
  has no equivalent here yet. The first PR that builds a real page
  (beyond this repo's single bring-up page) should establish one — a
  shared `substitutions:` block of y/x positions, per that document's
  own reasoning ("identical values in every page file, so merging
  collapses to one block") — and record it in this section.
- **Performance at 800×1280.** LVGL software rendering at this
  resolution is a tracked concern upstream —
  [esphome/esphome#16873](https://github.com/esphome/esphome/issues/16873)
  is specifically about making an even lower-resolution (800×800) P4
  MIPI-DSI UI usable. Keep an eye on redraw cost (avoid a
  whole-screen `draw_` script running every 200 ms if it turns out to
  cost real frame time) once more than the one bring-up page exists.
- **Multi-page navigation model.** The M5Dial pages via a rotary
  encoder; this is a touchscreen with no such knob. A swipe/tab-bar
  navigation scheme (matching `smartebl_display`'s five sections, per
  the README) needs designing once there's more than one page —
  nothing here should be read as assuming encoder-style paging.
- **Alarm/status-line precedence.** `m5dial_fram/design_rules.md` §6's
  "rank first, then damage before inconvenience" precedence rules apply
  to *values* this display also shows (fuses, tanks), but the actual
  Master Warning/Caution presentation is `smartebl_display`'s design,
  not `m5dial_fram`'s — reconcile the two once this display grows an
  alarm/status page, rather than picking one arbitrarily now.
