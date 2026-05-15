# LEM-Netz

Documentation-only repository (German decentralized energy management). Both phases in planning — no code, no builds, no tests.

## Repo Contents

| File | Role |
|------|------|
| `README.md` | Overview, household types, phases, cost targets |
| `Requirements_Specification_LEM-Netz.md` | Functional/non-functional requirements v3.0 (Phase 1+2) |
| `LEM-Netz_Technical_Concept.md` | Implementation guide v3.4 — components, MQTT topics, architecture |

If the two specs conflict, trust the Technical Concept (v3.4, implementation-focused, more recent).

## No Build / No Automation

There is no CI, no pre-commit, no linter, no test runner, no package manager, no Dockerfile. Do not look for or run any build/check commands. The files are plain Markdown, edited directly.

## Domain Constraints (agents constantly miss these)

- **Infrastructure safety first**: transformer limits are non-negotiable; optimization is suspended when limits require it.
- **Economic fairness**: optimization must NOT financially disadvantage any household type (fixed tariff, EEG feed-in, dynamic pricing, §14a EnWG, PV+battery — all must break even or benefit).
- **German market**: EEG, Tibber/aWATTar/Ostrom, §14a, EPEX Spot. All data stays local (GDPR, no cloud dependency).
- **Architecture**: Two networks connected by a bridge device per household. The **neighborhood network** (Home Assistant + Mosquitto + InfluxDB) coordinates load balancing. Each household has a **bridge device** (ESP32 + LoRa radio) that connects the household's internal devices (Modbus RS485) to the central server via **LoRaWAN 868 MHz** through the outdoor gateway (Milesight UG67). MQTT stays on the LAN between gateway and server — no WiFi or MQTT at the household level.
