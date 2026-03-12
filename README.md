# Home Assistant — Self-Parameterized Template Sensors

> A Home Assistant YAML pattern for building self-parameterized template sensors using `this.entity_id` introspection and naming conventions.

A pattern for Home Assistant showing how to **write a template sensor once and reuse it across many entities** without duplicating logic.

A common problem in Home Assistant configurations is repeating the same template logic for multiple rooms or devices. This pattern shows how to write the logic once and reuse it safely across many entities.

The core idea is **self-parameterized template sensors**.

This pattern relies on:

- **`this.entity_id` introspection** — derive source sensors dynamically from the entity name
- **strict naming conventions** — encode relationships directly in entity names

YAML anchors are an optional convenience tool to avoid duplicating the template logic across sensors.

This allows a template sensor to **configure itself from its own entity name**.

---

## The Problem

Many Home Assistant configurations duplicate the same template logic per room or device.
```yaml
sensor:
  - name: temperature_bedroom
    state: "{{ states('sensor.temperature_bedroom_1') }}"

  - name: temperature_living_room
    state: "{{ states('sensor.temperature_living_room_1') }}"
```

When the logic becomes more complex (fallbacks, validation, attributes), duplication grows and diverges over time.

---

## Why This Pattern Exists

Many Home Assistant configurations evolve organically and accumulate large amounts of duplicated template logic. When the same calculation is repeated across multiple rooms or devices, small fixes must be applied everywhere — and inconsistencies appear over time.

Self-parameterized template sensors solve this by turning entity naming into configuration, allowing one template to scale across many entities without modification.

---

## The Pattern

The goal of this pattern is to write **self-parameterized template sensors**.

Instead of hardcoding source entities inside the template, the sensor derives them from its own `entity_id`. This allows a single template to be reused across an entire family of entities without modification.

This approach works best with **trigger-based template sensors**, where the template runs when the relevant source entities update.

In short:

> entity name → determines dependencies → runs generic logic

This effectively turns the entity naming convention into configuration.

### Pattern overview
```
sensor.temperature_bedroom
            │
            ▼
sensor.temperature_bedroom_level
            │
            ▼
template derives dependency automatically
            │
            ▼
classification logic runs — same template, any room
```

### Minimal example
```yaml
sensor:
  - name: temperature_bedroom_level
    state: &level_logic >
      {% set room = this.entity_id
           | replace('sensor.temperature_', '')
           | replace('_level', '') %}
      {% set t = states('sensor.temperature_' ~ room) | float(none) %}
      {{ 'cold' if t < 18 else 'ok' if t < 24 else 'warm' if t < 27 else 'hot' }}

  - name: temperature_living_room_level
    state: *level_logic
```

Two sensors. One template. Zero duplication.

> For simplicity this example shows only the sensor block. In practice this logic would typically live inside a trigger-based template sensor under `template:`.

### Conceptual example

Suppose we define the following naming convention:
```
sensor.temperature_<room>        ← source
sensor.temperature_<room>_level  ← derived sensor
```

The `*_level` sensor classifies the temperature of its corresponding room. Instead of writing one template per room, the template derives the source entity automatically:
```jinja2
{% set room = this.entity_id | replace('sensor.temperature_', '') | replace('_level', '') %}
{% set t = states('sensor.temperature_' ~ room) | float(none) %}

{% if t is none %}    unknown
{% elif t < 18 %}     cold
{% elif t < 24 %}     ok
{% elif t < 27 %}     warm
{% else %}            hot
{% endif %}
```

The logic is written once, and reused across every room. A YAML anchor stores the template — each additional sensor is just a trigger block referencing it.

---

## Where This Pattern Is Useful

Any repeated template logic that follows a consistent naming structure:

- Temperature or humidity classification per room
- CO₂ air quality status per room
- Comfort level derived from a source sensor
- Device diagnostics or availability state
- Power level classification per device

One anchor, any number of rooms or devices.

---

## Example

The pattern works by deriving the source sensors from the entity name itself. Each consolidated sensor automatically resolves its own source sensors using the naming convention.
```
sensor.temperature_<room>_1   (main source)
sensor.temperature_<room>_2   (fallback source)
             ↓
sensor.temperature_<room>     (consolidated output)
```
```yaml
- trigger:
    - platform: state
      entity_id:
        - sensor.temperature_bedroom_1
        - sensor.temperature_bedroom_2
    - platform: homeassistant
      event: start

  sensor:
    - name: "Temperature Bedroom"
      unique_id: temperature_bedroom
      device_class: temperature
      unit_of_measurement: "°C"
      state: &temperature_logic >
        {% set suffix = this.entity_id | replace('sensor.temperature_', '') %}
        {% set src1 = 'sensor.temperature_' ~ suffix ~ '_1' %}
        {% set src2 = 'sensor.temperature_' ~ suffix ~ '_2' %}

        {% set main = states(src1) | float(none) %}
        {% set back = states(src2) | float(none) %}
        {% set last = this.state | float(none) %}

        {{ (main if main is not none
            else back if back is not none
            else last) | round(1)
           if (main is not none or back is not none or last is not none)
           else none }}

- trigger:
    - platform: state
      entity_id:
        - sensor.temperature_living_room_1
        - sensor.temperature_living_room_2
    - platform: homeassistant
      event: start

  sensor:
    - name: "Temperature Living Room"
      unique_id: temperature_living_room
      device_class: temperature
      unit_of_measurement: "°C"
      state: *temperature_logic
```

The logic is written **once**. The YAML anchor (`&temperature_logic`) stores the template, and `*temperature_logic` reuses it for every additional sensor. Every additional room is three lines of trigger + anchor reference.

---

## Fallback Chain

The template implements a three-level fallback:
```
main source (src1)
  → fallback source (src2) if main is unavailable
    → last known value (this.state) if both are unavailable
      → none if no value has ever existed
```

`this.state` exposes the sensor's previous state, allowing the template to retain the last known value when sources temporarily disappear (requires Home Assistant state restoration to be active).

On first boot, before any value has been recorded, the sensor honestly returns `none`.

---

## Naming Convention

This pattern relies on strict naming discipline.

| Entity | Role |
|--------|------|
| `sensor.temperature_<room>` | consolidated output |
| `sensor.temperature_<room>_1` | main source |
| `sensor.temperature_<room>_2` | fallback source |

The template derives its source sensors from its own entity name via `this.entity_id`. **If the naming convention is broken, the template will silently target incorrect sources.**

This pattern trades duplication for naming discipline.

---

## Extending the Pattern

The same approach applies to any repeated sensor logic.

Example: a color classification sensor derived from temperature + thresholds:
```
sensor.color_temperature_<room>
  → reads sensor.temperature_<room>
  → reads sensor.temperature_threshold_low_<room>
  → reads sensor.temperature_threshold_high_<room>
```

One anchor, any number of rooms.

---

## Examples

This repository includes several progressively more advanced examples of the pattern.

### Basic classification
```
examples/temperature_classification.yaml
```

A minimal example showing how a sensor can classify a temperature value using `this.entity_id` to automatically derive its source sensor.

### Consolidation with fallback
```
examples/temperature_consolidation.yaml
examples/temperature_fallback.yaml
```

These examples demonstrate how a sensor can consolidate multiple sources using a fallback chain:
```
main source
  → fallback source
    → last known state
```

The sensor derives its dependencies automatically from its own entity name.

### Advanced pipeline example
```
examples/temperature_minmax_daily/
```

A complete example demonstrating how **self-parameterized sensors can form a full sensor pipeline**.
```
temperature_min_daily_room_X
temperature_max_daily_room_X
        │
        ▼
temperature_minmax_daily_room_X
        │
        ▼
color_temperature_min_daily_room_X
color_temperature_max_daily_room_X
        │
        ▼
color_temperature_minmax_daily_room_X
```

This example shows how one template logic can power an entire **family of sensors**, naming conventions act as **implicit configuration**, and backend sensors can be composed into **multi-stage pipelines**.

See `examples/temperature_minmax_daily/README.md` for full details.

---

## Trade-offs

**Advantages**
- No template duplication
- Logic is centralized — fix once, fixed everywhere
- Consistent behavior across all entities

**Constraints**
- Requires strict naming conventions throughout the configuration
- Relies on template introspection (`this.entity_id`)
- Naming violations fail silently — no HA error, wrong sources used

---

## Compatibility

This pattern requires:

- Home Assistant template sensors supporting `this.entity_id`
- Trigger-based template sensors
- Home Assistant 2022.4 or newer (recommended)

---

## License

MIT
