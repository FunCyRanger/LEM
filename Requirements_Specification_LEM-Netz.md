# Requirements Specification: LEM-Netz

## Decentralized Energy Management System for Neighborhood Communities

**Version:** 3.0
**Status:** Requirements Draft
**Scope:** Phase 1 + Phase 2

---

## Table of Contents

- [1. Problem Statement](#1-problem-statement)
- [2. Scope](#2-scope)
  - [2.1 Phase 1: Data Collection](#21-phase-1-data-collection-current)
  - [2.2 Phase 2: Control Integration](#22-phase-2-control-integration-planning)
- [3. Functional Requirements](#3-functional-requirements)
  - [3.1 Data Acquisition](#31-data-acquisition-phase-1)
  - [3.2 Data Transmission](#32-data-transmission-phase-1)
  - [3.3 Data Processing](#33-data-processing-phase-1)
  - [3.4 System Management](#34-system-management-phase-1)
  - [3.5 Infrastructure Safety](#35-infrastructure-safety-constraints-phase-2---top-priority)
  - [3.6 Pricing Model Support](#36-pricing-model-support-phase-2)
  - [3.7 Control Capabilities](#37-control-capabilities-phase-2)
- [4. Non-Functional Requirements](#4-non-functional-requirements)
  - [4.1 Cost](#41-cost-requirements)
  - [4.2 Technical](#42-technical-requirements)
  - [4.3 Usability](#43-usability-requirements)
  - [4.5 Priority Hierarchy](#45-priority-hierarchy-critical)
  - [4.6 Economic Fairness](#46-economic-fairness-requirements)
- [5. Regulatory Requirements - Germany](#5-regulatory-requirements---germany)
  - [5.1 Smart Meter Interface](#51-smart-meter-interface)
  - [5.2 Grid Connection](#52-grid-connection)
  - [5.3 Energy Market](#53-energy-market-regulations)
  - [5.4 Data Protection](#54-data-protection)
  - [5.5 Radio Equipment](#55-radio-equipment)
- [6. German Market Pricing Models](#6-german-market-pricing-models)
- [7. General Architecture](#7-general-architecture)
- [8. Cost Targets](#8-cost-targets)
- [9. Future Phases (Roadmap)](#9-future-phases-roadmap)
- [10. Glossary](#10-glossary)
- [11. Revision History](#11-revision-history)

---

## 1. Problem Statement

### 1.1 Background

The energy transition (Energiewende) in Germany requires active participation from local communities. However, most households lack visibility into their neighborhood's total power consumption, making it difficult to coordinate local energy optimization.

### 1.2 Problem

Without localized data, community energy projects cannot:
- Optimize local energy consumption
- Coordinate charging of electric vehicles with available power
- Integrate renewable energy effectively
- Reduce grid strain at neighborhood level

Additionally, different households have different revenue models:
- Fixed feed-in tariffs (EEG)
- Dynamic/volldynamisch pricing (Tibber, aWATTar, etc.)
- Self-consumption focus
- Battery arbitrage opportunities
- §14a EnWG variable network charges

A community system must NOT disadvantage any household due to their individual pricing model.

### 1.3 Goal

Enable neighborhoods (Quartiere) to collect, aggregate, and utilize their own energy data locally—without dependency on external cloud services or utility companies. Phase 2 adds intelligent control that respects each household's unique economic situation.

### 1.4 Why This Matters

- **Data Sovereignty:** All data stays in the community
- **Local Optimization:** Enables coordination without internet
- **Community Control:** Residents own their data
- **Economic Fairness:** Optimization must not create financial losses for any household type
- **Future Foundation:** Phase 1 creates infrastructure for Phase 2+ (control, optimization)

---

## 2. Scope

### 2.1 Phase 1: Data Collection (Current)

This requirements specification covers **Phase 1**: collection of energy data from smart meters in multiple households and transmission to a central server.

**Included in Phase 1:**
- Measuring electrical power (W) per household
- Measuring cumulative energy consumption (kWh) per household
- Automatic wireless data transmission to central server
- Local data aggregation ("virtual transformer" total)
- Local data storage without cloud dependency

**Excluded from Phase 1:**
- Control of wallboxes, batteries, or other devices
- Pricing model integration
- Revenue-aware optimization

### 2.2 Phase 2: Control Integration (Planning)

Phase 2 adds intelligent control capabilities while ensuring economic fairness across all household types.

**Included in Phase 2:**
- Integration with wallbox control systems (EV charging)
- Integration with smart plugs/switches for load control
- Integration with PV inverters for generation monitoring
- Integration with battery storage systems
- Dynamic electricity price support (EPEX Spot market)
- §14a EnWG network charge optimization
- Revenue-aware optimization logic

### 2.3 Geographic Scope

| Parameter | Target |
|-----------|--------|
| Number of households | 100 - 1,000 |
| Maximum distance between households | 1,000+ meters |
| Location | Germany (outdoor installation required) |

---

## 3. Functional Requirements

### 3.1 Data Acquisition (Phase 1)

**FR01: Power Measurement**
- The system SHALL measure the current active power (in watts) at each household's electricity meter.
- The measurement SHALL be taken from the meter using the standardized SML (Smart Message Language) protocol or IEC 62056-21 interface.
- *Rationale:* Required for real-time load monitoring and future load management.

**FR02: Energy Measurement**
- The system SHALL measure cumulative energy consumption (in kilowatt-hours) at each household.
- The measurement SHALL be taken automatically at configurable intervals.
- *Rationale:* Required for billing reconciliation and consumption analysis.

**FR03: Sampling Rate**
- The system SHALL support transmission intervals of 15 seconds or less when powered via USB.
- The system SHALL support configurable transmission intervals (e.g., 1 minute, 5 minutes, 15 minutes).
- *Rationale:* Fast intervals needed for dynamic load management in future phases.

### 3.2 Data Transmission (Phase 1)

**FR04: Wireless Communication**
- The system SHALL transmit data wirelessly from each household to a central gateway.
- The wireless technology SHALL operate in license-free frequency bands (EU 868 MHz).
- The wireless range SHALL support at least 1,000 meters in open terrain.
- *Rationale:* Houses are spread over 1000m+; wireless eliminates need for building wiring.

**FR05: Encryption**
- All wireless communication SHALL be encrypted end-to-end (AES-128 or equivalent).
- *Rationale:* Required for data privacy and security.

**FR06: Offline Capability**
- The system SHALL continue to collect and buffer data if the central server is temporarily unreachable.
- *Rationale:* Ensures data continuity during network outages.

### 3.3 Data Processing (Phase 1)

**FR07: Central Aggregation**
- The system SHALL aggregate power readings from all households into a single "virtual transformer" value representing total neighborhood load.
- The aggregation SHALL update in real-time as new data arrives.
- *Rationale:* Core functionality for load monitoring and future coordination.

**FR08: Local Data Storage**
- All data SHALL be stored locally on the community's own server.
- No data SHALL be transmitted to external cloud services without explicit community consent.
- *Rationale:* Data sovereignty requirement.

**FR09: Standard Protocols**
- Data SHALL be accessible via standard protocols (MQTT and JSON).
- This enables integration with various home automation platforms.
- *Rationale:* Prevents vendor lock-in, enables flexibility.

### 3.4 System Management (Phase 1)

**FR10: Easy Installation**
- Each household sensor SHALL be installable without specialized tools or technical expertise.
- Installation SHALL be possible on standard German smart meters (moderne Messeinrichtungen, mME).
- *Rationale:* Community project must be maintainable by non-experts.

**FR11: Device Management**
- New sensors SHALL be registrable via the gateway's web interface.
- Failed sensors SHALL be identifiable and replaceable without system disruption.
- *Rationale:* Scalability and maintainability.

### 3.5 Infrastructure Safety Constraints (Phase 2) - TOP PRIORITY

> **PRIORITY HIERARCHY:** Infrastructure safety is the highest-priority constraint. The system MUST never allow optimization strategies that compromise grid stability. Economic optimization is SUBORDINATE to infrastructure safety.

**FR11a: Transformer Load Limiting**
- The system SHALL continuously monitor the virtual transformer total (sum of all household power).
- The system SHALL enforce a configurable maximum threshold (e.g., 80% of transformer rating) to prevent overload.
- When approaching the threshold, the system SHALL automatically shed or reduce load across households.
- *Rationale:* CRITICAL - Grid stability is non-negotiable. A transformer overload would damage infrastructure and blackout the entire community.

**FR11b: Coordinated Load Management**
- The system SHALL coordinate all controllable loads (wallboxes, batteries, smart plugs) to respect infrastructure limits.
- Load coordination SHALL happen automatically and in real-time (< 30 second response).
- Individual household optimization SHALL be suspended if infrastructure limits are at risk.
- *Rationale:* Without coordination, simultaneous charging/loading could exceed transformer capacity.

**FR11c: Graceful Degradation**
- If infrastructure limits are reached, economic optimization for all households SHALL be suspended.
- The system SHALL prioritize only essential load reduction.
- Households SHALL be notified when optimization is limited due to infrastructure constraints.
- *Rationale:* Economic fairness is secondary to grid stability.

### 3.6 Pricing Model Support (Phase 2)

**FR12: Household Configuration**
- The system SHALL allow configuration of each household's revenue/pricing model.
- The system SHALL store: tariff type, feed-in rate, import rate, dynamic pricing provider.
- *Rationale:* Required for revenue-aware optimization.

**FR13: Revenue-Neutral Optimization (SUBORDINATE TO INFRASTRUCTURE SAFETY)**
- The optimization logic SHALL NOT apply strategies that could result in financial loss to any household.
- For households with fixed feed-in tariffs (EEG), optimization SHALL prioritize maximizing self-consumption.
- For households with dynamic pricing, optimization SHALL leverage time-variable prices.
- For households with §14a network charges, optimization SHALL minimize network fees during peak periods.
- **THIS REQUIREMENT IS SUBORDINATE TO FR11a-FR11c:** Economic optimization SHALL be suspended when infrastructure safety requires it.
- *Rationale:* Economic fairness is important BUT secondary to grid stability. See FR11a-FR11c priority hierarchy.

**FR14: Dynamic Price Integration**
- The system SHALL import real-time electricity prices from EPEX Spot Day-Ahead market.
- The system SHALL support integration with multiple dynamic pricing providers (Tibber, aWATTar, Ostrom, etc.).
- Price data SHALL be updated at least hourly (preferably every 15 minutes).
- *Rationale:* Required for time-based optimization.

**FR15: §14a EnWG Network Charge Optimization**
- The system SHALL support the three §14a EnWG modules:
  - Module 1: Flat reduction (pauschale Reduzierung)
  - Module 2: Percentage reduction (prozentuale Reduzierung)
  - Module 3: Time-variable network charges (zeitvariable Netzentgelte)
- The system SHALL optimize consumption during low-tariff periods when Module 3 is active.
- *Rationale:* Network charges are ~20-25% of electricity cost; optimization saves money.

### 3.7 Control Capabilities (Phase 2)

**FR16: Wallbox/EV Charger Control**
- The system SHALL integrate with electric vehicle wallboxes via standard protocols (OCPP, Modbus TCP, or manufacturer APIs).
- The system SHALL control charging start/stop and charging power.
- The system SHALL coordinate charging across multiple households to avoid transformer overload.
- *Rationale:* EV charging is major load; coordination prevents grid issues.

**FR17: Smart Plug/Switch Control**
- The system SHALL integrate with smart plugs or switches for controllable loads.
- Supported protocols SHALL include at least: MQTT, HTTP/REST, or manufacturer APIs.
- *Rationale:* Enables load shifting for non-critical appliances.

**FR18: PV System Monitoring**
- The system SHALL monitor PV generation data from inverters.
- Supported protocols SHALL include at least: Modbus TCP, HTTP/REST, or manufacturer APIs.
- *Rationale:* Required for self-consumption optimization.

**FR19: Battery Storage Integration**
- The system SHALL monitor battery state of charge (SoC), charge/discharge power.
- The system SHALL control battery charging/discharging when integrated.
- Supported protocols SHALL include at least: Modbus TCP, or manufacturer APIs.
- *Rationale:* Battery arbitrage and self-consumption optimization.

**FR20: Load Coordination (INFRASTRUCTURE SAFETY)**
- The system SHALL coordinate loads across all households to prevent exceeding transformer capacity.
- The coordination SHALL be automatic and respect individual household preferences.
- **THIS IS A CORE INFRASTRUCTURE SAFETY REQUIREMENT** - takes priority over all economic optimization.
- *Rationale:* Prevents grid overload; enables maximum utilization of local generation. This is the PRIMARY mechanism for FR11a-FR11c compliance.

---

## 4. Non-Functional Requirements

### 4.1 Cost Requirements

| Category | Target Budget | Notes |
|----------|--------------|-------|
| Central infrastructure (server + gateway) | ≤ €300 | One-time cost |
| Per-household hardware (Phase 1) | ≤ €200 | Per household |
| Per-household hardware (Phase 2) | ≤ €350 | Including control capabilities |
| Ongoing costs | €0 | No subscriptions |

*Note: Budget targets may require trade-offs in features or component selection.*

### 4.2 Technical Requirements

| Requirement | Target |
|-------------|--------|
| Household capacity | 100 - 1,000 |
| Wireless range | ≥ 1,000 m (open terrain) |
| Data availability | 99.9% (local operation) |
| Power consumption (sensor) | < 1W |
| Control response time | < 30 seconds |
| Price update frequency | ≥ hourly (15 min preferred) |

### 4.3 Usability Requirements

| Requirement | Target |
|-------------|--------|
| Installation time per household | ≤ 30 minutes |
| Training required | None for residents |
| Maintenance | Minimal (no moving parts) |
| Configuration interface | Web-based, mobile-friendly |

### 4.4 Operational Requirements

| Requirement | Target |
|-------------|--------|
| Local operation without internet | Required (core) |
| Data backup | Local storage only |
| Uptime | 24/7 operation |
| Power outage resilience | Buffer data during outages |

### 4.5 Priority Hierarchy (CRITICAL)

> **PRIORITY: Infrastructure Safety > Economic Fairness**

The system MUST enforce the following priority hierarchy:

1. **TOP PRIORITY: Infrastructure Safety**
   - Never exceed transformer capacity limits
   - Automatically shed loads if approaching limits
   - Grid stability is non-negotiable

2. **SECOND PRIORITY: Economic Fairness**
   - Do not disadvantage any household economically
   - Optimize within infrastructure constraints
   - Transparent economic impact visibility

If infrastructure safety and economic fairness conflict, **infrastructure safety always wins**. Economic optimization is suspended when grid stability requires it.

### 4.6 Economic Fairness Requirements

| Requirement | Target |
|-------------|--------|
| Revenue impact visibility | Each household can see their economic impact |
| Optimization transparency | Clear explanation of optimization decisions |
| No financial disadvantage | System must never increase costs for any participant |
| Individual opt-out | Households can disable optimization for their devices |

---

## 5. Regulatory Requirements - Germany

### 5.1 Smart Meter Interface

**BSI TR-03109-1** (Technische Richtlinie)
- Specifies the interface for reading data from Smart Meter Gateways (SMG)
- For Phase 1 (data collection), the optical IR interface on modern meters (mME) is sufficient
- Phase 2 with §14a requires Smart Meter (iMSys) for quarter-hourly data
- *Status:* Binding for full functionality

### 5.2 Grid Connection

**VDE-AR-N 4100** (Anschlussregelung Niederspannung)
- Defines requirements for connecting controllable loads to the low-voltage grid
- Required for wallbox control in Phase 2
- Ensures community does not exceed grid capacity limits
- *Status:* Binding for Phase 2

### 5.3 Energy Market Regulations

**§14a EnWG** (Energiewirtschaftsgesetz)
- Since January 2025, variable network charges are mandatory
- Three modules available: flat %, percentage, time-variable
- Requires Smart Meter (iMSys) for Module 3
- *Status:* Binding for network charge optimization

**EEG (Erneuerbare-Energien-Gesetz)**
- Feed-in tariffs for PV: 5.5-7.8 ct/kWh (varies by size/date, 20-year guarantee)
- Self-consumption remains most economically efficient
- Direct marketing optional for larger systems
- *Status:* Reference for optimization logic

### 5.4 Data Protection

**GDPR (DSGVO)**
- All personal energy data belongs to the household owner
- Data processing must be transparent
- Data must be stored locally within Germany (no US/cloud servers)
- Community must have clear data handling policies
- Optimization data must not be shared without consent
- *Status:* Binding requirement

### 5.5 Radio Equipment

**RED (Radio Equipment Directive) 2014/53/EU**
- Wireless components must be CE-certified for EU use
- 868 MHz band is license-free for low-power devices in Germany
- *Status:* Binding requirement - only use CE-certified components

---

## 6. German Market Pricing Models

### 6.1 Overview of Household Types

The system MUST support the following German household configurations:

| # | Household Type | Energy Profile | Revenue/Cost Model |
|---|----------------|----------------|-------------------|
| 1 | No PV, fixed tariff | Import only | Static fixed price (~30-35 ct/kWh) |
| 2 | PV only (EEG) | Import + Export | Fixed feed-in tariff (5.5-8 ct/kWh) + self-use value |
| 3 | PV only (Dynamic) | Import + Export | EPEX Spot prices for both |
| 4 | PV + Battery | Import + Export + Storage | Dynamic + arbitrage |
| 5 | Battery only | Import + Storage | Charge when cheap, discharge when expensive |
| 6 | Heat pump | High consumption | §14a network charges apply |
| 7 | EV + Wallbox | High consumption | Dynamic charging optimization |
| 8 | Multi-generation | PV + Battery + EV | Full optimization potential |

### 6.2 Economic Optimization Goals

| Household Type | Primary Optimization Goal | Secondary Goal |
|----------------|---------------------------|----------------|
| No PV (fixed tariff) | Minimize consumption during peak price periods | - |
| PV only (EEG) | Maximize self-consumption | Minimize grid import |
| PV only (Dynamic) | Maximize export value during high-price periods | Self-consumption |
| PV + Battery | Arbitrage: charge battery low, discharge high | Self-consumption |
| Battery only | Charge during negative/low prices, discharge during peaks | Grid service |
| Heat pump | Shift consumption to §14a low-tariff periods | Minimize network charges |
| EV + Wallbox | Charge during low-price periods | Coordinate with household load |

### 6.3 Critical: Revenue-Neutral Requirement

> **REQUIREMENT FR13 (REITERATED):** The optimization logic MUST be configurable per household to match their specific revenue model. The system MUST NOT apply universal strategies that could disadvantage any household economically. Each household MUST be able to view the financial impact of optimization decisions.

> **IMPORTANT:** This requirement is **SUBORDINATE** to infrastructure safety requirements (FR11a-FR11c, FR20). Economic optimization is suspended when the virtual transformer approaches or exceeds capacity limits. Grid stability ALWAYS takes priority over economic gains.

---

## 7. General Architecture

### 7.1 Conceptual Overview - Phase 1

All communication between households and the central server uses LoRaWAN 868 MHz. MQTT is used only on the local LAN between the outdoor gateway and the server.

```
┌────────────────────────────────────────────────────────────────────┐
│                   LEM-NETZ SYSTEM (Phase 1)                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  HOUSEHOLD DOMAIN (×N)                                       │ │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌────────────┐ │ │
│  │  │   Household 1    │  │   Household 2    │  │  House N   │ │ │
│  │  │  ┌────────────┐  │  │  ┌────────────┐  │  │ ┌────────┐ │ │ │
│  │  │  │  Bridge    │  │  │  │  Bridge    │  │  │ │ Bridge │ │ │ │
│  │  │  │  Device    │  │  │  │  Device    │  │  │ │ Device │ │ │ │
│  │  │  │  (LoRaWAN) │  │  │  │  (LoRaWAN) │  │  │ │(LoRaWN)│ │ │ │
│  │  │  │  + IR      │  │  │  │  + IR      │  │  │ │ + IR   │ │ │ │
│  │  │  └─────┬──────┘  │  │  └──────┬─────┘  │  │ └───┬────┘ │ │ │
│  │  └────────┼─────────┘  └─────────┼─────────┘  └─────┼──────┘ │ │
│  │           │                       │                    │       │ │
│  │           └───────────────────────┴────────────────────┘       │ │
│  │                              │ LoRaWAN 868 MHz                  │ │
│  └──────────────────────────────┼──────────────────────────────────┘ │
│                                 ▼                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  OUTDOOR GATEWAY (IP67, Milesight UG67)                      │  │
│  │  LoRaWAN ↔ MQTT translator — receives uplinks, forwards     │  │
│  │  downlinks — connected via Ethernet to central server        │  │
│  └──────────────────────────────┬───────────────────────────────┘  │
│                                 │ Ethernet (MQTT)                    │
│                                 ▼                                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  NEIGHBORHOOD NETWORK                                        │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │  │
│  │  │ MQTT Broker  │  │  Database    │  │   Dashboard      │   │  │
│  │  │ (Mosquitto)  │  │ (InfluxDB)   │  │     (HA)         │   │  │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 7.2 Extended Architecture - Phase 2

Phase 2 adds Modbus control to the same bridge device. The bridge reads the smart meter (IR), controls home devices (Modbus RTU), and communicates with the central server via LoRaWAN — no WiFi needed.

```
┌────────────────────────────────────────────────────────────────────┐
│                    LEM-NETZ SYSTEM - PHASE 2                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  NEIGHBORHOOD NETWORK (Local Server)                        │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │  │
│  │  │MQTT Broker  │  │  Database   │  │  Dashboard       │    │  │
│  │  │ (Mosquitto) │  │ (InfluxDB)  │  │  (Home Assistant)│    │  │
│  │  └──────┬──────┘  └──────┬──────┘  └────────┬─────────┘    │  │
│  │         │                │                    │              │  │
│  │  ┌──────┴──────┐  ┌──────┴──────┐                            │  │
│  │  │Price Import │  │ Optimization│                            │  │
│  │  │(EPEX API)   │  │ Engine      │                            │  │
│  │  └─────────────┘  └─────────────┘                            │  │
│  │         │                │                                   │  │
│  │         ▼                ▼                                   │  │
│  │  ┌─────────────────────────────────────────────┐             │  │
│  │  │         Control Manager                     │             │  │
│  │  │  - Virtual Transformer Logic                │             │  │
│  │  │  - Household Revenue Model Config           │             │  │
│  │  │  - Load Coordination                        │             │  │
│  │  └─────────────────────────────────────────────┘             │  │
│  └───────────────────┬─────────────────────────────────────────┘  │
│                      │ MQTT over Ethernet (LAN)                    │
│  ┌───────────────────┴─────────────────────────────────────────┐  │
│  │  OUTDOOR GATEWAY (Milesight UG67, IP67)                      │  │
│  │  LoRaWAN ↔ MQTT — translates between wireless and LAN       │  │
│  └───────────────────┬─────────────────────────────────────────┘  │
│                      │ LoRaWAN 868 MHz (uplink + downlink)        │
│  ┌───────────────────┴─────────────────────────────────────────┐  │
│  │  BRIDGE DEVICE (×N) — one per household                     │  │
│  │  ┌──────────────────────────────────────────────────────┐   │  │
│  │  │  • LoRaWAN uplink (sensor data) + downlink (limits)  │   │  │
│  │  │  • IR optical: reads smart meter                      │   │  │
│  │  │  • Modbus RTU (RS485): controls home devices          │   │  │
│  │  │  • Enforces power limits, executes shed commands      │   │  │
│  │  │  • Buffers data during LoRaWAN outages                │   │  │
│  │  └──────────────────────────────────────────────────────┘   │  │
│  │                      │ Modbus RTU (RS485)                     │
│  │  ┌──────────────────────────────────────────────────────┐   │  │
│  │  │  HOUSEHOLD DEVICES (varies per home)                  │   │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────┐ │   │  │
│  │  │  │ Wallbox  │  │PV Inverter│  │ Battery  │  │Smart │ │   │  │
│  │  │  │ (Modbus) │  │ (Modbus) │  │ (Modbus) │  │Plug  │ │   │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────┘ │   │  │
│  │  └──────────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 7.3 Component Categories (Generic)

| Domain | Component Category | Phase | Description |
|--------|-------------------|-------|-------------|
| Household | Bridge Device | 1+2 | ESP32 with LoRa radio + IR read head + Modbus RS485; reads meter, controls devices, communicates via LoRaWAN |
| Household | Control Device | 2 | Wallbox, smart plug, battery inverter (Modbus RTU) |
| Gateway | Outdoor Gateway | 1 | LoRaWAN ↔ MQTT translator; weatherproof, Ethernet to LAN |
| Neighborhood | Server Platform | 1 | Runs locally, no cloud |
| Neighborhood | Message Broker | 1 | MQTT for data transport (LAN side only) |
| Neighborhood | Database | 1 | Local time-series storage |
| Neighborhood | Price Importer | 2 | EPEX Spot market integration |
| Neighborhood | Optimization Engine | 2 | Control logic with revenue awareness |
| Neighborhood | Dashboard | 1 | User interface for viewing data |

*Note: Specific products are NOT specified at requirements level.*

---

## 8. Cost Targets

### 8.1 Budget Breakdown - Phase 1

**Central Infrastructure (One-time)**

| Category | Target Cost | Notes |
|----------|-------------|-------|
| Central server platform | €100-150 | Single-board computer or dedicated appliance |
| Outdoor gateway | €150-200 | Weatherproof wireless receiver |
| Network equipment | €20-50 | Cables, PoE injector if needed |
| **Central Total** | **≤ €300** | |

**Per-Household Hardware**

| Category | Target Cost | Notes |
|----------|-------------|-------|
| Energy sensor + wireless | €100-150 | Optical reader with integrated radio |
| Power supply (optional) | €0-20 | If USB power not available near meter |
| Installation materials | €0-10 | Cable ties, mounting if needed |
| **Per Household Total** | **≤ €200** | |

### 8.2 Budget Breakdown - Phase 2 (Additional)

**Per-Household Hardware (Phase 2 add-on)**

| Category | Target Cost | Notes |
|----------|-------------|-------|
| Smart plug/switch | €20-40 | Per controllable device |
| Wallbox integration | €0 | Usually already present |
| Battery/PV integration | €0-50 | Depends on existing equipment |
| **Phase 2 Add-on Total** | **≤ €150** | |

### 8.3 Example Cost Scenarios

| Scenario | Households | Phase 1 Cost | Phase 2 Cost | Total |
|----------|------------|--------------|--------------|-------|
| Small (10 households) | 10 | €1,700 | €1,500 | €3,200 |
| Medium (50 households) | 50 | €7,300 | €7,500 | €14,800 |
| Large (100 households) | 100 | €14,300 | €15,000 | €29,300 |

---

## 9. Future Phases (Roadmap)

### 9.1 Phase 3: Advanced Optimization (Future)

- Add predictive algorithms based on weather forecasts
- Add battery degradation-aware optimization
- Add multi-community coordination
- Add peer-to-peer energy trading
- Add grid service provision (aggregator model)

---

## 10. Glossary

| Term | Definition |
|------|------------|
| **mME** | Moderne Messeinrichtung - Modern electricity meter with IR interface |
| **iMSys** | Intelligentes Messsystem - Smart Meter with remote reading |
| **SML** | Smart Message Language - Standard protocol for meter data |
| **IEC 62056-21** | International standard for meter reading |
| **MQTT** | Message Queuing Telemetry Transport - Lightweight messaging protocol |
| **EPEX Spot** | European Power Exchange - Day-ahead electricity market |
| **Virtual Transformer** | Aggregated total power consumption of all households |
| **VDE-AR-N 4100** | German association rule for low-voltage grid connection |
| **BSI TR-03109-1** | German technical guideline for smart meter interfaces |
| **GDPR/DSGVO** | General Data Protection Regulation |
| **RED** | Radio Equipment Directive - EU certification for wireless devices |
| **EEG** | Erneuerbare-Energien-Gesetz - Renewable Energy Act |
| **§14a EnWG** | Paragraph 14a of Energy Industry Act - Variable network charges |
| **OCPP** | Open Charge Point Protocol - Wallbox communication standard |
| **IP67** | Ingress Protection rating - Dust-tight and waterproof |

---

## 11. Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | May 2025 | Initial version (technical) |
| 2.0 | May 2026 | Requirements-focused, abstracted from specific products |
| 3.0 | May 2026 | Added Phase 2 requirements with pricing model diversity |

---

*This document defines the requirements for Phase 1 and Phase 2 of the LEM-Netz project. It is technology-agnostic and focuses on what the system must do, not how it is implemented. Economic fairness across all household types is a core requirement.*