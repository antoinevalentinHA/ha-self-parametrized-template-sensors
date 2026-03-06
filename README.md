# Home Assistant — Self-Parametrized Template Sensors

A pattern for Home Assistant showing how to **write a template sensor once and reuse it across many entities** without duplicating logic.

A common problem in Home Assistant configurations is repeating the same template logic for multiple rooms or devices. This pattern shows how to write the logic once and reuse it safely across many entities.

The pattern combines:

- **YAML anchors** — reuse the same template logic
- **`this.entity_id` introspection** — derive source sensors dynamically
- **strict naming conventions**

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

## The Pattern

Instead of duplicating the logic:

1. Write the logic **once**
2. Reuse it with a **YAML anchor**
3. Derive source sensors **from the entity name**

Expected naming structure:

```
sensor.temperature_<room>      ← consolidated sensor
sensor.temperature_<room>_1    ← main source
sensor.temperature_<room>_2    ← fallback source
```

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

See the full working example in `examples/temperature_consolidation.yaml`.

The logic is written **once**. The YAML anchor (`&temperature_logic`) stores the template, and `*temperature_logic` reuses it for every additional sensor.

---

## Fallback Chain

The template implements a three-level fallback:

```
main source (src1)
  → fallback source (src2) if main is unavailable
    → last known value (this.state) if both are unavailable
      → none if no value has ever existed
```

`this.state` exposes the sensor's previous state, effectively acting as persistent memory across updates and restarts. If all sources are temporarily unavailable, the sensor retains its last known value rather than producing an unknown state.

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
