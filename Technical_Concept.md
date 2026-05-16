# Technical Concept: LEM-Netz

**Version:** 1.0  
**Status:** Draft  
**Input:** [LEM-Requirements.md](LEM-Requirements.md)  
**Supersedes:** LEM-Netz_Technical_Concept.md (deprecated)

---

## 1. Architecture Overview

Three-tier model connecting the neighborhood network to each household's internal automation:

```
Neighborhood HA ←MQTT/LAN→ Gateway ←LoRaWAN 868 MHz→ ESP32 ←MQTT/WiFi→ Household Automation
```

| Tier | Role | Hardware |
|---|---|---|
| **Neighborhood coordination** | Virtual transformer, watchdog, fair-share, optional scheduler | Central server (RPi 5 or HA Green) + LoRaWAN gateway |
| **Bridge per household** | Translates between LoRaWAN and local MQTT; reads smart meter via IR | ESP32 + LoRa radio + IR read head |
| **Household automation** | Optimizes own devices within received constraints; varies per household | Wallbox, battery BMS, heat pump controller, local HA, EVCC, etc. |

**Key principle:** The ESP32 never makes optimization decisions. It relays constraints downward and status upward. The household automation decides how to comply.

---

## 2. Hardware

### 2.1 Per Household — Bridge Device

| Component | Purpose | Cost |
|---|---|---|
| ESP32 (DevKit C or similar) | MCU, WiFi for local MQTT, UART for IR reader, SPI for LoRa | €12–18 |
| LoRa radio module (RFM95W / SX1276) | 868 MHz LoRaWAN transceiver | €8–15 |
| IR optical read head | Reads smart meter via SML/IEC 62056-21 | €5–10 |
| RS485 module (MAX3485) | Fallback: direct Modbus RTU to devices (if no household automation) | €3–5 |
| USB power supply + enclosure | Power and mounting | €8–15 |
| **Total per household** | | **€36–63** |

The ESP32 connects to the household's local WiFi for MQTT communication with the household automation. The household needs a local MQTT broker (many automation systems include one, or a lightweight broker runs on the ESP32 itself).

### 2.2 Central Infrastructure

| Option | Components | Total | Notes |
|---|---|---|---|
| **Indoor (≤€300)** | RPi 5 (4GB) + LoRa concentrator hat (e.g. RAK2287) | ~€180–220 | Fits budget; good for pilot and small deployments |
| **Outdoor basic (~€305)** | RPi 5 + Dragino DLOS8N gateway | ~€305 | IP65 outdoor-rated; moderate scale |
| **Outdoor pro (~€733)** | HA Green + Milesight UG67 | ~€733 | Best range, 2000+ nodes; exceeds €300 target |

The concept assumes the indoor RPi+concentrator option as default. Communities can choose to invest in an outdoor gateway if their geography requires it.

---

## 3. Communication

| Path | Protocol | Details |
|---|---|---|
| ESP32 ↔ LoRaWAN gateway | LoRaWAN 868 MHz | Uplink every 2 min (sensor data, headroom). Downlink (constraints, shed commands) |
| Gateway ↔ Central HA | MQTT over Ethernet LAN | Gateway forwards LoRaWAN payloads to MQTT topics and vice versa |
| ESP32 ↔ Household automation | MQTT over household WiFi | ESP32 publishes constraints to topics the automation subscribes to; automation publishes device status |

### 3.1 LoRaWAN Uplink (ESP32 → Gateway)

| Data | Interval | Size |
|---|---|---|
| Smart meter power (W) + energy (kWh) | Every 2 min | 4 bytes |
| Device status summary (from household MQTT) | Every 2 min | 4–8 bytes |
| Available headroom (`max_power - current_load`) | Every 2 min | 2 bytes |
| Shed confirmation | On event | 1 byte |

Total typical payload: 10–14 bytes per uplink.

### 3.2 LoRaWAN Downlink (Gateway → ESP32)

| Command | Size | Effect |
|---|---|---|
| `max_power` (W) | 2 bytes | ESP32 publishes this to household MQTT; automation must stay within limit |
| `shed` | 1 byte | ESP32 publishes emergency shed to household MQTT |
| `restore` | 1 byte | ESP32 publishes restore signal |
| Schedule params (Phase 2) | 8–16 bytes | Compact parameters (charge window, max rate, price threshold) |

### 3.3 Local MQTT Topics (Household WiFi)

| Topic | Direction | Payload |
|---|---|---|
| `lem/household/<id>/constraints/max_power` | ESP32 → automation | `{"value": 5000, "unit": "W"}` |
| `lem/household/<id>/constraints/shed` | ESP32 → automation | `{"reason": "transformer_overload", "level": 1}` |
| `lem/household/<id>/constraints/restore` | ESP32 → automation | `{}` |
| `lem/household/<id>/constraints/schedule` | ESP32 → automation | `{"windows": [{"start": "02:00", "end": "04:00", "max_power": 7000}]}` |
| `lem/household/<id>/status/power` | automation → ESP32 | `{"wallbox": 3000, "battery": -1500, "heatpump": 2000}` |
| `lem/household/<id>/status/flexibility` | automation → ESP32 | `{"can_reduce": 2000, "can_increase": 3000}` |
| `lem/household/<id>/status/shed_ack` | automation → ESP32 | `{"status": "executed"}` |

---

## 4. Phase 1 — Monitoring and Infrastructure Protection

### 4.1 Data Collection

The ESP32 reads the smart meter via the IR optical head (SML protocol). The household automation publishes device-level power data to local MQTT. The ESP32 bundles both into LoRaWAN uplinks every 2 minutes.

### 4.2 Virtual Transformer

The central HA sums all household power readings into a virtual transformer sensor:

```
virtual_transformer = sum(power_household_1 ... power_household_N)
```

### 4.3 Two-Stage Watchdog

The watchdog monitors the virtual transformer and escalates through two stages:

| Stage | Threshold | Action | Wait |
|---|---|---|---|
| **1 — Limit tightening** | 75% of transformer rating | Calculate fair-share `max_power` per household → publish via LoRaWAN downlink | 3 min |
| **2 — Direct shed** | Still > 75% after Stage 1, or > 95% immediately | Publish `shed` to all households | — |
| **Recovery** | Drops below 55% | Restore normal `max_power` limits, publish `restore` | — |

### 4.4 Fair-Share Calculation During Overload

During Stage 1, each household receives a proportional reduction:

```
household_limit = household_base × (transformer_limit × 0.75 / current_total)
```

All households are reduced proportionally — no household is singled out. This satisfies FR-06 during constraint mode (equal sacrifice).

### 4.5 Shed Priority

When the household automation receives a `max_power` constraint that its current load exceeds, it sheds loads in this fixed order (UC-04):

1. EV wallbox (pause charging)
2. Battery charging (stop charging; discharging is allowed)
3. Heat pump (reduce or pause)
4. Balcony solar (via smart plug, if reverse power flow limits are exceeded)

Essential loads (lights, electronics, kitchen) are never controlled.

---

## 5. Phase 2 — Control and Optimization

### 5.1 Flexibility Offer/Request

Each household automation publishes its flexibility to local MQTT:

```json
{
  "can_reduce": 2000,     // can reduce load by 2 kW
  "can_increase": 3000,   // can increase load by 3 kW (e.g., charge battery)
  "price_min": 0.08,      // minimum price to accept reduction (€/kWh)
  "price_max": 0.25       // maximum price to pay for additional power
}
```

The ESP32 forwards a compact summary in the next LoRaWAN uplink. The central coordinator sees available flexibility across the neighborhood and can match surplus to demand within transformer limits.

### 5.2 Day-Ahead Scheduler (Optional)

The central HA runs a global optimization (using HAEO or similar) with:
- EPEX Spot prices
- Transformer capacity constraint
- Household flexibility forecasts

Output: compact schedule parameters per household (charge windows, max rates, price thresholds) sent via LoRaWAN downlink. The household automation follows the schedule but can deviate. The watchdog always overrides in real-time if limits are breached.

### 5.3 §14a EnWG Support

| Module | Mechanism | LEM Integration |
|---|---|---|
| **Module 1** (flat reduction) | Fixed annual discount on network charges | Purely financial — no technical integration needed. HA stores per-household tariff config |
| **Module 2** (percentage reduction) | 60% off network charges for separately metered devices | Purely financial — no technical integration needed. Grid operator's emergency dimming is handled via separate Steuerbox, orthogonal to LEM |
| **Module 3** (time-variable) | Network charges vary by time of day (typically 3 tiers) | Scheduler accounts for time-variable network charges in its optimization. Requires iMSys at the household |

---

## 6. Balcony Solar (Balkonkraftwerk)

A balcony solar system with battery is handled like any other household device:

- The household automation manages the battery (charge from solar or grid when cheap, discharge when expensive)
- The ESP32 coordinates constraints via MQTT — no special treatment
- If reverse power flow limits are exceeded (UC-04), the automation curtails generation by stopping battery charging from solar or reducing inverter output

If the balcony solar has no battery and no Modbus interface, curtailment requires a smart plug that the system can switch. This is an optional add-on, not a core requirement.

---

## 7. Per-Household Configuration

Each household has a configuration stored in the central HA:

```yaml
household:
  id: "hh_01"
  tariff_type: "dynamic"           # fixed, dynamic, eeg_feedin, 14a_module1, 14a_module2, 14a_module3
  feed_in_rate: 0.08               # €/kWh (EEG feed-in)
  import_rate: 0.30                # €/kWh (fixed tariff, or base for dynamic)
  base_allocation: 5000            # W (normal max_power)
  devices:                         # what this household controls
    - wallbox
    - battery
    - heat_pump
```

This configuration is used by:
- The fair-share calculator (can weight differently by tariff type if desired)
- The scheduler (optimizes for each household's specific pricing model)
- The financial impact tracker (FR-06, Phase 2)

---

## 8. Offline Operation

| Component | Without Internet | Without LoRaWAN |
|---|---|---|
| Central HA | Full operation (prices except EPEX unavailable; watchdog works) | Status frozen at last uplink |
| ESP32 | Reads meter, polls automation MQTT, enforces last known `max_power` | Falls back to autonomous household automation (no coordination) |
| Household automation | Full operation | Full operation |

The system is designed for graceful degradation. No component depends on external cloud services for core operation.

---

## 9. Cost Summary

| Category | Target | | |
|---|---|---|---|
| Per-household (Phase 1) | €36–63 | ✓ Under €200 | Bridge device |
| Per-household (Phase 2) | €36–63 | ✓ Reuses same hardware | No upgrade cost |
| Central (indoor) | ~€180–220 | ✓ Under €300 | RPi 5 + concentrator |
| Central (outdoor) | ~€305–733 | ⚠ Exceeds target | Dragino or UG67 |
| Recurring | €0 | ✓ | All open-source, EPEX free |

---

## 10. Limitations and Open Points

- **§14a Module 2 emergency dimming:** The grid operator's Steuerbox is completely separate from LEM. If the grid operator dimms a device, LEM will see the reduced load in its data but cannot influence the decision
- **Financial fairness baseline (FR-06, Phase 2):** The reference point for "no financial disadvantage" needs definition when Phase 2 optimization is designed. Options: historical baseline, equal proportional share, or status quo baseline
- **Onboarding flow (UC-05):** Not detailed yet. Likely pre-flashed ESP32 with LoRaWAN OTAA auto-join + web UI for household registration
- **ESP32 firmware updates:** Not detailed yet. Likely USB reflash per household (acceptable for a community project with periodic maintenance days)
