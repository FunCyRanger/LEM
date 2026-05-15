# Technical Concept: LEM-Netz

## Implementation Guide for Phase 1 + Phase 2

**Version:** 3.1
**Status:** Review (incorporating open-source optimization recommendations)
**Purpose:** Translate Requirements into Implementation

---

## 1. Implementation Approach

This document provides a technical implementation path for Phase 1 (Data Collection) and Phase 2 (Control Integration) as defined in the Requirements Specification. It describes **how** to achieve the **what** defined in the requirements.

### 1.1 Design Principles

1. **Infrastructure Safety FIRST:** Never compromise grid stability; transformer limits are non-negotiable
2. **Requirements-Driven:** Every technical choice must satisfy a requirement
3. **Cost-Conscious:** Stay within budget targets (€300 central, €200/household Phase 1, €350 Phase 2)
4. **Community-Friendly:** Simple installation, minimal maintenance
5. **Vendor-Agnostic:** Avoid lock-in where possible
6. **Regulatory-Compliant:** Meet German CE, BSI, VDE, GDPR, §14a EnWG requirements
7. **Economically Fair:** Optimization must NOT disadvantage any household type (SUBORDINATE to #1)

---

## 2. Technical Architecture

### 2.1 Component Selection Criteria

Based on the requirements, the following technical categories are needed:

| Requirement | Technical Category | Key Specifications | Phase |
|-------------|-------------------|-------------------|-------|
| FR01, FR02 | Energy Sensor | Optical IR reader, SML/IEC62056-21 support | 1 |
| FR04, FR05 | Wireless Communication | 868 MHz, AES-128 encryption, ≥1km range | 1 |
| FR13-15 | Price Importer | EPEX Spot API, provider integrations | 2 |
| FR16-20 | Control Integration | Wallbox, smart plug, PV, battery APIs | 2 |
| NF01 | Outdoor Gateway | IP67 rated, weatherproof | 1 |
| FR07, FR09 | Central Server | MQTT broker, local database, dashboard | 1 |

### 2.2 System Diagram - Phase 1 + 2

```
┌─────────────────────────────────────────────────────────────────┐
│                     LEM-NETZ IMPLEMENTATION                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [House 1]           [House 2]           [House N]              │
│  ┌─────────┐        ┌─────────┐        ┌─────────┐            │
│  │Smart    │        │Smart    │        │Smart    │            │
│  │Meter    │        │Meter    │        │Meter    │            │
│  │  IR     │        │  IR     │        │  IR     │            │
│  └────┬────┘        └────┬────┘        └────┬────┘            │
│       │                  │                  │                   │
│       ▼                  ▼                  ▼                   │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  ENERGY SENSOR (per household)                          │     │
│  │  • Optical IR reader                                   │     │
│  │  • SML decoder                                         │     │
│  │  • Wireless transmitter (868 MHz)                     │     │
│  │  • Power: USB or battery                               │     │
│  └──────────────────────┬────────────────────────┬────────┘     │
│                         │  Wireless (868 MHz)     │               │
│                         │  Range: 1km+            ▼               │
│                    ┌────┴─────────────────────────┴────┐         │
│                    │      CONTROL DEVICES             │         │
│                    │  • Wallbox (EV charging)        │         │
│                    │  • Smart plugs                  │         │
│                    │  • PV inverter                 │         │
│                    │  • Battery BMS                 │         │
│                    └──────────────────────────────────┘         │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  OUTDOOR GATEWAY                                        │     │
│  │  • IP67 weatherproof enclosure                         │     │
│  │  • 8-channel wireless receiver                        │     │
│  │  • Built-in network server (optional)                  │     │
│  │  • Ethernet/PoE output                                 │     │
│  └──────────────────────┬────────────────────────────────┘     │
│                         │  Ethernet                              │
│                         ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  CENTRAL SERVER                                         │     │
│  │  ┌─────────────┐  ┌─────────────┐  ┌───────────────┐    │     │
│  │  │MQTT Broker │  │  Database   │  │  Dashboard   │    │     │
│  │  │(Mosquitto) │  │ (InfluxDB)  │  │    (HA)      │    │     │
│  │  └─────────────┘  └─────────────┘  └───────────────┘    │     │
│  │         │                │                │                │     │
│  │         ▼                ▼                ▼                │     │
│  │  ┌─────────────────────────────────────────────────────┐  │     │
│  │  │         OPTIMIZATION ENGINE                         │  │     │
│  │  │  ┌────────────────┐  ┌─────────────────────────┐   │  │     │
│  │  │  │ Price Import   │  │ Control Logic           │   │  │     │
│  │  │  │ (EPEX, Tibber) │  │ (Revenue-aware)         │   │  │     │
│  │  │  └────────────────┘  └─────────────────────────┘   │  │     │
│  │  └─────────────────────────────────────────────────────┘  │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Implementation Steps - Phase 1

### Phase A: Central Infrastructure Setup

#### A.1 Server Installation (30-45 minutes)

1. **Hardware Setup**
   - Deploy Home Assistant Green (or equivalent)
   - Connect via Ethernet to network router
   - Power on and wait for boot (5-10 minutes)

2. **Software Configuration**
   - Access via `homeassistant.local:8123`
   - Create admin account
   - Complete initial setup wizard

3. **MQTT Broker Installation**
   - Navigate to: Settings → Add-ons → Add-on Store
   - Install "Mosquitto broker"
   - Configure: Enable "Watchdog" and "Start on boot"
   - Start the broker

4. **MQTT Integration**
   - Navigate to: Settings → Devices & Services
   - Configure MQTT when discovered
   - Default credentials auto-created

#### A.2 Gateway Installation (45-60 minutes)

1. **Physical Installation**
   - Mount gateway at outdoor location (pole, wall)
   - Ensure clear line-of-sight to most households
   - Connect via Ethernet to PoE injector
   - Connect PoE injector to network

2. **Network Configuration**
   - Access gateway web interface
   - Configure as packet forwarder or built-in network server
   - Set up MQTT forward to central server IP (port 1883)
   - Note: DevEUI and AppEUI for device registration

### Phase B: Household Sensor Installation (30-45 minutes per household)

#### B.1 Prerequisites

1. **Check Meter Compatibility**
   - Verify meter has IR interface (standard on mME)
   - Identify IR window location (usually left side)

2. **Request Grid Operator PIN** (optional, for full data access)
   - Contact your Netzbetreiber (grid operator)
   - Request "Freischaltung der SML-Datenschnittstelle"
   - PIN will be sent by mail/email
   - Enter PIN at meter to enable all OBIS values

#### B.2 Sensor Installation

1. **Mount Sensor**
   - Clean meter glass at IR window
   - Attach magnetic sensor head
   - Route antenna outside metallic cabinet (required for RF)
   - Connect Micro-USB power

2. **Configure Sensor**
   - Use vendor configuration tool
   - Set transmission interval: ≤15 seconds (FR03)
   - Configure SML filter for:
     - OBIS 1-0:0.2.1.255 (current power)
     - OBIS 1-0:1.8.0.255 (cumulative energy)

3. **Register in Gateway**
   - Add device via OTAA (Over-The-Air Activation)
   - Enter DevEUI from sensor label
   - Enter AppKey from configuration
   - Device will join automatically

### Phase C: Data Verification (30 minutes)

#### C.1 Verify Data Flow

1. **Check MQTT Messages**
   - Navigate to: Settings → Devices & Services → MQTT
   - Click "Listen to a topic"
   - Subscribe to: `application/#`
   - Verify JSON payloads arriving

#### C.2 Create Template Sensors

Example Home Assistant configuration:

```yaml
# Example: Create sensors from MQTT payload
sensor:
  - platform: mqtt
    name: "Household Power"
    state_topic: "application/1/device/+/event/up"
    value_template: "{{ value_json.object.measurements.power }}"
    unit_of_measurement: "W"

  - platform: mqtt
    name: "Household Energy"
    state_topic: "application/1/device/+/event/up"
    value_template: "{{ value_json.object.measurements.energy }}"
    unit_of_measurement: "kWh"
```

#### C.3 Create Virtual Transformer

```yaml
sensor:
  - platform: template
    sensors:
      virtual_transformer:
        friendly_name: "Virtual Transformer (Total)"
        unit_of_measurement: "W"
        value_template: >
          {% set total = 0 %}
          {% for i in range(1, 101) %}
            {% set total = total + states('sensor.house_' ~ i ~ '_power') | float(0) %}
          {% endfor %}
          {{ total }}
```

---

## 3.1 Offline Architecture (FR06)

The system SHALL remain functional without internet. Here is how each layer handles offline scenarios:

| Layer | Component | Offline Behavior | Data Retention |
|-------|-----------|-----------------|----------------|
| Edge | iOKE868 sensor | Continues measuring and transmitting; no store-and-forward | Last reading only (no buffer) |
| Gateway | Milesight UG67 | Continues receiving LoRa packets; forwards when connection restored | Built-in NS queues data |
| MQTT | Mosquitto broker | All local, no internet dependency | Retained messages on topics |
| Server | Home Assistant | Continues running; dashboard and automations work locally | Full database local |

**No component requires internet for core operation.** The price optimizations in Phase 2 require intermittent internet for EPEX/Tibber data, but the system degrades gracefully: if price data is unavailable, optimization falls back to default charging patterns.

---

## 4. Implementation Steps - Phase 2

### Phase D: Price Integration

#### D.1 Price Import via Home Assistant Integrations

Three options exist for importing electricity prices, depending on the household's provider:

| Method | Provider | Updates | Cost | HACS? |
|--------|----------|---------|------|-------|
| **EPEX Spot** (free) | Any (ENTSO-E data) | Every 15-60 min | Free | Yes |
| **Tibber official** | Tibber customers | 15-min intervals | Free (Tibber account) | Built-in |
| **Tibber Prices** (advanced) | Tibber customers | 15-min + price ratings | Free | Yes |

**Option 1: EPEX Spot (Recommended for non-Tibber users)**
- GitHub: [`mampfes/ha_epex_spot`](https://github.com/mampfes/ha_epex_spot) (291 stars, MIT license)
- Install via HACS → Integrations → Add repository → Install "EPEX Spot"
- Configure for German price zone (DE-LU)
- Provides sensors: market price, total price, daily avg, lowest/highest intervals
- Built-in service `epex_spot.get_lowest_price_interval` for determining optimal start times
- Update frequency: configurable (15-60 min)

**Option 2: Tibber Official (Tibber customers)**
- Built into Home Assistant — no HACS needed
- Navigate to: Settings → Devices & Services → Add Integration → Search "Tibber"
- Enter API token from [developer.tibber.com](https://developer.tibber.com)
- Provides: current price, 24h forecast, consumption data, monthly stats

**Option 3: Tibber Prices Custom Integration (Advanced)**
- GitHub: [`jpawlowski/hass.tibber_prices`](https://github.com/jpawlowski/hass.tibber_prices)
- 100+ sensors including price levels (VERY_CHEAP to VERY_EXPENSIVE)
- Best price / peak price period detection
- 15-min interval precision (quarter-hourly)
- Install via HACS → Custom repositories

#### D.2 Price Configuration per Household

1. **Create Household Price Configuration**
   - Configure tariff type per household
   - Set fixed feed-in rate (EEG) or dynamic rates
   - Configure network charge module (§14a)

```yaml
# Example: Household configuration
input_select:
  household_1_tariff_type:
    name: "Household 1 Tariff Type"
    options:
      - fixed_feed_in
      - dynamic_pricing
      - fixed_price_only

input_number:
  household_1_feed_in_rate:
    name: "Feed-in Rate (ct/kWh)"
    min: 0
    max: 20
    step: 0.1
```

### Phase E: Control Integration

#### E.1 Wallbox/EV Charger Integration

1. **Supported Protocols**
   - **OCPP** (via EVCC or custom integration)
   - Modbus TCP (for compatible wallboxes)
   - Manufacturer REST APIs

2. **Integration Methods**
   - **EVCC Add-on** (recommended for OCPP) — HA add-on, 6K+ stars, browser config since v0.300
   - Home Assistant wallbox integrations
   - Custom MQTT integration

3. **OCPP Compatibility Warning**
   - Some wallbox manufacturers have OCPP implementation issues:
     - **Wallbox Pulsar Plus/Pro**: Known OCPP disconnects every few days requiring restart of both wallbox and EVCC; firmware 6.7.x introduced daily forced reboots that break OCPP connections
     - **General**: Prefer local Modbus/REST APIs over OCPP when available
     - **EVCC** is actively developed (240+ manufacturers supported) — test compatibility before deployment
   - See [EVCC GitHub Issues](https://github.com/evcc-io/evcc/issues) for known problems

4. **Configuration**
   - Add wallbox to Home Assistant
   - Configure charging limits
   - Set up automation for price-based charging

#### E.2 Smart Plug/Switch Integration

1. **Supported Devices**
   - Shelly (via Shelly integration)
   - Tuya/Tuya Smart Life (via HA integration)
   - Any MQTT-controlled smart plug

2. **Configuration**
   - Add devices to Home Assistant
   - Configure power measurement (if available)
   - Set up control entities

#### E.3 PV System Integration

1. **Supported Inverters**
   - SolarEdge (Modbus TCP)
   - SMA (Sunny Home Manager)
   - Kostal (Modbus TCP)
   - Victron (Venus OS)

2. **Data Points**
   - Current PV production (W)
   - Daily energy (kWh)
   - Total energy (kWh)

#### E.4 Battery Storage Integration

1. **Supported Systems**
   - BYD (via Modbus)
   - Ford (via API)
   - Tesla Powerwall (via API)
   - Generic Modbus batteries

2. **Data & Control**
   - State of Charge (SoC) %
   - Charge/Discharge power (W)
   - Start/Stop control

### Phase F: Revenue-Aware Optimization

#### F.0 INFRASTRUCTURE SAFETY WATCHDOG (CRITICAL - IMPLEMENT FIRST)

> **IMPLEMENTATION ORDER:** Infrastructure safety MUST be implemented BEFORE economic optimization. The watchdog is the enforcement mechanism.

The virtual transformer YAML sensor from Phase C provides monitoring, but enforcement requires a **physical watchdog** that automatically sheds load:

**Watchdog Implementation:**
```yaml
# Watchdog: Automatically disable loads when transformer is over 80%
automation:
  - alias: "Transformer Overload Protection"
    trigger:
      - platform: numeric_state
        entity_id: sensor.virtual_transformer
        above: 8000  # 80% of 10kVA transformer
    action:
      - service: switch.turn_off
        entity_id: "{{ states.switch | selectattr('attributes.transformer_protected', 'defined') | selectattr('attributes.transformer_protected', 'eq', true) | map(attribute='entity_id') | list }}"
      - service: mqtt.publish
        data:
          topic: "lem-netz/watchdog/alert"
          payload: "Transformer at {{ states('sensor.virtual_transformer') | int }}W — loads shed"
          retain: true
      - service: persistent_notification.create
        data:
          title: "⚠ Transformer Overload"
          message: "Virtual transformer exceeded 80%. Non-essential loads have been disabled."
```

Mark each controlled device with `transformer_protected: true` so the watchdog targets them:
```yaml
switch:
  - platform: mqtt
    name: "Household 1 Wallbox"
    command_topic: "lem-netz/house1/wallbox/set"
    state_topic: "lem-netz/house1/wallbox/state"
    qos: 2
    device_class: outlet
    icon: mdi:ev-station
  - platform: mqtt
    name: "Household 2 Smart Plug"
    command_topic: "lem-netz/house2/plug/set"
    state_topic: "lem-netz/house2/plug/state"
    qos: 2
    device_class: outlet
    icon: mdi:power-socket-de

input_boolean:
  transformer_protection_armed:
    name: "Transformer Protection"
    initial: on
```

On recovery (load drops below 60%), the watchdog re-enables loads:
```yaml
  - alias: "Transformer Load Recovery"
    trigger:
      - platform: numeric_state
        entity_id: sensor.virtual_transformer
        below: 6000  # 60% — hysteresis prevents oscillation
    action:
      - service: switch.turn_on
        entity_id: "{{ states.switch | selectattr('attributes.transformer_protected', 'defined') | selectattr('attributes.transformer_protected', 'eq', true) | map(attribute='entity_id') | list }}"
```

#### F.1 Recommended Optimization Tools (Phase 2)

Instead of hand-rolling YAML automations for each household, use existing open-source optimization tools:

| Tool | Description | License | Integration | Best For |
|------|-------------|---------|-------------|----------|
| **HAEO** | Linear programming optimizer | MIT | HA custom integration | Battery + PV + grid optimization |
| **AkkudoktorEOS** | Predictive price + load optimizer | Open source | HA add-on | German dynamic pricing optimization |
| **EVCC** | EV charging + energy management | MIT | HA add-on (+390 HA sensors) | Wallbox coordination |
| **forty-two-watts** | Multi-device battery coordinator | Open source | MQTT autodiscovery | Multi-inverter sites |

#### F.2 HAEO (Recommended Primary Optimizer)

[Home Assistant Energy Optimizer](https://github.com/hass-energy/haeo) — solves the optimization as a linear programming problem:

- **48-hour horizon**, 5-minute resolution
- Supports: batteries, solar, grid, constant/forecast loads, multi-node networks
- **Constrained by infrastructure limits** (transformer capacity is a configurable constraint)
- Automatically respects household pricing models
- Install via HACS → Custom repositories → `https://github.com/hass-energy/haeo`
- Requires HiGHS solver (bundled)

```
Example HAEO network model:
  Grid (import/export limits, pricing)
    └─ Node: Street Transformer (max 10kVA)
         ├─ Household 1: Solar + Battery + Charger (EEG tariff)
         ├─ Household 2: Battery only (dynamic pricing)
         ├─ Household 3: Solar only (fixed tariff)
         └─ ...
```

**Why HAEO over manual YAML:**
- Proper optimization algorithm, not if-else rules
- Constraint-aware (transformer capacity is a hard constraint)
- 48-hour lookahead (not just current price)
- Automatically adapts to changing conditions

#### F.3 AkkudoktorEOS (Predictive Add-on)

[AkkudoktorEOS](https://github.com/akkudoktor-eos/eos) — specifically designed for German energy market:

- Predictive models for EPEX Spot prices
- Load forecasting based on historical data
- Optimizes: batteries, heat pumps, EV charging
- Available as Home Assistant add-on
- Designed by Dr. Andreas Schmitz (YouTube: @akkudoktor)

#### F.4 Per-Household Optimization Rules (When using HAEO/AkkudoktorEOS)

Configure each household's tariff type in the optimizer:

| Household Type | HAEO Configuration | Optimization Goal |
|---------------|--------------------|-------------------|
| No PV, fixed tariff | Set import price to fixed rate | Minimize consumption cost |
| PV only (EEG) | Set export price to feed-in rate | Maximize self-consumption |
| PV + Battery (Dynamic) | Enable dynamic import/export rates | Arbitrage |
| Battery only | Enable dynamic import, set export=0 | Charge cheap, discharge peak |
| Heat pump | Add §14a network charge schedule | Shift to low-tariff periods |

---

## 5. Product Selection (Reference Only)

*This section provides reference products that meet the requirements. Other products may also be suitable.*

### 5.0 Evaluation Criteria

Each component was evaluated against the following criteria:

| Criterion | Weight | Definition |
|-----------|--------|------------|
| Cost | High | Meets budget target (€300 central, €200/household) |
| Availability | High | Available from German distributors |
| CE/RED Certification | Required | Legal requirement for EU operation |
| Scalability | Medium | Supports 100-1000 households |
| Maintainability | High | No specialized tools or expertise needed |
| Offline Capability | High | Core function works without internet |
| SML Protocol Support | Required | Must read German smart meters |

### 5.1 Energy Sensors (Phase 1)

| Product | Type | Key Features |
|---------|------|---------------|
| IMST iOKE868 | Optical IR + LoRaWAN | SML support, 868 MHz, magnetic mount |
| ... | ... | ... |

### 5.2 Outdoor Gateways (Phase 1)

| Product | Type | Key Features |
|---------|------|---------------|
| Milesight UG67 | Outdoor IP67 | 8-channel, 868 MHz, PoE, built-in NS |
| ... | ... | ... |

### 5.3 Server Platforms (Phase 1)

| Product | Type | Key Features |
|---------|------|---------------|
| Home Assistant Green | Appliance | Pre-installed, 4GB RAM, 32GB storage |
| Raspberry Pi 5 + case | SBC | Alternative, requires HA OS installation |
| ... | ... | ... |

### 5.4 Control Devices (Phase 2)

| Category | Product Examples | Protocol |
|----------|------------------|----------|
| Wallbox | go-e, Wallbe, DaheimLader, Zaptec | OCPP, Modbus, REST |
| Smart Plug | Shelly Plus, Shelly Pro | MQTT, REST |
| PV Inverter | SolarEdge, SMA, Kostal | Modbus TCP |
| Battery | BYD, Tesla, FoxESS | Modbus, REST |

### 5.5 Optimization Software (Phase 2)

| Software | License | Integration | Key Features |
|----------|---------|-------------|--------------|
| **HAEO** ([hass-energy/haeo](https://github.com/hass-energy/haeo)) | MIT | HA custom integration via HACS | Linear programming, 48h horizon, multi-node, constraint-aware |
| **AkkudoktorEOS** ([akkudoktor-eos/eos](https://github.com/akkudoktor-eos/eos)) | Open source | HA add-on | German market focus, predictive price/load, battery/heat pump |
| **EVCC** ([evcc-io/evcc](https://github.com/evcc-io/evcc)) | MIT | HA add-on (official) | 240+ manufacturers, OCPP server, browser config v0.300+ |
| **forty-two-watts** ([frahlg/forty-two-watts](https://github.com/frahlg/forty-two-watts)) | Open source | MQTT autodiscovery | Multi-inverter, 48h MPC planner, Go binary on Pi |

**Recommendation:** Start with HAEO for core optimization. Add AkkudoktorEOS for German price predictions. Use EVCC for wallbox OCPP control.

---

## 6. Cost Implementation Guide

### 6.1 Budget Tracking - Phase 1

| Category | Target | Actual | Status |
|----------|--------|--------|--------|
| Central server | €100-150 | | |
| Outdoor gateway | €150-200 | | |
| Network equipment | €20-50 | | |
| **Central Total** | **≤ €300** | | |
| | | | |
| Sensor per household | €100-150 | | |
| Power supply (optional) | €0-20 | | |
| **Per Household** | **≤ €200** | | |

### 6.2 Budget Tracking - Phase 2

| Category | Target | Notes |
|----------|--------|-------|
| Control integration | €0-50 | Usually already present |
| Smart plugs (per device) | €20-40 | Optional |
| Additional software | €0 | Open source |
| **Phase 2 Add-on** | **≤ €150/household** | |

### 6.3 Cost Optimization Tips

- Use WiFi backhaul instead of cellular to reduce ongoing costs
- Choose sensors with USB power (avoid battery costs)
- Install gateway in central location to maximize range
- Bulk purchase sensors for volume discount
- Leverage existing devices (wallboxes, PV) already owned by households
- Use open-source software (EVCC, Home Assistant) to avoid licensing costs

---

## 7. Regulatory Compliance Checklist

| Requirement | Regulation | Phase 1 | Phase 2 |
|-------------|------------|---------|---------|
| CE certification | RED 2014/53/EU | ✓ | ✓ |
| Data sovereignty | GDPR | ✓ | ✓ |
| Smart meter interface | BSI TR-03109-1 | ✓ (mME) | ✓ (iMSys for §14a) |
| Grid connection | VDE-AR-N 4100 | - | Required for control |
| Network charges | §14a EnWG | - | Required for Module 3 |
| Energy market | EEG | Reference | Required for optimization |

---

## 8. Troubleshooting Guide

### 8.1 Common Issues - Phase 1

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| No data from sensor | Antenna inside metallic cabinet | Route antenna outside |
| Sensor not joining | Wrong AppKey | Re-register with correct key |
| Intermittent data | Weak wireless signal | Add intermediate sensor or move gateway |
| Gateway offline | Network issue | Check Ethernet/PoE connection |
| No MQTT data | Wrong MQTT config | Verify broker IP and port in gateway |

### 8.2 Common Issues - Phase 2

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| Wallbox not responding | OCPP connection failed | Check network, credentials; restart EVCC and wallbox |
| Wallbox frequent disconnects | Manufacturer firmware issue | Known problem with Wallbox Pulsar Plus firmware 6.7.x — try local Modbus instead of OCPP |
| Price data not updating | API rate limit | Check integration logs; EPEX API may throttle |
| Optimization causing loss | Wrong household config | Verify tariff type setting in HAEO/HA |
| §14a not working | No Smart Meter (iMSys) | Requires iMSys installation for Module 3 |
| Watchdog not firing | Entity ID mismatch | Verify all switches have `transformer_protected: true` |
| False positive overload | Sensor drift | Check virtual transformer calculation — sum of all households

### 8.3 Diagnostics

1. **Check sensor LED status** - Should indicate join and transmit
2. **Check gateway web interface** - Shows joined devices and uplink count
3. **Check MQTT topic** - Subscribe to `#` for all messages
4. **Check Home Assistant logs** - Settings → System → Logs
5. **Check price integration** - Verify EPEX/provider connectivity

---

## 9. German Market Reference

### 9.1 Pricing Model Summary

| Model | Provider Examples | Price Range | Optimization Strategy |
|-------|------------------|-------------|----------------------|
| Fixed Feed-in (EEG) | All grid operators | 5.5-8 ct/kWh (20 yr) | Maximize self-consumption |
| Dynamic (Volldynamisch) | Tibber, Ostrom, aWATTar | EPEX + 2-5 ct/kWh | Time-shift consumption |
| Fixed Price | E.ON, EnBW, etc. | 30-35 ct/kWh | Minimize consumption |
| §14a Network | All grid operators | Variable by module | Shift to low-tariff periods |

### 9.2 Key Regulations

- **§14a EnWG (2025):** Variable network charges, 3 modules
- **EEG 2023:** Feed-in tariffs, 20-year guarantee
- **BIS TR-03109-1:** Smart Meter Gateway interface
- **RED:** CE certification for wireless equipment

---

## 10. Future Expansion Path

### Phase 3 (Future)

- Add predictive algorithms based on weather forecasts
- Add battery degradation-aware optimization
- Add multi-community coordination
- Add peer-to-peer energy trading
- Add grid service provision (aggregator model)

---

## 11. Appendix: Reference Links

### Regulatory References
- BSI TR-03109-1: [BSI LMN Interface](https://www.bsi.bund.de)
- VDE-AR-N 4100: [VDE FNN](https://www.vde.com)
- §14a EnWG: [BNetzA Information](https://www.bundesnetzagentur.de)
- EEG: [Bundesjustizministerium](https://www.gesetze-im-internet.de/eeg_2014/)

### Technical References
- MQTT Protocol: [mqtt.org](https://mqtt.org)
- EPEX Spot: [EPEX SPOT](https://www.epexspot.com)
- EPEX Spot HA Integration: [mampfes/ha_epex_spot](https://github.com/mampfes/ha_epex_spot)
- OCPP: [OCPP Documentation](https://www.openchargealliance.org)
- EVCC: [evcc-io/evcc](https://github.com/evcc-io/evcc)
- EVCC HA Add-on: [evcc-io/hassio-addon](https://github.com/evcc-io/evcc-hassio-addon)
- HAEO Optimizer: [hass-energy/haeo](https://github.com/hass-energy/haeo)
- AkkudoktorEOS: [akkudoktor-eos/eos](https://github.com/akkudoktor-eos/eos)
- forty-two-watts: [frahlg/forty-two-watts](https://github.com/frahlg/forty-two-watts)
- Tibber HA Integration: [home-assistant.io/integrations/tibber](https://www.home-assistant.io/integrations/tibber/)

---

*This document provides implementation guidance for Phase 1 and Phase 2. It supports the requirements specification by showing how to achieve the defined goals while maintaining economic fairness across all household types.*

**Document Version:** 3.1
**Last Updated:** May 2026