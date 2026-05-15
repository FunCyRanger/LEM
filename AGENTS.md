# LEM-Netz Repository

This repository contains specification documents for a decentralized energy management system (German: "dezentrales Energiemanagementsystem").

## Project Status

- **Phase 1**: In planning - Data collection via Smart Meter → LoRaWAN → Home Assistant
- **Phase 2**: Planning - Control integration with revenue-aware optimization
- **Focus**: Multi-household energy monitoring with economic fairness across different pricing models

## Documents

| File | Description |
|------|-------------|
| `Requirements_Specification_LEM-Netz.md` | Functional and non-functional requirements (v3.0 - includes Phase 2) |
| `LEM-Netz_Technisches_Konzept.md` | Technical concept, installation guide, implementation steps (v3.0) |

## Key Specifications

- **Goal**: Neighborhood-level energy optimization via wireless (868 MHz)
- **Architecture**: Edge (sensors) → Gateway (outdoor) → Central (Home Assistant)
- **Cost Targets**: ~€300 central + ~€200/household (Phase 1), ~€350 total (Phase 2)
- **Coverage**: Up to 100-1000 households over 1000+ meters (outdoor IP67 gateway)

## German Market Focus

The system supports diverse household configurations common in Germany:
- Fixed EEG feed-in tariffs
- Dynamic pricing (Tibber, aWATTar, Ostrom)
- §14a EnWG variable network charges
- PV + battery systems
- EV/wallbox integration

**Critical**: Optimization must NOT cause financial disadvantage to any household type.

## No Code

This is a documentation-only repository. Do not attempt to:
- Run builds, tests, or linters
- Create code or configuration files
- Look for entrypoints or package managers