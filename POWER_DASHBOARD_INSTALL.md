# Power Monitoring Dashboard - Installation Guide

## Required Custom Cards

Install these custom cards via HACS before using the dashboard:

### 1. Mushroom Cards (Required)
- **HACS URL:** https://github.com/piitaya/lovelace-mushroom
- **Installation:**
  1. Open HACS -> Frontend
  2. Click "+ Explore & Download Repositories"
  3. Search for "Mushroom"
  4. Click "Mushroom Cards" and install

### 2. ApexCharts Card (Required)
- **HACS URL:** https://github.com/RomRider/apexcharts-card
- **Installation:**
  1. Open HACS -> Frontend
  2. Search for "ApexCharts"
  3. Install "ApexCharts Card"

### 3. Card-mod (Required)
- **HACS URL:** https://github.com/thomasloven/lovelace-card-mod
- **Installation:**
  1. Open HACS -> Frontend
  2. Search for "card-mod"
  3. Install it

### 4. Vertical Stack In Card (Required)
- **HACS URL:** https://github.com/ofekashery/vertical-stack-in-card
- **Installation:**
  1. Open HACS -> Frontend
  2. Search for "Vertical Stack In Card"
  3. Install it

## Dashboard Installation

1. Install all required custom cards (see above)
2. Restart Home Assistant after installing all cards
3. The Power Monitor dashboard should appear in the sidebar
4. If not visible, check `configuration.yaml` for the lovelace dashboard config

## Dashboard Features

### Hourly Average Tracking
- Current hour average power
- Projected hourly average (with dynamic trigger %)
- Time remaining in hour
- Dynamic trigger threshold

### Quick Stats Row
- Current instantaneous power
- 5-minute smoothed power (used for projections)
- Last hour average
- Shedding status indicator

### Fastledd (Monthly)
- Today's dognmaks (highest hourly avg)
- Fastledd average (top 3 peaks)
- Current billing tier
- Top 3 peak chips with dates

### Controls
- Power threshold (+/- 100W buttons)
- Spike duration assumption (+/- 5 min)
- Load shedding enable/disable
- Smart shedding mode toggle
- Auto-restore toggle

### Device Management
- Per-device load shedding toggles
- Priority sliders (1-10)
- Power draw configuration
- Device status display

## Troubleshooting

**If cards don't appear:**
1. Make sure all custom cards are installed
2. Restart Home Assistant
3. Clear browser cache (Ctrl+F5)
4. Check browser console for errors

**If templates show errors:**
1. Make sure `sensor.power_usage_status` exists
2. Restart Home Assistant after installing the power_management.yaml package
3. Check that all entity names match your setup

**If sensors show "unavailable":**
1. Check that the utility meter `sensor.hourly_energy_consumption` exists
2. Verify your power meter entity (e.g., `sensor.aidon_active_power_import`)
3. Wait a few minutes for statistics sensors to populate

## Files

| File | Description |
|------|-------------|
| `/config/packages/power_management.yaml` | All sensors, helpers, utility meters |
| `/config/automations.yaml` | Load shedding automations |
| `/config/lovelace/dashy.yaml` | Dashboard configuration |

## Documentation

- **POWER_MANAGEMENT_GUIDE.md** - Comprehensive system guide
- **ANTI_CYCLING_GUIDE.md** - Anti-cycling protection details
- **POWER_DASHBOARD_INSTALL.md** - This file
