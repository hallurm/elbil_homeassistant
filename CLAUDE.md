# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is **not** a runnable application. It's a curated, public export of the EV-charging-cost part of one Home Assistant installation's configuration — a documentation/reference repo, not a deployable config tree. There is no build system, no linter, no test suite, and nothing here executes on its own.

The files are snapshots taken from a live Home Assistant server via SSH and hand-filtered down to only the EV-related pieces; the server's full `configuration.yaml`/`automations.yaml`/`scripts.yaml` contain a large amount of unrelated household automation that is deliberately **not** included here. Editing files in this repo has **no effect** on any running system — there is no deploy step, no CI, and no connection back to the original server.

## Commands

There is no build, lint, or test tooling in this repo. The only useful local check is YAML syntax validation, since these files are hand-extracted fragments rather than complete, includable configs:

```bash
python -c "import yaml, sys; yaml.safe_load(open(sys.argv[1], encoding='utf-8'))" homeassistant/sensors.yaml
```

Run per-file for whichever file you changed. This only catches YAML syntax errors — it cannot catch Home Assistant template errors, wrong entity references, or Jinja2 mistakes, since there is no live Home Assistant instance to validate against here.

## Architecture

### The calculation chain (spans `homeassistant/sensors.yaml`, `utility_meter.yaml`, `automations.yaml`)

The core design problem this config solves: there is no physical power meter isolated on the EV charger itself, so "how much of the charging came from solar vs. the grid" is *derived*, not measured. The full reasoning is written up in [elbilsensorer.md](elbilsensorer.md) — read that before changing any formula. In short, the chain is:

```
raw hardware sensors (phase currents/voltages, grid meter, inverter)
  → elbil_oplader_ac_effekt              (instant AC power the charger draws, W)
  → elbil_oplader_effekt_fra_nettet /    (instant grid vs. solar split, W —
    elbil_oplader_effekt_fra_sol          proportional to the whole house's
                                           live sol/net mix at that instant)
  → elbil_oplader_dkk_per_time           (instant cost rate, DKK/h)
  → [Riemann integration] → energi_fra_nettet / energi_fra_sol / pris_akkumuleret
                                           (lifetime accumulators, kWh/DKK)
  → [utility_meter]        → dag/uge/måned/år period counters
  → [combined with externally-logged public charging] → "hjemme og ekstern" totals
```

`homeassistant/sensors.yaml` contains both the three `platform: integration` (Riemann sum) accumulators and the `template:` sensors that do the instant-power math. `homeassistant/utility_meter.yaml` derives the day/week/month/year period counters from those accumulators. `homeassistant/automations.yaml` holds a single automation that zeroes the home battery's discharge limit while the car is charging — without it, battery discharge would silently pollute the grid/solar split.

### External (public) charging

Public-charger sessions can't be measured automatically, so `homeassistant/input_number.yaml` defines manual entry fields plus a running total, and `homeassistant/scripts.yaml`'s single script folds an entry into the total and resets the input fields. The `_total_*_alt` sensors at the bottom of `sensors.yaml` add home + external together *without* feeding external kWh into the solar-share percentage, since external charging has no known solar fraction by definition.

### Two dashboards, two different mechanisms

- `dashboards/elbil.yaml` — a plain YAML-mode Lovelace dashboard (one page, everything on it). This format is directly includable by Home Assistant as-is.
- `dashboards/elbil_g6.yaml` — a five-tab "sections" dashboard using Mushroom cards and ApexCharts. On the live server this lives in `.storage/lovelace.dashboard_g6` as **JSON** (storage-mode dashboards are UI-managed); the copy here was converted to YAML purely for readability/diffing. It is not something Home Assistant can load from this file directly.

### Entity IDs are derived from `name:`, not from `unique_id`

A recurring gotcha reflected throughout these files: Home Assistant generates an entity's `entity_id` by slugifying its **`name:`** field, not its `unique_id` or its YAML dict key. Several sensor and `utility_meter` names intentionally spell out full Danish phrases (e.g. `"Elbil opladning kWh fra nettet denne måned"`) because the resulting entity_id (`sensor.elbil_opladning_kwh_fra_nettet_denne_maned`, note "å" → "a") is what every other template's `states('sensor...')` call must reference. If you rename a `name:` field, every downstream template referencing its old entity_id breaks silently (falls back to `float(0)`) rather than erroring loudly.

### Display precision is split across two mechanisms

Template sensors carry `suggested_display_precision: 1` directly in `sensors.yaml`. The `integration`-platform accumulators and all `utility_meter` period counters have no YAML equivalent for this — on the live instance their 1-decimal display is set via the entity registry (`options.sensor.display_precision`), which isn't captured in these exported files. See the header comments in `sensors.yaml` and `utility_meter.yaml`.

## Working in this repo

- **Never add secrets.** The repo is public. Don't copy in recorder/MQTT/InfluxDB credentials, SSH details, or any part of the source `configuration.yaml` beyond the EV-specific fragments already here.
- **Entity names are installation-specific.** Anything matching `sensor.fsp_ne_<digits>_*` is this one household's charger device ID; `sensor.inverter_*`, `sensor.power_meter_*`, and `sensor.el_kob_pris` come from the three integrations listed in the README. Don't invent new entity names that don't follow this pattern without noting they're placeholders.
- **Keep `elbilsensorer.md` in sync with `homeassistant/*.yaml`.** It's the prose explanation of every formula; if you change a template's math, update the corresponding section there too.
