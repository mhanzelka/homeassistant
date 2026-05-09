# Home Assistant + ESPHome

Blueprints, ESPHome device configs, and a few helper YAMLs I run at home and decided to share.

The blueprints are written for current Home Assistant releases (selectors with sections, `device_class` filters, statistics/schedule helpers) and are designed to be imported and configured entirely from the UI — no YAML editing required after import.

## Blueprints

### Air Purifier Schedule V1

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fmhanzelka%2Fhomeassistant%2Fmain%2Fblueprints%2Fair_purifier_schedule_v1.yaml)

Drives a `fan`-domain air purifier from a Home Assistant **schedule helper** and a list of window/door contact sensors.

- The purifier is turned **off** whenever any monitored window or door is open.
- Otherwise the schedule decides when it runs.
- Each schedule block can carry an additional data field (default name `mode`) with one of: `turbo`, `sleep`, `auto`, `manual`. The automation reads it and applies the matching `preset_mode` on the fan; `manual` (or unknown / missing values) falls back to a configured percentage.
- **External overrides (optional):** two `input_select` helpers driven from your own automations can override the schedule. Both expose the same options: `schedule` (= no override), `off`, `turbo`, `sleep`, `auto`, `manual`. The **hard** override wins absolutely (ignores windows and schedule). The **soft** override wins over the schedule but is blocked by an open window or by an active schedule block whose mode is `sleep`. Hard always wins over soft.
- **Window debounce (hysteresis):** a configurable delay between any window/door state change and the resulting fan action (default 30 s, set to 0 to disable). Brief open/close bounces never reach the fan because the automation runs in `mode: restart` and re-evaluates after the delay.

**Inputs:** purifier fan, window/door sensors, schedule helper, preset names for turbo/sleep/auto, manual fallback speed, optional hard/soft override helpers, window debounce delay.

**Setup:**
1. Create a *Schedule* helper (Settings → Devices & Services → Helpers → Schedule) and draw the time blocks you want.
2. On each block add an additional data field named `mode` (or whatever you configure in the blueprint) with one of `turbo` / `sleep` / `auto` / `manual`.
3. Look up the exact `preset_modes` strings for your fan in Developer Tools → States and put them into the matching inputs. Leave a preset empty to disable it (that mode then falls back to manual).
4. (Optional) For external overrides, create one or two *Dropdown* (`input_select`) helpers with options `schedule, off, turbo, sleep, auto, manual` (initial value `schedule`). Wire them into the *Hard override* / *Soft override* inputs and drive them from your automations via `input_select.select_option`.

---

### Motion Activated Lights V1

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fmhanzelka%2Fhomeassistant%2Fmain%2Fblueprints%2Fmotion_activated_lights_v1.yaml)

A simple motion / presence-driven lights blueprint. Pick one or more motion sensors and one or more lights — the blueprint turns the lights on when any sensor detects activity and turns them off after a configurable delay once **all** sensors are clear.

- Lights turn on whenever **any** selected sensor switches to `on`. Already-on lights are left untouched (plain `light.turn_on` with no parameters — no brightness or colour override).
- The off-delay starts only after **all** monitored sensors report no motion, and resets if any sensor detects motion again before it expires.
- **Take control** (optional): when off (default), lights that were already on at the moment motion was first detected are left alone after the delay. When on, the automation turns the lights off regardless.

**Inputs:** motion / presence sensors, lights, wait time after no motion (duration), take control toggle.

**Setup:** pick the sensors and lights and set the delay — that's it. No helpers required.

---

### Display Orchestrator V1

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2Fmhanzelka%2Fhomeassistant%2Fmain%2Fblueprints%2Fdisplay_orchestrator_v1.yaml)

Designed to drive the ESPHome [`max7219_8x32`](esphome/max7219_8x32/) device that lives in this repo — pair it with that firmware for an immediately usable setup. The blueprint also works with any other ESPHome display that exposes the same `switch` + `select` interface (and optionally a `text` entity for per-slot openings push).

It exposes a `switch` for power on/off and a `select` for the active screen, then combines a schedule, a list of window/door sensors and a 3D-printer progress sensor into a single controller.

- A schedule helper decides when the display is allowed to be on; outside its blocks the switch is forced off and the active screen is left alone.
- While the display is on, an open window/door switches the active screen to the configured **openings** screen (highest priority).
- With no window open, a non-zero printer progress sensor switches to the configured **progress** screen.
- Falls back to a configurable default screen, or leaves the current selection alone if none is configured.
- **Per-slot openings push (optional):** when an *Openings text entity* is set, the blueprint also writes a comma-separated state string (`"1,0,1,0,0"`) to it on every change, so individual slots on the device update with the actual window/door states. The order in which sensors are picked under *Window / door sensors* defines slot 0 → slot 4 on the device.
- **Reconnect handler:** when the device's entities transition out of `unavailable` (boot, OTA reflash, WiFi drop, USB unplug/replug), the automation re-pushes the current window states and forces the openings screen, so the display always recovers to a known-good view.

**Supported screens** — option strings exposed by the `max7219_8x32` device's `select.<device>_active_screen` entity. Use these values verbatim in the `windows_screen_option` / `printer_screen_option` / `default_screen_option` inputs:

| Option | Description |
| --- | --- |
| `window_and_door_openings` | 5-slot grid of window/door states (filled = open, outline = closed). Driven by `text.<device>_openings`. |
| `text` | Configurable text from HA (`text.<device>_text_screen_content`) with alignment + optional marquee scroll. |
| `counter` | Three-level visual counter (units → 30-unit ticks → 30-min ticks). Drives `number.<device>_counter_value`; `switch.<device>_counter_auto_tick` enables a 1 Hz on-device auto-increment. |
| `progress` | Generic 8×8 icon + horizontal progress bar 0-100 %. Picks an icon by name from `select.<device>_progress_icon` and reads the percentage from `number.<device>_progress_value`. |
| `time` | Centred `HH:MM` clock from a `time:` `homeassistant` source. |

**Inputs:** display switch, active screen select, optional openings text entity, on-schedule helper, printer progress sensor + screen option name + threshold, window sensors + screen option name, optional default screen option name.

**Setup:**
1. Match the *active screen* option strings (`window_and_door_openings`, `progress`, …) to the option names exposed by your device's `select.<device>_active_screen` entity.
2. Create a *Schedule* helper for when the display should be on.
3. Pick the window/door binary sensors. **Order matters** if you use the openings text push — the first one becomes slot 0, etc.
4. Wire the printer progress sensor and tune the threshold if the sensor reports tiny non-zero noise while idle.
5. (Optional) Set an "Openings text entity" pointing at `text.<device>_openings` for per-slot state push.

---

### Window Ventilation Guard

> **Status: in development.** The YAML is in this repo but the blueprint isn't ready for general use yet — no one-click import button on purpose. Expect breaking changes; import only if you're ready to read the source.

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
   - `https://raw.githubusercontent.com/mhanzelka/homeassistant/main/blueprints/motion_activated_lights_v1.yaml`
   - `https://raw.githubusercontent.com/mhanzelka/homeassistant/main/blueprints/display_orchestrator_v1.yaml`

## ESPHome devices

The `esphome/` directory holds two physical devices I drive from Home Assistant. Each device lives in its own subdirectory with its main YAML, screen / tab render lambdas, and a symlink back to the canonical `secrets.yaml` at the repo root. Shared boilerplate (logger, OTA, captive portal, encrypted API) is factored into `esphome/common.yaml`.

```
esphome/
├── secrets.yaml           # gitignored; per-device subdirs symlink ../secrets.yaml
├── common.yaml            # logger / ota / captive_portal / api
├── data/                  # shared HA-entity imports (currently TTGO only)
├── ttgo/
│   ├── ttgo.yaml          # TTGO T-Display main config
│   ├── colors.yaml        # color palette
│   └── tabs/              # per-tab render lambda substitutions
└── max7219_8x32/
    ├── matrix.yaml        # MAX7219 main config ("HA Display V1 Red")
    ├── icons/icons.yaml   # 8×8 pixel icon catalog for the progress screen
    └── screens/           # per-screen render lambda substitutions
```

### TTGO T-Display

ESP32 + ST7789v 135 × 240 colour LCD with two on-board buttons.

- Two tabs switchable from the right button: **openings** (5 windows/doors as 3 × 2 grid) and **climate** (6 per-room temperatures with colour-coded ranges).
- Left button publishes `esphome.ttgo_button` events to HA on `single` / `double` / `hold`, ready to wire into automations.
- Deep sleep after 20 s of inactivity; the right button doubles as the wake pin (GPIO0 is a strapping pin and unsuitable for wake on classic ESP32, so left can't wake the device).
- Backlight is a separate `gpio` switch turned off explicitly before `deep_sleep.enter`, so a sleeping device is genuinely dark.

### MAX7219 8×32 matrix ("HA Display V1 Red")

ESP32 DevKit V1 + 4× cascaded FC-16 MAX7219 modules (32 cols × 8 rows monochrome LEDs).

- Five swappable screens picked via `select.<device>_active_screen`: `window_and_door_openings`, `text`, `counter`, `progress`, `time`.
- Generic interface — the device knows nothing about the user's specific HA entities. All data flows in via writeable HA entities:
  - `text.<device>_openings` — comma-separated open/closed state for 5 slots.
  - `text.<device>_text_screen_content`, `select.<device>_text_screen_alignment`, `switch.<device>_text_screen_scrolling` — text screen config.
  - `number.<device>_counter_value` (+ `switch.<device>_counter_auto_tick`) — counter screen value and 1 Hz autonomous tick.
  - `select.<device>_progress_icon` (catalog from `icons/icons.yaml`) + `number.<device>_progress_value` — progress screen.
  - `switch.<device>_display_on` and `number.<device>_display_intensity` — power toggle and 0-15 brightness.
- Pairs with the **Display Orchestrator V1** blueprint above for the schedule + windows + printer wiring; everything else is plain HA service calls.

### Build & flash

```sh
esphome compile esphome/ttgo/ttgo.yaml
esphome compile esphome/max7219_8x32/matrix.yaml
```

First flash over USB; subsequent reflashes can use OTA at `<device>.local` (e.g. `esphome run ... → [2] Over The Air`). For TTGO the OTA window is short because of deep sleep — wake the device first by pressing the right button.

## License

MIT — use, fork, modify, share. No warranty; test in your own setup before relying on these for anything safety-relevant.
