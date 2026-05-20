# Sensorbox V2 — ESPHome Configuration Notes

This repository contains modified ESPHome configurations for the **3D Printer Emission Sensor Array (Sensorbox V2)** by Thomas Sanladerer ([@MadeWithLayers](https://www.printables.com/@MadeWithLayers)).

> **Original project:** [https://www.printables.com/model/1079858-3d-printer-emission-sensor-array-sensorbox-v2](https://www.printables.com/model/1079858-3d-printer-emission-sensor-array-sensorbox-v2)
>
> Full credit to Thomas for the outstanding hardware design, PCB layout, sensor selection and original firmware concept. This is an exceptional project and well worth building. Go give it a like on Printables!

---

## Files

| File | Description |
|------|-------------|
| `airquality01.yaml` | Non-LVGL version using ESPHome's native display lambda renderer |
| `airquality01_lvgl.yaml` | LVGL version using arc gauge widgets |

---

## Before You Flash

Before compiling either config, you will need to supply your own credentials. There are two ways to do this:

**Option A — Use a `secrets.yaml` file (recommended)**
Add the following entries to your ESPHome `secrets.yaml` file:

```yaml
wifi_ssid: "your_wifi_ssid"
wifi_password: "your_wifi_password"
api_encryption_key: "your_api_encryption_key"
ota_password: "your_ota_password"
```

Then update the config files to reference them:

```yaml
api:
  encryption:
    key: !secret api_encryption_key

ota:
  - platform: esphome
    password: !secret ota_password
```

**Option B — Enter credentials directly in the yaml**
Replace the placeholder values in the config file directly:

```yaml
api:
  encryption:
    key: "your_api_encryption_key_here"

ota:
  - platform: esphome
    password: "your_ota_password_here"
```

A new API encryption key can be generated from the ESPHome dashboard when creating a new device, or via the ESPHome CLI with `esphome generate-api-key`.

> **Note:** Do not commit your actual credentials to a public repository. If using Option B, ensure your yaml files are in `.gitignore` or that credentials have been removed before pushing.

### Altitude Calibration — QNH Pressure Correction

> ⚠️ **You must set your altitude before flashing** if you want accurate sea-level (QNH) pressure readings.

The BMP280 measures absolute station pressure at your physical location. To convert this to a sea-level equivalent (QNH) — which is what weather apps and meteorological services report — a correction based on your altitude is applied.

Both config files are set to **120m** (the author's installation altitude). Find the following line in whichever yaml you are using and change `120.0` to your actual altitude in metres:

```yaml
filters:
  - lambda: return x / pow(1.0 - (120.0 / 44330.0), 5.255);
```

For example, for an altitude of 50m:
```yaml
filters:
  - lambda: return x / pow(1.0 - (50.0 / 44330.0), 5.255);
```

If you are at sea level, you can remove the filter entirely or set the value to `0.0`. If you don't know your altitude, any of these methods work reliably:

- [whatismyelevation.com](https://www.whatismyelevation.com) — shows your elevation automatically using your browser's location
- [elevatr R package / open-elevation.com](https://api.open-elevation.com) — open API, no account needed
- Your phone's built-in compass or GPS app often shows altitude
- Search **"[your suburb] elevation"** on Google — for most locations this returns a direct answer in the search results even if Maps itself doesn't show it

---

## Non-LVGL Version (`airquality01.yaml`)

### Overview

This version uses ESPHome's native drawing API via a `lambda:` block in the `display:` component. All sensor readings are drawn as text with colour-coded values, divided into sections by horizontal lines.

### Changes from Original

#### Display Driver Migration
The original config used `platform: ili9xxx` which has been deprecated in favour of `platform: mipi_spi`. The display section has been updated accordingly:

```yaml
display:
  - platform: mipi_spi      # was: ili9xxx
    model: ILI9341
    data_rate: 40MHz         # explicit SPI rate — default was 10MHz
    transform:
      swap_xy: false         # all three transform options now required
      mirror_x: false
      mirror_y: true
    dimensions: 240x320      # inline format required by mipi_spi
```

**Key differences from `ili9xxx`:**
- All three `transform` options (`swap_xy`, `mirror_x`, `mirror_y`) must be explicitly declared — omitting any one causes a validation error
- `dimensions` must use the inline `WxH` format rather than a nested `height`/`width` block
- `data_rate` defaults to 10MHz in `mipi_spi` vs 40MHz in `ili9xxx` — explicitly setting 40MHz restores previous performance

#### Display Lambda Refactor
The original display lambda was functional but contained repeated code blocks. The refactored version introduces:

- A **`threshold_color` helper lambda** replacing 9 identical 4-line colour cascade blocks (one per sensor). Thresholds are now expressed as a single line per sensor
- A **`draw_section` helper lambda** replacing 3 identical section-drawing blocks (title, unit, value loop). Sections are now drawn with a single function call
- All sensor state variables are read once at the top of the frame rather than inline
- Unused variables (`h`, `col`, `title`, `unit`, `names[]`, `values[]`, `colors[]`, `count`) removed

#### BMP280 QNH Pressure Correction
The BMP280 reports absolute station pressure. A sea-level correction filter has been added for an installation altitude of **120m**:

```yaml
pressure:
  name: "BMP280 Pressure (QNH)"
  filters:
    - lambda: return x / pow(1.0 - (120.0 / 44330.0), 5.255);
```

Adjust `120.0` to match your actual altitude in metres. The corrected value is directly comparable to Bureau of Meteorology QNH readings and weather station output.

#### Logger Level
Changed from `VERY_VERBOSE` to `DEBUG` for normal operation. `VERY_VERBOSE` generates significant log noise and has a minor performance impact on a busy device like this one.

#### SGP30 eCO2 Display
The SGP30 eCO2 estimate has been removed from the display (though it continues to be reported to Home Assistant). The SCD40 provides a real NDIR CO2 measurement — the SGP30 eCO2 is a correlation-based estimate derived from VOC levels and is not reliable as a standalone CO2 reading.

---

## LVGL Version (`airquality01_lvgl.yaml`)

### Overview

This version replaces the display lambda entirely with [LVGL (Light and Versatile Graphics Library)](https://esphome.io/components/lvgl/) widgets. Rather than redrawing the full screen on a timer, LVGL only redraws widgets that have changed — triggered directly by each sensor's `on_value:` event.

The display shows arc/gauge style indicators for each sensor reading, colour-coded using the same thresholds as the non-LVGL version.

### Layout

```
┌─────────────────────────┐
│ TEMP  HUM   QNH   DEW   │  ← colour-coded; datetime below
│ 22°C  54%  1031  13°C   │
├─────────────────────────┤
│ PARTICLES CUMUL  µg/m³  │  ← tap to toggle cumulative/diff
│ PM1   [████░░░░░░░]  9  │
│ PM2.5 [█████░░░░░░] 13  │  ← gradient bars: green→amber→red
│ PM10  [█████░░░░░░] 14  │
├─────────────────────────┤
│ CO2  ppm                │
│     [SCD40]  [ENS*]     │  ← arc gauges, ENS160 hidden
├─────────────────────────┤
│ VOC  ppb                │
│ SGP30  [████░░░░░░] 245 ppb │
│ CH2O   [██░░░░░░░░]  8µg/m³ │  ← gradient bars
│ ENS160 [hidden until connected] │
│ VOC Idx: ---  NOx Idx: ---    │  ← SGP41, hidden until connected
└─────────────────────────┘
```

*Hidden until sensor is physically connected*

### Key Features

#### Sensor-Driven Updates
Every sensor widget updates only when its sensor fires an `on_value:` event. No polling, no full redraws. This means:
- Arc value updates
- Label text updates
- Arc colour changes (white → yellow → orange → red → purple based on thresholds)

All happen instantly when new data arrives, independently of each other.

#### Hidden-Until-Connected Widgets
All sensor sections and individual arcs start with `hidden: true`. They reveal themselves automatically on first `on_value:` trigger. This means:

- **Currently disconnected sensors** (ENS160, SGP4x, SHT40) produce no visible empty space
- **Adding a new sensor** requires no config changes — plug it in and it appears on the display
- **Removing a sensor** causes its arc to simply remain hidden on next boot

The sensors currently confirmed connected on this hardware:

| Sensor | Interface | Status |
|--------|-----------|--------|
| AHT20 | I2C main (0x38) | ✅ Connected |
| BMP280 | I2C main (0x77) | ✅ Connected |
| SCD40 | I2C main (0x62) | ✅ Connected |
| SGP30 | I2C main (0x58) | ✅ Connected |
| SGP41 | I2C main (0x59) | ✅ Connected |
| PMSX003 | UART (GPIO7/8) | ✅ Connected |
| ZE08-CH2O | UART (GPIO9/10) | ✅ Connected |
| ENS160 + AHT20 | I2C secondary (0x53/0x38) | ✅ Connected |
| SHT40 | I2C main (0x44) | ❌ Not present |

#### Mixed Arc and Bar Layout
The display uses a mixed widget approach:

- **CO2 section** — arc gauges, giving a prominent visual indicator for the most important air quality metric
- **Particles and VOC sections** — horizontal bar widgets, providing space-efficient display of multiple sensors with name, bar and value on each row

#### Gradient Bars
All bar indicators use a green→amber→red horizontal gradient defined once in the `lvgl: gradients:` section:

```yaml
gradients:
  - id: bar_grad
    direction: hor
    stops:
      - color: 0x00C800   # green at 0%
      - color: 0xFF9900   # amber at 50%
      - color: 0xFF1616   # red at 100%
```

The gradient spans the full bar width but LVGL only renders up to the current value — so low readings show green, mid readings show green fading to amber, and high readings reveal the full spectrum. No per-update colour calculations needed.

#### Datetime Display
The datetime label is updated via the `time` component's `on_time_sync` and `on_time` triggers rather than being redrawn every frame:

```yaml
time:
  - platform: homeassistant
    id: esptime
    on_time_sync:
      then:
        - lvgl.label.update:
            id: lbl_datetime
            ...
    on_time:
      - seconds: 0
        minutes: /1
        then:
          - lvgl.label.update:
              id: lbl_datetime
              ...
```

#### Display Configuration
As with the non-LVGL version, the display requires `auto_clear_enabled: false` and `update_interval: never` — LVGL manages all rendering directly:

```yaml
display:
  - platform: mipi_spi
    id: main_display
    auto_clear_enabled: false
    update_interval: never
    ...
```

### Temperature, Humidity, Pressure and Dewpoint

The header row displays four values:

- **TEMP** — temperature from AHT20, colour-coded based on ASHRAE 55 home comfort range
- **HUM** — relative humidity from AHT20, colour-coded based on ASHRAE 62.1 and CIBSE Guide A
- **QNH** — sea-level corrected pressure from BMP280, colour-coded for weather indication
- **DEW** — dewpoint temperature calculated from AHT20 temp + humidity

#### Temperature thresholds (ASHRAE 55)

| Colour | Range | Meaning |
|--------|-------|---------|
| 🟢 Green | 19–26°C | ASHRAE 55 comfort zone |
| 🟡 Amber | 16–19°C or 26–30°C | Cool or warm but tolerable |
| 🔴 Red | <16°C or >30°C | Too cold or too hot |

#### Humidity thresholds (ASHRAE 62.1 / CIBSE Guide A)

| Colour | Range | Meaning |
|--------|-------|---------|
| 🟢 Green | 40–60% | Optimal indoor range |
| 🟡 Amber | 30–40% or 60–65% | Dry or humid but acceptable |
| 🔴 Red | <30% or >65% | Too dry (irritation, static) or mould risk |

#### QNH pressure thresholds

| Colour | Range | Meaning |
|--------|-------|---------|
| 🔴 Red | <980 hPa | Storm / severe weather approaching |
| 🟡 Amber | 980–1000 hPa | Low pressure, unsettled conditions |
| 🟢 Green | 1000–1030 hPa | Normal range, settled weather |
| 🟡 Amber | >1030 hPa | Very high pressure, very dry conditions |

QNH is the sea-level corrected barometric pressure — the same value reported by weather services and apps. A falling QNH indicates deteriorating weather; a rising QNH indicates improving conditions. Remember to set your altitude correctly (see [Altitude Calibration](#altitude-calibration--qnh-pressure-correction) above).

#### Dewpoint (DEW) thresholds

| Colour | Range | Meaning |
|--------|-------|---------|
| 🟡 Amber | <10°C | Dry — low humidity, possible irritation |
| 🟢 Green | 10–16°C | Comfortable indoor dewpoint range |
| 🟡 Amber | 16–21°C | Humid — getting uncomfortable |
| 🔴 Red | >21°C | Muggy — uncomfortably humid |

Dewpoint is a more reliable indicator of perceived humidity than relative humidity alone, as it is independent of temperature. A dewpoint above 21°C is generally considered uncomfortable regardless of the actual temperature.

Dewpoint is calculated on-device using the Magnus approximation:
```
dewpoint ≈ T - ((100 - RH) / 5)
```

Accurate to within 0.35°C for typical indoor temperature ranges.

---

### Colour Thresholds

Bar indicators use a continuous **green→amber→red gradient** that fills as values increase. Arc gauges (CO2 section) use a dynamic colour change at defined thresholds. Both approaches are based on indoor air quality guidelines for a home environment. Thresholds are intentionally tighter than general/outdoor or workplace standards, reflecting that occupants may have prolonged exposure in a living space.

| Colour | Meaning |
|--------|---------|
| 🟢 Green | Good — within recommended indoor limits |
| 🟡 Amber | Warning — elevated, investigate sources or increase ventilation |
| 🔴 Red | Dangerous — take action, ventilate immediately |

#### Thresholds by sensor

| Sensor | Green | Amber | Red | Reference |
|--------|-------|-------|-----|-----------|
| **PM1** (µg/m³) | 0–10 | 10–25 | >25 | WHO 2021 (mirrors PM2.5) |
| **PM2.5** (µg/m³) | 0–10 | 10–25 | >25 | WHO 2021 indoor guideline |
| **PM10** (µg/m³) | 0–20 | 20–45 | >45 | WHO 2021 indoor guideline |
| **CO2** (ppm) | 400–800 | 800–1200 | >1200 | ASHRAE / RESET Air standard |
| **TVOC** (ppb) | 0–300 | 300–500 | >500 | WELL Building Standard |
| **CH2O** (µg/m³) | 0–62 | 62–123 | >123 | WHO indoor guideline (≈80ppb) |
| **SGP41 VOC Index** | 0–100 | 100–300 | >300 | Sensirion VOC Index scale (100 = baseline) |
| **SGP41 NOx Index** | 0–100 | 100–300 | >300 | Sensirion NOx Index scale (100 = baseline) |

> **Note on PM1:** No official WHO standard exists specifically for PM1. The PM2.5 thresholds are applied as a conservative approximation given the smaller particle size.
>
> **Note on ZE08 display:** The ZE08-CH2O bar displays the µg/m³ value from the sensor packet rather than ppb. This is because some sensors produce a variable µg/m³ reading while the ppb output remains clamped at 5000 during extended burn-in. Both values are reported to Home Assistant. The WHO indoor guideline of 80ppb ≈ 98µg/m³ is used for the red threshold.
>
> **Note on formaldehyde:** Thresholds are especially relevant when monitoring a 3D printer enclosure, as several common filaments (particularly ABS, ASA and resin) are known sources of formaldehyde and VOC emissions.

---

## Particle Matter — Cumulative vs Differential Display

The PMS5003 reports **cumulative** particle concentrations, meaning each band includes all smaller particles:

- **PM1** = all particles ≤1.0µm
- **PM2.5** = all particles ≤2.5µm (includes PM1)
- **PM10** = all particles ≤10µm (includes PM2.5 and PM1)

This means PM10 ≥ PM2.5 ≥ PM1 at all times in cumulative mode, which is the industry standard way of reporting and the basis for all WHO and regulatory guidelines.

An alternative **differential** view is also available, which shows the actual particle size distribution:

- **PM1** = particles ≤1.0µm (unchanged)
- **PM1–2.5** = particles in the 1.0–2.5µm band only (PM2.5 − PM1)
- **PM2.5–10** = particles in the 2.5–10µm band only (PM10 − PM2.5)

> **Note for 3D printer monitoring:** Emissions from common filaments (particularly ABS, ASA and resin) are predominantly ultrafine particles in the PM0.1–PM1 range. In differential mode you will typically see most of the mass in the PM1 band when printing, with very little in the PM1–2.5 or PM2.5–10 bands. This is expected and confirms the ultrafine nature of 3D printer emissions.

> **Note on thresholds in differential mode:** The WHO/indoor air quality thresholds used for arc colouring are defined for cumulative values. In differential mode the arc colours remain active but the thresholds are less meaningful — treat them as relative indicators rather than absolute health limits when in differential mode.

### Switching modes

The display mode can be changed two ways:

1. **Home Assistant** — a switch entity named **PM Differential Mode** is exposed to HA. Toggle it from any HA dashboard or automation
2. **On-screen** — tap the `PARTICLES CUMUL` / `PARTICLES DIFF` heading label directly on the display

In differential mode the bar values change but the labels remain `PM1`, `PM2.5` and `PM10` — the section heading switches between `PARTICLES CUMUL` and `PARTICLES DIFF` to indicate the current mode.

Both methods are synchronised — toggling from HA updates the display label and arc names immediately, and vice versa.

---

## Hardware Notes

### mipi_spi Transform Requirements
Unlike `ili9xxx`, the `mipi_spi` driver requires **all three** transform options to be explicitly set. Missing any one will cause a validation error at compile time.

### SPI Data Rate
The default data rate for `mipi_spi` is 10MHz. This config explicitly sets `data_rate: 40MHz` which is reliable on the ESP32-S2 with PSRAM and noticeably improves display refresh performance given the volume of content being rendered.

### PSRAM
The ESP32-S2 Saola used in this build has 2MB quad-mode PSRAM. This must be explicitly declared:

```yaml
psram:
  mode: quad
```

Without this, the display buffer allocation will fail silently and produce a blank screen.

---

## Hardware Notes — ZE08 5V Power Bodge

The original Sensorbox V2 PCB routes **3.3V** to the ZE08-CH2O connector pin 4 (VIN). However the ZE08 datasheet specifies **5V** as the required supply voltage for correct electrochemical cell operation. Running the sensor on 3.3V may result in incorrect readings, slow or incomplete burn-in, or the sensor failing to release its warmup sentinel value.

> ⚠️ **If your ZE08 is not producing correct readings, check the supply voltage first.**

### Fix

The PCB provides several convenient 5V sources that can be used for the bodge wire:

- **Unused sensor headers** — any unpopulated sensor header that has a 5V pin can supply the wire
- **Power rail header rows** — the PCB has dedicated rows of through-holes for 5V, 3.3V and GND along the board edges, specifically intended for power distribution

Run a short bodge wire from any of these 5V sources to **pin 4 of the ZE08 connector**. Pin 4 is the VIN supply pin — verify against the ZE08 datasheet pinout before connecting. This overrides the 3.3V trace and supplies the correct voltage to the sensor.

This fix should be applied when building new boards — route the 5V bodge wire before populating any other components so it is easy to access.

> **Note:** Thomas Sanladerer has acknowledged the 3.3V routing was an oversight in the original PCB design. Future PCB revisions may correct this.

---

## Credits

**Original project design, PCB, enclosure and firmware concept:**
Thomas Sanladerer ([@MadeWithLayers](https://www.printables.com/@MadeWithLayers))
[https://www.printables.com/model/1079858-3d-printer-emission-sensor-array-sensorbox-v2](https://www.printables.com/model/1079858-3d-printer-emission-sensor-array-sensorbox-v2)

This is a truly excellent project. The PCB design in particular — breaking out all ESP32-S2 GPIO pins, supporting multiple interchangeable sensor configurations and fitting everything into a compact 3D-printed enclosure — makes it one of the most practical and well-executed open hardware air quality monitors available. Highly recommended.

**ESPHome configuration modifications:**
Developed with the assistance of [Claude](https://claude.ai) (Anthropic), an AI assistant. The modifications — including the LVGL display implementation, sensor threshold research, display driver migration, comfort index calculations, and various ESPHome-specific fixes — were developed collaboratively through an iterative session of hardware debugging, firmware development and testing.

If you find these configs useful, consider giving the original Sensorbox V2 project a like on Printables and checking out Thomas's YouTube channel [@MadeWithLayers](https://www.youtube.com/@MadeWithLayers).
