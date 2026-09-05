# Smart EBL Link Protocol

As of: 2026-09-05. Applies to the RS232 wire between `smartebl` (main unit)
and this display. If a device's behavior deviates from this document,
either the device is wrong or this document is outdated — either way it
must be corrected, not ignored (same house rule as
`m5dial_fram/design_rules.md`, adopted here on purpose for consistency
across the project).

**Status: this document specifies both ends. Only the display end
(this repo) is implemented.** `smartebl` is read-only from this
session — its half (broadcasting telemetry, listening for command
frames, exactly as specified below) is separate follow-up work in that
repo. Nothing here describes code that exists in `smartebl` today.

## 1. Why a custom protocol, not Modbus/MQTT/native API

- Both devices are ESPHome, but they are not the same device anymore —
  the previous generation (`smartebl_display`) worked because the
  Nextion display shared `smart-ebl.yaml`'s entities directly; this one
  is genuinely a second MCU on the far end of a wire.
- No network hop is acceptable on this path (the whole point is
  control without Wi-Fi/HA in the loop), which rules out ESPHome's
  native API.
- The link is a single point-to-point RS232 pair, not a shared bus —
  there is no addressing or arbitration problem to solve, which is most
  of what Modbus RTU exists for. A small line protocol is simpler to
  implement on both ends in plain ESPHome `lambda:` blocks (no custom
  C++ component, no external library) and easy to read on a serial
  monitor while debugging the harness.

## 2. Physical layer

- UART, 115200 baud, 8N1 — same baud rate `smartebl` already uses for
  `display_uart` (GPIO18/19 through the TRS3221 RS232 transceiver), so
  the existing cable and its electrical characteristics don't change.
- **The display side needs its own RS232-to-3.3V-TTL transceiver**
  between the RJ45 jack and the ESP32-P4's UART pins — see the
  README's "RS232 link" section. Wiring the P4's 3.3V GPIOs straight to
  an RS232-level line risks damaging them.

## 3. Framing

One frame per line, terminated by `\n` (a trailing `\r` before it is
tolerated and stripped, not required). Two frame types, identified by
the first character:

```
T<group>;key=value;key=value;...;*XX\n     -- telemetry (smartebl -> display)
C;key=value;*XX\n                          -- command   (display -> smartebl)
```

- `<group>` (telemetry only): a single digit, 1–6, see §5. Lets the
  display update only the section that just arrived instead of
  invalidating everything on every line, and lets `smartebl` round-robin
  through groups instead of building one very long line every cycle.
- `key=value` pairs, separated by `;`, no whitespace. `key` is always
  one of the entity ids from §5's tables — the exact same ids used in
  `smartebl`'s own `smart-ebl.yaml`, on purpose, so there is exactly one
  name for a value across the whole project.
- `value` is ASCII: a decimal number for `float`/`bool` (bool as `0`/
  `1`), or a bare string for `string`-typed keys (select options,
  error codes) — must not itself contain `;`, `=`, or `*`. `smartebl`'s
  Truma select options are plain words today (no punctuation), so this
  isn't expected to bite, but a future option string that does contain
  one of those characters needs re-encoding (not addressed here).
- `*XX`: an 8-bit XOR checksum of every byte in the line **before** the
  `*`, as two uppercase hex digits. A receiver that doesn't match drops
  the whole line silently — never acts on a corrupted frame, never
  crashes on one either. Deliberately simple (not CRC8/16): this is a
  short, shielded, point-to-point wire, not a noisy multi-drop bus: XOR
  catches single-byte corruption, which is the failure mode that
  actually matters here.
- Max line length 480 bytes. A receiver that hits it without seeing a
  `\n` clears its buffer and resyncs on the next line rather than
  growing without bound — protects against a wiring fault or a garbled
  boot-time UART glitch turning into an unbounded allocation.
- Unknown keys are ignored, not an error. This is what makes the
  protocol forward-compatible: a display running an older key table can
  sit on the same wire as a `smartebl` that has grown new sensors,
  and vice versa, without either side refusing to parse.

## 4. Timing & staleness

- `smartebl` broadcasts one group every ~200 ms, cycling 1→6→1→... — a
  full refresh of every group takes ~1.2 s, in line with the previous
  Nextion generation's 500 ms `update_interval` for the values that
  mattered most (tanks, pump), faster for the rest.
  Group priority — how often each group's turn comes up — is a
  `smartebl`-side tuning knob (e.g. group 4, switch states, changing on
  every user action, could get 2 slots per cycle instead of 1); nothing
  on the display side assumes a fixed cadence beyond "if I haven't
  heard a group in N seconds, its values are stale."
- The display tracks the last-received time per group (or globally, in
  the current basic implementation — see §7). If nothing arrives for
  3 s, every value fed by this link falls back to the same `---`
  invalid-value convention as `m5dial_fram/design_rules.md` §7 — a
  disconnected link must never keep showing the last good number.
- Command frames are fire-and-forget — no ack, because RS232 is
  already always-connected point-to-point (nothing to negotiate a
  session over). `smartebl` re-broadcasts the switch's *actual* new
  state in the very next group-4 telemetry frame regardless of whether
  the command took effect, so the display's UI always converges to
  ground truth within one cycle even if a command frame is lost —
  mirroring the "optimistic locally, corrected by the next real update"
  pattern already used by `m5dial_fram`'s adjustable-value handling.

## 5. Telemetry groups & keys

Every id below is exactly the id/name already defined in `smartebl`'s
`smart-ebl.yaml` — cross-reference that file, don't rename here.

**Implemented** = wired into `smart-ebl-display.yaml`'s frame parser
today. **Planned** = specified, not yet wired on the display side —
add an `else if` in the same parser function, following the existing
entries, per the README's roadmap.

### Group 1 — Fuses (16 keys, all `float`, volts)

`fuse_f1_voltage` … `fuse_f16_voltage`. Planned (none implemented yet —
the derived `fuse_fN_ok` binary sensors are display-side computation,
`> 1.0 V`, not separate wire keys, same as `smartebl` computes them
itself: no need to send both the raw value and its derived boolean).

### Group 2 — Tanks (`float`, %)

| key | status |
|---|---|
| `tank1_level` | **implemented** |
| `tank2_level` | **implemented** |
| `tank3_level` | planned |

### Group 3 — Electrical (`float`, volts unless noted)

| key | status |
|---|---|
| `leisure_battery_voltage` | **implemented** |
| `starter_battery_voltage` | **implemented** |
| `light_output_voltage` | planned |
| `user_12v_output_voltage` | planned |
| `aux_12v_output_voltage` | planned |
| `pump_output_voltage` | planned |
| `light_l_output_voltage` | planned |
| `contour_light_output_voltage` | planned |
| `voltage_3v3_bus` | planned |
| `ac_charger_voltage` | planned |
| `duocontrol_voltage` | planned |
| `dplus_voltage` | planned |
| `f10_abs_voltage` | planned |
| `dplus_active` (`bool`, derived on `smartebl` side same as today) | planned |
| `shore_power_connected` (`bool`) | planned |

### Group 4 — Switches (`bool`, also the command target)

| key | status |
|---|---|
| `switch_pump` | **implemented** (mirror + command) |
| `switch_light` | planned |
| `switch_12v` | planned |
| `switch_aux` | planned |
| `eisex_output` | planned |
| `dplus_relay` | planned |
| `fridge_control` | planned |
| `tank1_dc_control` / `tank2_dc_control` / `tank3_dc_control` | planned |

### Group 5 — Truma

| key | type | status |
|---|---|---|
| `truma_room_temp` | float, °C | planned |
| `truma_water_temp` | float, °C | planned |
| `truma_error_code` | string | planned |
| `truma_connected` | bool | planned |
| `truma_has_error` | bool | planned |
| `truma_target_room_temp` | float, °C — also a command target | planned |
| `truma_target_water_temp` | float, °C — also a command target | planned |
| `truma_fan_mode` | string — also a command target | planned |
| `truma_energy_mix` | string — also a command target | planned |

### Group 6 — System

| key | type | status |
|---|---|---|
| `esp32_temperature` | float, °C | planned |
| `uptime` | float, seconds | planned |
| `proto_version` | string, e.g. `"1"` — see §6 | planned |

## 6. Versioning

`proto_version` (group 6) exists from day one so a future breaking
change to the framing itself (not just an added key, which §3 already
handles for free) has somewhere to signal from. No consumer needs to
check it yet — there is only version "1".

## 7. Command frames (display → smartebl)

```
C;switch_pump=1;*7F\n
```

One key=value pair per command frame (multiple pairs are valid framing
per §3 but there's no current reason to batch a command). `smartebl`
applies it as if the corresponding switch/number/select entity had been
set locally — `switch_pump=1` behaves exactly like calling
`switch.turn_on: switch_pump` in that yaml, `truma_target_room_temp=21.5`
like a `number.set_value` call, `truma_fan_mode=Auto` like a
`select.set_value` call.

Implemented on the display side today: `switch_pump` (both directions
— see `smart-ebl-display.yaml`'s `send_link_command` script). Every
other row marked "also a command target" above is planned the same way
the corresponding telemetry key is.

## 8. What `smartebl`'s counterpart needs to do

(Specification only — not implemented, see the top of this document.)

1. Own a second `uart:` bus (or reuse `display_uart` now that it's a
   generic link rather than Nextion-specific — either is a `smartebl`
   design decision, not this repo's to make).
2. A round-robin `interval:` that composes and writes one group's
   telemetry line per tick, in the format of §3, over the keys of §5 —
   same `lambda:` style already used throughout `smart-ebl.yaml` for
   binary-sensor thresholds.
3. A line reader (same byte-accumulate-until-`\n` shape as this repo's
   own reader, see §9) that recognizes `C;key=value;*XX` frames,
   validates the checksum, and dispatches known keys to the matching
   `switch.turn_on`/`turn_off`, `number.set_value`, or
   `select.set_value` action.

## 9. Reference implementation (display side)

`smart-ebl-display.yaml` implements §3's reader as a 20 ms `interval:`
lambda that byte-accumulates into a `std::string` global, and §7's
writer as a parameterized `script:` (`send_link_command`) that composes
the line and its checksum and calls `uart::UARTDevice::write_str`.
Extending either to a new key is adding one more `else if` / one more
call site — see that file's comments at the reader/writer for the exact
shape to copy.
