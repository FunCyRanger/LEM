# LEM-Netz

Documentation-only repository (German decentralized energy management). Phase 1 (data collection) in planning; Phase 2 (control integration) in planning.

## No Code

This repo contains only Markdown specs. Do **not** run builds, tests, linters, or look for entrypoints/package managers.

## Documents

| File | What it contains |
|------|-----------------|
| `README.md` | Project overview, supported household types, phases, cost targets |
| `Requirements_Specification_LEM-Netz.md` | Functional/non-functional requirements v3.0 (Phase 1+2), 604 lines |
| `LEM-Netz_Technical_Concept.md` | Implementation guide v3.4, component selection, MQTT topics, 1333 lines |

## Domain Constraints (agents constantly miss these)

- **Infrastructure safety first**: transformer limits are non-negotiable; optimization is suspended when limits require it.
- **Economic fairness**: optimization must NOT financially disadvantage any household type (fixed tariff, EEG feed-in, dynamic pricing, §14a EnWG, PV+battery — all must break even or benefit).
- **German market**: supports EEG, Tibber/aWATTar/Ostrom, §14a, EPEX Spot. All data stays local (GDPR, no cloud dependency).
- **Architecture**: Edge (iOKE868 LoRaWAN sensors) → Outdoor Gateway (Milesight UG67, IP67) → Central (Home Assistant + Mosquitto + InfluxDB). Phase 2 adds a single ESP32/ESPHome per household bridging Modbus/WiFi devices to MQTT. One controller type for all households.

## Repo Structure

No subdirectories, no generated files. The two spec docs are the authoritative sources. If they conflict, trust the Technical Concept (it is implementation-focused and more recent v3.4 vs v3.0).
