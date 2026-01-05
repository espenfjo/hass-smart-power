# Power Management & Load Shedding Guide

## Overview

This power management system is designed for the **Norwegian electricity billing model** where your monthly "fastledd" (grid fee) is calculated based on the **average of your 3 highest hourly peaks** (døgnmaks) from different days in a month.

**Key Principle**: We care about **hourly averages**, not instantaneous power spikes. A 10-second spike to 8 kW doesn't matter if your hourly average stays under 5 kW.

---

## Norwegian Billing Model (Fastledd)

### How It Works

1. **Døgnmaks**: The highest hourly average power for each day
2. **Top 3 Peaks**: At month end, your 3 highest døgnmaks (from different days) are identified
3. **Fastledd Average**: These 3 peaks are averaged to determine your billing tier
4. **Tiers**: 0-2, 2-5, 5-10, 10-15, 15-20, 20-25 kW

### Example

| Day | Døgnmaks |
|-----|----------|
| Jan 3 | 4.2 kW |
| Jan 7 | 6.1 kW |
| Jan 12 | 5.3 kW |
| Jan 15 | 3.8 kW |
| Jan 20 | 5.8 kW |

**Top 3**: 6.1 + 5.8 + 5.3 = 17.2 kW
**Average**: 17.2 / 3 = **5.73 kW** → Tier 3 (5-10 kW)

---

## Smart Tier-Based Shedding

### The "Day Lost" Logic

If today's peak has already exceeded your target threshold, there's no point protecting it anymore. The system automatically shifts to protect the **next tier up**.

**Example**:
- Your target: 5 kW (`input_number.power_threshold`)
- Today's peak so far: 5.3 kW (already exceeded!)
- System now protects: 10 kW (next tier)

### Key Sensors

| Sensor | Description |
|--------|-------------|
| `sensor.next_tier_threshold` | Current protection target (adjusts if day is "lost") |
| `sensor.smart_shedding_required` | True/False - whether shedding is needed now |
| `sensor.fastledd_average` | Real-time average of top 3 peaks (includes today) |
| `sensor.fastledd_tier` | Current billing tier based on fastledd average |

---

## Projection System

### How Projections Work

The system predicts what your hourly average will be at the end of the current hour:

```
projected = (past_avg × minutes_elapsed) + (smoothed_power × spike_minutes) + (baseline × remaining_minutes)
            ─────────────────────────────────────────────────────────────────────────────────────────────────
                                                        60
```

### Anti-Spike Measures

#### 1. 5-Minute Smoothed Power
Instead of reacting to instantaneous power, we use a 5-minute rolling average (`sensor.power_5min_average`). This prevents the oven preheating from triggering shedding.

#### 2. Spike Duration Assumption
The system assumes high-power events only last for a configurable duration (default: 20 min), not the entire remaining hour.

**Example at minute 10**:
- Without spike limit: Assumes oven runs for 50 more minutes
- With 20-min spike limit: Assumes oven runs for 20 more minutes, then baseline

**Configure**: `input_number.spike_duration_assumption` (5-55 minutes)

#### 3. Dynamic Trigger Thresholds

The system becomes more tolerant as the hour progresses:

| Time in Hour | Trigger Threshold |
|--------------|-------------------|
| 0-15 min | 95% of target |
| 15-30 min | 97% of target |
| 30-45 min | 98% of target |
| 45-60 min | 99% of target |

**Why?** Early in the hour, there's more uncertainty. Late in the hour, the average is mostly locked in.

---

## Configuration

### Main Settings

| Helper | Description | Default |
|--------|-------------|---------|
| `input_number.power_threshold` | Target hourly average to protect | 5000 W |
| `input_number.spike_duration_assumption` | How long spikes are assumed to last | 20 min |
| `input_boolean.load_shedding_enabled` | Enable automatic load shedding | ON |
| `input_boolean.smart_shedding_mode` | Shed minimum devices (vs all) | ON |
| `input_boolean.auto_restore_devices` | Auto-restore when safe | ON |

### Device Configuration

For each device, configure:

1. **Enable for shedding**: `input_boolean.load_shedding_<device>`
2. **Priority** (1=first, 10=last): `input_number.priority_<device>`
3. **Expected power draw**: `input_number.power_draw_<device>`

### Fairness-Based Shedding

The system tracks how long each device has been shed in the last 24 hours. Devices that have been shed more get a "burden" penalty, making them less likely to be shed again.

---

## Key Sensors

### Hourly Tracking

| Sensor | Description |
|--------|-------------|
| `sensor.current_hour_average_power` | Average power this hour so far |
| `sensor.projected_hourly_power` | Predicted hourly average at :59 |
| `sensor.hourly_power_margin` | Watts remaining before exceeding threshold |
| `sensor.power_5min_average` | 5-minute smoothed power (for projections) |
| `input_number.last_hour_average_power` | Previous hour's final average |

### Døgnmaks & Fastledd

| Sensor/Helper | Description |
|---------------|-------------|
| `input_number.todays_peak_power` | Highest hourly avg today |
| `input_number.monthly_peak_1/2/3` | Top 3 peaks this month |
| `input_text.monthly_peak_1/2/3_date` | Dates of top 3 peaks |
| `sensor.fastledd_average` | Average of top 3 (includes today if applicable) |
| `sensor.fastledd_tier` | Current billing tier |

### Smart Shedding

| Sensor | Description |
|--------|-------------|
| `sensor.next_tier_threshold` | Current target (may shift if day lost) |
| `sensor.smart_shedding_required` | Whether shedding is currently needed |

**Attributes on `sensor.smart_shedding_required`**:
- `projected_w`: Current projection
- `next_threshold_w`: Current target
- `trigger_percentage`: Active trigger % (95-99%)
- `margin_to_threshold_w`: Buffer before shedding
- `reason`: Human-readable explanation

---

## Automations

### Hourly Cycle

| Time | Automation | Action |
|------|------------|--------|
| :00:05 | Save Last Hour Average | Stores previous hour's avg from utility meter `last_period` |
| :01:00 | Update Today's Peak | Compares last hour avg with today's peak, updates if higher |

### Daily Cycle

| Time | Automation | Action |
|------|------------|--------|
| 00:00:05 | Save Døgnmaks | Saves today's peak to monthly top 3 if applicable, resets today's peak |

### Monthly Cycle

| Time | Automation | Action |
|------|------------|--------|
| 1st @ 00:00:10 | Reset Monthly Peaks | Clears all monthly peaks for new billing period |

### Load Shedding

| Trigger | Automation | Action |
|---------|------------|--------|
| `smart_shedding_required` = True | Load Shedding | Sheds devices to get below dynamic trigger threshold |
| `smart_shedding_required` = False (2 min) | Restore Devices | Restores previously shed devices |

---

## Dashboard

The Power Monitor dashboard shows:

### Hourly Average Tracking
- Current hour average
- Projected hourly average (with trigger %)
- Time remaining in hour
- Dynamic trigger threshold

### Quick Stats
- Current instantaneous power
- 5-minute smoothed power
- Last hour average
- Shedding status

### Fastledd (Monthly)
- Today's døgnmaks
- Fastledd average (top 3)
- Current tier
- Top 3 peak chips with dates

### Controls
- Power threshold (+/- 100W buttons)
- Spike duration assumption (+/- 5 min buttons)
- Load shedding enable/disable
- Smart shedding mode toggle
- Auto-restore toggle

---

## Troubleshooting

### Shedding triggers too quickly?
1. Increase `spike_duration_assumption` (try 30 min)
2. Check that `sensor.power_5min_average` is working (should smooth spikes)
3. Verify trigger thresholds are at 95/97/98/99% (not older 85/90/95/98%)

### Today's peak not updating?
1. Check `input_number.last_hour_average_power` - should update at :00:05
2. Verify utility meter has `last_period` attribute
3. Peak only updates at :01:00 each hour

### Fastledd average seems wrong?
1. Check if today's peak is included (see `includes_today` attribute)
2. Verify monthly peaks are set correctly
3. Check dates on monthly peaks (should be different days)

### Devices not restoring?
1. Verify `auto_restore_devices` is ON
2. Check `smart_shedding_required` is False
3. Ensure `margin_to_threshold_w` > 500
4. Verify device was shed by system (not manually off)

---

## Files

| File | Contents |
|------|----------|
| `/config/packages/power_management.yaml` | All sensors, helpers, utility meters |
| `/config/automations.yaml` | Load shedding, peak tracking automations |
| `/config/lovelace/dashy.yaml` | Power Monitor dashboard |

---

## Summary

This system protects your electricity bill by:

1. **Tracking hourly averages** (not spikes)
2. **Predicting** where the hour is heading
3. **Smoothing** power readings to avoid false triggers
4. **Dynamically adjusting** thresholds based on time in hour
5. **Shifting targets** when a day is already "lost"
6. **Shedding intelligently** - minimum devices, fairness-based
7. **Auto-restoring** when safe

The goal: Keep your fastledd average in the lowest practical tier while maintaining comfort.
