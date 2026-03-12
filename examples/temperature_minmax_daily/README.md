# Example — Daily Temperature Min/Max Pipeline

This example demonstrates how **self-parameterized template sensors** can be used to build a complete sensor pipeline with minimal duplication.

The entire system relies on a single convention:
```
<this.entity_id> → derive suffix → derive source entities
```

Using this pattern, a single template logic can serve an entire family of sensors.

---

## Architecture
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

Each stage is generic and uses `this.entity_id` to determine which entities it should read. No room names are hardcoded in the logic.

---

## Naming convention

All sensors in this example rely on a strict naming convention. The `<room>` suffix acts as the configuration key used by the templates.

| Entity | Purpose |
|---|---|
| `sensor.temperature_min_daily_<room>` | Daily minimum temperature |
| `sensor.temperature_max_daily_<room>` | Daily maximum temperature |
| `sensor.temperature_lower_threshold_<room>` | Lower comfort threshold |
| `sensor.temperature_upper_threshold_<room>` | Upper comfort threshold |
| `sensor.temperature_minmax_daily_<room>` | Display sensor (`min / max °C`) |
| `sensor.color_temperature_min_daily_<room>` | Color classification of the minimum |
| `sensor.color_temperature_max_daily_<room>` | Color classification of the maximum |
| `sensor.color_temperature_minmax_daily_<room>` | Synthetic color summarizing the daily range |

### Example for `room_1`
```
sensor.temperature_min_daily_room_1
sensor.temperature_max_daily_room_1
sensor.temperature_lower_threshold_room_1
sensor.temperature_upper_threshold_room_1
sensor.temperature_minmax_daily_room_1
sensor.color_temperature_min_daily_room_1
sensor.color_temperature_max_daily_room_1
sensor.color_temperature_minmax_daily_room_1
```

The templates derive the `<room>` suffix automatically from `this.entity_id`. The suffix can represent any logical area — the templates never reference room names directly.

---

## Files

### `temperature_minmax_jour.yaml`

Produces a compact display sensor for dashboards. Example output: `18.8 / 24.1 °C`

Combines `sensor.temperature_min_daily_<room>` and `sensor.temperature_max_daily_<room>`. Does **not** implement any color or classification logic.

### `couleur_temperature_min_jour.yaml`

Classifies the daily minimum temperature using configurable thresholds from `sensor.temperature_lower_threshold_<room>` and `sensor.temperature_upper_threshold_<room>`. Output: `sensor.color_temperature_min_daily_<room>`.

### `couleur_temperature_max_jour.yaml`

Same classification logic applied to the daily maximum. Output: `sensor.color_temperature_max_daily_<room>`.

### `couleur_temperature_minmax_jour.yaml`

Produces a synthetic color summarizing the daily range by combining both color sensors. Priority rule — worst condition wins:
```
grey → red → yellow → blue → light_blue → green
```

---

## Key patterns

**Self-parameterized sensors.** Each sensor derives its source entities automatically from `this.entity_id`. A single logic block serves an entire room family.

**Sensor families.** Sensors share the same suffix (`room_1`, `room_2`, …). The suffix becomes the implicit configuration key.

**Backend/UI separation.** Display sensors only format data. Color classification happens in dedicated backend sensors.

---

## Scaling

Adding a new room only requires two new instances:
```
sensor.temperature_min_daily_room_5
sensor.temperature_max_daily_room_5
```

No template logic needs to be modified. Naming conventions act as implicit configuration — the room name never appears in the logic itself.
