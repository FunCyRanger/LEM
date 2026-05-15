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
│  │ (ESP32/ESPHome)  │    │ (ESP32/ESPHome)  │    │(ESP32/ESPHome)│  │
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

All households use the same controller: ESP32 + ESPHome. See Section 3.2.2 for hardware details.

**ESP32/ESPHome Setup:**
- Flash ESPHome firmware via USB (first-time) or OTA (updates)
- Connect MAX3485 RS485 transceiver: ESP32 GPIO16 (TX)→ RS485, GPIO17 (RX) ← RS485
- Wire RS485 A/B to wallbox or inverter Modbus terminals
- Write ESPHome YAML with Modbus client, MQTT topics per Section 3.2.1
- Configure WiFi SSID/password for household network
- Set MQTT broker address to central server IP
- Test: verify sensor data appears on `lem-netz/house/<id>/sensor/...`

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

The ESP32 communicates with the central server via a simple MQTT topic structure. Two message types exist: **sensor** data (house → central) and **control** commands (central → house).

**Topic patterns:**

```
lem-netz/house/<household_id>/sensor/<device>/<metric>       # Data from house to central
lem-netz/house/<household_id>/control/<type>                  # Limits/commands from central to house
lem-netz/house/<household_id>/status/<type>                   # Confirmations from house to central
```

**Example topics:**

| Topic | Direction | Payload | Description |
|-------|-----------|---------|-------------|
| `lem-netz/house/1/sensor/wallbox/power` | House → Central | `{"value": 4500, "unit": "W"}` | Wallbox current power consumption |
| `lem-netz/house/1/sensor/pv/production` | House → Central | `{"value": 3200, "unit": "W"}` | PV current production |
| `lem-netz/house/1/sensor/battery/soc` | House → Central | `{"value": 65, "unit": "%"}` | Battery state of charge |
| `lem-netz/house/1/control/max_power` | Central → House | `{"value": 5000, "unit": "W"}` | Hard limit on total household import power |
| `lem-netz/house/1/control/shed` | Central → House | `{"reason": "transformer_overload", "severity": "critical"}` | Shed non-essential loads (emergency fallback only) |
| `lem-netz/house/1/control/restore` | Central → House | `{"reason": "load_recovered"}` | Restore previously shed loads |
| `lem-netz/house/1/status/shed` | House → Central | `{"status": "executed", "loads_shed": ["wallbox", "smart_plug_1"]}` | Shed confirmation |

`control/max_power` replaces the previous `constraint/max_power`. The ESP32 enforces this hard limit locally by shedding or throttling devices. `control/shed` is the emergency fallback used only when limits alone cannot resolve an overload. No compliance reporting or heartbeat topics are needed — the central system verifies load by reading sensor data directly.

### 3.2.2 Local Controller (ESP32 + ESPHome)

Every household with controllable devices uses the same local controller: an **ESP32 running ESPHome**. There is one configuration approach for all households.

| Component | Purpose | Cost |
|-----------|---------|------|
| ESP32 DevKit C | WiFi + Bluetooth MCU, 2 UARTs for Modbus RS485 | €12-18 |
| MAX3485 RS485 Module | Converts ESP32 UART to RS485 for Modbus RTU | €3-5 |
| USB power supply | 5V power (can share outlet with iOKE868 sensor) | €5-10 |
| USB cable + enclosure | Mounting and connectivity | €3-5 |
| **Total** | | **€20-25** |

**Capabilities:**
- Runs ESPHome firmware — all configuration via YAML, no coding required
- Connects to central HA via MQTT over household WiFi
- Modbus RTU (RS485) for wallbox, PV inverter, battery BMS (±1-2 Modbus devices)
- WiFi for smart plugs (Shelly local HTTP API) or direct ESPHome switches
- Enforces hard limits: if total household power exceeds `control/max_power`, sheds loads in fixed priority order
- Executes emergency `control/shed` immediately when received
- ESPHome native encryption + MQTT TLS for security

**Setup:**
1. Flash ESPHome firmware via USB (first-time) or OTA (updates)
2. Connect MAX3485: ESP32 GPIO16 (TX) → RS485, GPIO17 (RX) ← RS485, wire A/B to wallbox/inverter Modbus terminals
3. Write ESPHome YAML with Modbus client and MQTT topics
4. Configure WiFi SSID/password for household network
5. Set MQTT broker address to central server IP
6. Test: `mosquitto_pub` and `mosquitto_sub` to verify MQTT link

### 3.2.3 Backhaul (Household to Central)

The ESP32 connects to the central MQTT broker over the household's existing WiFi network. No additional infrastructure is needed.

| Option | Description | Setup |
|--------|-------------|-------|
| **MQTT over WiFi (default)** | ESP32 connects to household WiFi, publishes to central MQTT broker | Configure WiFi SSID/password + broker IP in ESPHome YAML |
| **MQTT over Ethernet** | For households with Ethernet near the controller (more reliable) | Use ESP32 with Ethernet module (e.g., Olimex ESP32-POE) |

No internet connection is required — MQTT stays on the local network if the broker is reachable via LAN/WiFi. TLS is optional for local-only deployments.

### 3.2.4 Device Support on ESP32

| Device | Connection | ESPHome Configuration |
|--------|-----------|----------------------|
| Wallbox (Modbus RTU) | RS485 via MAX3485 | Modbus RTU client in YAML |
| Wallbox (Modbus TCP) | WiFi (if wallbox supports TCP) | Prefer RS485 for reliability |
| PV Inverter (Modbus) | RS485 via MAX3485 | Modbus RTU client (1-2 devices total per ESP32) |
| Battery BMS (Modbus RTU) | RS485 via MAX3485 | Modbus RTU client (if registers are documented) |
| Smart Plug (Shelly) | WiFi local HTTP API | `http_request` component to poll Shelly REST |
| Smart Plug (generic MQTT) | WiFi | ESPHome MQTT output component |

If a household has more than 2 Modbus devices, use a second ESP32 or one of the alternative controller options documented in the appendix (Section 12).

### 3.2.5 Control Philosophy

The system uses a **constraint-based control model**: the central system sets hard limits, the ESP32 enforces them locally. This provides household autonomy, privacy (central system never sees individual device states), and resilience (ESP32 continues enforcing the last known limit even if MQTT drops).

**Responsibility split:**

| Concern | Central System (HA) | Household (ESP32) |
|---------|--------------------|--------------------|
| Transformer safety | Monitors virtual transformer, calculates fair-share limits | Enforces `control/max_power` by shedding loads in priority order |
| Load optimization | Broadcasts price signals | Decides locally which loads to run within limits |
| Emergency shutdown | Issues `control/shed` as last resort | Executes shed immediately (all non-essential off) |

**Normal operation:**
```
Central HA                                ESP32
   │                                         │
   ├─ control/max_power = 5000W ───────────►│
   │                                         ├─ "I have 5kW budget"
   │                                         │  → Heat pump at 2kW
   │                                         │  → EV charges at 3kW
   │◄── sensor/wallbox/power = 3000W ────────┤
   │◄── sensor/plug/power = 2000W ───────────┤
```

**Emergency shed:**
```
Central HA                                ESP32
   │                                         │
   ├─ control/max_power = 3000W ───────────►│  Step 1: tighten limit
   │  (30s — still overloaded)               │
   ├─ control/shed ────────────────────────►│  Step 2: emergency shed
   │◄── status/shed = {executed} ───────────┤
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

### Phase E: Control Integration via Local Controller

> **Architecture:** All device integration (wallbox, PV, battery, smart plugs) happens on the **per-household ESP32**, not on the central server. The ESP32 publishes sensor data to central MQTT and subscribes to control commands. See Section 3.2 for the full topic hierarchy.

#### E.1 Integration Architecture

```
┌─ HOUSEHOLD ─────────────────────────────────────────────────────┐
│                                                                  │
│  Wallbox ───Modbus RTU──┐                                        │
│  PV Inverter ─Modbus RTU─┤──→ ESP32 (ESPHome) ──MQTT──→ Central │
│  Battery ─────Modbus RTU─┤        (via WiFi)      (TLS)    HA   │
│  Smart Plug ────WiFi────┘                                        │
│                                                                  │
│  ESP32 handles:                                                  │
│  • Protocol translation (Modbus RTU, Shelly HTTP)                │
│  • Data normalization to MQTT topic schema                       │
│  • Hard limit enforcement: stay within control/max_power         │
│  • Emergency shed execution on control/shed                      │
│  • Data buffering during network outages (last values)           │
└──────────────────────────────────────────────────────────────────┘
```

#### E.1.1 Control Topic Handling

> **Primary mode:** The central system sends hard limits via `control/max_power`. The ESP32 enforces these locally by shedding loads in priority order. `control/shed` is the emergency fallback.

| Topic | Purpose | ESP32 Action |
|-------|---------|-------------|
| `control/max_power` | Hard limit on total household import | Sum all device power; if exceeding limit, shed lowest-priority device |
| `control/shed` | Emergency shutdown | Turn off all non-essential loads immediately |

No compliance reporting or heartbeat — the central system monitors actual load via sensor data.

#### E.2 Wallbox/EV Charger Integration (on ESP32)

1. **Supported Protocol**

   | Protocol | ESP32 Support | Notes |
   |----------|--------------|-------|
   | Modbus RTU (RS485) | ✓ via MAX3485 | Preferred — simple, reliable |
   | Modbus TCP | — | Not recommended on ESP32; use RS485 instead |
   | OCPP 1.6/2.0 | — | Not available on ESP32. Use EVCC on RPi (see appendix) |
   | Manufacturer REST | — | Not available on ESP32 |

   **Recommendation:** Connect wallbox via Modbus RTU (RS485). If your wallbox only supports OCPP, see the RPi-based alternative in Section 12.

2. **Data Published to Central MQTT**

   | Topic | Payload | Interval |
   |-------|---------|----------|
   | `lem-netz/house/<id>/sensor/wallbox/power` | `{"value": 4500, "unit": "W"}` | Every 15s or on change |
   | `lem-netz/house/<id>/sensor/wallbox/energy` | `{"value": 12.5, "unit": "kWh"}` | Every 5 min |
   | `lem-netz/house/<id>/sensor/wallbox/status` | `{"state": "charging"}` | On change |
   | `lem-netz/house/<id>/status/shed` | `{"status": "executed", "loads_shed": ["wallbox"]}` | On watchdog command |

3. **Control Commands from Central MQTT**

   | Topic | Payload | ESP32 Action |
   |-------|---------|-------------|
   | `control/max_power` | `{"value": 5000, "unit": "W"}` | Reduce wallbox current so total household stays below limit |
   | `control/shed` | `{"reason": "transformer_overload"}` | Immediately stop wallbox charging |

4. **OCPP Compatibility Note**
   - If your wallbox only supports OCPP (e.g., Wallbox Pulsar Plus), you will need the RPi-based alternative controller (see Section 12) since OCPP is not supported on ESP32
   - Prefer wallboxes with Modbus RTU for compatibility with this architecture

#### E.3 Smart Plug/Switch Integration (on ESP32)

Smart plugs connect via WiFi and are polled by the ESP32:

1. **Supported Devices**

   | Device | Connection | ESPHome Configuration |
   |--------|-----------|----------------------|
   | Shelly Plus Plug S | WiFi local HTTP API | `http_request` component to poll REST API |
   | Shelly Pro 4PM | WiFi local HTTP API | `http_request` component (4 channels) |
   | Generic MQTT plug | WiFi | ESPHome MQTT output component |

2. **Data Flow**
   - **Sensor data**: ESP32 polls Shelly at `http://<shelly-ip>/rpc/Switch.GetStatus` → publishes to `lem-netz/house/<id>/sensor/plug/<name>/power`
   - **Limit**: ESP32 sums all plug power; if approaching `control/max_power`, turns off lowest-priority plug
   - **Emergency**: ESP32 receives `control/shed` → sends HTTP POST to Shelly to turn off

3. **Configuration**
   - Assign static IP to each smart plug via DHCP reservation
   - Assign each plug a shed priority (1 = shed first) in ESPHome YAML

#### E.4 PV System Integration (on ESP32)

1. **Supported Inverters**

   | Inverter | ESP32 Connection | Notes |
   |----------|-----------------|-------|
   | Kostal (Modbus RTU) | RS485 via MAX3485 | Works with Modbus RTU |
   | Kostal (Modbus TCP) | — | Not supported on ESP32 |
   | SolarEdge | — | Use HA SolarEdge integration for monitoring on central server |
   | SMA | — | Use HA SMA integration for monitoring on central server |
   | Victron | — | Use HA Victron integration for monitoring on central server |

   The ESP32 can read a PV inverter with Modbus RTU output. For monitoring-only (no control), publish inverter data directly to HA via the built-in integration and skip the ESP32 connection.

2. **Data Published**

   | Topic | Payload |
   |-------|---------|
   | `lem-netz/house/<id>/sensor/pv/production` | `{"value": 3200, "unit": "W"}` |
   | `lem-netz/house/<id>/sensor/pv/daily_yield` | `{"value": 14.2, "unit": "kWh"}` |

3. **Control**
   - The ESP32 only reads PV data; it does not curtail PV production (this requires inverter-specific Modbus registers beyond ESP32 scope)
   - If PV curtailment is needed, implement it in HA automations using the inverter's native integration

#### E.5 Battery Storage Integration (on ESP32)

1. **Supported Systems**

   | System | ESP32 Connection | Notes |
   |--------|-----------------|-------|
   | Generic Modbus BMS | RS485 via MAX3485 | Works if Modbus registers are documented and simple (SoC, power) |
   | BYD (Modbus) | — | Use HA Modbus integration on central server instead |
   | Tesla Powerwall | — | Use HA Powerwall integration on central server |
   | FoxESS (Modbus) | — | Use HA Modbus integration on central server |

   For most battery systems, use the HA integration directly on the central server rather than routing through ESP32. The ESP32 connection is only needed for batteries with simple Modbus RTU interfaces.

2. **Data Published**

   | Topic | Payload |
   |-------|---------|
   | `lem-netz/house/<id>/sensor/battery/soc` | `{"value": 65, "unit": "%"}` |
   | `lem-netz/house/<id>/sensor/battery/power` | `{"value": -1500, "unit": "W"}` (negative = charging) |

3. **Control**
   - The ESP32 reads battery state for load management decisions
   - If battery control is needed (charge/discharge scheduling), it is handled by HA automations using the battery's native HA integration

### Phase F: Revenue-Aware Optimization

#### F.0 INFRASTRUCTURE SAFETY WATCHDOG (CRITICAL - IMPLEMENT FIRST)

> **IMPLEMENTATION ORDER:** Infrastructure safety MUST be implemented BEFORE economic optimization. The watchdog is the enforcement mechanism.

The watchdog operates in **two stages**: first by tightening `control/max_power` limits (allowing ESP32s to self-regulate), then by direct shed commands if the limit tightening does not resolve the overload.

```
Central HA detects overload (80%)
  │
  ├─ Stage 1: control/max_power = reduced_limit ──► ESP32s shed loads autonomously
  │     Wait 30s
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
| 1.1 | Detect threshold | `sensor.virtual_transformer` exceeds 80% (8000W for 10kVA) |
| 1.2 | Calculate fair share | Reduce each household's `max_power` proportionally |
| 1.3 | Publish limit | `lem-netz/house/<id>/control/max_power` = new limit per household |
| 1.4 | Wait | 30-second window for ESP32s to self-regulate |
| 1.5 | Check | If total < 80% → resolved. If still > 80% → proceed to Stage 2 |

**Stage 2 — Direct Shed (escalation, only if Stage 1 fails):**

| Step | Action | Detail |
|------|--------|--------|
| 2.1 | Send shed | `lem-netz/house/<id>/control/shed` to ALL households |
| 2.2 | Acknowledge | ESP32s confirm via `status/shed` |
| 2.3 | Verify | If still > 80%, alert and escalate manually |
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
# Stage 1: Tighten limits when transformer exceeds 80%
automation:
  - alias: "Transformer Stage 1 — Limit Tightening"
    trigger:
      - platform: numeric_state
        entity_id: sensor.virtual_transformer
        above: 8000
    mode: single
    action:
      - variables:
          overload_factor: "{{ 8000 / states('sensor.virtual_transformer') | float }}"
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
      - delay: "00:00:30"
      - if:
          - condition: numeric_state
            entity_id: sensor.virtual_transformer
            above: 8000
        then:
          - service: mqtt.publish
            data:
              topic: "lem-netz/watchdog/escalation"
              payload: '{"stage": 2}'
              qos: 2
```

```yaml
# Stage 2: Direct shed (triggered by escalation or 95% threshold)
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
        below: 6000
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

#### F.0.4 ESP32 Handler (ESPHome YAML)

```yaml
# ESPHome: Handle control/max_power and control/shed
api:
  on_mqtt_message:
    - topic: lem-netz/house/1/control/max_power
      then:
        - lambda: |-
            int new_limit = parse_json_value(x, "value");
            id(current_max_power) = new_limit;
            if (id(total_power) > new_limit) {
              id(smart_plug_1).turn_off();
            }

    - topic: lem-netz/house/1/control/shed
      then:
        - lambda: |-
            id(smart_plug_1).turn_off();
            id(wallbox_modbus).write_register(1000, 0);
        - mqtt.publish:
            topic: lem-netz/house/1/status/shed
            payload: '{"status": "executed"}'
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
| **ESP32 DevKit C** | €12-18 | [reichelt.de](https://www.reichelt.de) · [amazon.de](https://www.amazon.de) | WiFi MCU, 2 UARTs for Modbus RS485, runs ESPHome |
| **MAX3485 RS485 Module** | €3-5 | [reichelt.de](https://www.reichelt.de) · [iot-shop.de](https://iot-shop.de) | Required for Modbus RTU to wallbox/inverter |
| **USB power supply** | €5-10 | — | 5V USB power (can share outlet with iOKE868) |

| Component | Cost |
|-----------|------|
| **ESP32 + RS485 + PSU** | **~€20-25** |

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
| Sensor (per household) | IMST iOKE868 + USB PSU | €119 + €7.50 | 1 | **€126.50** |
| Local Controller | ESP32 + RS485 + PSU | €20-25 | 1 | **€20-25** |
| Smart Plug (Phase 2, optional) | Shelly Plus Plug S | €18-21 | 1 | **€18-21** |
| **Per Household (Phase 1)** | | | | **~€147-152** |
| **Per Household (Phase 1+2)** | | | | **~€166-173** |

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
| Local Controller | €15-25 | €20-25 (ESP32) | ✓ Within budget |
| **Per Household (Phase 1)** | **≤ €200** | **~€147-152** | ✓ Within budget |
| Smart Plug (Phase 2, optional) | €0-50 | €18-21 | ✓ Within budget |
| **Per Household (Phase 1+2)** | **≤ €350** | **~€166-173** | ✓ Within budget |

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
- Use open-source software (ESPHome, Home Assistant) to avoid licensing costs
- If a household already runs Home Assistant, consider integrating directly via Remote HA add-on instead of adding an ESP32

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
| Wallbox not responding | Modbus connection failed | Check RS485 A/B polarity; verify termination resistors; check USB power on ESP32 |
| Wallbox only supports OCPP | No OCPP support on ESP32 | Use RPi-based alternative controller (Section 12) |
| Price data not updating | API rate limit | Check integration logs; EPEX API may throttle |
| Optimization causing loss | Wrong household config | Verify tariff type setting in HA |
| §14a not working | No Smart Meter (iMSys) | Requires iMSys installation for Module 3 |
| Watchdog not firing | MQTT topic mismatch | Verify `control/max_power` topic matches ESP32 subscription |
| Watchdog Stage 1 not reducing load | ESP32 not receiving limit | Subscribe to `lem-netz/house/<id>/control/#` to verify messages arrive |
| Watchdog Stage 2 shed not executed | ESP32 offline | Check WiFi connection; verify sensor data still arriving via LoRaWAN |
| False positive overload | Sensor drift | Check virtual transformer calculation — sum of all households |
| Shed not acknowledged | ESP32 crashed | Restart ESP32; check ESPHome logs |

### 8.3 Diagnostics

1. **Check sensor LED status** - Should indicate join and transmit
2. **Check gateway web interface** - Shows joined devices and uplink count
3. **Check MQTT topics** - Subscribe to `lem-netz/#` for all LEM messages
4. **Check ESPHome logs** - Connect to ESP32 via ESPHome dashboard or serial
5. **Check wallbox Modbus** - On ESP32, verify Modbus register reads in ESPHome logs
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

If a household has devices that only support **OCPP** (e.g., Wallbox Pulsar Plus) or **Modbus TCP** (no RS485 port), the ESP32 cannot interface with them. Use a Raspberry Pi Zero 2W as the local controller instead.

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
3. Configure `config.yaml` with household ID, MQTT broker URL, device list
4. Set up systemd service for auto-start
5. Use EVCC as OCPP client for wallbox control if needed

The RPi uses the same MQTT topics as the ESP32 (`lem-netz/house/<id>/control/*`, `sensor/*`, `status/*`). All watchdog, price integration, and optimization logic on the central server remains identical — only the local controller hardware changes.

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