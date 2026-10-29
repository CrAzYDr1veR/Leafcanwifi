# Home Assistant — Revision 3 Migration Patch

The `esp-leaf.yaml` firmware (Revision 3) renamed or removed a few legacy entities and introduced new ones. Your Home Assistant `configuration.yaml` **must be updated** if you use custom template helpers, otherwise those helpers will report `unavailable` or `unknown`.

> Your `configuration.yaml` lives in `/homeassistant` on your Home Assistant server — this file provides the ready-to-apply patch (make a backup before editing).

---

## 1. Renamed Entities (Find & Replace)

| Legacy Reference (in `configuration.yaml`) | New Entity ID |
| :--- | :--- |
| `sensor.garagem_leaf_esp_remaining_time_6kw_wallbox` | `sensor.garagem_leaf_esp_remaining_charge_time` |
| `sensor.garagem_leaf_esp_remaining_time_3kw_plug` | `sensor.garagem_leaf_esp_remaining_charge_time` |
| `sensor.garagem_leaf_esp_remaining_time_50kw_qc` | `sensor.garagem_leaf_esp_remaining_charge_time` |
| `sensor.garagem_leaf_esp_charge_status_sensor` | `sensor.garagem_leaf_esp_charge_status` |
| `sensor.garagem_leaf_esp_bms_active_balancing_shunts` | `sensor.garagem_leaf_esp_bms_group_6_active_shunts` |
| `sensor.garagem_leaf_esp_lbc_soc` | `sensor.garagem_leaf_esp_usable_soc` (dash) **or** `sensor.garagem_leaf_esp_battery_soc_lbc_group_1` (LBC, 0.01%) |
| `sensor.garagem_leaf_esp_battery_temperature_dash` | `sensor.garagem_leaf_esp_battery_gauge_segment_dash` |

### 1.1 `configuration.yaml` (~line 121-122) — Remaining Charge Time

The three separate "remaining time" helpers (6 kW / 3 kW / QC) have been replaced by a **single** real value decoded directly from frame `0x5BC` (`LB_Remain_Charge_Time`).

```yaml
# BEFORE
- sensor:
    name: Tempo restante carga 6kW
    unique_id: garagem_leaf_esp_remaining_time_6kw_wallbox
    state: "{{ states('sensor.garagem_leaf_esp_remaining_time_6kw_wallbox') }}"
    unit_of_measurement: min
    device_class: duration
- sensor:
    name: Tempo restante carga 3kW
    unique_id: garagem_leaf_esp_remaining_time_3kw_plug
    state: "{{ states('sensor.garagem_leaf_esp_remaining_time_3kw_plug') }}"
    unit_of_measurement: min
    device_class: duration
```

```yaml
# AFTER
- sensor:
    name: Tempo restante de carga
    unique_id: garagem_leaf_esp_remaining_charge_time
    state: "{{ states('sensor.garagem_leaf_esp_remaining_charge_time') }}"
    unit_of_measurement: min
    device_class: duration
```

> The value is only published when physically plausible (remaining energy / 60 kW max, up to 720 min) and after passing the `0x5BC` variant gate.

### 1.2 `configuration.yaml` (~line 395) — Charge Status

```yaml
# BEFORE
state: "{{ states('sensor.garagem_leaf_esp_charge_status_sensor') }}"
# AFTER
state: "{{ states('sensor.garagem_leaf_esp_charge_status') }}"
```

### 1.3 `configuration.yaml` (~line 411) — Combined Leaf SOC

The combined SOC helper now uses the real LBC SOC (Group 1), with the dashboard SOC source as an optional alternative.

```yaml
# BEFORE
- sensor:
    name: Leaf SOC Combinado
    unique_id: garagem_leaf_esp_soc_combinado
    state: "{{ states('sensor.garagem_leaf_esp_lbc_soc') }}"
    unit_of_measurement: "%"
    device_class: battery
```

```yaml
# AFTER (recommended: LBC Group 1, 0.01% resolution)
- sensor:
    name: Leaf SOC Combinado
    unique_id: garagem_leaf_esp_soc_combinado
    state: "{{ states('sensor.garagem_leaf_esp_battery_soc_lbc_group_1') }}"
    unit_of_measurement: "%"
    device_class: battery

# Or, if you prefer the dashboard SOC (0x1DB) with automatic fallback:
- sensor:
    name: Leaf SOC Combinado
    unique_id: garagem_leaf_esp_soc_combinado
    state: >
      {% if is_state('sensor.garagem_leaf_esp_usable_soc_source', 'Dash 0x1DB') %}
        {{ states('sensor.garagem_leaf_esp_usable_soc') }}
      {% else %}
        {{ states('sensor.garagem_leaf_esp_battery_soc_lbc_group_1') }}
      {% endif %}
    unit_of_measurement: "%"
    device_class: battery
```

---

## 2. Lovelace Dashboard (`nissan-leaf` view)

The updated dashboard layout is included in [`dashboard_layout.yaml`](dashboard_layout.yaml) in this repository. If your Home Assistant server still uses the older view, replace it with the contents of `dashboard_layout.yaml`.

Entities to **remove** from old views (no longer exist):

* `sensor.garagem_leaf_esp_remaining_time_50kw_qc`
* `sensor.garagem_leaf_esp_remaining_time_6kw_wallbox`
* `sensor.garagem_leaf_esp_remaining_time_3kw_plug`
* `sensor.garagem_leaf_esp_bms_active_balancing_shunts`

Entities **added** in Revision 3:

| Entity | Dashboard Card |
| :--- | :--- |
| `sensor.garagem_leaf_esp_soc_gids` | Battery ➔ Gauge "SOC (GIDs)" |
| `sensor.garagem_leaf_esp_battery_soc_lbc_group_1` | Battery ➔ "SOC (LBC group 1)" |
| `sensor.garagem_leaf_esp_battery_capacity_ah` | Battery ➔ "Capacity (Ah)" |
| `sensor.garagem_leaf_esp_battery_health_hx` | Battery ➔ "Health (HX)" |
| `sensor.garagem_leaf_esp_battery_pack_temperatures` | Temperatures ➔ "Pack Temperatures" |
| `sensor.garagem_leaf_esp_battery_temperature_source` | Temperatures ➔ "Temperature Source" |

---

## 3. Database Optimization (`home-assistant_v2.db`)

Revision 3 publishes numeric sensors **only when the value changes** beyond a deadband threshold (0.05–0.3 units). Previously, helpers triggered at 5–10 Hz created thousands of rows from oscillations between two adjacent steps.

Custom user helpers to review:

| Helper | Issue | Recommendation |
| :--- | :--- | :--- |
| `sensor.garagem_leaf_esp_potencia_carregamento` | Triggered at 5–10 Hz from `0x1DB` | Use `sensor.garagem_leaf_esp_hv_instant_power` (already filtered in firmware) or remove |
| `sensor.garagem_leaf_esp_energia_carregamento` | Integrates power at 5–10 Hz | Use `sensor.garagem_leaf_esp_remaining_energy` from the firmware (`0x59E`) |

---

## 4. Validation Checklist

1. Run `python -m esphome config esp-leaf.yaml` ➔ should output **`INFO Configuration is valid!`**
2. Compile and flash (`esphome run esp-leaf.yaml`).
3. Check the ESPHome logs for:

```text
leaf_7BB First Frame len=53 group=0x01 ...
leaf_7BB message complete: 53 bytes, group 0x01
leaf_7BB group 1: SOC=80.40 % AHR=111.71 Ah HX=96.50 %
leaf_7BB group 4 pack temps: 19.2 / 20.2 / 0.0 / 19.4 C
leaf_7BB LBC group 61 SOH = 82.49 %
leaf_5BC ACCEPTED gids=214 (17120 Wh) t=... cond=...
```

* You should **not** see repeated `out of sequence frame` or `timeout, dropping incomplete message` errors (a single timeout may occur if the car is asleep).
* `leaf_5BC REJECTED` indicates the cluster scale variant (`500 GIDs`) did not match remaining energy; the firmware retains the last valid reading as intended.
* `leaf_54C raw=` should show `x[1] != 0xFF` when the air conditioning is active.
