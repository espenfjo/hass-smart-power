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

If today's **finalized peak** (from completed hours) has already exceeded your target threshold, there's no point protecting it anymore. The system automatically shifts to protect the **next tier up**.

**Important**: Only finalized hourly peaks trigger "day lost" - not mid-hour averages which can be misleading.

**Example**:
- Your target: 5 kW (`input_number.power_threshold`)
- Today's finalized peak: 5.3 kW (already exceeded!)
- System now protects: 10 kW (next tier)

### Key Sensors

| Sensor | Description |
|--------|-------------|
| `sensor.next_tier_threshold` | Current protection target (adjusts if day is "lost") |
| `sensor.smart_shedding_required` | True/False - whether shedding is needed now |
| `sensor.fastledd_average` | Average of top 3 finalized peaks |
| `sensor.fastledd_tier` | Current billing tier based on fastledd average |

---

## Budget System

### Remaining Hour Budget

The system calculates how much power you can use for the **rest of the hour** while staying under your threshold:

```
budget = (threshold × 60 - current_avg × minutes_elapsed) / minutes_remaining
```

**Example** at minute 34 with 6.39 kW current average:
- Total budget (5 kW threshold) = 5000 × 60 = 300,000 watt-minutes
- Used = 6390 × 34 = 217,260 watt-minutes
- Remaining = (300,000 - 217,260) / 26 = **3182W**

This means you need to average only 3.18 kW for the remaining 26 minutes to hit exactly 5 kW for the hour.

### Budget-Based Shedding

The system triggers shedding when:

1. **Projection-based**: `projected > threshold × trigger_pct` (traditional)
2. **Budget-based**: After minute 15, if `current_power > budget × 1.1` (10% over budget)

The smart mode calculates how much to reduce:
```
to_reduce = max(current - budget, projected - threshold, 0)
```

This ensures shedding happens when you're over budget, even if the projection is optimistic.

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

#### 4. Finalized Peaks Only

**Critical**: Sensors like `next_tier_threshold` and `fastledd_average` only use **finalized** hourly peaks from `input_number.todays_peak_power`, not the current hour average. This prevents mid-hour spikes from incorrectly marking the day as "lost".

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
| `input_boolean.anti_cycling_protection` | Prevent rapid on-off cycling | ON |

### Device Configuration

For each device, configure:

1. **Enable for shedding**: `input_boolean.load_shedding_<device>`
2. **Priority** (1=first, 10=last): `input_number.priority_<device>`
3. **Expected power draw**: `input_number.power_draw_<device>`

### Fairness-Based Shedding

The system tracks how long each device has been shed in the last 24 hours. Devices that have been shed more get a "burden" penalty, making them less likely to be shed again.

---

## Smart Anti-Cycling

### How It Works

When devices are shed and restored repeatedly, they accumulate a cycle count which adds backoff time before restoration. However, this is **bypassed when safe**:

| Condition | Behavior |
|-----------|----------|
| Budget > current × 1.5 | Skip backoff (plenty of headroom) |
| Minute < 15 | Skip backoff (early in hour, time to shed again if needed) |
| Otherwise | Apply backoff: `min(cycle_count × 10, 20)` minutes |

### Backoff Cap

Maximum backoff is capped at **20 minutes** regardless of cycle count. This prevents devices from being off for excessive periods.

---

## Key Sensors

### Hourly Tracking

| Sensor | Description |
|--------|-------------|
| `sensor.current_hour_average_power` | Average power this hour so far |
| `sensor.projected_hourly_power` | Predicted hourly average at :59 |
| `sensor.remaining_hour_budget` | Average power you can use for rest of hour |
| `sensor.hourly_power_margin` | Watts remaining before exceeding threshold |
| `sensor.power_5min_average` | 5-minute smoothed power (for projections) |
| `input_number.last_hour_average_power` | Previous hour's final average |

### Døgnmaks & Fastledd

| Sensor/Helper | Description |
|---------------|-------------|
| `input_number.todays_peak_power` | Highest **finalized** hourly avg today |
| `input_number.monthly_peak_1/2/3` | Top 3 peaks this month |
| `input_text.monthly_peak_1/2/3_date` | Dates of top 3 peaks |
| `sensor.fastledd_average` | Average of top 3 finalized peaks |
| `sensor.fastledd_tier` | Current billing tier |

### Smart Shedding

| Sensor | Description |
|--------|-------------|
| `sensor.next_tier_threshold` | Current target (based on finalized peaks) |
| `sensor.smart_shedding_required` | Whether shedding is currently needed |

**Attributes on `sensor.smart_shedding_required`**:
- `projected_w`: Current projection
- `next_threshold_w`: Current target
- `budget_w`: Remaining hour budget
- `current_power_w`: Current power usage
- `trigger_percentage`: Active trigger % (95-99%)
- `margin_to_threshold_w`: Buffer before shedding
- `over_budget`: True if current > budget
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
| `smart_shedding_required` = True | Load Shedding | Sheds devices based on `max(over_budget, over_projection)` |
| `smart_shedding_required` = False (2 min) | Restore Devices | Restores devices (with smart anti-cycling) |

---

## Dashboard

The Power Monitor dashboard shows:

### Hourly Average Tracking
- Current hour average
- Projected hourly average (with trigger %)
- Time remaining in hour
- Dynamic trigger threshold

### Remaining Hour Budget
- Budget left (kW you can use for rest of hour)
- Current usage
- Over/Under budget indicator
- Graph: Budget vs Current Power (this hour)

### Quick Stats
- Current instantaneous power
- 5-minute smoothed power
- Last hour average
- Shedding status

### Fastledd (Monthly)
- Today's døgnmaks (finalized)
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
3. Verify trigger thresholds are at 95/97/98/99%

### Shedding not triggering when over budget?
1. Check `sensor.smart_shedding_required` state and attributes
2. Verify `input_boolean.load_shedding_enabled` is ON
3. Check that devices have `input_boolean.load_shedding_*` ON
4. Verify devices are actually ON (can't shed what's already off)

### Today's peak not updating?
1. Check `input_number.last_hour_average_power` - should update at :00:05
2. Verify utility meter has `last_period` attribute
3. Peak only updates at :01:00 each hour

### Fastledd average seems wrong?
1. Only **finalized** peaks are included (not current hour average)
2. Check `todays_finalized_peak_w` attribute
3. Verify monthly peaks are set correctly

### Devices not restoring?
1. Verify `auto_restore_devices` is ON
2. Check `smart_shedding_required` is False
3. Ensure `margin_to_threshold_w` > 500
4. Check if budget is low (anti-cycling may delay restore)
5. Early in hour or high budget should restore immediately

### Threshold showing wrong value (e.g., 10kW instead of 5kW)?
1. Check `input_number.todays_peak_power` - this is the finalized peak
2. Only finalized peaks (not current hour avg) affect the threshold
3. If todays_peak > your threshold, system protects next tier up

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
2. **Budget-based shedding** - knows exactly how much power you can use
3. **Projection-based shedding** - predicts where the hour is heading
4. **Smoothing** power readings to avoid false triggers
5. **Dynamically adjusting** thresholds based on time in hour
6. **Using finalized peaks only** - mid-hour spikes don't mark day as "lost"
7. **Shifting targets** when a day is already "lost"
8. **Shedding intelligently** - minimum devices, fairness-based
9. **Smart anti-cycling** - quick restore when safe, protection when budget is tight

The goal: Keep your fastledd average in the lowest practical tier while maintaining comfort.
