# Deye SUN-6K-OG01 + Home Assistant: monitoring, grid outage alerts, grid input protection

[Українська](README.md) | **English**

How my **Deye SUN-6K-OG01LP1-EU-AM2** off-grid inverter with 48 V batteries is connected to Home Assistant. What's included: grid lost/restored notifications with a chronology and a battery runtime estimate, outage statistics, protection of the grid input against overload while charging from the grid, electricity cost with day/night tariffs, and two dashboards.

Everything runs locally through the inverter's Wi-Fi logger (Modbus), without the Solarman cloud.

> Entity names and texts are in Ukrainian («Мережа» = grid, «Батарея» = battery, «Без світла» = without power, «Відключення» = outage). Entity IDs are transliterated.

---

## 1. Hardware

| What | Details |
|---|---|
| Inverter | Deye **SUN-6K-OG01LP1-EU-AM2**: off-grid, 6 kW, 48 V, max charge/discharge current 135 A, 2 MPPT, single phase |
| Batteries | 3 × Deye 5 kWh Pro B (100 Ah each), 300 Ah ≈ **15.4 kWh** total, BMS over the Pylon protocol |
| PV | 5.46 kWp, azimuth 142°, tilt 25° |
| Link | Solarman Wi-Fi logger stick on the LAN, Modbus TCP |
| House supply | 3 phases; the inverter is on one phase with a 32 A breaker (≈ 7.4 kW) |
| Server | single-board computer with Armbian, Home Assistant in Docker |

## 2. Integration

- **[ha-solarman](https://github.com/davidrapan/ha-solarman)** (davidrapan), installed via HACS. Version 26.01.31 at the time of writing.
- Settings: logger IP, `lookup_file: deye_hybrid.yaml`, `mppt: 2`, `phase: 1`.
- There is no dedicated definition file for the OG01, but `deye_hybrid.yaml` works: the device type reads as "Off-Grid Inverter", and the main sensors are correct (grid, load, PV, battery, temperature, energy counters).
- It provides `sensor.deye_*`, `binary_sensor.deye_grid`, `number.deye_*` (charge currents, capacity, shutdown SOC), `select.deye_time_of_use`.
- The built-in **Forecast.Solar** integration is used for the production forecast.

## 3. What's in the package

Everything is in one file, `homeassistant/packages/deye.yaml`:

| What | Entity / automation |
|---|---|
| Grid charge current slider 0–120 A | `number.deye_zariad_vid_merezhi` (template over the raw `number.deye_battery_grid_charging_current`, whose scale is 0–240) |
| 3-day outage log | `sensor.merezha_khronologiia` (attribute `events`: time, on/off, SOC %, duration) |
| Outages today | `sensor.vidkliuchen_za_sogodni` (blips < 3 min are not counted) |
| Without power today / yesterday / 7 days / month | `history_stats` + `utility_meter` on `binary_sensor.deye_grid` |
| Battery runtime | `sensor.batareia_vistachit_na`, hours |
| Average load over 5 and 30 min | `statistics` on `sensor.deye_load_power` |
| Grid import day/night, monthly production | `utility_meter` with tariffs |
| Grid cost (day, night, total) and solar savings | template sensors × prices `input_number.tariff_price_day/night` |
| Grid notifications | `deye_grid_battery_alerts` |
| Heatsink overheat, battery 100 % | `deye_radiator_overheat_2026`, `deye_battery_full_2026` |
| Tariff switching | `grid_tariff_switch_2026` (23:00 night, 07:00 day, and at HA start) |
| Grid input protection | `deye_grid_input_protection` + `deye_grid_input_protection_restore` |
| Quiet morning | `deye_quiet_morning`: 06:00–09:30 charge current 2 A, then 100 A |

## 4. Grid notifications

When the grid goes down (after a 30 s delay, to ignore flicker), a high-priority push arrives on the phone and the same message appears in HA:

```
⚠️ Мережа вимкнена о 18:44                    (Grid off at 18:44)
Живлення будинку перейшло на батарею. Заряд: 80%   (House is on battery. SOC: 80%)
🔋 Вистачить ~19 год 24 хв при ~607 Вт        (Enough for ~19 h 24 min at ~607 W)

Хронологія:                                   (Chronology)
📅 Сьогодні, 25.10                             (Today)
без відключень                                (no outages)

📅 Вчора, 24.10                                (Yesterday)
⚡ 04:32 блимання 1 хв                         (blip 1 min)
🔴 08:05 вимкнено · 🔋100%                      (off)
🟢 12:07 увімкнено · 🔋95% · 4 год 2 хв          (on, outage lasted 4 h 2 min)
🔴 18:44 вимкнено · 🔋80%
🟢 22:34 увімкнено · 🔋50% · 3 год 49 хв
⏱ без світла 7 год 53 хв · відключень: 2 · блимань: 1   (total 7 h 53 min, 2 outages, 1 blip)
```

- When power returns: "✅ Grid restored at …, no power for N h M min", with the same chronology. If the outage was shorter than 3 min, the title is "⚡ Short blip".
- All grid notifications share one `tag`, so a new one replaces the previous one instead of piling up.
- If SOC drops below 20 % with no grid: "🔋 Critically low charge!".
- The chronology lives in an attribute of the trigger-template sensor `sensor.merezha_khronologiia` and survives HA restarts. It can be seeded from history with the `grid_log_seed` event (`data.events` = list of events).

**Battery runtime:**

```
(SOC − shutdown SOC) / 100 × capacity (Ah × 51.2 V) / average load over 5 min
```

The 5-minute average keeps a kettle or water heater from collapsing the estimate. Capacity comes from `number.deye_battery_capacity` (Ah), because `sensor.deye_battery_capacity` does not update in the integration.

## 5. Grid input protection

The inverter is on one phase with a 32 A breaker, i.e. ≈ 7.4 kW. When it charges the battery from the grid at a high current and something powerful is switched on in the house at the same time, the input can be overloaded.

- If the grid draw is **> 6 kW for 5 s**, the grid charge current is reduced to 20 A. If it is still > 6 kW after 15 s, it is reduced to 0.
- The previous value is saved in `input_number.deye_grid_charge_saved`.
- When the load plus the restored charge (`(saved − current) × voltage × 1.08`) stays **< 5.5 kW for 1 min**, the current is restored.
- Every step is written to the HA logbook ("Захист вводу"). Moving the charge slider by hand cancels the protection.

## 6. Dashboards

- `homeassistant/dashboards/inverter.json`: inverter. Energy flow (sunsynk-power-flow-card), controls (TOU, grid charge slider), PV1/PV2 strings, grid and output (voltage, current, frequency), battery, temperature, production forecast, day/night import.
- `homeassistant/dashboards/energy.json`: energy. Flows (power-flow-card-plus), production, daily battery charge and discharge, grid import, consumption, production forecast, battery and connection alarms.

HACS cards needed: `sunsynk-power-flow-card`, `power-flow-card-plus`, `apexcharts-card`, `mushroom`.

## 7. Installation

1. Install **ha-solarman** via HACS and add the inverter (logger IP, `deye_hybrid.yaml`, MPPT 2, phase 1). Check that `sensor.deye_battery`, `sensor.deye_grid_power` and `binary_sensor.deye_grid` exist.
2. Copy `homeassistant/packages/deye.yaml` to `config/packages/` and add to `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
3. In the package, replace `notify.mobile_app_YOUR_PHONE` with your notify service. Adjust the protection thresholds (6000 / 5500 W) to your breaker.
4. Restart HA and set the tariff prices in `input_number.tariff_price_day` / `tariff_price_night`.
5. Dashboards: "New dashboard → Edit → Raw configuration editor", paste the JSON (JSON is valid YAML).

If your solarman entity IDs differ, change them in the package.

## 8. Notes

1. **TOU (Time of Use).** The TOU registers read plausibly on the OG01, but I haven't fully verified this feature. By default all 6 programs have SOC 5 %: enabling TOU as is may drain the battery almost completely. Set the program SOCs first.
2. **`sensor.deye_battery_capacity`** (kWh from the BMS) does not update, so you can't count batteries with it. Use `number.deye_battery_capacity` (Ah) and the Deye app.
3. The raw `number.deye_battery_grid_charging_current` has a 0–240 scale, while the OG01 physically handles up to 135 A. That's why control goes through the 0–120 A slider.
4. My own notifications also say whether an outage follows the DTEK schedule or is an emergency. That part depends on a separate outage-schedule system and is not in the package.

---

Questions and additions are welcome in Issues. I'd like to compare notes with other Deye OG01 / SG models.

My other project: [Protherm Puma Condens + eBUS + Home Assistant](https://github.com/aviator1989/protherm-puma-ebus-ha).
