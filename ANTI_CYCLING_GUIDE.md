# Anti-Cycling Protection Guide

## Overview

The anti-cycling protection system prevents devices from being turned on and off too rapidly, which can damage equipment and reduce efficiency. This feature was inspired by the [HA-PowerManager](https://codeberg.org/rsolva/HA-PowerManager) project.

## Key Features

### 1. Minimum Off-Time Protection
Each device has a configurable minimum off-time that must elapse before it can be turned back on.

**Default Values:**
- **Varmtvannsbereder** (Water Heater): 30 minutes
- **Oljeovn Stikk** (Oil Heater): 60 minutes
- **Fryser** (Freezer): 120 minutes
- **Vaskemaskin** (Washing Machine): 90 minutes
- **Hot Tub Heater**: 60 minutes

### 2. Cycle Counter with Backoff
If a device is shed multiple times within a short period (10 minutes), the system adds a backoff delay:
- **Backoff Formula**: `minimum_off_time + (cycle_count × 10 minutes)`
- **Maximum Cycle Count**: 12 (adds up to 2 hours of backoff)

**Example:**
- Oil Heater minimum off-time: 60 minutes
- First shed: Must wait 60 minutes
- Second shed within 10 min: Must wait 70 minutes (60 + 10)
- Third shed within 10 min: Must wait 80 minutes (60 + 20)

### 3. Automatic Counter Reset
Cycle counters automatically reset to 0 after 2 hours of stability (no shedding).

## How It Works

### Tracking System

**Input Helpers Created:**

1. **Minimum Off-Time** (5 input_numbers):
   - `input_number.min_off_time_varmtvannsbereder`
   - `input_number.min_off_time_oljeovn_stikk`
   - `input_number.min_off_time_fryser`
   - `input_number.min_off_time_vaskemaskin`
   - `input_number.min_off_time_layzspa_heat`

2. **Cycle Counters** (5 input_numbers):
   - `input_number.cycle_count_varmtvannsbereder`
   - `input_number.cycle_count_oljeovn_stikk`
   - `input_number.cycle_count_fryser`
   - `input_number.cycle_count_vaskemaskin`
   - `input_number.cycle_count_layzspa_heat`

3. **Last Shed Times** (5 input_datetimes):
   - `input_datetime.last_shed_varmtvannsbereder`
   - `input_datetime.last_shed_oljeovn_stikk`
   - `input_datetime.last_shed_fryser`
   - `input_datetime.last_shed_vaskemaskin`
   - `input_datetime.last_shed_layzspa_heat`

4. **Last Restore Times** (5 input_datetimes):
   - `input_datetime.last_restore_varmtvannsbereder`
   - `input_datetime.last_restore_oljeovn_stikk`
   - `input_datetime.last_restore_fryser`
   - `input_datetime.last_restore_vaskemaskin`
   - `input_datetime.last_restore_layzspa_heat`

5. **Protection Toggle** (1 input_boolean):
   - `input_boolean.anti_cycling_protection`

### Automations

#### 1. Record Shed Times
**Automation ID**: `1767180000000`

**Triggers**: When any monitored device turns OFF

**Actions**:
1. Records current timestamp to `last_shed_*` datetime helper
2. Checks if device was shed < 10 minutes ago
3. If yes, increments `cycle_count_*` (max 12)

**Example Logic**:
```yaml
Device turns OFF at 14:00
→ Records 14:00 to last_shed_varmtvannsbereder
→ Checks previous shed time: 13:58 (2 minutes ago)
→ Rapid cycling detected!
→ Increments cycle_count_varmtvannsbereder from 0 to 1
```

#### 2. Reset Cycle Counters
**Automation ID**: `1767180100000`

**Triggers**: Every 30 minutes (time pattern)

**Actions**:
1. For each device, checks time since last shed
2. If > 2 hours (120 minutes) AND cycle_count > 0
3. Resets cycle_count to 0

**Example Logic**:
```yaml
Last shed: 12:00
Current time: 14:35
Time elapsed: 155 minutes > 120 minutes
Cycle count: 3
→ Reset cycle_count to 0
```

#### 3. Record Restore Times
**Automation ID**: `1767180200000`

**Triggers**: When any monitored device turns ON

**Actions**:
1. Records current timestamp to `last_restore_*` datetime helper

#### 4. Updated Restore Devices Logic

The existing "Power Usage Monitor - Restore Devices" automation now includes anti-cycling checks:

**For Each Device Being Restored**:
1. Check if anti-cycling protection is enabled
2. Get last shed time
3. Get minimum off-time
4. Get cycle count
5. Calculate required off-time: `min_off + (cycle_count × 10)`
6. Calculate actual time off: `now - last_shed`
7. Only restore if: `time_off >= required_off_time`

**Example**:
```yaml
Device: Oljeovn Stikk (Oil Heater)
Last shed: 14:00
Min off-time: 60 minutes
Cycle count: 2
Required off-time: 60 + (2 × 10) = 80 minutes
Current time: 15:10 (70 minutes elapsed)
→ NOT READY - needs 10 more minutes
Current time: 15:20 (80 minutes elapsed)
→ READY TO RESTORE
```

## Configuration

### Enable/Disable Protection

Toggle in dashboard or via:
```yaml
input_boolean.anti_cycling_protection: on/off
```

**When OFF**: Devices restore immediately (no minimum off-time checks)
**When ON**: All anti-cycling logic applies

### Adjust Minimum Off-Times

Set appropriate values based on your equipment:

**Compressor-Based Devices** (Freezer, AC):
- Longer off-times protect compressor
- Recommended: 90-120 minutes

**Heating Elements** (Water Heater, Oil Heater):
- Medium off-times prevent thermal cycling
- Recommended: 30-60 minutes

**Other Devices** (Washing Machine):
- Can typically cycle faster
- Recommended: 30-90 minutes

### View Current Status

Check device history in Developer Tools → States:
```
input_datetime.last_shed_varmtvannsbereder: "2025-01-08 14:23:15"
input_datetime.last_restore_varmtvannsbereder: "2025-01-08 15:05:30"
input_number.cycle_count_varmtvannsbereder: 2
input_number.min_off_time_varmtvannsbereder: 30
```

## Real-World Example

### Scenario: High Power Usage on Cold Day

**Timeline:**

**13:00** - Power spikes to 6.5 kW (threshold: 5 kW)
- System sheds Oil Heater (priority 1)
- Records: `last_shed_oljeovn_stikk = 13:00`
- Cycle count: 0

**13:20** - Power drops to 4 kW, auto-restore triggers
- Time since shed: 20 minutes
- Required: 60 minutes (min_off_time)
- **NOT RESTORED** - needs 40 more minutes

**13:55** - Another power spike to 6.2 kW
- System sheds Oil Heater again
- Previous shed: 13:00 (55 minutes ago, > 10 min)
- **No cycle increment** (previous shed was long ago)
- Records: `last_shed_oljeovn_stikk = 13:55`

**14:05** - Power spike to 6.8 kW
- System sheds Oil Heater AGAIN
- Previous shed: 13:55 (10 minutes ago, = 10 min threshold)
- **Cycle count INCREMENTED** to 1
- Records: `last_shed_oljeovn_stikk = 14:05`

**14:25** - Power normalizes to 4.2 kW
- Time since shed: 20 minutes
- Required: 60 + (1 × 10) = 70 minutes
- **NOT RESTORED** - needs 50 more minutes

**15:15** - Still at 4.2 kW
- Time since shed: 70 minutes
- Required: 70 minutes
- **RESTORED** - minimum time met!

**17:30** - Counter reset automation runs
- Last shed: 14:05
- Time elapsed: 205 minutes > 120 minutes
- Cycle count reset: 1 → 0

## Dashboard Integration

Add anti-cycling status to your dashboard:

### Cycle Count Display
```yaml
- type: entities
  title: Anti-Cycling Status
  entities:
    - entity: input_boolean.anti_cycling_protection
      name: Protection Enabled
    - type: divider
    - entity: input_number.cycle_count_varmtvannsbereder
      name: Water Heater Cycles
    - entity: input_number.cycle_count_oljeovn_stikk
      name: Oil Heater Cycles
    - entity: input_number.cycle_count_layzspa_heat
      name: Hot Tub Cycles
    - entity: input_number.cycle_count_vaskemaskin
      name: Washer Cycles
    - entity: input_number.cycle_count_fryser
      name: Freezer Cycles
```

### Last Shed Times
```yaml
- type: entities
  title: Last Device Shed Times
  entities:
    - entity: input_datetime.last_shed_varmtvannsbereder
      name: Water Heater
    - entity: input_datetime.last_shed_oljeovn_stikk
      name: Oil Heater
    - entity: input_datetime.last_shed_layzspa_heat
      name: Hot Tub
    - entity: input_datetime.last_shed_vaskemaskin
      name: Washer
    - entity: input_datetime.last_shed_fryser
      name: Freezer
```

### Minimum Off-Time Configuration
```yaml
- type: entities
  title: Minimum Off-Time Settings
  entities:
    - entity: input_number.min_off_time_varmtvannsbereder
      name: Water Heater
    - entity: input_number.min_off_time_oljeovn_stikk
      name: Oil Heater
    - entity: input_number.min_off_time_layzspa_heat
      name: Hot Tub
    - entity: input_number.min_off_time_vaskemaskin
      name: Washer
    - entity: input_number.min_off_time_fryser
      name: Freezer
```

## Troubleshooting

### Device Not Restoring?

**Check:**
1. Is anti-cycling protection enabled?
   - `input_boolean.anti_cycling_protection` = ON
2. How long since last shed?
   - View `input_datetime.last_shed_*`
3. What's the cycle count?
   - View `input_number.cycle_count_*`
4. Calculate required time:
   - `min_off_time + (cycle_count × 10)`
5. Compare with time elapsed

**Force Restore:**
- Disable `input_boolean.anti_cycling_protection`
- Wait for auto-restore to trigger
- Re-enable protection

### Cycle Counter Stuck at High Value?

**Check:**
1. When was last shed?
   - Counter resets after 2 hours
2. Is reset automation enabled?
   - Check automation: `Power Management - Reset Cycle Counters`

**Manual Reset:**
```yaml
service: input_number.set_value
target:
  entity_id: input_number.cycle_count_oljeovn_stikk
data:
  value: 0
```

### Device Cycling Too Fast?

**Solutions:**
1. Increase `min_off_time_*` for that device
2. Ensure anti-cycling protection is enabled
3. Check that automations are running
4. Verify device priority settings

## Benefits

### Equipment Protection
- Prevents compressor damage in freezers/AC units
- Reduces thermal stress on heating elements
- Extends device lifespan

### Efficiency
- Avoids startup inefficiencies
- Reduces power spikes from repeated startups
- More stable overall power consumption

### System Stability
- Prevents automation loops
- Reduces notification spam
- More predictable behavior

## Advanced: Template Sensors

Create a sensor to show devices blocked by anti-cycling:

```yaml
template:
  - sensor:
      - name: "Devices Awaiting Minimum Off-Time"
        state: >
          {% set devices = [] %}
          {% set anti_cycling = is_state('input_boolean.anti_cycling_protection', 'on') %}
          {% if anti_cycling %}
            {% if is_state('input_boolean.was_shed_varmtvannsbereder', 'on') %}
              {% set last_shed = states('input_datetime.last_shed_varmtvannsbereder') %}
              {% set min_off = states('input_number.min_off_time_varmtvannsbereder') | int %}
              {% set cycle = states('input_number.cycle_count_varmtvannsbereder') | int %}
              {% set required = min_off + (cycle * 10) %}
              {% set elapsed = ((as_timestamp(now()) - as_timestamp(last_shed)) / 60) | int if last_shed not in ['unknown', 'unavailable', ''] else 999 %}
              {% if elapsed < required %}
                {% set devices = devices + ['Varmtvannsbereder (' + (required - elapsed) | string + ' min)'] %}
              {% endif %}
            {% endif %}
            {# Repeat for other devices #}
          {% endif %}
          {{ devices | length }}
        attributes:
          devices: >
            {# Same logic but return full device list #}
```

## Summary

Anti-cycling protection makes your power management system:
- ✅ **Equipment-Safe**: Protects devices from rapid cycling
- ✅ **Intelligent**: Adapts backoff based on cycling frequency
- ✅ **Automatic**: Resets counters when stable
- ✅ **Configurable**: Adjust per-device settings
- ✅ **Transparent**: Full tracking and visibility

Your devices are now protected from harmful rapid cycling! 🛡️⚡
