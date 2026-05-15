# Technical Concept: LEM-Netz

## Implementation Guide for Phase 1 + Phase 2

**Version:** 3.0
**Status:** Planning
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

## 4. Implementation Steps - Phase 2

### Phase D: Price Integration

#### D.1 EPEX Spot Price Import

1. **Install Price Import Integration**
   - Use Home Assistant EPEX Spot integration or custom integration
   - Configure for German price zone (DE-LU)
   - Set update frequency: 15 minutes (or hourly minimum)

2. **Configure Dynamic Provider Integration** (optional)
   - If using Tibber, aWATTar, Ostrom: install provider-specific integration
   - Import provider-specific prices if available
   - OR use EPEX as fallback

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
   - OCPP (via EVCC or custom integration)
   - Modbus TCP (for compatible wallboxes)
   - Manufacturer REST APIs

2. **Integration Methods**
   - EVCC Add-on (recommended for OCPP support)
   - Home Assistant wallbox integrations
   - Custom MQTT integration

3. **Configuration**
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

#### F.0 PRIORITY HIERARCHY (CRITICAL - IMPLEMENT FIRST)

> **IMPLEMENTATION ORDER:** Infrastructure safety MUST be implemented BEFORE economic optimization.

1. **FIRST: Infrastructure Safety**
   - Implement transformer load monitoring (virtual transformer sensor)
   - Set max threshold (e.g., 80% of transformer rating)
   - Implement automatic load shedding when approaching threshold
   - Test: Verify loads automatically reduce when threshold exceeded

2. **SECOND: Economic Optimization** (only after #1 is verified)
   - Implement per-household revenue model configuration
   - Implement price-based optimization (within infrastructure limits)
   - Test: Verify economic optimization works correctly

> **IF INFRASTRUCTURE SAFETY AND ECONOMIC FAIRNESS CONFLICT:** Infrastructure safety ALWAYS wins. Suspend economic optimization when grid stability requires it.

#### F.1 Optimization Logic Requirements

**CRITICAL:** The optimization MUST respect different household revenue models, BUT only within infrastructure safety limits.

1. **For Households with Fixed EEG Feed-in Tariff:**
   - Maximize self-consumption (use PV power locally)
   - Do NOT optimize for export price timing
   - Priority: minimize grid import

2. **For Households with Dynamic Pricing:**
   - Charge devices when prices are low
   - Discharge batteries when prices are high (arbitrage)
   - Shift flexible loads to low-price periods

3. **For Households with §14a Network Charges:**
   - Shift consumption to low-tariff periods
   - Avoid high-tariff peak periods
   - Monitor grid load via virtual transformer

#### F.2 Example Optimization Automation

> **IMPORTANT:** All optimization automations MUST include a check for infrastructure limits BEFORE applying economic optimization. Never optimize if transformer is near capacity.

```yaml
# Example: Price-aware EV charging WITH infrastructure safety check
automation:
  - alias: "Optimize EV Charging"
    trigger:
      - platform: time
        at: "00:00:00"
    condition:
      # FIRST: Check infrastructure is safe
      - condition: numeric_state
        entity_id: sensor.virtual_transformer
        below: 8000  # e.g., 80% of 10kVA transformer
      # SECOND: Check household has dynamic pricing
      - condition: state
        entity_id: input_select.household_1_tariff_type
        state: "dynamic_pricing"
    action:
      - service: shell_command.get_current_price
      - service: evcc.set_charge_limit
        data:
          limit: >
            {% if states('sensor.current_price') | float < 10 %}
              100
            {% elif states('sensor.current_price') | float < 20 %}
              50
            {% else %}
              0
            {% endif %}
```

---

## 5. Product Selection (Reference Only)

*This section provides reference products that meet the requirements. Other products may also be suitable.*

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
| Wallbox not responding | OCPP connection failed | Check network, credentials |
| Price data not updating | API rate limit | Check integration logs |
| Optimization causing loss | Wrong household config | Verify tariff type setting |
| §14a not working | No Smart Meter (iMSys) | Requires iMSys installation |

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
- OCPP: [OCPP Documentation](https://www.openchargealliance.org)

---

*This document provides implementation guidance for Phase 1 and Phase 2. It supports the requirements specification by showing how to achieve the defined goals while maintaining economic fairness across all household types.*

**Document Version:** 3.0
**Last Updated:** May 2026