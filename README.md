# Attic office climate control

A self-contained Home Assistant package that keeps an attic office comfortable
(around a configurable target, default **24 °C**) while running a portable air
conditioner **as little as possible**. It prefers free cooling — opening the
window and running the fan when the outdoor air is cooler than the room — and
only falls back to the AC when free cooling can't do the job.

## Entities used

| Role | Entity |
|---|---|
| Room temperature | `sensor.attic_environment_temperature` |
| Outdoor temperature | `sensor.hue_outdoor_motion_sensor_1_temperature` |
| Window contact (`on` = open) | `binary_sensor.attic_window_contact` |
| Air conditioner | `climate.pro_breeze_12000_btu_air_conditioner` |
| Fan (on/off switch) | `switch.attic_fan` |
| Mobile push notifications | `notify.mobile_app_kens_iphone` |

> `weather.forecast_home` is **not** used yet — see [Ideas](#future-ideas).

## How it works

A single automation (`Attic office climate control`) re-evaluates whenever the
temperatures or window change, every 5 minutes, and at each schedule boundary.
It decides in priority order:

1. **Outside the pre-cool / work window** → AC off, fan off.
2. **Window open** → AC never runs; the fan runs only if the room is warm *and*
   the outdoor air is cooler (i.e. the open window is actually helping).
3. **Warm, window shut, outside cooler** → AC stays off, fan runs (free cooling);
   you get a push to **open the window**.
4. **Warm, outside not cooler** → AC switches to `cool` at the effective target;
   the room fan turns **off** (running both is too noisy — the AC's own fan does
   the work).
5. **Cool enough** → AC off, fan off.
6. **In between** the on/off thresholds nothing changes — a hysteresis dead-band
   that stops the AC short-cycling.

### Pre-cooling

Before the work day there is a **pre-cool window** (default 06:00 → 08:00). In
that window the *effective target* is lowered by the pre-cool offset
(`target − offset`, default 1 °C) so the attic starts the day a little cooler.
Free cooling is still preferred; the AC only fills the gap.

### Window notifications

A second automation (`Attic office window prompts`) sends a phone push to:

- **Open** the window when free cooling is available (room warm, outside cooler).
- **Close** the window when it's open but the outside air is *not* cooler, so the
  AC can take over.

Each prompt only fires after the condition holds for 2 minutes (no spam), and the
two share a notification `tag` so they replace each other rather than stack.

## Tunable helpers

All exposed in the UI (Settings → Devices & Services → Helpers):

| Helper | Default | Purpose |
|---|---|---|
| `input_number.attic_target_temp` | 24 °C | Comfort target |
| `input_number.attic_temp_tolerance` | 0.5 °C | Hysteresis half-band (AC on at +tol, off at −tol) |
| `input_number.attic_free_cooling_delta` | 1.5 °C | How far below the room the outside must be to free-cool |
| `input_number.attic_precool_offset` | 1 °C | How far below target to pre-cool |
| `input_datetime.attic_office_precool_start` | 06:00 | Pre-cool window start |
| `input_datetime.attic_office_start` | 08:00 | Work window start |
| `input_datetime.attic_office_end` | 18:00 | Work/pre-cool window end |
| `input_boolean.attic_climate_control` | — | Master on/off for both automations |

Turn `input_boolean.attic_climate_control` **on** to enable the system; turning it
off makes both automations no-op.

## Installation

### Fresh Home Assistant install
Copy `configuration.yaml` and the `packages/` directory into your HA config
folder, then restart Home Assistant.

### Existing Home Assistant install
1. Copy `packages/attic_office_climate.yaml` into your config's `packages/`
   directory (create it if needed).
2. Ensure packages are enabled in your existing `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
   Do **not** overwrite your existing `configuration.yaml` with the one here.
3. Restart Home Assistant.

## Notes

- Push notifications go to `notify.mobile_app_kens_iphone` (the HA companion app
  on the iPhone). If the phone is renamed or re-registered, update that service
  name in `packages/attic_office_climate.yaml`.
- The control automation only sends a command when the AC/fan isn't already in
  the desired state (idempotent `if/then` guards). This prevents the Pro Breeze
  from beeping on every periodic re-evaluation.
- `climate.set_temperature` is called with an inline `hvac_mode: cool`, which
  requires a reasonably recent HA core. If your AC integration rejects it, split
  it into a `climate.set_hvac_mode` (`cool`) step followed by
  `climate.set_temperature`.

## Future ideas

- Use `weather.forecast_home` to pre-cool harder before a forecast hot spell, or
  to skip the AC when a cool change is imminent.
