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

**Implemented** (2026-09-06): tap opens a popup (`obj_daynight_popup_panel`,
top_layer) with a brightness slider + an AUTO/DAY/NIGHT selector.

- **Day/Night color scheme** (dark-on-black ↔ light, the same swap the
  `sbb_clock` widget already does via `set_night_mode()`) is software-only
  and ships regardless of hardware status. Automatic default: `sun.sun`
  from Home Assistant (`s_sun`), the same source `m5dial_fram`'s clock
  page uses — there's no confirmed ambient-light sensor on this board.
  The popup's AUTO/DAY/NIGHT selector (`act_daynight_set_mode`) overrides
  this per user choice, persisted (`g_daynight`, `restore_value: true`).
- **Brightness slider is built, but still has nothing real to drive**:
  `docs/hardware.md` still leaves `reset_pin`/`enable_pin` unconfigured on
  this board — no PWM-capable backlight GPIO confirmed yet. The slider
  and its backing global (`g_backlight_pct`, persisted) are in place;
  `apply_backlight` is the one place a real `output:`/`ledc` write
  belongs once that pin is found (see §7's open item — the user-supplied
  Waveshare wiki page for this exact panel is the next place to check
  once this session's network block on `waveshare.com` doesn't apply).

### Sleep mode

**Implemented** (2026-09-06), per repo-owner request: between 22:00 and
08:00, the screen blanks after 10 minutes with no touch anywhere on the
panel (`check_sleep_idle`, driven by `touchscreen: on_touch:`'s activity
timer) — any touch wakes it immediately. A small manual SLEEP button next
to the day/night one (`toggle_sleep_manual`) blanks on demand at any time,
not just inside the window. Simulated with a full-screen black overlay
(`obj_sleep_overlay`, top_layer, drawn last so it also catches the waking
tap itself) rather than a real backlight-off, for the same unconfirmed-pin
reason as the brightness slider above.

## 5. Page catalog

Order is provisional (unlike `m5dial_fram`, there's no rotary encoder
address depending on it — a touchscreen can jump to any tab directly), but
listed in the nav rail's intended top-to-bottom order.

| # | Section | Sub-pages | Content | Data source |
|---|---|---|---|---|
| 0 | Home | — | Clock (`sbb_clock`, right half) + 5 switch tiles + Fridge tile + Alarmo tile (§6) | link (switches) + HA (clock temp) |
| 1 | Electric | 1: Overview, 2: Fuses | Victron-style power flow (Shore→Inverter→AC Loads / Solar→Battery→DC Loads) per `smartebl_display/docs/design.md`'s existing mockup — reuse directly, don't redesign; page 2: 16-fuse grid, `fuse_fN_ok = voltage > 1.0 V` per `design_rules.md` §2 | link (`docs/protocol.md` groups 1, 3) |
| — | *implemented, page 1* | — | **Superseded finding, corrected:** this repo initially assumed the vehicle had no Victron gear behind `smartebl`, so AC Loads/Solar Yield/DC Loads rendered permanently invalid. The repo owner confirmed a real SmartShunt (battery monitor) and MultiPlus (inverter/charger), integrated into HA via the exact same entities `m5dial_fram`'s `page_power_1/2/3.yaml` already use — now wired: **Battery** box shows SmartShunt SOC%/voltage/current (same 20%/40% thresholds as `page_power_1`); **Inverter/Charger** box (now a button) shows the MultiPlus's real mode (`select.multiplus` — ON/CHG/INV/OFF) and state (`sensor.multiplus_zustand`), tapping it opens a popup to set the mode directly and adjust the shore/grid current limit (`number.multiplus_strombegrenzung`, clamped 0-16A per `page_power_2`'s own real-wiring-ceiling note); **AC Loads** shows the MultiPlus's WR output power (`sensor.multiplus_0_wr_leistung`) as a % of the installed model's 1600W nominal rating, same caveat as `page_power_3`'s identical calculation (not exact, thermal derating untracked). **Solar Yield and DC Loads also confirmed real and wired**: an MPPT 190W solar charge controller (`sensor.mppt_190w_leistung_pv_ertrag` for the yield figure, `sensor.mppt_190w_zustand` for its status line — same "main value + humanized state" shape as the Charger box) and the GX device's own aggregate DC-consumption sensor (`sensor.gx_device_dc_verbrauch`) — not the SmartShunt's net current shown in the Battery box, a genuinely separate figure. Shore connected/disconnected stays `smartebl`'s own link reading (`shore_power_connected`) — that detection is real and cheap regardless. Page 2 (fuse grid) is not built yet. Page also recentered (both horizontally and vertically within the content area) per repo-owner feedback that it wasn't. | — |
| — | *reworked, page 1 (2026-09-06)* | — | Per repo-owner spec: boxes grew (row heights 130/170px → 210/290px, matched top/bottom margins - the page used to leave most of its content area empty). **Shore**'s main value is now the MultiPlus's real input power (`sensor.multiplus_eingangsleistung_l1`), sub-left/right voltage/frequency (`_eingangsspannung_l1`/`_eingangsfrequenz_l1`) - replaces the old combined W+A sub-line; connected/disconnected (still `shore_power_connected`, smartebl's own link) is now its own status line above the main value instead of replacing it. **AC Loads** mirrors that shape with output power/voltage/frequency (`sensor.multiplus_ausgangsleistung_l1`/`_ausgangsspannung_l1`/`_ausgangsfrequenz_l1`) - replaces the old %-of-1600W-nominal figure (`sensor.multiplus_0_wr_leistung`), though the same 80%/95% warning coloring carries over against the new sensor. **Solar Yield** is now a button, opening its own popup (same shape as the Charger one) for ON/OFF (`switch.mppt_190w`) and charge-current limit (`number.mppt_190w_stromstarke`, no confirmed real ceiling yet - see the popup's own comment); sub-left/right are now the MPPT's own DC battery-bus voltage/current (`sensor.mppt_190w_dc_batterie_bus_spannung`/`_stromstarke`), replacing the humanized-zustand sub line. **Battery** gained an IDLE/CHARGING/DISCHARGING line (interpreted from the SmartShunt's net current, `sensor.smartshunt_dc_bus_stromstarke`, ±0.5A treated as idle) and a remaining-time line (`sensor.smartshunt_verbleibende_zeit`, shown only while discharging) between the SOC and a proper 3-across V/A/W bottom row (`smartshunt_dc_bus_spannung`/`_stromstarke`/`sensor.smartshunt_leistung`) - fixes a real overlap bug between the old combined mid-line and the starter-voltage line at the bottom; also gained its own popup for `switch.fram_parkmodus`. Starter battery voltage is unchanged (still `smartebl`'s own link reading, at the very bottom). **DC Loads**' main value is still a placeholder - repo owner: "tbd", no entity picked yet. Also fixed on the Charger popup: mode buttons were left-to-right ON/CHG/INV/OFF, now OFF/CHARGER/INVERTER/ON per repo-owner spec; the LIMIT label drifted off-center as its digit count changed for want of `text_align: CENTER`, now fixed; the OFF button's own "active" highlight used to be styled identically to "inactive". **Open question, not yet answered:** Victron's own GUIv2 Inverter/Charger detail screen has a vertical bar this page has no equivalent for - repo owner asked what it represents; best unconfirmed guess is the AC input current as a % of the configured input current limit (`sensor.multiplus_eingangsstromstarke_l1` vs. `number.multiplus_strombegrenzung`, the same limit already wired into this popup) - needs checking against a real GX Touch/VRM before building anything against it. | — |
| — | *implemented, page 2 (`page_electric_fuses`)* | — | **Fuse grid built** - all 16 (`fuse_fN_ok = voltage > 1.0 V`, group 1, real on both ends now - `smartebl` already broadcast it, this display just hadn't parsed it). Reached by re-tapping the ELECTRIC nav icon while already on Electric (§2's "1/2" cycle, implemented for real here for the first time). Real fuse names/functions from `smartebl`'s own hardware table (CLAUDE.md), not the old Nextion mockup's generic placeholders. One refinement beyond the naive `fuse_fN_ok` rule: these ADCs read the LOAD side, so a switched group (Light/12V/AUX/Pump) reads near 0V any time its own switch is simply off - `draw_fuses` cross-references each switched fuse against that group's own switch mirror (already on this display via the link) so "AUS" (switch off, expected) and "AUSGEFALLEN?" (should be live per its own switch, isn't - a real fault) read as two different things instead of one ambiguous red dot. F2/F3 (vehicle/D+-controlled) and F10 (fridge D+/programmable) have no switch mirror on this display to check against - shown read-only, never alarmed. | — |
| 2 | Levels | — | **Implemented (`page_levels`), exactly as originally cataloged** - Fresh (`tank1_level`) / Waste (`tank2_level`) over the link, same 10%/25%/80%/95% thresholds as `m5dial_fram/page_water.yaml`'s fresh/grey rings (must not disagree with the M5Dial, design_rules.md §2); Gas A (außen/"in use") / Gas I (innen/"reserve") - HA, the exact same real `sensor.gas_1_stand`/`sensor.gas_2_stand` entities `m5dial_fram/page_gas.yaml` already reads, same precedent as the Electric page reusing `m5dial_fram`'s real Victron entities; Diesel - HA, `sensor.nw19939_fuel_level` (repo-owner-supplied), assumed already a 0-100% reading like the other four - flag if that's a raw liter value instead. An earlier round of this page substituted `tank3_level` for Gas A/Gas I/Diesel, reasoning that smartebl's hardware has no dual-bottle gas sensor or fuel-tank input of its own - correct about the hardware, wrong call: the repo owner wanted the original 5-column catalog kept regardless, sourcing what smartebl can't from HA (the same pattern the Electric page already established). `tank3_level` stays wired (real, `smartebl` already broadcasts it) but isn't shown on any page - reserved, same status `tank1`/`tank2` themselves had before this page existed. Gas combined-supply logic reused verbatim from `m5dial_fram/design_rules.md` §6 (`GAS LEER` only when **both** < 10%); Diesel's own 10%/25% thresholds are an unconfirmed guess, same shape as Fresh's. Rendered as vertical LVGL `bar` widgets (fills bottom-up by construction at width<height, no hand-rolled segment geometry needed) rather than smartebl_display's original mockup's hand-drawn segmented bars - same "Victron-style vertical tank column" idea, LVGL-native implementation. Tank alarms surface in the status bar concept (§4) at the cross-category level - not implemented yet, see §7 - and a page-local combined precedence in the meantime, not a page footer (deviation from `m5dial_fram`'s per-page-footer convention, deliberate per §1 above). | link (Fresh/Waste) + HA (Gas A/I, Diesel) |
| 3 | Climate | — | Truma Combi 4 room-heating side: current/target temp, on/off, via the `womolin_controller` MQTT integration — same entities `m5dial_fram/page_climate.yaml` already uses | HA |
| 4 | Boiler | — | Truma Combi 4 water-heating side: 3 fixed tiers (ECO/mid/BOOST), same `womolin_controller` integration as Climate | HA |
| — | *implemented, merged* | — | **Per repo-owner request, Climate and Boiler share ONE page and ONE nav entry ("TRUMA")** — left half Heizung (room), right half Boiler (water), not two separate sections as originally cataloged above. Each half now has an arc (room temp as % of 5-30°C, water temp as % of 40-80°C — same "arc is always a % of a sensible range" rule as `m5dial_fram/design_rules.md` §8, since there's no natural 0-100% for a temperature otherwise) plus the touch-adapted +/- buttons (no arm/disarm step — there's no rotary encoder here to contend with page navigation over). Fan level, room-heating side only: turned out to be `climate.womolin_controller_mqtt_truma_room`'s own `fan_mode` attribute (a standard HA climate-domain concept, not a separate `select.*` entity as first assumed) — confirmed real values `off`/`low`/`medium`/`high`, read AND written now (`climate.set_fan_mode`) via a tap-to-cycle button, same single-button-cycle shape as the Electric page's `select.multiplus` MODE button used before it grew into a 4-button popup. A shared status/error footer (Truma CP Plus connectivity, error code, operating status) spans the bottom of the page once, not duplicated per half. | — |
| — | *renamed/relaid-out (2026-09-06)* | — | Per repo-owner spec: on-screen headings shortened from "HEIZUNG (RAUM)"/"BOILER (WASSER)" to just "HEIZUNG"/"BOILER". The "SOLL nn°C" target line moved from a single line above the +/- buttons to two lines ("SOLL" / "nn°C", plus a third "ECO"/"BOOST" line on the Boiler side only) centered in the gap BETWEEN the +/- buttons instead. | — |
| 5 | Fans | — | **Implemented (`page_fans`)** - FANBOARD setpoint `number.fan_board_fan_speed` + its own 4 individual fans (`sensor.fan_board_fan_speed_1`..`_4`, read-only readout row under the FANBOARD arc), HVAC `number.relay_2ch_hvac_hvac_fan_battery` - real entity_ids confirmed by the repo owner (2026-09-06), correcting this page's earlier `m5dial_fram`-borrowed guesses (`number.fan_speed_control`/`number.hvac_fan_battery`). Same two-halves-with-a-divider layout as the Truma page rather than that file's concentric double-ring (round-panel-specific, design_rules.md §3). No rotary-encoder "arm" step to port - each half's own +/- buttons write directly, same touch-adapted debounced-write shape `page_climate`'s +/- buttons already established on this display. FANBOARD steps 5%, HVAC 10% (matches each entity's own `step:`), both clamped 0-100. | HA |
| 6 | Light/Features | — | Floor-plan view (§8) — room lights, lock/unlock, step, awning, iPixel LEDs | HA |
| 7 | Sensors | 1: Sensors, 2: Levelling | Page 1: door/window/presence contacts, grid layout per `smartebl_display/docs/design.md`'s "Status Grid Overview" template; page 2: spirit level + per-corner cm-to-add, same 4 entities `m5dial_fram/page_levelling.yaml` reads | HA |
| — | *implemented, page 2 only (`page_sensors_levelling`)* | — | **Levelling built**, ported from `m5dial_fram/page_levelling.yaml` verbatim (same bubble-target-rings metaphor, same worst-corner-wins bubble color, same still-open item there: the four corner cm sensors read 0 until an HA-side template sensor does the real degrees-to-cm math - nothing to change here once that lands). Read-only, no button, matches that file. **Page 1 (`page_sensors_overview`) is an honest placeholder** - §7's own open item (no door/window/presence entity list yet) is unchanged; the page exists (with the SENSORS nav entry + its own "1/2" indicator, §2's pattern, implemented for real here alongside Electric's) so Levelling is reachable without waiting on page 1's content. | HA |
| — | *relayout, page 2 (2026-09-06)* | — | Per repo-owner spec ("das Widget ist VIEL zu klein! eine Seite der freien Flaeche benutzen. Die andere Haelfte ist fuer die Huebstuetzen-Kontrolle reserviert"): split at x=705 (same left/right convention as the Fans page's own divider). **Left half:** the same bubble-level widget, everything doubled (200x200 root -> 400x400, every ring/bubble/offset x2 with it) so it actually fills its half instead of floating small in the middle of the whole content area. **Right half:** Huebstuetzen (leveling jacks) - repo-owner-confirmed the jacks aren't physically installed in this motorhome yet, so this is a genuinely **reserved placeholder** (same treatment as the Home page's own Fridge tile: permanently invalid/grey style, no live entity, no on_click) rather than a panel wired against guessed entity_ids that would just log HA warnings for nothing. Laid out for the control the repo owner described for once the hardware exists - one ON/OFF switch, one AUTOLEVEL button, a status indicator (green = eingefahren/retracted, red = ausgefahren/extended) - "maybe others", so not assumed final. An earlier round of this file built a fully-wired per-corner AUSFAHREN/EINFAHREN panel against placeholder entity_ids including a retract-confirm popup (`m5dial_fram/design_rules.md` §14's long-open "Confirmation for dangerous buttons" item) - reverted once the repo owner confirmed the hardware doesn't exist; that confirm-popup pattern is still the right one to reuse once real entities land. | HA |

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
| Alarmo | HA (`alarm_control_panel.alarmo` — **implemented** 2026-09-06, see below) | Tap → PIN popup (numeric keypad) → arm/disarm. Deliberately **not** in the quick-switch popup (§3) and **not** in the status bar (§4) — security action, home page only, confirmation required (same "confirmation for dangerous actions" reasoning as `m5dial_fram/design_rules.md` §14's `MP OFF`/`RETRACT` item) |

**Implemented** (2026-09-06, repo-owner: "können wir noch irgendwo die Alarmanlage integrieren?"): exactly the design above, built for real. `alarm_control_panel.alarmo` is the "Alarmo" HACS integration's own default single-instance entity id — repo-owner-confirmed correct, along with which of Alarmo's six possible states this installation actually uses: **Armed home, Armed away, and Disarmed only** (not night/vacation/custom-bypass). The 2x3 switch-tile grid (§6 above) had no free slot for a 7th tile, so Alarmo's tile sits below the clock instead (the x815-1135/y605-755 strip that was empty), showing a humanized state (AUS/SCHARF H/SCHARF A/ALARM/...) colored the same green/orange/red scheme as everything else on this display. Tapping it opens a numeric keypad popup (`obj_alarm_popup_panel`) with **three separate action buttons instead of one generic "OK"** — ARM HOME/ARM AWAY (both hidden while already armed) and DISARM (hidden while disarmed) — each a fixed `alarm_control_panel.alarm_arm_home`/`alarm_arm_away`/`alarm_disarm` service call with the entered code (a service call's own service: string isn't templatable in ESPHome, same reason the Electric page's MultiPlus mode popup uses one button per option rather than templating the service). CLR clears the entry; the popup clears its own PIN buffer on open/close/submit so a code never lingers on screen.

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

- **Sensors page 1 content.** Category confirmed (2026-09-06, repo
  owner): **door/window contacts** — presence/motion, water leak, and
  smoke/CO were offered and not selected. The page (`page_sensors_overview`)
  now has a shaped 6-cell placeholder grid (same dot+cap+state cell as
  `page_electric_fuses.yaml`'s real fuse grid, `smartebl_display/docs/design.md`
  Template 2) instead of just a caption, but every cell is still greyed
  out with no entity behind it — which specific doors/windows exist and
  how many actually fit the grid isn't enumerated yet, needs the real HA
  entity list (device_class `door`/`window` binary_sensors) before this
  is more than a mockup. This item is now "wire real entities into the
  existing 6 cells (resizing the grid if there end up being more or
  fewer than 6)", not "design the page".
