# LEM-Netz

Decentralized energy management system for neighborhood communities in Germany.

## Overview

LEM-Netz enables neighborhoods (Quartiere) to collect, aggregate, and utilize their own energy data locally—without dependency on external cloud services. The system monitors power consumption across multiple households and coordinates controllable loads to optimize energy usage while ensuring infrastructure stability.

## Key Features

- **Data Collection**: Read smart meter data (power, energy) via optical IR interface
- **Wireless Transmission**: LoRaWAN 868 MHz for distances up to 1000m+
- **Local Processing**: All data stored locally, no cloud dependency
- **Control Integration** (Phase 2): Coordinate wallboxes, batteries, smart plugs
- **Revenue-Aware Optimization**: Support for diverse German pricing models

## Priority Hierarchy

> **Infrastructure Safety > Economic Fairness**

The system NEVER allows optimization strategies that could overload the local transformer. Grid stability is non-negotiable—economic optimization is suspended when infrastructure limits require it.

## Supported Household Types

| Type | Pricing Model | Optimization Goal |
|------|---------------|-------------------|
| No PV | Fixed tariff | Minimize consumption cost |
| PV only (EEG) | Fixed feed-in | Maximize self-consumption |
| PV + Battery | Dynamic | Arbitrage (charge low, discharge high) |
| Heat pump | §14a network charges | Shift to low-tariff periods |
| EV + Wallbox | Dynamic | Coordinate charging |

## Phases

| Phase | Focus | Status |
|-------|-------|--------|
| **Phase 1** | Data collection | Planning |
| **Phase 2** | Control integration | Planning |

## Cost Targets

| Category | Budget |
|----------|--------|
| Central infrastructure | ≤ €300 |
| Per household (Phase 1) | ≤ €200 |
| Per household (Phase 2) | ≤ €350 |

## Documentation

- [Requirements Specification](Requirements_Specification_LEM-Netz.md) - Functional and non-functional requirements
- [Technical Concept](LEM-Netz_Technisches_Konzept.md) - Implementation guide
- [AGENTS.md](AGENTS.md) - Developer notes

## Tech Stack (Reference)

- **Sensors**: Optical IR readers with LoRaWAN
- **Gateway**: Outdoor IP67 LoRaWAN gateway
- **Server**: Home Assistant (local)
- **Protocols**: MQTT, SML, JSON

## Regulatory Compliance

- BSI TR-03109-1 (Smart Meter interface)
- VDE-AR-N 4100 (Grid connection)
- §14a EnWG (Variable network charges)
- GDPR (Data sovereignty)
- RED (Radio equipment)

## Why This Matters

- **Data Sovereignty**: All data stays in the community
- **Local Optimization**: Coordination without internet dependency
- **Economic Fairness**: No household disadvantaged by optimization
- **Community Control**: Residents own their data

---

*Project Status: Planning - Phase 1*