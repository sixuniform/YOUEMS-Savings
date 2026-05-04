# YOUEMS Savings Tracker

A Home Assistant package that tracks electricity savings from solar production, battery discharge, and arbitrage trading — measured against what you would have paid importing everything from the grid.

## How It Works

Every 15 minutes (aligned to Nordpool slot boundaries), the automation computes:

```
slot_savings = (house_delta - import_delta) × buy_price
             + export_delta × sell_price
```

| Term | Meaning |
|------|---------|
| `house_delta` | kWh consumed by house this slot |
| `import_delta` | kWh imported from grid this slot |
| `house - import` | kWh served by solar or battery (free/earned) |
| `export_delta` | kWh exported to grid this slot |
| `buy_price` | SEK/kWh you pay for grid import |
| `sell_price` | SEK/kWh you receive for grid export |

**What counts as a saving:**
- ☀️ Solar covering house load → positive (avoided grid cost)
- 🔋 Battery discharging → positive (avoided grid cost)  
- ⚡ Exporting to grid → positive (revenue)
- 🔌 Charging battery from grid → negative (cost, recovered later when discharging)
- 🏠 Normal house load from grid → zero (baseline, no saving or cost)

Prices are looked up from the `raw_today` attribute for the exact slot that just ended — not the current price — ensuring accuracy even when consecutive slots have identical prices.

---

## Required Sensors

### Grid meter (cumulative totals — resets OK if using `_total`)

| Sensor | Description |
|--------|-------------|
| `sensor.solax_inverter_grid_export_total` | Cumulative grid export in kWh |
| `sensor.solax_inverter_grid_import_total` | Cumulative grid import in kWh |
| `sensor.solax_inverter_house_energy_total` | Cumulative house consumption in kWh |

> These should be ever-increasing totals. If you only have `_today` variants, the tracker will reset at midnight giving incorrect savings for the first slot of each day.

### Nordpool price sensors

The tracker requires **two Nordpool integration sensors** with different fee configurations. Both must expose `raw_today` and `raw_tomorrow` attributes (list of `{start, value}` dicts where `value` is in SEK/kWh).

#### Buy price — what you pay per kWh from grid

Includes spot price + grid fees + taxes + VAT.

Example for **Vattenfall SE3** (adjust for your grid operator):

```yaml
# In your Nordpool integration config:
sensor:
  - platform: nordpool
    region: SE3
    currency: SEK
    additional_costs: >
      {{ ((0.49 + 0.36 + 0.02 + 0.02 + (current_price * 0.225)) * 1.225) | float }}
```

This gives entity: `sensor.nordpool_kwh_se3_sek_4_10_0` (naming depends on your config)

**Fee breakdown (Vattenfall SE3):**
- 0.49 SEK — energy tax (energiskatt)
- 0.36 SEK — grid fee (nätavgift effektdel)
- 0.02 SEK — electricity certificate (elcertifikat)
- 0.02 SEK — origin guarantee (ursprungsgaranti)
- `current_price × 0.225` — VAT on spot price (25% VAT × 0.9 factor)
- `× 1.225` — 25% VAT on all components

#### Sell price — what you receive per kWh exported

Spot price plus/minus any compensation or fees.

Example for **Vattenfall SE3**:

```yaml
sensor:
  - platform: nordpool
    region: SE3
    currency: SEK
    additional_costs: >
      {{ (0.104 - 0.04) | float }}
```

This gives entity: `sensor.nordpool_kwh_se3_sek_5_00_0`

**Adjustment breakdown:**
- +0.104 SEK — grid benefit (nätnytta)
- −0.040 SEK — selling fee

> **Important:** Update the entity names in `youems_savings.yaml` to match your actual Nordpool sensor names.

---

## Installation

### 1. Copy the package file

```bash
cp youems_savings.yaml /config/integrations/youems_savings.yaml
```

Ensure your `configuration.yaml` includes the integrations folder as packages:

```yaml
homeassistant:
  packages: !include_dir_named integrations/
```

### 2. Seed the previous readings

Before the first slot runs, set these three entities to their **current meter values** in  
Developer Tools → States → search → `set_value`:

| Entity | Set to current value of |
|--------|------------------------|
| `input_number.youems_savings_export_prev` | `sensor.solax_inverter_grid_export_total` |
| `input_number.youems_savings_import_prev` | `sensor.solax_inverter_grid_import_total` |
| `input_number.youems_savings_house_prev` | `sensor.solax_inverter_house_energy_total` |

Without seeding, the first slot will compute a huge delta from 0 and give incorrect results.

### 3. Reload YAML

Developer Tools → YAML → Reload All YAML (or restart HA)

### 4. Add the dashboard card

Copy the contents of `youems_savings_dashboard.yaml` into a new manual card in your HA dashboard, or add the glance card to an existing dashboard.

---

## Entities Created

### Template sensors (read-only, recorder tracked)

| Entity | Description |
|--------|-------------|
| `sensor.youems_savings_today` | Savings since midnight |
| `sensor.youems_savings_yesterday` | Savings 00:00-23:45 yesterday |
| `sensor.youems_savings_this_month` | Savings since 1st of this month |
| `sensor.youems_savings_last_month` | Savings for previous calendar month |
| `sensor.youems_savings_total` | All-time cumulative savings |

### Helper entities (backing store — do not edit during operation)

| Entity | Description |
|--------|-------------|
| `input_number.youems_savings_*_kr` | Backing accumulators (5 entities) |
| `input_number.youems_savings_*_prev` | Previous meter readings (3 entities) |

---

## Adapting to Other Inverters / Meters

Replace the three sensor names in `youems_savings.yaml` with your own:

```yaml
export_now: "{{ states('sensor.YOUR_GRID_EXPORT_TOTAL') | float(0) }}"
import_now: "{{ states('sensor.YOUR_GRID_IMPORT_TOTAL') | float(0) }}"
house_now:  "{{ states('sensor.YOUR_HOUSE_ENERGY_TOTAL') | float(0) }}"
```

And the two Nordpool sensor names:

```yaml
state_attr('sensor.YOUR_BUY_PRICE_SENSOR', 'raw_today')
state_attr('sensor.YOUR_SELL_PRICE_SENSOR', 'raw_today')
```

---

## Troubleshooting

**Savings always 0:** Check that your grid meter sensors are reporting non-zero values and that the `_prev` entities have been seeded.

**Large spike at startup:** The `_prev` entities were at 0 — reseed them and reset the accumulators to 0.

**Wrong price used:** Verify `raw_today` attribute exists on your Nordpool sensors using Developer Tools → States → click sensor → Attributes tab.

**Month not resetting:** The rollover only runs at `00:00` on the 1st of the month. If HA was down at that moment, trigger the automation manually.

---

## Part of YOUEMS

This package is extracted from [YOUEMS](https://github.com/sixuniform/youems) — a full battery scheduler and inverter control system for SolaX/Solinteg inverters with Pylontech batteries and Home Assistant.
