# Device API: real values in, named parts out

**Status:** 2026-10-04, design (proposal). For the wire protocol's commands and telemetry
([wire protocol](wire-protocol.md)); Stackchan's Embody Mode first, the car and other devices
after it. Early stage: names change without keeping the old ones ([principles](principles.md)
27).

Two rules, after Linux:

1. **Sensors report real values**, in units, the way Linux IIO and hwmon do: a reading is a
   number with its unit in its name. Meaning ("someone is near", "Ema is home") stays on the
   node ([principles](principles.md) 2).
2. **Actuators are named parts with attributes**, the way `/sys/class/leds/<name>/brightness`
   is: `left1` is the first LED strip on the left, `yaw` the head's turning servo. A command
   says which part and what to set; the device knows how.

## 1. Sensors: values with units

| Rule | Example |
|---|---|
| The unit ends the name | `light_lux`, `chip_temp_c`, `battery_v`, `wifi_rssi_dbm`, `head_yaw_deg`, `imu_gyro_dps` |
| SI or what people read, one per quantity | `_c` (°C), `_v`, `_a`, `_w`, `_pct`, `_deg`, `_dps` (°/s), `_g`, `_ut` (µT), `_mm`, `_lux`, `_hz`, `_db` |
| No physical unit exists: `_raw`, with its scale or range in the capability descriptor (IIO's `in_*_raw` + `in_*_scale`) | `proximity_raw` (0–2047, LTR-553 counts: closer is higher, no distance calibration) |
| Several of a kind: a number after the part | `temp1_c`, `temp2_c` (hwmon's `temp1_input`) |
| States as numbers 0/1, never strings | `charging`, `proximity_on` |

Today almost all Stackchan telemetry already follows this. To change: `proximity` →
`proximity_raw`; `mag_raw_x/y/z` and `mag_rhall` drop (the `mag_*_ut` values stay; the raw ones
were for checking the BMM150 conversion, done).

## 2. Actuators: named parts

A device lists its parts when it registers (in the capability descriptors the wire protocol
plans, [wire protocol](wire-protocol.md) §9), so an app, a loop or an AI agent can use a new
device without code written for it:

```json
"parts": {
  "led":   {"left1": {"pixels": 6, "color": "rgb"}, "right1": {"pixels": 6, "color": "rgb"}},
  "servo": {"yaw": {"min_deg": -128, "max_deg": 128, "rotate": true}, "pitch": {"min_deg": 5, "max_deg": 85}},
  "display": {"display1": {"width": 320, "height": 240}},
  "speaker": {"speaker1": {}}
}
```

**Names:** a place and a number, counted from 1 in a fixed order the device documents (`left1`,
`left2`, `right1`, `front1`, `head1`); a part that is one of a kind may have a plain name (`yaw`,
`pitch`, `display1`). Linux names LEDs `device:color:function`; the device is implied here (the
command goes to one device), the colour is an attribute.

**One command per kind of part, the part by name, attributes as in sysfs:**

| Command | Arguments | Linux model |
|---|---|---|
| `led` | `name` (`left1`, or `*` for all), `color` (`#rrggbb`), `brightness_pct`, `pixel` (1–n) or `pixels` ([…]), `trigger` (`none`, `timer`, `heartbeat`, `rainbow`, `chase`) with `period_ms`, `seconds` | `/sys/class/leds/<name>/{brightness,trigger,delay_on}` |
| `servo` | `name` (`yaw`), `angle_deg`, `speed_dps`, or `velocity_pct` (continuous, where `rotate` is listed), `torque` (on/off) | — (IIO output channels) |
| `display` | `name`, `brightness_pct`, `power` (on, blank, off) | `/sys/class/backlight/<name>/brightness` |
| `speaker` | `name`, `volume_pct` | ALSA mixer |

Effects are triggers, as in Linux: the device runs them itself (25 fps on the robot), the
server only names them.

## 3. Higher-level commands stay where timing matters

Some commands carry logic that must run on the device, because the network is too slow or may
drop: `look {yaw_deg, pitch_deg}` (smooth motion), `nod`, `shake`, `rotate` (with the stall stop
and the confirmation on the robot), the car's watchdog stop. They stay as commands of their
own, next to the part commands, and say so in their descriptor ("motion", "safety: confirmed
on the device"). Everything that only needs meaning (when to nod, what a reading means) is on
the node, in apps and loops.

## 4. From today's names

| Today | Proposed |
|---|---|
| `leds {left, right, pixels[0–11], effect, color, speed, seconds}` | `led {name: left1 \| right1 \| *, color, pixels[1–6], trigger, period_ms, seconds}` |
| `brightness {value \| auto}` | `display {name: display1, brightness_pct \| auto}` |
| `volume {value}` | `speaker {name: speaker1, volume_pct}` |
| `look`, `nod`, `shake`, `home`, `hold`, `rotate` | stay (§3); `hold` → `servo {name: *, torque: on, seconds}` |
| `servo_power {on}` | `servo {name: *, power: on \| off}` |
| `car_headlights {color}` | on the car: `led {name: front1 \| front2, color}` |
| `car_drive {left, right}` | stays (a safety command with the watchdog, §3) |
| telemetry `proximity`, `mag_raw_*`, `mag_rhall` | `proximity_raw`; the raw magnetometer values go |

## 5. Order

1. Descriptors: devices list their parts and each command's arguments with units (the wire
   protocol's planned capability descriptors).
2. `led`, `display`, `speaker`, `servo` on the robot; the dashboard, the pet and sbot switch at
   the same time (no old names kept).
3. The car's parts through sbot and tpbot-ble.
4. The renamed telemetry.
