# Technical Concept: LEM-Netz

## Implementation Guide for Phase 1 + Phase 2

**Version:** 3.4
**Status:** Review (incorporating open-source optimization recommendations)
**Purpose:** Translate Requirements into Implementation

---

## Table of Contents

### Phase 1
- [1. Implementation Approach](#1-implementation-approach)
- [2. Technical Architecture](#2-technical-architecture)
- [3. Implementation Steps - Phase 1](#3-implementation-steps---phase-1)
  - [Phase A: Central Infrastructure Setup](#phase-a-central-infrastructure-setup)
  - [Phase B: Household Sensor Installation](#phase-b-household-sensor-installation)
  - [Phase C: Data Verification](#phase-c-data-verification)
  - [3.1 Offline Architecture](#31-offline-architecture-fr06)
  - [3.2 Per-Household Local Integration](#32-per-household-local-integration)

### Phase 2
- [4. Implementation Steps - Phase 2](#4-implementation-steps---phase-2)
  - [Phase D: Price Integration](#phase-d-price-integration)
  - [Phase E: Control Integration](#phase-e-control-integration-via-local-controller)
  - [Phase F: Revenue-Aware Optimization / Watchdog](#phase-f-revenue-aware-optimization)

### Reference
- [5. Product Selection](#5-product-selection-reference-only)
- [6. Cost Implementation Guide](#6-cost-implementation-guide)
- [7. Regulatory Compliance Checklist](#7-regulatory-compliance-checklist)
- [8. Troubleshooting Guide](#8-troubleshooting-guide)
- [9. German Market Reference](#9-german-market-reference)
- [10. Future Expansion Path](#10-future-expansion-path)
- [11. Appendix A: Optimization Tools](#11-appendix-a-optimization-tools)
- [12. Appendix B: Alternative Controller](#12-appendix-b-alternative-controller-rpi-for-ocppmodbus-tcp)
- [13. Appendix C: Reference Links](#13-appendix-c-reference-links)

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
7. **Separation of Concerns:** Central system sends constraints and recommendations; households decide how to comply autonomously. Direct control is last-resort only.
8. **Economically Fair:** Optimization must NOT disadvantage any household type (SUBORDINATE to #1)

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
| FR16-20, FR06 | Per-Household Bridge Device | Connects local Modbus devices and meter IR to central via LoRaWAN; buffers data during outages | 1+2 |

### 2.2 System Diagram - Phase 1 + 2

The system uses a single wireless technology — LoRaWAN 868 MHz — for all communication between households and the central server. MQTT is used only on the local LAN between the gateway and the server.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LEM-NETZ SYSTEM                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │  NEIGHBORHOOD NETWORK — Central Server (Home Assistant Green/RPi)│      │
│  │                                                                  │      │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │      │
│  │  │ MQTT Broker  │  │  Database    │  │  Dashboard           │   │      │
│  │  │ (Mosquitto)  │  │ (InfluxDB)   │  │  (Home Assistant)    │   │      │
│  │  └──────┬───────┘  └──────┬───────┘  └────────┬─────────────┘   │      │
│  │         │                  │                    │                │      │
│  │         ▼                  ▼                    ▼                │      │
│  │  ┌────────────────────────────────────────────────────────────┐  │      │
│  │  │  OPTIMIZATION ENGINE (Phase 2)                             │  │      │
│  │  │  ┌─────────────────┐  ┌──────────────────────────────┐    │  │      │
│  │  │  │  Price Import   │  │  Control Logic               │    │  │      │
│  │  │  │ (EPEX, Tibber)  │  │  (Revenue-aware)             │    │  │      │
│  │  │  └─────────────────┘  └──────────────────────────────┘    │  │      │
│  │  │  ┌────────────────────────────────────────────────────┐   │  │      │
│  │  │  │  INFRASTRUCTURE WATCHDOG                           │   │  │      │
│  │  │  │  • Monitors virtual transformer                    │   │  │      │
│  │  │  │  • Issues max_power / shed commands                │   │  │      │
│  │  │  │  • 80% trigger / 60% hysteresis                    │   │  │      │
│  │  │  └────────────────────────────────────────────────────┘   │  │      │
│  │  └────────────────────────────────────────────────────────────┘  │      │
│  └───────────────────────┬──────────────────────────────────────────┘      │
│                          │ MQTT over Ethernet (LAN)                         │
│                          ▼                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  OUTDOOR GATEWAY (Milesight UG67, IP67 roof-mount)                   │  │
│  │  LoRaWAN ↔ MQTT packet forwarder — translates between               │  │
│  │  the wireless (868 MHz) and wired (Ethernet) domains                │  │
│  └───────────────────────┬──────────────────────────────────────────────┘  │
│                          │ LoRaWAN 868 MHz (bidirectional, uplink+downlink) │
│                          ▼                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  HOUSEHOLD DOMAIN (×N) — one bridge device per household              │  │
│  │                                                                      │  │
│  │  ┌────────────────────────────────────────────────────────────┐     │  │
│  │  │  BRIDGE DEVICE (ESP32 + LoRa radio)                         │     │  │
│  │  │                                                              │     │  │
│  │  │  • LoRaWAN uplink: meter readings, device power data       │     │  │
│  │  │  • LoRaWAN downlink: max_power limits, shed commands        │     │  │
│  │  │  • IR optical: reads smart meter (SML/IEC 62056-21)        │     │  │
│  │  │  • Modbus RTU (RS485): controls home devices               │     │  │
│  │  │  • Local limit enforcement: stays within control/max_power  │     │  │
│  │  │  • Data buffering during LoRaWAN outages (last known vals) │     │  │
│  │  └──────────┬─────────────────────────────────────────────────┘     │  │
│  │             │ Modbus RTU (RS485)                                      │
│  │  ┌──────────┴───────────────────────────────────────────────────┐     │  │
│  │  │  HOUSEHOLD DEVICES (varies per household)                    │     │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │     │  │
│  │  │  │ Wallbox  │  │PV Inverter│  │ Battery  │  │Smart Plug│    │     │  │
│  │  │  │ (Modbus) │  │ (Modbus) │  │ (Modbus) │  │ (Modbus) │    │     │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │     │  │
│  │  └─────────────────────────────────────────────────────────────┘     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Data flow summary:**

| Direction | Path | Protocol |
|-----------|------|----------|
| House → Central | ESP → LoRaWAN uplink → UG67 → LAN (MQTT) → HA | LoRaWAN + MQTT |
| Central → House | HA → LAN (MQTT) → UG67 → LoRaWAN downlink → ESP | MQTT + LoRaWAN |
| Local household | ESP ↔ home devices (Modbus RTU RS485) | Modbus |
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
   - Contact your grid operator (Netzbetreiber)
   - Request activation of the SML data interface ("Freischaltung der SML-Datenschnittstelle")
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

#### B.3 Local Controller Setup (20-45 minutes)

All households use the same bridge device: ESP32 + LoRa radio module. See Section 3.2.2 for hardware details.

**Bridge Device Setup:**
- Flash firmware via USB (first-time) — Arduino/PlatformIO with LoRaWAN + Modbus + IR libraries
- Connect MAX3485 RS485 transceiver: ESP32 UART2 (GPIO16 TX, GPIO17 RX) → RS485
- Wire RS485 A/B to wallbox or inverter Modbus terminals
- Connect IR optical read head to ESP32 UART1
- Connect LoRa radio module to ESP32 SPI bus (MOSI, MISO, SCK, NSS, DIO0, RST)
- Register on LoRaWAN network server: assign DevEUI, AppKey, AppEUI
- Set uplink interval (default: 2 min) in firmware config
- Test: verify uplink data arrives at HA via MQTT topics `lem-netz/house/<id>/sensor/...`

For households with devices requiring OCPP or Modbus TCP (not supported on ESP32), see the alternative controller setup in Section 12.

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

The system SHALL remain functional without internet. Here is how each domain handles offline scenarios:

| Domain | Component | Offline Behavior | Data Retention |
|--------|-----------|-----------------|----------------|
| Neighborhood Network | Mosquitto MQTT broker | All local, no internet dependency | Retained messages on topics |
| Neighborhood Network | Home Assistant + InfluxDB | Dashboard and automations continue locally | Full database local |
| Gateway | Milesight UG67 | Continues receiving LoRa packets; forwards when connection restored | Built-in NS queues data |
| Household Bridge | ESP32 + LoRa radio | Continues reading meter (IR) and polling local devices; buffers readings when LoRaWAN link is down; enforces last known max_power limit | Configurable buffer (limited on ESP32) |

**No component requires internet for core operation.** The price optimizations in Phase 2 require intermittent internet for EPEX/Tibber data, but the system degrades gracefully: if price data is unavailable, optimization falls back to default charging patterns.

---

## 3.2 Per-Household Local Integration

Control devices (wallbox, PV inverter, battery BMS, smart plugs) are located inside each household and cannot connect directly to the central server. Each house requires a **bridge device** that connects household devices to the central server via LoRaWAN through the outdoor gateway (UG67).

### 3.2.1 LoRaWAN Data Model and MQTT Topic Hierarchy

All household-to-central communication uses LoRaWAN 868 MHz. The Milesight UG67 gateway forwards LoRaWAN payloads to the central MQTT broker over Ethernet. MQTT topics exist only on the LAN between gateway and server — the ESP does not use MQTT.

**LoRaWAN payload format (ESP → UG67):**

| Data | Uplink Interval | Encoding | Size |
|------|----------------|----------|------|
| Smart meter power (W) | Every 2 min | 2-byte unsigned int | 2 bytes |
| Wallbox power (W) | Every 2 min | 2-byte signed int | 2 bytes |
| PV production (W) | Every 2 min | 2-byte unsigned int | 2 bytes |
| Battery SoC (%) | Every 2 min | 1-byte unsigned int | 1 byte |
| Battery power (W) | Every 2 min | 2-byte signed int | 2 bytes |
| Smart plug power (W) | Every 2 min | 2-byte unsigned int | 2 bytes |
| Shed status | On event | 1-byte bitmask | 1 byte |

Total typical payload: 6-12 bytes per uplink (well within LoRaWAN limits).

**Downlink payload format (UG67 → ESP):**

| Command | Encoding | Size | ESP Action |
|---------|----------|------|------------|
| `max_power` (W) | 2-byte unsigned int | 2 bytes | Stay within this total household import limit |
| `shed` | 1-byte flag | 1 byte | Shed all non-essential loads immediately |
| `restore` | 1-byte flag | 1 byte | Restore previously shed loads |

**MQTT topics (LAN side only — between UG67/network server and HA):**

| Topic | Direction | Payload |
|-------|-----------|---------|
| `lem-netz/house/<id>/sensor/power` | Gateway → HA | `{"value": 4500, "unit": "W"}` |
| `lem-netz/house/<id>/sensor/wallbox/power` | Gateway → HA | `{"value": 4500, "unit": "W"}` |
| `lem-netz/house/<id>/sensor/pv/production` | Gateway → HA | `{"value": 3200, "unit": "W"}` |
| `lem-netz/house/<id>/sensor/battery/soc` | Gateway → HA | `{"value": 65, "unit": "%"}` |
| `lem-netz/house/<id>/control/max_power` | HA → Gateway | `{"value": 5000, "unit": "W"}` |
| `lem-netz/house/<id>/control/shed` | HA → Gateway | `{"reason": "transformer_overload"}` |
| `lem-netz/house/<id>/control/restore` | HA → Gateway | `{"reason": "load_recovered"}` |

The gateway or a lightweight LoRaWAN network server (e.g., ChirpStack running on the central server) translates between LoRaWAN payloads and these MQTT topics. The ESP32 enforces the last received `max_power` limit locally by shedding or throttling devices in fixed priority order. `control/shed` is the emergency fallback used only when limits alone cannot resolve an overload.

### 3.2.2 Bridge Device Hardware (ESP32 + LoRa)

Every household uses the same bridge device: an **ESP32 with a LoRa radio module**. One design for all households.

| Component | Purpose | Cost |
|-----------|---------|------|
| ESP32 DevKit C | MCU, 2 UARTs for Modbus RS485, SPI for LoRa module | €12-18 |
| LoRa radio module (e.g., RFM95W, SX1276) | 868 MHz LoRaWAN transceiver, SPI interface | €8-15 |
| MAX3485 RS485 Module | Converts ESP32 UART to RS485 for Modbus RTU | €3-5 |
| IR optical read head | Reads smart meter via IR interface (SML protocol) | €5-10 |
| USB power supply | 5V power | €5-10 |
| USB cable + enclosure | Mounting and connectivity | €5-10 |
| **Total** | | **€38-68** |

**Capabilities:**
- LoRaWAN uplink: sends meter reading and device power data every 2 min

- Uplink interval: 2 min (default, adjustable 1-10 min per household)
- Activation: OTAA (Over-The-Air Activation) with individual DevEUI/AppKey per device

**Setup:**
1. Flash firmware via USB (first-time) using Arduino IDE or PlatformIO with MCCI LoRaWAN library
2. Connect MAX3485: ESP32 UART2 (GPIO16 TX, GPIO17 RX) → RS485, wire A/B to wallbox/inverter Modbus terminals
3. Connect IR read head: ESP32 UART1 to IR phototransistor circuit
4. Connect LoRa module: ESP32 SPI (MOSI, MISO, SCK) + NSS, DIO0, RST
5. Register device on LoRaWAN network server: DevEUI, AppKey, AppEUI
6. Test: verify uplink appears on MQTT topics at HA

### 3.2.3 Backhaul (Household to Central)

The ESP32 communicates with the central server exclusively via **LoRaWAN 868 MHz** through the outdoor gateway (Milesight UG67). No WiFi, no Ethernet, and no MQTT are needed at the household.

**Data flow:**

```
ESP (LoRa radio) ───┐
                    ├── LoRaWAN 868 MHz ──→ UG67 Gateway ── MQTT (LAN) ──→ HA
Smart Meter (IR) ───┘
Home Devices (Modbus) ─── Modbus RTU ──→ ESP
```

The UG67 acts as the protocol translator: LoRaWAN ↔ MQTT. On the household side, LoRaWAN handles all wireless communication. On the LAN side, MQTT transports data between UG67 and Home Assistant.

**Uplink intervals and duty cycle:**

| Setting | Value | Note |
|---------|-------|------|
| Report interval | 2 min (default) | Adjustable 1-10 min per household |
| Duty cycle limit | 1% (EU868) | ~36 seconds transmit per hour |
| Typical airtime per uplink | ~50-200 ms | Depends on SF setting |
| Max uplinks per hour (SF7) | ~720 | More than sufficient at 2 min intervals |

At a 2-minute interval (30 uplinks/hour), the duty cycle usage is under 0.4%. Even at 1-minute intervals (60 uplinks/hour), SF7-9 stay well within the 1% limit.

### 3.2.4 Device Support on Bridge

| Device | Connection | Notes |
|--------|-----------|-------|
| Wallbox (Modbus RTU) | RS485 via MAX3485 | Preferred — simple, reliable |
| PV Inverter (Modbus RTU) | RS485 via MAX3485 | 1-2 Modbus devices total per ESP32 |
| Battery BMS (Modbus RTU) | RS485 via MAX3485 | If Modbus registers are documented |
| Smart Plug (Modbus RTU) | RS485 via MAX3485 | Use Modbus-capable relays (e.g., Shelly Pro with RS485 addon) |
| Smart Meter (IR) | IR optical read head | Reads SML/IEC 62056-21 data via serial UART |

**Modbus TCP** or **OCPP** devices require a Raspberry Pi alternative (see Section 12). Prefer Modbus RTU devices for compatibility.

If a household has more than 2 Modbus devices, use a second ESP32.

### 3.2.5 Control Philosophy

The system uses a **constraint-based control model**: the central system sets hard limits (via LoRaWAN downlink), the ESP32 enforces them locally. This provides household autonomy, privacy (central system never sees individual device states), and resilience (ESP32 continues enforcing the last known `max_power` limit even if LoRaWAN downlink is temporarily unavailable).

**Latency and safety margin:**

Because the bridge device reports power data every 2 minutes (LoRaWAN uplink), the central watchdog cannot detect overloads instantly. The system compensates with a moderate safety margin:

| Parameter | WiFi/MQTT model (old) | LoRaWAN model (this design) |
|-----------|----------------------|---------------------------|
| Max detection delay | ~1 second | Up to 2 minutes |
| Watchdog trigger threshold | 80% | 75% |
| Watchdog recovery threshold | 60% | 55% |
| Stage 1 → Stage 2 wait | 30 seconds | 3 minutes (allowing for LoRaWAN downlink cycle) |

These thresholds are configurable per installation. The key tradeoff: simpler deployment (no WiFi per household) for slightly reduced transformer utilization.

**Responsibility split:**

| Concern | Central System (HA) | Household (ESP32) |
|---------|--------------------|--------------------|
| Transformer safety | Monitors virtual transformer, calculates fair-share limits | Enforces last received `max_power` by shedding loads in priority order |
| Load optimization | Broadcasts price signals via LoRaWAN downlink | Decides locally which loads to run within limits |
| Emergency shutdown | Issues `shed` downlink as last resort | Executes shed immediately (all non-essential off) |

**Normal operation:**
```
Central HA ── LAN MQTT ── UG67 ── LoRaWAN ── ESP32
   │                                              │
   ├─ max_power = 5000W ────────downlink────────►│
   │                                              ├─ "I have 5kW budget"
   │                                              │  → Heat pump at 2kW
   │                                              │  → EV charges at 3kW
   │◄──── uplink: wallbox=3000W, plug=2000W ──────┤
```

**Emergency shed:**
```
Central HA ── LAN MQTT ── UG67 ── LoRaWAN ── ESP32
   │                                              │
   ├─ max_power = 3000W ──downlink (Stage 1)───►│  tighter limit
   │  (wait 5-10 min for next uplink check)      │
   ├─ shed ──────────────downlink (Stage 2)─────►│  emergency shed
   │◄──── uplink: shed=executed ──────────────────┤
```

**Load shed priority (ESP32, fixed order):**
1. Smart plugs (non-essential loads)
2. Wallbox charging (pause EV)
3. Battery charging (stop charging)
4. Heat pump / other controllable loads

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

### Phase E: Control Integration via Bridge Device

> **Architecture:** All device integration (wallbox, PV, battery, smart plugs) happens on the **per-household bridge device (ESP32)**, not on the central server. The ESP32 sends sensor data via LoRaWAN uplink and receives control commands via LoRaWAN downlink, routed through the UG67 gateway. Communication on the LAN side uses MQTT between UG67 and HA. See Section 3.2 for the full data model.

#### E.1 Integration Architecture

```
┌─ HOUSEHOLD ────────────────────────────────────────────────────────────┐
│                                                                         │
│  Wallbox ────────Modbus RTU──┐                                          │
│  PV Inverter ───Modbus RTU───┤──→ ESP32 (LoRa radio) ──LoRaWAN──→ UG67 │
│  Battery ───────Modbus RTU───┤         (868 MHz)           │           │
│  Smart Plug ──Modbus RTU─────┘                              │ MQTT LAN │
│                                                              ▼          │
│  Smart Meter ────IR optical──→ ESP32                        Central HA │
│                                                                         │
│  ESP32 handles:                                                         │
│  • Protocol translation (Modbus RTU, IR optical)                       │
│  • LoRaWAN uplink: sensor data every 2 min                           │
│  • LoRaWAN downlink: receive limits and commands                       │
│  • Hard limit enforcement: stay within last received max_power         │
│  • Data buffering during LoRaWAN outages                               │
└─────────────────────────────────────────────────────────────────────────┘
```

#### E.1.1 Control Command Handling

> **Primary mode:** The central system sends hard limits via LoRaWAN downlink (`max_power`). The ESP32 enforces these locally by shedding loads in priority order. `shed` is the emergency fallback.

| Command | Encoding | ESP32 Action |
|---------|----------|-------------|
| `max_power` (2 bytes) | Unsigned int (watts) | Sum all device power; if exceeding limit, shed lowest-priority device |
| `shed` (1 byte) | Flag | Turn off all non-essential loads immediately |

No compliance reporting or heartbeat — the central system monitors actual load via uplink sensor data.

#### E.2 Wallbox/EV Charger Integration (on ESP32)

1. **Supported Protocol**

   | Protocol | ESP32 Support | Notes |
   |----------|--------------|-------|
   | Modbus RTU (RS485) | ✓ via MAX3485 | Preferred — simple, reliable |
   | Modbus TCP | — | Not available; use RS485 instead |
   | OCPP 1.6/2.0 | — | Not available on ESP32. Use EVCC on RPi (see appendix) |
   | Manufacturer REST | — | Not available on ESP32 |

   **Recommendation:** Connect wallbox via Modbus RTU (RS485). If your wallbox only supports OCPP, see the RPi-based alternative in Section 12.

2. **Data Sent via LoRaWAN Uplink**

   | Data Point | Encoding | Interval |
   |------------|----------|----------|
   | Wallbox power (W) | 2-byte signed int | Every 2 min |
   | Wallbox energy (kWh) | 4-byte unsigned int | Every 5 min |
   | Wallbox status | 1-byte enum | On change |
   | Shed confirmation | 1-byte flag | On watchdog command |

3. **Control Commands via LoRaWAN Downlink**

   | Command | Encoding | ESP32 Action |
   |---------|----------|-------------|
   | `max_power` | 2-byte unsigned int (W) | Reduce wallbox current so total household stays below limit |
   | `shed` | 1-byte flag | Immediately stop wallbox charging |

4. **OCPP Compatibility Note**
   - If your wallbox only supports OCPP (e.g., Wallbox Pulsar Plus), you will need the RPi-based alternative controller (see Section 12) since OCPP is not supported on ESP32
   - Prefer wallboxes with Modbus RTU for compatibility with this architecture

#### E.3 Smart Plug/Switch Integration (on ESP32)

Smart plugs must support Modbus RTU (RS485) — the bridge device has no WiFi:

1. **Supported Devices**

   | Device | Connection | Notes |
   |--------|-----------|-------|
   | Shelly Pro with RS485 addon | Modbus RTU via MAX3485 | Shelly Pro 4PM has optional RS485 adapter |
   | Generic Modbus relay (e.g., Finders) | Modbus RTU via MAX3485 | DIN-rail mount, power measurement optional |
   | ESP32 GPIO relay | Direct GPIO | Simple on/off, no power measurement |

2. **Data Flow**
   - **Sensor data**: ESP32 reads plug power via Modbus registers → includes in LoRaWAN uplink
   - **Limit**: ESP32 sums all plug power; if approaching `max_power`, turns off lowest-priority plug
   - **Emergency**: ESP32 receives `shed` downlink → sets GPIO/Modbus coil to off

3. **Configuration**
   - Assign each plug a Modbus slave ID and shed priority in firmware
   - Priority 1 = shed first during limit enforcement

#### E.4 PV System Integration (on ESP32)

1. **Supported Inverters**

   | Inverter | ESP32 Connection | Notes |
   |----------|-----------------|-------|
   | Kostal (Modbus RTU) | RS485 via MAX3485 | Works with Modbus RTU |
   | Kostal (Modbus TCP) | — | Not available |
   | SolarEdge | — | Use HA SolarEdge integration on central server |
   | SMA | — | Use HA SMA integration on central server |
   | Victron | — | Use HA Victron integration on central server |

   The ESP32 can read a PV inverter with Modbus RTU output. For monitoring-only (no control), use HA's built-in integration on the central server and skip the ESP32 connection.

2. **Data Sent via LoRaWAN Uplink**

   | Data Point | Encoding | Interval |
   |------------|----------|----------|
   | PV production (W) | 2-byte unsigned int | Every 2 min |
   | PV daily yield (kWh) | 4-byte unsigned int | Every 5 min |

3. **Control**
   - The ESP32 only reads PV data; it does not curtail PV production (this requires inverter-specific Modbus registers beyond ESP32 scope)
   - If PV curtailment is needed, implement it in HA automations using the inverter's native integration

#### E.5 Battery Storage Integration (on ESP32)

1. **Supported Systems**

   | System | ESP32 Connection | Notes |
   |--------|-----------------|-------|
   | Generic Modbus BMS | RS485 via MAX3485 | Works if Modbus registers are documented (SoC, power) |
   | BYD (Modbus) | — | Use HA Modbus integration on central server instead |
   | Tesla Powerwall | — | Use HA Powerwall integration on central server |
   | FoxESS (Modbus) | — | Use HA Modbus integration on central server |

   For most battery systems, use the HA integration directly on the central server rather than routing through ESP32. The ESP32 connection is only needed for batteries with simple Modbus RTU interfaces.

2. **Data Sent via LoRaWAN Uplink**

   | Data Point | Encoding | Interval |
   |------------|----------|----------|
   | Battery SoC (%) | 1-byte unsigned int | Every 2 min |
   | Battery power (W) | 2-byte signed int (negative = charging) | Every 2 min |

3. **Control**
   - The ESP32 reads battery state for local load management decisions
   - If battery control is needed (charge/discharge scheduling), it is handled by HA automations using the battery's native HA integration

### Phase F: Revenue-Aware Optimization

#### F.0 INFRASTRUCTURE SAFETY WATCHDOG (CRITICAL - IMPLEMENT FIRST)

> **IMPLEMENTATION ORDER:** Infrastructure safety MUST be implemented BEFORE economic optimization. The watchdog is the enforcement mechanism.

The watchdog operates in **two stages**: first by tightening `control/max_power` limits (allowing ESP32s to self-regulate), then by direct shed commands if the limit tightening does not resolve the overload.

```
Central HA detects overload (75%)
  │
  ├─ Stage 1: control/max_power = reduced_limit ──► LoRaWAN downlink ──► ESP32s shed loads
  │     Wait 3 min
  │
  ├─ Stage 2: control/shed ──► Immediate load shed on all households
  │     (only if still overloaded after Stage 1)
  │
  └─ Recovery: restore limits → restore loads
```

#### F.0.1 Watchdog Stages

**Stage 1 — Limit Tightening (normal response):**

| Step | Action | Detail |
|------|--------|--------|
| 1.1 | Detect threshold | `sensor.virtual_transformer` exceeds 75% (7500W for 10kVA) |
| 1.2 | Calculate fair share | Reduce each household's `max_power` proportionally |
| 1.3 | Publish limit via MQTT → UG67 → LoRaWAN downlink | `lem-netz/house/<id>/control/max_power` = new limit per household |
| 1.4 | Wait | 3 minutes (allow LoRaWAN downlink cycle) |
| 1.5 | Check | If total < 75% → resolved. If still > 75% → proceed to Stage 2 |

**Stage 2 — Direct Shed (escalation, only if Stage 1 fails):**

| Step | Action | Detail |
|------|--------|--------|
| 2.1 | Send shed | `lem-netz/house/<id>/control/shed` to ALL households |
| 2.2 | Acknowledge | ESP32s confirm via `status/shed` |
| 2.3 | Verify | If still > 75%, alert and escalate manually |
| 2.4 | Alert | Create persistent notification and log event |

#### F.0.2 Central Configuration

```yaml
# Home Assistant: Virtual transformer and thresholds
sensor:
  - platform: template
    sensors:
      virtual_transformer:
        unit_of_measurement: "W"
        value_template: >
          {% set total = 0 %}
          {% for i in range(1, 101) %}
            {% set total = total + states('sensor.house_' ~ i ~ '_power') | float(0) %}
          {% endfor %}
          {{ total }}

# Per-household base allocation (normal max_power per house)
input_number:
  household_1_base_allocation:
    name: "Household 1 base allocation"
    min: 0
    max: 10000
    step: 100
    unit_of_measurement: "W"
```

#### F.0.3 Watchdog Automation (Central HA)

```yaml
# Stage 1: Tighten limits when transformer exceeds 75%
automation:
  - alias: "Transformer Stage 1 — Limit Tightening"
    trigger:
      - platform: numeric_state
        entity_id: sensor.virtual_transformer
        above: 7500
    mode: single
    action:
      - variables:
          overload_factor: "{{ 7500 / states('sensor.virtual_transformer') | float }}"
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/1/control/max_power"
          payload: '{"value": {{ (2000 * overload_factor) | int }}, "unit": "W"}'
          qos: 2
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/2/control/max_power"
          payload: '{"value": {{ (2000 * overload_factor) | int }}, "unit": "W"}'
          qos: 2
      - delay: "00:03:00"
      - if:
          - condition: numeric_state
            entity_id: sensor.virtual_transformer
            above: 7500
        then:
          - service: mqtt.publish
            data:
              topic: "lem-netz/watchdog/escalation"
              payload: '{"stage": 2}'
              qos: 2
```

```yaml
# Stage 2: Direct shed (triggered by Stage 1 escalation or critical threshold)
automation:
  - alias: "Transformer Stage 2 — Direct Shed"
    trigger:
      - platform: numeric_state
        entity_id: sensor.virtual_transformer
        above: 9500
      - platform: mqtt
        topic: "lem-netz/watchdog/escalation"
    mode: single
    action:
      - repeat:
          for_each: ["lem-netz/house/1/control/shed", "lem-netz/house/2/control/shed"]
          sequence:
            - service: mqtt.publish
              data:
                topic: "{{ repeat.item }}"
                payload: '{"reason": "transformer_critical"}'
                qos: 2
      - service: persistent_notification.create
        data:
          title: "⚠ Critical Transformer Overload"
          message: "Transformer exceeded 95%. Shed commands issued to all households."
```

**Load Recovery:**

```yaml
automation:
  - alias: "Transformer Load Recovery"
    trigger:
      - platform: numeric_state
        entity_id: sensor.virtual_transformer
        below: 5500
    mode: single
    action:
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/1/control/max_power"
          payload: '{"value": 2000, "unit": "W"}'
          qos: 2
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/2/control/max_power"
          payload: '{"value": 2000, "unit": "W"}'
          qos: 2
      - delay: "00:00:05"
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/1/control/restore"
          payload: '{"reason": "load_recovered"}'
          qos: 2
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/2/control/restore"
          payload: '{"reason": "load_recovered"}'
          qos: 2
```

#### F.0.4 ESP32 Handler (Pseudocode)

The ESP32 receives downlink commands via LoRaWAN and processes them locally:

```c
// On LoRaWAN downlink received
void on_downlink(uint8_t fport, uint8_t *payload, uint8_t len) {
    if (fport == 1 && len >= 2) {
        // Command: max_power (2-byte unsigned int)
        uint16_t new_limit = (payload[0] << 8) | payload[1];
        current_max_power = new_limit;

        if (total_power > new_limit) {
            shed_lowest_priority_load();
        }
    }
    else if (fport == 2 && len >= 1 && payload[0] == 0x01) {
        // Command: shed
        shed_all_non_essential();       // GPIO off, Modbus coil off
        send_uplink(3, shed_confirmed); // Status: shed executed
    }
    else if (fport == 2 && len >= 1 && payload[0] == 0x00) {
        // Command: restore
        restore_all_loads();
    }
}
```

#### F.0.5 Failure Detection

| Situation | Detection | Action |
|-----------|-----------|--------|
| Transformer still > 80% after Stage 2 | `sensor.virtual_transformer` > 80% for 60s after shed | Alert admin; check individual household sensor data |
| ESP32 not responding | Sensor data stops updating on LoRaWAN path | Flag household for maintenance; check power/WiFi |

#### F.1 Optional: Advanced Optimization

For revenue-aware optimization (Phase 2), Home Assistant integrations can import dynamic prices (EPEX Spot, Tibber — see Phase D) and HA automations can implement basic rules (e.g., "charge battery when price < X"). For more sophisticated optimization with 48-hour lookahead, see the optimization tools in Section 12 (Appendix).

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

| Product | Price (incl. VAT) | German Shop | Key Features |
|---------|-------------------|-------------|---------------|
| **IMST iOKE868** | €119-129 (plus €7.50 USB power supply or €10 battery holder) | [shop.imst.de](https://shop.imst.de/wireless-solutions/lora-products/69/ioke868-lorawan-smart-metering-kit) · [iot-shop.de](https://iot-shop.de/shop/category/marke-imst-954) €126 | SML protocol, 868 MHz LoRaWAN, magnetic mount, USB or battery power, external antenna (magnetic base, 2m cable), configurable TX interval |
| **KLAX 2.0** | €149.90 | [iot-shop.de](https://iot-shop.de/shop/klax-2-0-lorawan-sml-opto-head-for-modern-power-meters-4365) | SML protocol, 868 MHz LoRaWAN, battery-powered (AA lithium, replaceable), IP21, compact (96×35×40mm), LoRaWAN Class A |
| **Fludia FM432ir** | €156 (direct) · €184.33 (iot-shop.de) | [fludia.com](https://shop.fludia.com/shop/en/isoUS/26-67-fm432ir-iot-sensor-for-german-electricity-meters-lorawan.html) · [iot-shop.de](https://iot-shop.de/shop/fl-fm432ir-fludia-fm432ir-lorawan-sml-optokopf-fur-moderne-stromzahler-6180) | SML protocol, 868 MHz LoRaWAN (also LTE-M/NB-IoT variants), battery-powered, configurable 1-15 min interval, 3.5-8 year battery life, made in France |

**Recommendation:** IMST iOKE868 is the most cost-effective choice for Phase 1. It offers flexible power options (USB or battery), external antenna for metallic cabinets, and the lowest per-unit cost. KLAX 2.0 is a solid battery-only alternative at €149.90. Fludia FM432ir offers the longest battery life but at higher cost.

### 5.2 Outdoor Gateways (Phase 1)

| Product | Price (incl. VAT) | German Shop | Key Features |
|---------|-------------------|-------------|---------------|
| **Milesight UG67-868M** | €588.19 | [reichelt.de](https://www.reichelt.de/outdoor-lorawan-gateway-mil-ug67-p305301.html) | IP67, 8-channel (SX1302), 868 MHz, 2000+ nodes, PoE PD, built-in Network Server, quad-core 1.5GHz, 512MB RAM, 8GB eMMC, MQTT/HTTP/BACnet/Modbus, external N-Female antenna connectors, supercapacitor backup |
| **Dragino DLOS8N-868** | €224.95 (no 4G) · €299.95 (with 4G) | [antratek.de](https://www.antratek.de/dlos8n-outdoor-lorawan-gateway) · [industry-electronics.de](https://industry-electronics.com/dragino/dlos8n-868-ec25-gateway-lora-outdoor-lorawan-lieske_1704078.htm) €392 | IP65, 10-channel (SX1302), 868 MHz, OpenWrt open source, PoE, GPS, WiFi, Ethernet, external fiber glass antenna |

**Recommendation:** Milesight UG67 is the preferred gateway for 100-1000 household deployments. It has proven scalability (2000+ nodes), built-in Network Server (reduces dependency on external NS), IP67 rating, and is available from reichelt.de with reliable stock. Dragino DLOS8N is a lower-cost alternative but less proven at scale — suitable for smaller deployments or as a development testbed. The UG67's built-in supercapacitor provides ~1 minute of operation after power loss for orderly shutdown.

**Note:** The UG67-868M model (no cellular) is sufficient since WiFi/Ethernet backhaul is used. The LTE variant (UG67-L04EU) is available if cellular failover is needed.

### 5.3 Server Platforms (Phase 1)

| Product | Price (incl. VAT) | German Shop | Key Features |
|---------|-------------------|-------------|---------------|
| **Home Assistant Green** | €139-€179 (Nabu Casa SRP: €179 as of Apr 2026) | [nabucasa.com](https://www.nabucasa.com) · [smartdomo.de](https://shopforward.de/i/31280-860011789703) €139 · [reichelt.de](https://www.reichelt.de) €145 | Pre-installed HA OS, 4GB RAM, 32GB eMMC, Gigabit Ethernet, 2×USB 2.0, HDMI, ARM A55 quad-core, 112×112×32mm, plug-and-play |
| **Raspberry Pi 5 (4GB) + case + PSU + SD** | ~€100-120 | [reichelt.de](https://www.reichelt.de) · [raspberrypi.com](https://www.raspberrypi.com) | Requires manual HA OS install, 4GB/8GB RAM options, USB boot, PCIe 2.0, more flexible but higher maintenance |

**Recommendation:** Home Assistant Green is the recommended platform. It eliminates setup complexity, has sufficient performance for 100+ household MQTT handling, and is the officially supported path. The RPi 5 alternative is suitable for technically experienced users or when budget is the primary constraint. At the current HA Green street price of ~€139-145, the price difference to an RPi 5 setup (~€100-120) is marginal given the convenience and support benefits.

### 5.4 Control Devices (Phase 2)

| Category | Product Examples | Price (approx.) | Protocol | Notes |
|----------|------------------|-----------------|----------|-------|
| Wallbox | go-e Charger Gemini, Wallbe Eco 2.0, DaheimLader, Zaptec Go | €500-1200 | OCPP, Modbus, REST | go-e and DaheimLader have native HA integrations; prefer local Modbus/REST over OCPP where possible |
| Smart Plug | **Shelly Pro with RS485 addon** | ~€100+ | Modbus RTU via RS485 | DIN-rail mount, Shelly Pro 4PM supports optional RS485 adapter |
| Smart Plug | **Generic Modbus relay** (e.g., Finders) | €20-50 | Modbus RTU via RS485 | DIN-rail mount, power measurement optional; use Modbus-capable |
| PV Inverter | SolarEdge, SMA, Kostal | varies | Modbus TCP | Most modern inverters support Modbus; check manufacturer documentation |
| Battery | BYD, Tesla Powerwall, FoxESS | varies | Modbus, REST | Battery BMS integration depends on manufacturer API openness |

**Note:** Smart plugs must support **Modbus RTU** (RS485) for direct connection to the bridge device. If a plug is WiFi-only, use a Modbus-capable relay instead (e.g., Shelly Pro with RS485 addon, or a generic Modbus relay module). Avoid cloud-only smart plugs.

### 5.5 Local Controller Hardware (Phase 1+2)

| Product | Price (incl. VAT) | German Shop | Use Case |
|---------|-------------------|-------------|----------|
| **ESP32 DevKit C** | €12-18 | [reichelt.de](https://www.reichelt.de) · [amazon.de](https://www.amazon.de) | MCU, 2 UARTs for Modbus RS485, SPI for LoRa module |
| **MAX3485 RS485 Module** | €3-5 | [reichelt.de](https://www.reichelt.de) · [iot-shop.de](https://iot-shop.de) | Required for Modbus RTU to wallbox/inverter |
| **LoRa radio module** (RFM95W / SX1276) | €8-15 | [reichelt.de](https://www.reichelt.de) · [iot-shop.de](https://iot-shop.de) | 868 MHz LoRaWAN transceiver, SPI interface |
| **IR optical read head** | €5-10 | [amazon.de](https://www.amazon.de) | Reads smart meter IR interface (SML protocol) |
| **USB power supply** | €5-10 | — | 5V USB power |
| **USB cable + enclosure** | €3-5 | — | Mounting and connectivity |

| Component Set | Cost |
|-----------|------|
| **ESP32 + RS485 + LoRa + IR + PSU + enclosure** | **~€36-63** |

For households requiring OCPP or Modbus TCP control (wallboxes without Modbus RTU), see the RPi-based alternative controller in Section 12.

### 5.6 Optimization Software (Phase 2)

| Software | License | Integration | Key Features |
|----------|---------|-------------|--------------|
| **HAEO** ([hass-energy/haeo](https://github.com/hass-energy/haeo)) | MIT | HA custom integration via HACS | Linear programming, 48h horizon, multi-node, constraint-aware |
| **AkkudoktorEOS** ([akkudoktor-eos/eos](https://github.com/akkudoktor-eos/eos)) | Open source | HA add-on | German market focus, predictive price/load, battery/heat pump |
| **EVCC** ([evcc-io/evcc](https://github.com/evcc-io/evcc)) | MIT | HA add-on (official) | 240+ manufacturers, OCPP server, browser config v0.300+ |
| **forty-two-watts** ([frahlg/forty-two-watts](https://github.com/frahlg/forty-two-watts)) | Open source | MQTT autodiscovery | Multi-inverter, 48h MPC planner, Go binary on Pi |

**Recommendation:** Start with HAEO for core optimization. Add AkkudoktorEOS for German price predictions. Use EVCC for wallbox OCPP control.

### 5.7 Price Comparison Summary

| Component | Recommended Product | Price (incl. VAT) | Quantity | Total |
|-----------|-------------------|-------------------|----------|-------|
| Central Server | Home Assistant Green | €145 | 1 | **€145** |
| Outdoor Gateway | Milesight UG67-868M | €588 | 1 | **€588** |
| PoE Injector | Included in UG67 bundle (reichelt.de) | €0 | — | **€0** |
| **Central Total** | | | | **€733** |
| | | | | |
| Bridge Device (per household) | ESP32 + LoRa + RS485 + IR head + PSU | €36-63 | 1 | **€36-63** |
| Smart Plug (Phase 2, optional) | Modbus relay (e.g., Shelly Pro RS485) | €50-100 | 1 | **€50-100** |
| **Per Household (Phase 1)** | | | | **~€36-63** |
| **Per Household (Phase 1+2)** | | | | **~€86-163** |

**Central cost exceeds the €300 target.** This is driven by the UG67 gateway (€588). Options to reduce:
- Use Dragino DLOS8N (€225) instead of UG67 → central total ~€370, but risks at scale
- Share gateway across multiple communities (up to 2000 nodes supported)
- Use RPi 5 (€100) instead of HA Green → saves ~€45

**Per-household cost is within the €200 target** for Phase 1 and within €350 for Phase 2.

---

## 6. Cost Implementation Guide

### 6.1 Budget Tracking - Phase 1

| Category | Target | Actual | Status |
|----------|--------|--------|--------|
| Central server | €100-150 | €145 (HA Green) | ⚠ Over budget |
| Outdoor gateway | €150-200 | €588 (UG67-868M) | ⚠ Over budget |
| Network equipment | €20-50 | €0 (PoE included) | ✓ Under budget |
| **Central Total** | **≤ €300** | **€733** | ⚠ Exceeds target |
| | | | |
| Bridge Device per household | €30-70 | €36-63 (ESP32 + LoRa + RS485 + IR + PSU) | ✓ Within budget |
| **Per Household (Phase 1)** | **≤ €200** | **~€36-63** | ✓ Well within budget |
| Smart Plug (Phase 2, per device) | €20-100 | €50-100 (Modbus relay) | ✓ Within budget |
| **Per Household (Phase 1+2)** | **≤ €350** | **~€86-163** | ✓ Within budget |

**Note:** Central total exceeds the €300 target primarily due to the Milesight UG67 gateway (€588). See Section 5.7 for cost reduction options.

### 6.2 Budget Tracking - Phase 2

| Category | Target | Actual | Notes |
|----------|--------|--------|-------|
| Control integration | €0-50 | €0 | Use existing devices where possible; EVCC/HAEO are free open source |
| Smart plugs (per device) | €20-100 | €50-100 | Modbus-capable relay (e.g., Shelly Pro RS485) |
| Additional software | €0 | €0 | HAEO, EVCC, AkkudoktorEOS — all open source |
| **Phase 2 Add-on** | **≤ €150/household** | **€50-100** (or €0 if existing Modbus devices) | ✓ Within budget |

### 6.3 Cost Optimization Tips

- Use WiFi backhaul instead of cellular to reduce ongoing costs
- Choose sensors with USB power (avoid battery costs)
- Install gateway in central location to maximize range
- Bulk purchase sensors for volume discount
- Leverage existing devices (wallboxes, PV) already owned by households
- Use open-source software (Arduino/PlatformIO, Home Assistant) to avoid licensing costs
- If a household already runs Home Assistant and has Modbus devices, consider integrating directly via HA Modbus integration instead of adding an ESP32

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
| Bridge device offline | Power or LoRa issue | Check USB power; verify device is joined to LoRaWAN network |
| No uplink data from bridge | Wrong DevEUI/AppKey | Verify device registration on network server |
| No Modbus data from wallbox | RS485 wiring issue | Check A/B polarity and termination resistors on RS485 bus |

### 8.2 Common Issues - Phase 2

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| Wallbox not responding | Modbus connection failed | Check RS485 A/B polarity; verify termination resistors; check USB power on ESP32 |
| Wallbox only supports OCPP | No OCPP support on ESP32 | Use RPi-based alternative controller (Section 12) |
| Price data not updating | API rate limit | Check integration logs; EPEX API may throttle |
| Optimization causing loss | Wrong household config | Verify tariff type setting in HA |
| §14a not working | No Smart Meter (iMSys) | Requires iMSys installation for Module 3 |
| Watchdog not firing | Network server config | Verify MQTT forwarding from gateway/network server to HA |
| Watchdog Stage 1 not reducing load | ESP32 not receiving downlink | Check LoRaWAN downlink queue; verify ESP32 is not in silent mode |
| Watchdog Stage 2 shed not executed | ESP32 offline | Check power; verify device last seen on network server |
| False positive overload | Sensor drift | Check virtual transformer calculation — sum of all households |
| Shed not acknowledged | ESP32 crashed | Restart ESP32; check serial logs |

### 8.3 Diagnostics

1. **Check sensor LED status** - Should indicate join and transmit
2. **Check gateway web interface** - Shows joined devices and uplink count
3. **Check MQTT topics** - Subscribe to `lem-netz/#` for all LEM messages
4. **Check bridge serial logs** - Connect to ESP32 via USB serial monitor
5. **Check wallbox Modbus** - On ESP32, verify Modbus register reads in serial debug output
6. **Check Home Assistant logs** - Settings → System → Logs
7. **Check price integration** - Verify EPEX/provider connectivity

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

## 11. Appendix A: Optimization Tools

The core architecture (watchdog + ESP32 limit enforcement) handles infrastructure safety. For revenue-aware optimization, these open-source tools can be added to Home Assistant.

### 11.1 HAEO (Recommended Primary Optimizer)

[Home Assistant Energy Optimizer](https://github.com/hass-energy/haeo) — linear programming optimizer:

- 48-hour horizon, 5-minute resolution
- Supports: batteries, solar, grid, constant/forecast loads, multi-node networks
- Constrained by infrastructure limits (transformer capacity is a configurable constraint)
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

### 11.2 AkkudoktorEOS (Predictive Add-on)

[AkkudoktorEOS](https://github.com/akkudoktor-eos/eos) — designed for German energy market:
- Predictive models for EPEX Spot prices
- Load forecasting based on historical data
- Optimizes: batteries, heat pumps, EV charging
- Available as Home Assistant add-on

### 11.3 EVCC (EV Charging)

[EVCC](https://github.com/evcc-io/evcc) — EV charging + energy management:
- 240+ manufacturers, OCPP server
- HA add-on (+390 HA sensors)
- Useful if wallboxes require OCPP control (see Appendix B)

### 11.4 Per-Household Configuration (for HAEO/AkkudoktorEOS)

| Household Type | Configuration | Goal |
|---------------|--------------|------|
| No PV, fixed tariff | Set import price to fixed rate | Minimize cost |
| PV only (EEG) | Set export price to feed-in rate | Maximize self-consumption |
| PV + Battery (Dynamic) | Enable dynamic rates | Arbitrage |
| Battery only | Enable dynamic import, export=0 | Charge cheap, discharge peak |
| Heat pump | Add §14a schedule | Shift to low-tariff periods |

---

## 12. Appendix B: Alternative Controller (RPi for OCPP/Modbus TCP)

If a household has devices that only support **OCPP** (e.g., Wallbox Pulsar Plus) or **Modbus TCP** (no RS485 port), the ESP32 cannot interface with them. Use a Raspberry Pi Zero 2W with WiFi/MQTT direct to the central server instead.

| Component | Purpose | Cost |
|-----------|---------|------|
| Raspberry Pi Zero 2W | WiFi + quad-core, runs Python agent | €18-22 |
| MicroSD card (32GB) | OS and data buffering | €8-12 |
| USB power supply | 5V power | €5-10 |
| USB RS485 adapter | Optional, for Modbus RTU devices | €8-12 |
| **Total** | | **€39-56** |

**Setup:**
1. Flash Raspberry Pi OS Lite to SD card
2. Install Python agent (pre-built script from central server repo)
3. Configure `config.yaml` with household ID, MQTT broker URL, device list, IR reader
4. Set up systemd service for auto-start
5. Use EVCC as OCPP client for wallbox control if needed

Unlike the ESP32 bridge, the RPi connects via **MQTT over WiFi** directly to the central broker (bypassing LoRaWAN). This is the exception — only for households that require OCPP or Modbus TCP. All MQTT topics and central server logic remain identical.

---

## 13. Appendix C: Reference Links

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

**Document Version:** 4.0
**Last Updated:** May 2026