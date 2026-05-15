# Technical Concept: LEM-Netz

## Implementation Guide for Phase 1 + Phase 2

**Version:** 3.4
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
| FR16-20, FR06 | Per-Household Local Controller | Bridges local Modbus/WiFi devices to central MQTT; buffers data during outages | 1+2 |

### 2.2 System Diagram - Phase 1 + 2

```
┌─────────────────────────────────────────────────────────────────────┐
│                      LEM-NETZ IMPLEMENTATION                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  [HOUSE 1]                 [HOUSE 2]                 [HOUSE N]     │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────┐  │
│  │  ENERGY METER    │    │  ENERGY METER    │    │ ENERGY METER │  │
│  │  (IR interface)  │    │  (IR interface)  │    │ (IR inter.)  │  │
│  └──────┬───────────┘    └──────┬───────────┘    └──────┬───────┘  │
│         │                       │                       │          │
│         ▼                       ▼                       ▼          │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────┐  │
│  │ iOKE868 SENSOR   │    │ iOKE868 SENSOR   │    │ iOKE868 SEN. │  │
│  │ (LoRaWAN 868MHz) │    │ (LoRaWAN 868MHz) │    │ (LoRaWAN)    │  │
│  └──────┬───────────┘    └──────┬───────────┘    └──────┬───────┘  │
│         │                       │                       │          │
│  ┌──────┴───────────┐    ┌──────┴───────────┐    ┌──────┴───────┐  │
│  │ CONTROL DEVICES  │    │ CONTROL DEVICES  │    │CONTROL DEV.  │  │
│  │ • Wallbox        │    │ • Wallbox        │    │ • Smart Plug │  │
│  │ • PV inverter    │    │ • Smart Plug     │    │ • Battery    │  │
│  │ • Battery BMS    │    │                   │    │              │  │
│  └──────┬───────────┘    └──────┬───────────┘    └──────┬───────┘  │
│         │ Modbus/WiFi           │ Modbus/WiFi           │ Modbus   │
│         ▼                       ▼                       ▼          │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────┐  │
│  │ LOCAL CONTROLLER │    │ LOCAL CONTROLLER │    │LOCAL CONTR.  │  │
│  │ (Tier 1/2/3)     │    │ (Tier 1/2/3)     │    │(Tier 1/2/3)  │  │
│  │ ESP32 / RPi / HA │    │ ESP32 / RPi / HA │    │ESP32 / RPi   │  │
│  └──────┬───────────┘    └──────┬───────────┘    └──────┬───────┘  │
│         │    MQTT over          │    MQTT over          │   MQTT    │
│         │    WiFi/Internet      │    WiFi/Internet      │   over    │
│         │    (TLS)              │    (TLS)              │   Mesh    │
│         │         │             │         │             │     │     │
│         └─────────┼─────────────┴─────────┼─────────────┴─────┘     │
│                   │                       │                         │
│          ┌────────┴───────────────────────┴──────────┐              │
│          │    OUTDOOR GATEWAY (Milesight UG67)       │              │
│          │    • LoRaWAN 868 MHz packet forwarder      │              │
│          │    • MQTT (via MQTT) to central server     │              │
│          └──────────────────────┬────────────────────┘              │
│                                 │ Ethernet                          │
│                                 ▼                                   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  CENTRAL SERVER (Home Assistant Green / RPi 5)              │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │   │
│  │  │ MQTT Broker  │  │  Database    │  │   Dashboard      │  │   │
│  │  │ (Mosquitto)  │  │ (InfluxDB)   │  │     (HA)         │  │   │
│  │  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │   │
│  │         │                  │                    │            │   │
│  │         ▼                  ▼                    ▼            │   │
│  │  ┌────────────────────────────────────────────────────────┐  │   │
│  │  │              OPTIMIZATION ENGINE                       │  │   │
│  │  │  ┌─────────────────┐  ┌──────────────────────────┐    │  │   │
│  │  │  │  Price Import   │  │  Control Logic           │    │  │   │
│  │  │  │ (EPEX, Tibber)  │  │  (Revenue-aware)         │    │  │   │
│  │  │  └─────────────────┘  └──────────────────────────┘    │  │   │
│  │  │         │                                               │  │   │
│  │  │         ▼                                               │  │   │
│  │  │  ┌──────────────────────────────────────────────────┐   │  │   │
│  │  │  │  INFRASTRUCTURE WATCHDOG                         │   │  │   │
│  │  │  │  • Monitors virtual transformer                  │   │  │   │
│  │  │  │  • Issues MQTT shed commands to local controllers│   │  │   │
│  │  │  │  • 80% trigger / 60% hysteresis                  │   │  │   │
│  │  │  └──────────────────────────────────────────────────┘   │  │   │
│  │  └────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
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

#### B.3 Local Controller Setup (20-45 minutes, depends on tier)

1. **Select Tier (see Section 3.2.2)**
   - **Tier 1 (ESP32):** For households with ≤2 Modbus devices, basic load shedding only
   - **Tier 2 (RPi Zero 2W):** For households with PV, battery, wallbox requiring full integration
   - **Tier 3 (Existing HA):** If household already runs Home Assistant

2. **Tier 1 — ESP32/ESPHome Setup**
   - Flash ESPHome firmware via USB (first-time) or OTA (updates)
   - Connect MAX3485 RS485 transceiver: ESP32 GPIO16 (TX)→ RS485, GPIO17 (RX) ← RS485
   - Wire RS485 A/B to wallbox or inverter Modbus terminals
   - Write ESPHome YAML with Modbus client, MQTT topics per Section 3.2.1
   - Configure WiFi SSID/password for household network
   - Set MQTT broker address to central server IP (or DDNS hostname)
   - Enable MQTT TLS with self-signed or Let's Encrypt certificate
   - Test: verify sensor data appears on `lem-netz/house/<id>/sensor/...`

3. **Tier 2 — RPi Zero 2W Setup**
   - Flash Raspberry Pi OS Lite to SD card
   - Install Node-RED or Python agent (pre-built script from central server repo)
   - Optional: Connect RS485 USB dongle for Modbus RTU devices
   - Configure `config.yaml` with:
     - Household ID (1, 2, 3, ...)
     - MQTT broker URL, port, TLS certificate path
     - Device list (wallbox, PV, battery) with Modbus register maps or API endpoints
   - Enable local buffering: SQLite database logs all readings
   - Set up systemd service for auto-start
   - Test: `mosquitto_pub` and `mosquitto_sub` to verify MQTT link

4. **Tier 3 — Existing HA Bridge**
   - Install "Remote Home-Assistant" add-on in local HA instance
   - Configure connection to central HA URL and API token
   - Select entities to expose to central
   - OR: Install Mosquitto broker locally on existing HA and bridge to central broker

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
| Local Control | ESP32 / RPi / existing HA | Continues polling local devices (wallbox, PV, battery); buffers readings when MQTT link to central is down; executes watchdog shed commands regardless of connectivity | Configurable buffer (hours to days on RPi; limited on ESP32) |
| Gateway | Milesight UG67 | Continues receiving LoRa packets; forwards when connection restored | Built-in NS queues data |
| MQTT | Mosquitto broker | All local, no internet dependency | Retained messages on topics |
| Server | Home Assistant | Continues running; dashboard and automations work locally | Full database local |

**No component requires internet for core operation.** The price optimizations in Phase 2 require intermittent internet for EPEX/Tibber data, but the system degrades gracefully: if price data is unavailable, optimization falls back to default charging patterns.

---

## 3.2 Per-Household Local Integration

Control devices (wallbox, PV inverter, battery BMS, smart plugs) are located inside each household and cannot connect directly to the central server. Each house requires a **local controller** that bridges household devices to the central MQTT broker.

### 3.2.1 MQTT Topic Hierarchy

All local controllers communicate with the central server via a standardized MQTT topic structure. Three message types exist: **constraints** (hard limits), **recommendations** (suggestions), and **commands** (direct control, fallback only).

**Topic patterns:**

```
lem-netz/house/<household_id>/sensor/<device>/<metric>       # Data from house to central
lem-netz/house/<household_id>/constraint/<type>               # Hard limits from central to house
lem-netz/house/<household_id>/recommendation/<type>           # Non-binding suggestions from central
lem-netz/house/<household_id>/command/<action>                # Direct commands (fallback only)
lem-netz/house/<household_id>/status/<action>                 # Confirmations from house to central
lem-netz/house/<household_id>/heartbeat                       # Connectivity alive signal
```

**Example topics:**

| Topic | Direction | Payload | Description |
|-------|-----------|---------|-------------|
| `lem-netz/house/1/sensor/wallbox/power` | House → Central | `{"value": 4500, "unit": "W"}` | Wallbox current power consumption |
| `lem-netz/house/1/sensor/pv/production` | House → Central | `{"value": 3200, "unit": "W"}` | PV current production |
| `lem-netz/house/1/sensor/battery/soc` | House → Central | `{"value": 65, "unit": "%"}` | Battery state of charge |
| `lem-netz/house/1/constraint/max_power` | Central → House | `{"value": 5000, "unit": "W", "valid_from": "...", "valid_until": "..."}` | Hard limit on total household import power |
| `lem-netz/house/1/constraint/max_export` | Central → House | `{"value": 3000, "unit": "W"}` | Hard limit on grid feed-in (PV, battery discharge) |
| `lem-netz/house/1/recommendation/charge_window` | Central → House | `{"start": "14:00", "end": "16:00", "price": 0.25}` | Suggested EV charging period |
| `lem-netz/house/1/command/shed` | Central → House | `{"reason": "transformer_overload", "severity": "critical"}` | Shed non-essential loads (emergency fallback only) |
| `lem-netz/house/1/command/restore` | Central → House | `{"reason": "load_recovered"}` | Restore previously shed loads |
| `lem-netz/house/1/status/shed` | House → Central | `{"status": "executed", "loads_shed": ["wallbox", "smart_plug_1"]}` | Shed confirmation |
| `lem-netz/house/1/status/compliance` | House → Central | `{"constraint": "max_power", "value": 4800, "compliant": true}` | Compliance report to watchdog |
| `lem-netz/house/1/heartbeat` | House → Central | `{"uptime": 3600, "tier": "2"}` | Alive signal |

### 3.2.2 Local Controller Tiers

Different households have different needs and existing equipment. The architecture supports three tiers:

| Tier | Hardware | Cost (approx.) | Protocol Support | Data Buffering | Best For |
|------|----------|----------------|-----------------|----------------|----------|
| **1** | ESP32 + RS485 module + ESPHome | €15-25 | Modbus RTU (1-2 devices), WiFi MQTT | Limited (last values) | Basic load shedding, single wallbox or smart plug |
| **2** | Raspberry Pi Zero 2W + Python/Node-RED agent | €35-55 | Modbus TCP/RTU, OCPP client, REST APIs, WiFi MQTT | Hours to days (SD card) | Full integration with PV, battery, wallbox, multiple devices |
| **3** | Existing Home Assistant instance | €0 (already present) | All HA integrations, Remote HA or MQTT bridge | Full (HA database) | Households already running HA |

**Constraint handling per tier:**

| Tier | How Constraints Are Enforced | Local Optimization Capability |
|------|------------------------------|-------------------------------|
| **1** (ESP32) | Hard limit: if `constraint/max_power = 5000W` exceeded, turns off non-essential loads in fixed priority order (plug first, wallbox second). Simple threshold enforcement — no scheduling. | None — fixed priority only |
| **2** (RPi Zero 2W) | Intelligent enforcement: receives constraint and decides locally *how* to meet it. Can delay EV charging by 1h, reduce heat pump temperature, or discharge battery — whichever best fits household preferences. | Can run local optimizer (Node-RED, simple Python scheduler) |
| **3** (Existing HA) | Fully integrated: constraint becomes a condition in local HA automations. Household defines its own rules for how to respect the limit while maximizing comfort/cost savings. | Full HA automations + local add-ons |

**Tier 1 (ESP32/ESPHome):**
- Runs ESPHome firmware — connects to central HA via native API or MQTT
- Modbus RTU via UART + MAX3485 transceiver for wallbox/inverter
- Simple YAML configuration, no coding required
- Power: USB 5V (can share with iOKE868 power supply)
- Security: MQTT over TLS, unique device authentication per house
- Constraint enforcement via ESPHome lambda: compare current power sum against `constraint/max_power` value, shed loads accordingly

**Tier 2 (RPi Zero 2W):**
- Runs a lightweight Python agent or Node-RED flows
- Supports Modbus TCP (Ethernet/WiFi) and Modbus RTU (USB/GPIO)
- Can run EVCC as OCPP client for wallbox control
- Local SQLite database buffers all readings when MQTT link is down
- Periodic heartbeat confirms connectivity; watchdog checks time since last heartbeat
- Constraint handler: Python callback on `constraint/#` topics that adjusts local device schedules
- Compliance reporter: periodically publishes current load against limit to `status/compliance`

**Tier 3 (Existing HA):**
- Use "Remote Home-Assistant" add-on to bridge entities to central HA
- Or: MQTT auto-discovery — local HA publishes all device entities to central MQTT broker
- Zero additional hardware cost
- Requires household to already run HA competently
- Constraint handling: HA automation triggers on `constraint/max_power` changes, adjusts device states accordingly

### 3.2.3 Backhaul Options

The link between the local controller and the central MQTT broker depends on the household's network situation:

| Option | Description | Internet Required? | Security | Latency |
|--------|-------------|-------------------|----------|---------|
| **MQTT over LAN/WiFi** | Controller connects to central broker via local Ethernet/WiFi | No | TLS + password | <1ms |
| **MQTT over Internet** | Controller connects to central broker via house's existing internet (DDNS/VPN) | Yes | TLS + client certificate | 10-50ms |
| **Community WiFi Mesh** | Dedicated neighborhood mesh (e.g., batman-adv, OpenWrt mesh) | No | WPA2 + TLS | 1-5ms |
| **LoRaWAN status only** | Basic heartbeat and watchdog status via LoRaWAN downlink | No | AES-128 | seconds |

**Recommendation:** Default to MQTT over the household's existing internet connection with TLS + client certificate. This requires no additional infrastructure and works for any tier. For offline-critical deployment (FR06), add a community WiFi mesh or accept that watchdog commands are sent via LoRaWAN downlink as fallback.

### 3.2.4 Device Mapping Per Tier

| Device | Tier 1 (ESP32) | Tier 2 (RPi Zero) | Tier 3 (Existing HA) |
|--------|----------------|-------------------|----------------------|
| Wallbox (Modbus TCP) | — (needs Modbus RTU via RS485) | ✓ Modbus TCP or EVCC OCPP | ✓ Any HA integration |
| Wallbox (Modbus RTU) | ✓ RS485 + MAX3485 | ✓ RS485 USB dongle | ✓ RS485 USB dongle |
| Wallbox (OCPP) | — | ✓ EVCC as OCPP client | ✓ EVCC add-on |
| PV Inverter (Modbus) | ✓ (limited to 1-2 devices) | ✓ Full Modbus TCP | ✓ Any HA integration |
| Battery BMS | — | ✓ Modbus or REST | ✓ Any HA integration |
| Smart Plug (Shelly) | ✓ Shelly local HTTP API | ✓ Shelly local HTTP API | ✓ Shelly integration |
| Smart Plug (MQTT) | ✓ ESPHome native | ✓ Node-RED / Python | ✓ HA MQTT integration |

### 3.2.5 Control Philosophy — Separation of Concerns

The system uses a **two-tier control model**: constraints and recommendations are the primary mode; direct commands are the emergency fallback.

**Why constraints over commands:**
- **Household autonomy**: Each household decides how to meet limits based on its own priorities (comfort, cost, PV yield)
- **Economic fairness (FR20)**: Different pricing models (EEG, dynamic, §14a) require different optimization strategies — only the household knows its tariff
- **Privacy**: Central system does not need to know which devices are in each house or their schedules
- **Resilience**: A household can continue operating normally even if the MQTT link to central is temporarily lost — it stays within the last known constraint
- **Future-proofing**: New device types can be added at the household level without central coordination

**Responsibility split:**

| Concern | Central System | Household System |
|---------|---------------|------------------|
| Transformer safety | Monitors virtual transformer, calculates fair-share limits per household | Respects `max_power` and `max_export` constraints |
| Load optimization | Broadcasts price signals and recommended charge windows | Decides device schedules within constraints (or ignores recommendations) |
| Compliance enforcement | Checks if household stays within limits; escalates if violated | Reports compliance status periodically |
| Emergency shutdown | Issues direct `command/shed` only as last resort | Local controller always listens for and executes shed commands |
| Local preferences | Unknown — does not track household devices or behavior | Full knowledge of devices, tariffs, comfort requirements |

**Normal operation flow (constraint mode):**

```
Central                                                   Household
   │                                                          │
   ├─ constraint/max_power = 5000W ──────────────────────────►│
   │                                                          ├─ Decides: "I have 5kW budget"
   │                                                          │  → Run heat pump at 2kW
   │                                                          │  → Charge EV at 3kW
   │                                                          │  (within limit)
   │◄── sensor/wallbox/power = 3000W ─────────────────────────┤
   │◄── sensor/plug/power = 2000W ────────────────────────────┤
   │◄── status/compliance = {compliant: true} ────────────────┤
   │                                                          │
```

**Emergency flow (command fallback):**

```
Central                                                   Household
   │                                                          │
   ├─ constraint/max_power = 3000W ──────────────────────────►│  Step 1: tighten constraint
   │◄── status/compliance = {compliant: true} ────────────────┤
   │  (30s later — transformer still overloaded)               │
   ├─ command/shed ──────────────────────────────────────────►│  Step 2: direct command
   │◄── status/shed = {executed} ─────────────────────────────┤
   │                                                          │
```

**Constraint priority order (when limits conflict):**

1. **`command/shed`** — highest priority, must be executed immediately (transformer emergency)
2. **`constraint/max_power`** — hard limit on total import, household must stay below
3. **`constraint/max_export`** — hard limit on grid feed-in
4. **`recommendation/*`** — non-binding, household may ignore based on local optimization

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

### Phase E: Control Integration via Local Controller

> **Architecture:** All device integration (wallbox, PV, battery, smart plugs) happens on the **per-household local controller**, not on the central server. The local controller publishes normalized data to central MQTT and subscribes to control commands. See Section 3.2 for the full topic hierarchy and tier options.

#### E.1 Integration Architecture

```
┌─ HOUSEHOLD ─────────────────────────────────────────────────────┐
│                                                                  │
│  Wallbox ──Modbus──┐                                             │
│  PV Inverter ─Modbus┤──→ Local Controller ──MQTT──→ Central HA  │
│  Battery ───Modbus──┤       (Tier 1/2/3)    (TLS)               │
│  Smart Plug ──WiFi──┘                                             │
│                                                                  │
│  Local Controller handles:                                       │
│  • All protocol translation (Modbus, OCPP, REST)                 │
│  • All local device discovery and polling                         │
│  • Data normalization to MQTT topic schema                        │
│  • Constraint enforcement: stay within max_power/max_export      │
│  • Recommendation processing: decide locally whether to follow   │
│  • Local execution of shed commands (emergency fallback only)    │
│  • Data buffering during network outages                          │
│  • Compliance reporting: publish current status against limits   │
└──────────────────────────────────────────────────────────────────┘
```

#### E.1.1 Constraint and Recommendation Handling

> **Primary mode:** The central system sends constraints (hard limits) and recommendations (suggestions). The local controller is responsible for staying within constraints while deciding autonomously how to use its available budget. Direct commands are only used in emergencies.

**Constraint flow per device type:**

| Constraint Topic | Applies To | Local Controller Action |
|-----------------|------------|------------------------|
| `constraint/max_power` | All devices (total household import) | Sum all device power; if approaching limit, reduce non-essential loads first |
| `constraint/max_export` | PV inverter, battery | Limit grid feed-in by curtailing PV or stopping battery discharge |
| `command/shed` | All protected devices | Immediate shutdown — emergency only, overrides all constraints |

**Recommendation flow:**

| Recommendation Topic | Meaning | Household Decision |
|--------------------|---------|-------------------|
| `recommendation/charge_window` | "Cheapest prices 14:00-16:00" | Tier 2/3: schedule EV charging accordingly. Tier 1: ignore (no scheduling ability). |
| `recommendation/price_signal` | Current market price signal | Tier 2/3: use as input to local optimization. Tier 1: ignore. |

**Local controller constraint enforcement examples per tier:**

```
Tier 1 (ESP32):
  Subscribe: constraint/max_power
  On update: if total_power > max_power, shed lowest-priority device
  Logic:     "I have 5kW limit. Currently using 5.2kW. Turn off smart_plug_3."

Tier 2 (RPi Zero):
  Subscribe: constraint/max_power + recommendation/price_signal
  On update: re-run local optimizer with new constraint
  Logic:     "I have 5kW limit and prices peak at 18:00.
              Option A: reduce EV from 11kW to 5kW, charge battery from grid.
              Option B: delay EV until 22:00, use PV for battery.
              → Pick Option B (better for household cost)."

Tier 3 (Existing HA):
  Subscribe: constraint/max_power
  Trigger HA automation: "When max_power changes, adjust device schedules"
  Full HA automation capabilities
```

**Compliance reporting (all tiers):**

Each local controller periodically publishes its compliance status:

```json
topic: lem-netz/house/<id>/status/compliance
payload: {
  "constraint": "max_power",
  "limit": 5000,
  "current_load": 4800,
  "compliant": true,
  "devices": {
    "wallbox": 3000,
    "smart_plug_1": 1200,
    "smart_plug_2": 600
  }
}
```

If a household is non-compliant (current_load > limit), the central watchdog can:
1. Log warning
2. Tighten constraint further
3. If persistent: escalate to direct `command/shed`

#### E.2 Wallbox/EV Charger Integration (on Local Controller)

1. **Supported Protocols per Tier**

   | Protocol | Tier 1 (ESP32) | Tier 2 (RPi Zero) | Tier 3 (Existing HA) |
   |----------|----------------|-------------------|----------------------|
   | Modbus RTU (RS485) | ✓ (MAX3485) | ✓ (USB dongle) | ✓ (USB dongle) |
   | Modbus TCP | — | ✓ | ✓ |
   | OCPP 1.6/2.0 | — | ✓ (via EVCC) | ✓ (via EVCC add-on) |
   | Manufacturer REST | — | ✓ (curl/Python) | ✓ (HA integration) |

2. **Data Published to Central MQTT**

   | Topic | Payload | Interval |
   |-------|---------|----------|
   | `lem-netz/house/<id>/sensor/wallbox/power` | `{"value": 4500, "unit": "W"}` | Every 15s or on change |
   | `lem-netz/house/<id>/sensor/wallbox/energy` | `{"value": 12.5, "unit": "kWh"}` | Every 5 min |
   | `lem-netz/house/<id>/sensor/wallbox/status` | `{"state": "charging", "phase_count": 3}` | On change |
   | `lem-netz/house/<id>/status/shed` | `{"status": "executed", "loads_shed": ["wallbox"]}` | On watchdog command |

3. **Constraints and Commands Received from Central MQTT**

   | Topic | Type | Payload | Local Controller Action |
   |-------|------|---------|------------------------|
   | `constraint/max_power` | Constraint | `{"value": 5000, "unit": "W"}` | Reduce wallbox current so total household stays below limit. Tier 1: hard-cut at threshold. Tier 2/3: smart reduction within local schedule. |
   | `constraint/max_export` | Constraint | `{"value": 3000, "unit": "W"}` | Limit PV export or battery discharge to stay below feed-in cap |
   | `recommendation/charge_window` | Recommendation | `{"start": "14:00", "end": "16:00"}` | Tier 2/3: schedule charging within window if beneficial. Tier 1: ignore. |
   | `command/shed` | Fallback command | `{"reason": "transformer_overload", "priority": 1}` | Immediately stop wallbox charging regardless of local preferences. Only used after constraint escalation fails. |

4. **OCPP Compatibility Warning**
   - Some wallbox manufacturers have OCPP implementation issues:
     - **Wallbox Pulsar Plus/Pro**: Known OCPP disconnects every few days requiring restart of both wallbox and EVCC; firmware 6.7.x introduced daily forced reboots that break OCPP connections
     - **General**: Prefer local Modbus/REST APIs over OCPP when available
     - **EVCC** is actively developed (240+ manufacturers supported) — test compatibility before deployment
   - On Tier 2, run EVCC locally on the RPi Zero as an OCPP client
   - On Tier 3, use the EVCC HA add-on in the local HA instance
   - See [EVCC GitHub Issues](https://github.com/evcc-io/evcc/issues) for known problems

#### E.3 Smart Plug/Switch Integration (on Local Controller)

All smart plug integration runs on the local controller, which reads power data and relays on/off commands:

1. **Per Tier**

   | Device | Tier 1 (ESP32) | Tier 2 (RPi Zero) | Tier 3 (Existing HA) |
   |--------|----------------|-------------------|----------------------|
   | Shelly Plus Plug S | HTTP GET local IP for power | HTTP API via requests/curl | Shelly HA integration |
   | Shelly Pro 4PM | HTTP GET for each channel | HTTP API | Shelly HA integration |
   | Generic MQTT plug | ESPHome output component | Python MQTT client | HA MQTT integration |
   | Tuya Smart Plug | — | Tuya API (if local) | Tuya HA integration |

2. **Data Flow**
   - **Sensor data**: Local controller polls Shelly at `http://<shelly-ip>/rpc/Switch.GetStatus` → publishes to `lem-netz/house/<id>/sensor/plug/<name>/power`
   - **Constraint**: Central publishes `constraint/max_power` → local controller sums all plug power, turns off lowest-priority plugs if approaching limit
   - **Control** (fallback): Central publishes `command/shed` → local controller sends HTTP POST to Shelly → confirms via `lem-netz/house/<id>/status/plug/<name>`

3. **Configuration**
   - Assign static IP to each smart plug via DHCP reservation
   - Register plug MAC address in local controller config
   - Assign each plug a shed priority (1 = shed first, 3 = shed last) for constraint enforcement
   - Tag each plug with `transformer_protected: true` in controller config for watchdog

#### E.4 PV System Integration (on Local Controller)

1. **Per Tier**

   | Inverter | Tier 1 (ESP32) | Tier 2 (RPi Zero) | Tier 3 (Existing HA) |
   |----------|----------------|-------------------|----------------------|
   | SolarEdge (Modbus TCP) | — | ✓ modbus_read.py | ✓ HA SolarEdge integration |
   | SMA (Modbus TCP) | — | ✓ SMA Modbus client | ✓ HA SMA integration |
   | Kostal (Modbus TCP) | ✓ (RS485 to Modbus RTU) | ✓ Modbus RTU or TCP | ✓ HA Kostal integration |
   | Victron (Venus OS) | — | ✓ REST API on Venus | ✓ HA Victron integration |

2. **Data Published**

   | Topic | Payload |
   |-------|---------|
   | `lem-netz/house/<id>/sensor/pv/production` | `{"value": 3200, "unit": "W"}` |
   | `lem-netz/house/<id>/sensor/pv/daily_yield` | `{"value": 14.2, "unit": "kWh"}` |
   | `lem-netz/house/<id>/sensor/pv/total_yield` | `{"value": 4520, "unit": "kWh"}` |

3. **Constraints**
   - `constraint/max_export`: If set, local controller must limit PV feed-in. Tier 1 can only disconnect inverter (on/off). Tier 2/3 can set power curtailment via Modbus register if inverter supports it.
   - `recommendation/sell_signal`: Central suggests "export now" (high prices) — Tier 2/3 may increase feed-in from battery if profitable.

#### E.5 Battery Storage Integration (on Local Controller)

1. **Per Tier**

   | System | Tier 1 (ESP32) | Tier 2 (RPi Zero) | Tier 3 (Existing HA) |
   |--------|----------------|-------------------|----------------------|
   | BYD (Modbus) | — | ✓ Modbus TCP/RTU | ✓ HA Modbus integration |
   | Tesla Powerwall | — | ✓ Local Gateway API | ✓ HA Powerwall integration |
   | FoxESS (Modbus) | — | ✓ Modbus client | ✓ HA Modbus integration |
   | Generic Modbus BMS | ✓ (simple registers) | ✓ Full register map | ✓ HA Modbus integration |

2. **Data Published**

   | Topic | Payload |
   |-------|---------|
   | `lem-netz/house/<id>/sensor/battery/soc` | `{"value": 65, "unit": "%"}` |
   | `lem-netz/house/<id>/sensor/battery/power` | `{"value": -1500, "unit": "W"}` (negative = charging) |
   | `lem-netz/house/<id>/sensor/battery/status` | `{"state": "discharging"}` |

3. **Constraints**
   - `constraint/max_export`: Limit battery discharge to grid. Tier 2/3 can adjust charge/discharge setpoints.
   - `constraint/max_power`: Total household limit — battery can help by discharging to cover load instead of importing from grid.
   - `recommendation/charge_window`: Tier 2/3 can schedule battery charging during low-price periods to reduce cost.

### Phase F: Revenue-Aware Optimization

#### F.0 INFRASTRUCTURE SAFETY WATCHDOG (CRITICAL - IMPLEMENT FIRST)

> **IMPLEMENTATION ORDER:** Infrastructure safety MUST be implemented BEFORE economic optimization. The watchdog is the enforcement mechanism.

The watchdog operates in **two stages**: first by tightening constraints (allowing households to self-regulate), then by direct shed commands only if the constraint tightening does not resolve the overload within a timeout.

**Two-stage architecture:**
```
Central HA detects overload (80%)
  │
  ├─ Stage 1: constraint/max_power = reduced_limit ──► Household autonomously complies
  │     Wait 30s for compliance
  │
  ├─ Stage 2: command/shed ──► Immediate load shed
  │     (only if still overloaded after Stage 1)
  │
  └─ Recovery: restore constraints → restore loads
```

#### F.0.1 Watchdog Stages

**Stage 1 — Constraint Tightening (normal response):**

| Step | Action | Detail |
|------|--------|--------|
| 1.1 | Detect threshold | `sensor.virtual_transformer` exceeds 80% (8000W for 10kVA) |
| 1.2 | Calculate fair share | Reduce each household's `max_power` proportionally: e.g., total load 10kW across 10 houses → each gets 800W instead of 1000W |
| 1.3 | Publish constraint | `lem-netz/house/<id>/constraint/max_power` = new limit per household |
| 1.4 | Wait for compliance | 30-second window for households to self-regulate |
| 1.5 | Check compliance | Sum reported loads from `status/compliance` topics. If total < 80% → resolved. If still > 80% → proceed to Stage 2. |

**Stage 2 — Direct Shed (escalation, only if Stage 1 fails):**

| Step | Action | Detail |
|------|--------|--------|
| 2.1 | Identify non-compliant | Households whose `status/compliance` shows current_load > limit |
| 2.2 | Send shed commands | `lem-netz/house/<id>/command/shed` to non-compliant households |
| 2.3 | Acknowledge | Local controllers confirm execution via `status/shed` |
| 2.4 | Verify resolution | If transformer still > 80%, widen shed to all households, not just non-compliant |
| 2.5 | Alert | Create persistent notification and log event |

#### F.0.2 Household Compliance Configuration

```yaml
# Central HA: Household power limit configuration
lem_netz:
  transformer:
    max_power: 10000           # Transformer rating (VA)
    warning_threshold: 0.8      # 80% — trigger Stage 1
    critical_threshold: 0.95    # 95% — trigger Stage 2 immediately (skip Stage 1)
    recovery_threshold: 0.6     # 60% — restore loads
    stage1_timeout: 30          # Seconds to wait before escalating to Stage 2

  households:
    1:
      name: "Household 1"
      base_allocation: 2000     # Normal max_power (W)
      devices:
        - id: wallbox
          type: ev_charger
          max_power: 11000
        - id: plug_pool_pump
          type: smart_plug
          max_power: 1500
        - id: plug_heater
          type: smart_plug
          max_power: 2000
```

#### F.0.3 Watchdog Automation (Central HA)

```yaml
# Stage 1: Tighten constraints when transformer exceeds 80%
automation:
  - alias: "Transformer Stage 1 — Constraint Tightening"
    trigger:
      - platform: numeric_state
        entity_id: sensor.virtual_transformer
        above: 8000  # 80% of 10kVA
    mode: single
    action:
      # Calculate reduction factor
      - variables:
          overload_factor: "{{ 8000 / states('sensor.virtual_transformer') | float }}"
      
      # Publish reduced max_power to each household
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/1/constraint/max_power"
          payload: '{"value": {{ (2000 * overload_factor) | int }}, "unit": "W", "reason": "transformer_overload", "valid_until": "{{ (now() + timedelta(minutes=15)).isoformat() }}"}'
          qos: 2
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/2/constraint/max_power"
          payload: '{"value": {{ (2000 * overload_factor) | int }}, "unit": "W", "reason": "transformer_overload", "valid_until": "..."}'
          qos: 2
      # ... repeat per household ...

      # Wait for households to self-regulate
      - delay: "00:00:30"

      # Check if resolved — if not, escalate to Stage 2
      - if:
          - condition: numeric_state
            entity_id: sensor.virtual_transformer
            above: 8000
        then:
          - service: mqtt.publish
            data:
              topic: "lem-netz/watchdog/escalation"
              payload: '{"stage": 2, "reason": "constraint_tightening_insufficient"}'
              qos: 2
```

```yaml
# Stage 2: Direct shed commands (triggered by escalation or 95% threshold)
automation:
  - alias: "Transformer Stage 2 — Direct Shed"
    trigger:
      - platform: numeric_state
        entity_id: sensor.virtual_transformer
        above: 9500  # 95% — emergency, skip Stage 1
      - platform: mqtt
        topic: "lem-netz/watchdog/escalation"
    mode: single
    action:
      # Send direct shed to all households
      - repeat:
          for_each: ["lem-netz/house/1/command/shed", "lem-netz/house/2/command/shed"]
          sequence:
            - service: mqtt.publish
              data:
                topic: "{{ repeat.item }}"
                payload: '{"priority": "all", "reason": "transformer_critical"}'
                qos: 2

      - service: persistent_notification.create
        data:
          title: "⚠ Critical Transformer Overload"
          message: "Transformer exceeded 95%. Direct shed commands issued to all households."

      # Log for audit
      - service: system_log.write
        data:
          message: "WATCHDOG: Stage 2 shed triggered at {{ states('sensor.virtual_transformer') }}W"
          level: warning
```

**Load Recovery (two-stage, reverse order):**

```yaml
automation:
  - alias: "Transformer Load Recovery"
    trigger:
      - platform: numeric_state
        entity_id: sensor.virtual_transformer
        below: 6000  # 60% — hysteresis
    mode: single
    action:
      # Stage 1: Restore constraints to normal allocation
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/1/constraint/max_power"
          payload: '{"value": 2000, "unit": "W", "reason": "load_recovered"}'
          qos: 2
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/2/constraint/max_power"
          payload: '{"value": 2000, "unit": "W", "reason": "load_recovered"}'
          qos: 2
      # ... repeat per household ...

      # Stage 2: Restore any previously shed loads
      - delay: "00:00:05"
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/1/command/restore"
          payload: '{"reason": "load_recovered"}'
          qos: 2
      - service: mqtt.publish
        data:
          topic: "lem-netz/house/2/command/restore"
          payload: '{"reason": "load_recovered"}'
          qos: 2
```

#### F.0.4 Local Controller Handler (on each local controller)

The local controller subscribes to both `constraint/#` and `command/#` topics:

**Constraint handler (all tiers, primary mode):**

```python
# Every local controller handles constraint/max_power updates
def on_constraint_max_power(client, userdata, msg):
    payload = json.loads(msg.payload)
    new_limit = payload["value"]
    
    # Tier 1: simple threshold check and shed
    if total_power() > new_limit:
        shed_lowest_priority_device()
    
    # Tier 2/3: re-run local optimizer with new constraint
    local_optimizer.set_max_power(new_limit)
    local_optimizer.optimize()
    
    # Publish compliance
    client.publish("lem-netz/house/1/status/compliance",
        json.dumps({
            "constraint": "max_power",
            "limit": new_limit,
            "current_load": total_power(),
            "compliant": total_power() <= new_limit
        }))

client.subscribe("lem-netz/house/1/constraint/#")
client.on_message = on_constraint_max_power
```

**Shed handler (all tiers, fallback):**

```python
def on_shed_command(client, userdata, msg):
    payload = json.loads(msg.payload)
    
    # Immediate shutdown of all non-essential loads
    for device in protected_devices:
        device.turn_off()
    
    # Acknowledge
    client.publish("lem-netz/house/1/status/shed",
        json.dumps({"status": "executed", "reason": payload.get("reason", "unknown")}))

client.subscribe("lem-netz/house/1/command/shed")
client.on_message = on_shed_command
```

**Tier 1 (ESP32/ESPHome) full handler:**

```yaml
# ESPHome: Handle both constraint and command topics
api:
  on_mqtt_message:
    - topic: lem-netz/house/1/constraint/max_power
      then:
        - lambda: |-
            // Extract limit value from JSON
            int new_limit = parse_json_value(x, "value");
            id(current_max_power) = new_limit;
            
            // Check if current load exceeds limit
            if (id(total_power) > new_limit) {
              id(smart_plug_1).turn_off();
            }
        - mqtt.publish:
            topic: lem-netz/house/1/status/compliance
            payload: !lambda |-
              return "{\"constraint\": \"max_power\", \"limit\": " + 
                     String(id(current_max_power)) + 
                     ", \"compliant\": " + 
                     (id(total_power) <= id(current_max_power) ? "true" : "false") + "}";

    - topic: lem-netz/house/1/command/shed
      then:
        - lambda: |-
            id(smart_plug_1).turn_off();
            id(wallbox_modbus).write_register(1000, 0);
        - mqtt.publish:
            topic: lem-netz/house/1/status/shed
            payload: '{"status": "executed"}'
```

#### F.0.5 Watchdog Failure Detection and Audit

| Situation | Detection | Action |
|-----------|-----------|--------|
| Household not publishing compliance | No `status/compliance` within 60s of constraint update | Mark as unresponsive; retry via LoRaWAN downlink; log warning |
| Household non-compliant after Stage 1 | `status/compliance.compliant = false` after 30s timeout | Escalate to Stage 2 for that specific household |
| Transformer still overloaded after full Stage 2 | `sensor.virtual_transformer` > 80% 10s after shed commands | Broaden shed to all households including non-protected devices |
| Local controller crash | No heartbeat for >120s | Flag for maintenance; check power/WiFi at household |

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

| Product | Price (incl. VAT) | German Shop | Key Features |
|---------|-------------------|-------------|---------------|
| **IMST iOKE868** | €119-129 (plus €7.50 USB power supply or €10 battery holder) | [shop.imst.de](https://shop.imst.de/wireless-solutions/lora-products/69/ioke868-lorawan-smart-metering-kit) · [iot-shop.de](https://iot-shop.de/shop/category/marke-imst-954) €126 | SML protocol, 868 MHz LoRaWAN, magnetic mount, USB or battery power, external antenna (magnetic base, 2m cable), configurable TX interval |
| **KLAX 2.0** | €149.90 | [iot-shop.de](https://iot-shop.de/shop/klax-2-0-lorawan-sml-opto-head-for-modern-power-meters-4365) | SML protocol, 868 MHz LoRaWAN, battery-powered (AA lithium, replaceable), IP21, compact (96×35×40mm), LoRaWAN Class A |
| **Fludia FM432ir** | €156 (direct) · €184.33 (iot-shop.de) | [fludia.com](https://shop.fludia.com/shop/en/isoUS/26-67-fm432ir-iot-sensor-for-german-electricity-meters-lorawan.html) · [iot-shop.de](https://iot-shop.de/shop/fl-fm432ir-fludia-fm432ir-lorawan-sml-optokopf-fur-moderne-stromzahler-6180) | SML protocol, 868 MHz LoRaWAN (also LTE-M/NB-IoT variants), battery-powered, 1-min or 15-min interval, 3.5-8 year battery life, made in France |

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
| Smart Plug | **Shelly Plus Plug S** | **€17.84-20.99** | WiFi (MQTT/REST), Bluetooth | Available at [reichelt.de](https://www.reichelt.de/shelly-plus-plug-s-schwarz-shelly-plusplugb-p353366.html); power measurement included, 2500W max, local API (no cloud required) |
| Smart Plug | Shelly Pro 4PM | €89-99 | WiFi (MQTT/REST), Ethernet | DIN-rail mount, 4 channels, power measurement per channel |
| PV Inverter | SolarEdge, SMA, Kostal | varies | Modbus TCP | Most modern inverters support Modbus; check manufacturer documentation |
| Battery | BYD, Tesla Powerwall, FoxESS | varies | Modbus, REST | Battery BMS integration depends on manufacturer API openness |

**Note:** Smart plugs must support **local control** (MQTT or REST API) without cloud dependency to comply with FR06 (offline capability). Shelly devices meet this requirement with their local HTTP/MQTT API. Avoid cloud-only smart plugs.

### 5.5 Local Controller Hardware (Phase 1+2)

| Product | Price (incl. VAT) | German Shop | Use Case |
|---------|-------------------|-------------|----------|
| **ESP32 DevKit C** | €12-18 | [reichelt.de](https://www.reichelt.de) · [amazon.de](https://www.amazon.de) | Tier 1 controller: WiFi + Bluetooth, 2 UARTs for Modbus RS485, runs ESPHome, low power |
| **MAX3485 RS485 Module** | €3-5 | [reichelt.de](https://www.reichelt.de) · [iot-shop.de](https://iot-shop.de) | Required for Tier 1 ESP32 Modbus RTU connection to wallbox/inverter |
| **USB RS485 Adapter** | €8-12 | [reichelt.de](https://www.reichelt.de) | Required for Tier 2 RPi Zero Modbus RTU connection |
| **Raspberry Pi Zero 2W** | €18-22 | [reichelt.de](https://www.reichelt.de) · [raspberrypi.com](https://www.raspberrypi.com) | Tier 2 controller: quad-core, WiFi, 512MB RAM, runs Python/Node-RED agent |
| **RPi Zero 2W Case + PSU + SD** | €15-20 | [reichelt.de](https://www.reichelt.de) | Accessories for Tier 2 controller setup |

| Tier | Total Hardware Cost (per household) |
|------|-------------------------------------|
| **Tier 1** (ESP32 + RS485 + PSU) | **~€20-25** |
| **Tier 2** (RPi Zero 2W kit + RS485) | **~€45-55** |
| **Tier 3** (Existing HA) | **~€0** |

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
| Sensor (per household) | IMST iOKE868 + USB PSU | €119 + €7.50 | 1 | **€126.50** |
| Local Controller (Tier 1) | ESP32 + RS485 + PSU | €20-25 | 1 | **€20-25** |
| Local Controller (Tier 2) | RPi Zero 2W kit + RS485 | €45-55 | 1 | **€45-55** |
| Smart Plug (Phase 2, optional) | Shelly Plus Plug S | €18-21 | 1 | **€18-21** |
| **Per Household — Tier 1 (Phase 1)** | | | | **~€147-152** |
| **Per Household — Tier 2 (Phase 1)** | | | | **~€172-182** |
| **Per Household — cumul. Tier 1 (Phase 1+2)** | | | | **~€166-173** |
| **Per Household — cumul. Tier 2 (Phase 1+2)** | | | | **~€191-203** |

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
| Sensor per household | €100-150 | €119 (iOKE868) | ✓ Within budget |
| Power supply (optional) | €0-20 | €7.50 (USB PSU) | ✓ Within budget |
| Local Controller (Tier 1) | €15-25 | €20-25 (ESP32) | ✓ Within budget |
| Local Controller (Tier 2) | €30-60 | €45-55 (RPi Zero 2W) | ✓ Within budget |
| **Per Household — Tier 1 (Phase 1)** | **≤ €200** | **~€147-152** | ✓ Within budget |
| **Per Household — Tier 2 (Phase 1)** | **≤ €200** | **~€172-182** | ✓ Within budget |
| Smart Plug (Phase 2, optional) | €0-50 | €18-21 | ✓ Within budget |
| **Per Household — Tier 1 (cumul. Phase 1+2)** | **≤ €350** | **~€166-173** | ✓ Within budget |
| **Per Household — Tier 2 (cumul. Phase 1+2)** | **≤ €350** | **~€191-203** | ✓ Within budget |

**Note:** Central total exceeds the €300 target primarily due to the Milesight UG67 gateway (€588). See Section 5.7 for cost reduction options.

### 6.2 Budget Tracking - Phase 2

| Category | Target | Actual | Notes |
|----------|--------|--------|-------|
| Control integration | €0-50 | €0 | Use existing devices where possible; EVCC/HAEO are free open source |
| Smart plugs (per device) | €20-40 | €18-21 | Shelly Plus Plug S from reichelt.de |
| Additional software | €0 | €0 | HAEO, EVCC, AkkudoktorEOS — all open source |
| **Phase 2 Add-on** | **≤ €150/household** | **€18-21** (or €0 if existing devices) | ✓ Within budget |

### 6.3 Cost Optimization Tips

- Use WiFi backhaul instead of cellular to reduce ongoing costs
- Choose sensors with USB power (avoid battery costs)
- Install gateway in central location to maximize range
- Bulk purchase sensors for volume discount
- Leverage existing devices (wallboxes, PV) already owned by households
- Use open-source software (EVCC, Home Assistant) to avoid licensing costs
- Choose the right local controller tier: Tier 1 (ESP32) for basic load shedding; Tier 2 (RPi) for full PV+battery+wallbox integration
- Deploy Tier 3 (existing HA) whenever a household already runs Home Assistant — zero additional hardware cost

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
| Local controller offline | Power or WiFi issue | Check USB power and WiFi connection at household; verify MQTT broker address |
| Local controller not publishing | Wrong topic prefix | Verify household ID in config matches `lem-netz/house/<id>/...` |
| No Modbus data from wallbox | RS485 wiring issue | Check A/B polarity and termination resistors on RS485 bus |

### 8.2 Common Issues - Phase 2

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| Wallbox not responding | OCPP connection failed | Check network, credentials; restart EVCC and wallbox |
| Wallbox frequent disconnects | Manufacturer firmware issue | Known problem with Wallbox Pulsar Plus firmware 6.7.x — try local Modbus instead of OCPP |
| Price data not updating | API rate limit | Check integration logs; EPEX API may throttle |
| Optimization causing loss | Wrong household config | Verify tariff type setting in HAEO/HA |
| §14a not working | No Smart Meter (iMSys) | Requires iMSys installation for Module 3 |
| Watchdog not firing | MQTT topic mismatch | Verify shed command topic matches local controller subscription |
| Watchdog Stage 1 not reducing load | Local controller not processing constraint | Subscribe to `lem-netz/house/<id>/status/compliance` to see if controller acknowledges constraint |
| Watchdog Stage 2 shed not executed | Local controller offline | Check heartbeat on `lem-netz/house/<id>/heartbeat`; if stale, check controller power/WiFi |
| False positive overload | Sensor drift | Check virtual transformer calculation — sum of all households |
| Shed acknowledgement missing | Local controller crashed | Restart controller; check watchdog failure detection in Section F.0.5 |
| Household exceeding constraint | Local optimizer malfunction | Verify local controller correctly sums device power and compares to `constraint/max_power`. Tier 1: check ESPHome lambda logic. Tier 2: check Python agent logs. |

### 8.3 Diagnostics

1. **Check sensor LED status** - Should indicate join and transmit
2. **Check gateway web interface** - Shows joined devices and uplink count
3. **Check MQTT topic** - Subscribe to `lem-netz/#` for all LEM messages
4. **Check local controller heartbeat** - Subscribe to `lem-netz/house/+/heartbeat` to verify all controllers are alive
5. **Check wallbox Modbus** - On local controller, run Modbus scanner to verify device responds
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

**Document Version:** 3.4
**Last Updated:** May 2026