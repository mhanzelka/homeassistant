# Home Assistant Blueprints

A small collection of Home Assistant automation blueprints I use at home and decided to share.

Both blueprints are written for current Home Assistant releases (selectors with sections, `device_class` filters, statistics/schedule helpers) and are designed to be imported and configured entirely from the UI — no YAML editing required after import.

## Blueprints

### Air Purifier Schedule

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fmhanzelka%2Fhomeassistant%2Fmain%2Fblueprints%2Fair_purifier_schedule_v1.yaml)

Drives a `fan`-domain air purifier from a Home Assistant **schedule helper** and a list of window/door contact sensors.

- The purifier is turned **off** whenever any monitored window or door is open.
- Otherwise the schedule decides when it runs.
- Each schedule block can carry an additional data field (default name `mode`) with one of: `turbo`, `sleep`, `auto`, `manual`. The automation reads it and applies the matching `preset_mode` on the fan; `manual` (or unknown / missing values) falls back to a configured percentage.

**Inputs:** purifier fan, window/door sensors, schedule helper, preset names for turbo/sleep/auto, manual fallback speed.

**Setup:**
1. Create a *Schedule* helper (Settings → Devices & Services → Helpers → Schedule) and draw the time blocks you want.
2. On each block add an additional data field named `mode` (or whatever you configure in the blueprint) with one of `turbo` / `sleep` / `auto` / `manual`.
3. Look up the exact `preset_modes` strings for your fan in Developer Tools → States and put them into the matching inputs. Leave a preset empty to disable it (that mode then falls back to manual).

---

### Window Ventilation Guard

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fmhanzelka%2Fhomeassistant%2Fmain%2Fblueprints%2Fwindow_ventilation_guard.yaml)

Watches open windows and fires an alert action when ventilating becomes counter-productive — different logic for winter and summer, switched automatically by a rolling-average outdoor temperature sensor.

- **Winter mode:** alert if outdoor temperature is below the safe minimum, or immediately when *eco mode* is on. Above the anomaly threshold (warm winter day) ventilation is always considered safe.
- **Summer mode:** alert when outdoor temperature ≥ (indoor − offset), i.e. it's getting warmer outside than inside.
- **Hysteresis:** a dead-band between the summer and winter thresholds keeps the season stable on transitional days; the previous mode is preserved.
- **Recommendation entity (optional):** an `input_boolean` driven by the automation that says whether the window *should* be open right now. In winter it's only active during configurable morning/evening ventilation windows, with a per-day minute quota tracked in an `input_text` helper (auto-resets on date change — no midnight cron needed).

**Inputs:** window sensors, outdoor and indoor temperature sensors, outdoor temperature statistics sensor (24–72 h mean), eco mode `input_boolean`, season mode `input_select` / `input_text`, season auto-switch toggle, thresholds, alert action, optional recommendation entity, optional ventilation tracker.

**Setup:**
1. Create a *Statistics* helper targeting your outdoor temperature entity, with `mean` over a 24–72 h window — this drives the season switching.
2. Create an `input_boolean` for eco mode.
3. Create an `input_select` (options: `summer`, `winter`) **or** an `input_text` for the season mode and pre-set it to your current season before enabling the automation.
4. (Optional) Create an `input_boolean` for the window-open recommendation and an `input_text` for the ventilation tracker.
5. Hook up your alert (mobile push, siren, TTS, …) into the *Alert action* input.

## Importing

Click the blue **Open your Home Assistant instance** button next to a blueprint above. Home Assistant opens the blueprint import dialog with the URL pre-filled — confirm and the blueprint is installed.

If the button doesn't work (e.g. you haven't set up [My Home Assistant](https://my.home-assistant.io/) yet), import manually:

1. Settings → Automations & Scenes → Blueprints → **Import Blueprint**.
2. Paste the raw URL of the YAML file:
   - `https://raw.githubusercontent.com/mhanzelka/homeassistant/main/blueprints/air_purifier_schedule_v1.yaml`
   - `https://raw.githubusercontent.com/mhanzelka/homeassistant/main/blueprints/window_ventilation_guard.yaml`

## License

MIT — use, fork, modify, share. No warranty; test in your own setup before relying on these for anything safety-relevant.
