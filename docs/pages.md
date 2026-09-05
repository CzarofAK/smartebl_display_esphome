# Page & Navigation Concept

As of: 2026-09-05. Applies to `smart-ebl-display.yaml`. Resolves
`design_rules.md` §4's "Multi-page navigation model" open item **at the
concept level** — which pages exist, what's on them, how you get between
them, and where each value's alarm surfaces. The other two open items in
that section (the exact rectangular pixel grid, and performance at
800×1280) are **not** resolved here on purpose — per that document's own
reasoning, pixel positions get decided by looking at the actual device
once a page is actually built, not guessed in advance. Treat this
document as "what goes where", `design_rules.md` as "how a page is built"
once it exists.

If a page deviates from this document, either the page is wrong or this
document is outdated — either way it must be corrected, not ignored (same
house rule as `design_rules.md` and `docs/protocol.md`).

## 1. Why this display's navigation differs from both references

`smartebl_display` (Nextion, 800×480) has 5 sections with a resistive,
text-only touch panel. `m5dial_fram` (240×240, round) has no navigation
UI at all — a rotary encoder pages through a fixed list. This display is
capacitive multi-touch, rectangular, and roughly 3× the pixel area of
either — big enough that per-page footers for alarms (`m5dial_fram`'s
approach) stop making sense once there are 7+ pages: a fuse alarm sitting
in the water page's footer would only be seen by someone currently on the
water page. Per user decision, **this display centralizes every alarm
into one persistent status bar** (`smartebl_display`'s original Master
Warning/Caution concept) instead — see §4.

## 2. Left navigation rail

Persistent on every page except possibly the home page itself (open —
decide once a page exists to check whether repeating it on the home page
reads as redundant or as helpful consistency).

| | |
|---|---|
| Width | ~100–120 px (≈15–20 mm at this panel's ~5.9 px/mm density — the width the user originally asked about) |
| Content | One icon+label per section (§5's page catalog), stacked vertically |
| Sub-pages | A section with more than one page shows a page indicator under its icon when active (`smartebl_display`'s "1/2" pattern, `docs/design.md` — reused verbatim, don't reinvent) — tapping the active icon again cycles to the next sub-page, exactly like that document's "Vertical Navigation" section |
| Bottom slot | One extra icon, visually separated by a divider from the section icons above it — **not a section**, an action: opens the quick-switch popup (§3). Reachable from every page. |

## 3. Quick-switch popup (bottom-left)

Per user decision, the switches are **not** a permanently reserved
column on every page (unlike `smartebl_display`'s original mockups) —
every other page needs its full content width (a Victron-style flow
diagram or 5 tank columns doesn't have 100px to spare, and duplicating
controls on every page invites accidental taps while just reading a
value). Instead:

- Triggered from the nav rail's bottom icon (§2), on any page.
- Opens as a popup/bottom-sheet anchored bottom-left.
- Contents: exactly the 5 switches from `docs/protocol.md` group 4 that
  the home page also shows directly — `switch_light`, `switch_12v`,
  `switch_aux`, `switch_pump`, `eisex_output`. Same widgets, same command
  path (`send_link_command`), just reachable without navigating home
  first.
- Fridge and Alarmo are deliberately **not** in this popup — see §6.

## 4. Status bar (top)

Full width, persistent on every page (including the home page).

| Zone | Normal state | Alert state |
|---|---|---|
| Left/center | Time + date, plus inside/outside temperature — **implemented**: inside reuses `page_climate`'s own Truma room sensor (no second HA entity needed); outside (`s_temp_outside`) is a LOCAL two-wire NTC probe moved over from the old Nextion panel, read directly over ADC (`adc_temp_outside`→`r_temp_outside`→`s_temp_outside`), not an HA entity - GPIO and NTC calibration constants are unverified guesses, see `README.md`'s roadmap | Master Warning (red) / Caution (orange) scrolling text — `smartebl_display/docs/design.md`'s existing color/priority scheme, reused unchanged |
| Right, icon cluster | WLAN status · RS232 link status (`docs/protocol.md`'s existing "LINK OK"/"NO LINK", already implemented in `smart-ebl-display.yaml`) · Brightness/Day-Night | — |

**Alarmo is explicitly not in this bar** — homepage tile + PIN popup only
(§6). Keeping a security-panel shortcut out of the always-visible bar was
a deliberate user call, not an oversight.

### Brightness / Day-Night icon

Tap opens a popup: a brightness slider + an AUTO/DAY/NIGHT selector.

- **Day/Night color scheme** (dark-on-black ↔ light, the same swap the
  `sbb_clock` widget already does via `set_night_mode()`) is software-only
  and can ship regardless of hardware status. Automatic default: base it
  on `sun.sun` elevation from Home Assistant, the same source
  `m5dial_fram`'s clock page uses (`id(my_sun).is_below_horizon()`) —
  there's no confirmed ambient-light sensor on this board. The popup's
  AUTO/DAY/NIGHT selector overrides this per user choice.
- **Brightness slider is blocked on a hardware unknown**: `docs/hardware.md`
  currently leaves `reset_pin`/`enable_pin` unconfigured on this board —
  the backlight is "either always-on or controlled some other way", no
  PWM-capable GPIO confirmed. The slider UI can be built now, but it has
  nothing to drive until that pin is found and wired (see §7's open item —
  the user-supplied Waveshare wiki page for this exact panel is the next
  place to check once this session's network block on `waveshare.com`
  doesn't apply).

## 5. Page catalog

Order is provisional (unlike `m5dial_fram`, there's no rotary encoder
address depending on it — a touchscreen can jump to any tab directly), but
listed in the nav rail's intended top-to-bottom order.

| # | Section | Sub-pages | Content | Data source |
|---|---|---|---|---|
| 0 | Home | — | Clock (`sbb_clock`, right half) + 5 switch tiles + Fridge tile + Alarmo tile (§6) | link (switches) + HA (clock temp) |
| 1 | Electric | 1: Overview, 2: Fuses | Victron-style power flow (Shore→Inverter→AC Loads / Solar→Battery→DC Loads) per `smartebl_display/docs/design.md`'s existing mockup — reuse directly, don't redesign; page 2: 16-fuse grid, `fuse_fN_ok = voltage > 1.0 V` per `design_rules.md` §2 | link (`docs/protocol.md` groups 1, 3) |
| — | *implemented, page 1* | — | **Superseded finding, corrected:** this repo initially assumed the vehicle had no Victron gear behind `smartebl`, so AC Loads/Solar Yield/DC Loads rendered permanently invalid. The repo owner confirmed a real SmartShunt (battery monitor) and MultiPlus (inverter/charger), integrated into HA via the exact same entities `m5dial_fram`'s `page_power_1/2/3.yaml` already use — now wired: **Battery** box shows SmartShunt SOC%/voltage/current (same 20%/40% thresholds as `page_power_1`); **Inverter/Charger** box (now a button) shows the MultiPlus's real mode (`select.multiplus` — ON/CHG/INV/OFF) and state (`sensor.multiplus_zustand`), tapping it opens a popup to set the mode directly and adjust the shore/grid current limit (`number.multiplus_strombegrenzung`, clamped 0-16A per `page_power_2`'s own real-wiring-ceiling note); **AC Loads** shows the MultiPlus's WR output power (`sensor.multiplus_0_wr_leistung`) as a % of the installed model's 1600W nominal rating, same caveat as `page_power_3`'s identical calculation (not exact, thermal derating untracked). **Solar Yield and DC Loads remain invalid** — no solar-yield entity has been named yet, and the SmartShunt's net current (already shown in the Battery box) isn't a clean "loads-only" figure; still open items, not guessed. Shore connected/disconnected stays `smartebl`'s own link reading (`shore_power_connected`) — that detection is real and cheap regardless. Page 2 (fuse grid) is not built yet. Page also recentered (both horizontally and vertically within the content area) per repo-owner feedback that it wasn't. | — |
| 2 | Levels | — | Fresh / Waste / Gas A / Gas I / Diesel as vertical tank columns, per `smartebl_display/docs/design.md`'s existing Levels mockup. Gas combined-supply logic reused verbatim from `m5dial_fram/design_rules.md` §6 (`GAS LEER` only when **both** < 10%). Tank alarms surface in the status bar (§4), not a page footer — a deviation from `m5dial_fram`'s per-page-footer convention, deliberate per §1 above | link (group 2) |
| 3 | Climate | — | Truma Combi 4 room-heating side: current/target temp, on/off, via the `womolin_controller` MQTT integration — same entities `m5dial_fram/page_climate.yaml` already uses | HA |
| 4 | Boiler | — | Truma Combi 4 water-heating side: 3 fixed tiers (ECO/mid/BOOST), same `womolin_controller` integration as Climate | HA |
| — | *implemented, merged* | — | **Per repo-owner request, Climate and Boiler share ONE page and ONE nav entry ("TRUMA")** — left half Heizung (room), right half Boiler (water), not two separate sections as originally cataloged above. Each half now has an arc (room temp as % of 5-30°C, water temp as % of 40-80°C — same "arc is always a % of a sensible range" rule as `m5dial_fram/design_rules.md` §8, since there's no natural 0-100% for a temperature otherwise) plus the touch-adapted +/- buttons (no arm/disarm step — there's no rotary encoder here to contend with page navigation over). Fan level (`select.womolin_controller_mqtt_fan_mode`) is shown read-only on the room-heating side — the write side (`select.select_option`) needs the entity's real option strings, which nobody has confirmed yet; don't guess them the way `select.multiplus`'s options were already confirmed real (see the Electric row below). A shared status/error footer (Truma CP Plus connectivity, error code, operating status) spans the bottom of the page once, not duplicated per half. | — |
| 5 | Fans | — | Fan board speed + Sprinter HVAC fan, same two entities as `m5dial_fram/page_fans.yaml` | HA |
| 6 | Light/Features | — | Floor-plan view (§8) — room lights, lock/unlock, step, awning, iPixel LEDs | HA |
| 7 | Sensors | 1: Sensors, 2: Levelling | Page 1: door/window/presence contacts, grid layout per `smartebl_display/docs/design.md`'s "Status Grid Overview" template; page 2: spirit level + per-corner cm-to-add, same 4 entities `m5dial_fram/page_levelling.yaml` reads | HA |

Not on this display (per explicit user decision): a pre-flight-check
overlay (`m5dial_fram`'s `overlay_preflight`) — not needed here, that
device stays the one place for it.

## 6. Home page detail

Right half: `sbb_clock` widget, sized to fill it (the widget scales by
`width`/`height` alone — see `m5dial_clock_sbb`'s README, no changes
needed to reuse it here at a much larger size than the M5Dial's 240×240).
No date/day shown (`show_date` left at its default `false`) — asked for
explicitly, also a small mercy on the PSRAM-canvas redraw cost the
README's roadmap already flags.

Left half / remainder: tiles, one per switch, as a 2×3 grid **vertically
centered in the content area and horizontally centered between the nav
rail and the clock** (repo-owner-specified layout, not the reading
order of the table below):

```
LIGHT   ICE-EX
AUX     FRIDGE
POWER   PUMP
```

Table below still lists which tile maps to which switch, just not the
on-screen position:

| Tile | Source | Notes |
|---|---|---|
| Power (12V) | `switch_12v` (link) | |
| Aux | `switch_aux` (link) | |
| Light | `switch_light` (link) | |
| Pump | `switch_pump` (link) | already implemented in `smart-ebl-display.yaml` today |
| IceEx | `eisex_output` (link) | |
| Fridge | TBD | **Reserve the slot regardless of outcome** — user is still checking whether this vehicle wires the fridge to its own controllable line (`smartebl` itself supports it, F9/F10). If not: render the tile in the `0x555555` invalid-value style permanently rather than removing it, so the layout doesn't shift later once it is wired up |
| Alarmo | HA (`alarm_control_panel` or similar, entity TBD) | Tap → PIN popup (numeric keypad) → arm/disarm. Deliberately **not** in the quick-switch popup (§3) and **not** in the status bar (§4) — security action, home page only, confirmation required (same "confirmation for dangerous actions" reasoning as `m5dial_fram/design_rules.md` §14's `MP OFF`/`RETRACT` item) |

## 7. Open items

- **Alarm priority ranking across categories.** §4's status bar shows
  exactly one message at a time, same as `smartebl_display`'s original
  design — but that design only ever had to rank alarms *within* one
  category. Centralizing fuses/battery/tanks/gas/sensors/levelling into
  one bar needs an explicit cross-category ranking, which doesn't exist
  yet. Draft, unconfirmed — needs a decision, not a guess:

  1. Fuse/electrical critical
  2. Waste tank full (damage)
  3. Gas empty (both bottles < 10%, combined-supply logic)
  4. Starter battery critical (< 12.0 V)
  5. Fresh water empty (inconvenience only)
  6. Door/window open while driving
  7. Levelling not set (irrelevant while driving; lowest, if shown at all while in motion)

- **Backlight PWM pin unconfirmed** (§4) — blocks the brightness slider
  from doing anything real. The user-supplied
  [Waveshare wiki page for this exact panel](https://www.waveshare.com/wiki/10.1-DSI-TOUCH-A)
  is the next thing to check once off this session's network block —
  `docs/hardware.md` should get the finding, not this file.

- **Light/Features floor plan — source image landed.**
  `docs/assets/floorplan-source.jpg` is now in the repo (rear twin-bed
  layout, corner wet bath, wardrobe, rear lounge bench, kitchenette
  bottom-center, second lounge seat, cab up front). Still needed before
  a real overlay can be built: the actual HA entity_ids behind each zone
  below (none confirmed yet — this table is a photo-based guess at which
  rooms have a switchable light at all, not a wired-up list), and pixel
  coordinates on the image once someone sits down with it in an editor.
  Zone breakdown from that photo, rear to front:

  | Zone | Likely features |
  |---|---|
  | Rear twin beds | Reading/ambient light(s) |
  | Wet bath (toilet/shower, mid-rear) | Bathroom light (+ fan, if separately switched) |
  | Wardrobe (next to wet bath) | none assumed |
  | Rear lounge bench (opposite wardrobe) | Ambient/reading light |
  | Kitchenette (bottom-center: sink + 2-burner hob) | Kitchen light |
  | Second lounge seat (front of kitchenette) | Ambient light |
  | Cab (swivel captain's chairs, front) | Cab light, if separately switched |
  | Entrance (side door — position not visible in a top-down interior shot) | Step, lock |
  | Exterior (not in this photo at all) | Awning light, outside light, iPixel LEDs |

  All of the above are Home Assistant entities (Shellys etc.), **not**
  `docs/protocol.md` keys — `smartebl` only exposes one combined `Light`
  relay group (F4-F5), not per-room switching. This page therefore
  depends on Wi-Fi/HA being up, unlike the home page's 5 switches, which
  keep working over the RS232 link with HA down. Worth a small
  "no HA connection" state on this page specifically, distinct from the
  link-down state the rest of the display uses.

- **Home page layout with the nav rail.** Open whether the left nav rail
  repeats on the home page (consistency) or the home page is the one
  place it's hidden (more room for the switch tiles). Decide once a
  mockup exists.

- **Sensors page 1 content.** Which specific door/window/presence
  entities exist and how many fit the grid template isn't enumerated yet
  — needs the actual HA entity list before that page can be more than a
  placeholder.
