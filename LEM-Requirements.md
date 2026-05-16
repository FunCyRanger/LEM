**Requirements Analysis and Use-Case Documentation: Decentralized LEM-Netz (Simplified Version)**

**Status:** Draft

### 1. Introduction and Purpose

The decentralized Local Energy Management System (LEM-Netz) enables neighborhoods to robustly and cost-effectively coordinate generation and consumption. It prioritizes the protection of grid infrastructure and realizes economic benefits primarily through § 14a EnWG and optimized self-consumption. Formal balancing energy-sharing accounting is deliberately avoided to minimize complexity and additional regulatory hurdles.

### 2. Goals

- Highest priority: Ensuring grid infrastructure safety (transformers and lines).
- Increasing local self-consumption and load flexibility.
- Leveraging existing and future regulatory incentives (§ 14a EnWG).
- High robustness and self-governance capability.
- Low entry barriers and easy extensibility.
- Ensuring data sovereignty of participants.

### 3. Functional Requirements (FR)

**FR-01 Grid Infrastructure Protection (highest priority)**  
The system must periodically determine the maximum permissible net export/import limit for the neighborhood or affected grid branch and distribute it bindingly to all participants.

**FR-02 Measurement Data Acquisition**  
Provision of time-resolved consumption and generation data through a suitable, certified metering device. As a private individual, access to this measurement data must be possible.

**FR-03 Decentralized Agents**  
Each participant operates an autonomous agent that processes local measurement data, offers or requests flexibility, and optimizes their own installations.

**FR-04 Local Coordination**  
Support for coordinating local surplus and demand within applicable grid limits to increase self-consumption and grid-serving behavior (without balancing accounting).

**FR-05 Simple Onboarding**  
New participants must be able to integrate into the system without extensive administrative effort.

**FR-06 Support for Grid-Serving Control**  
Provision of mechanisms for grid-oriented adjustment of controllable consumption devices according to § 14a EnWG.

### 4. Non-Functional Requirements

- **Robustness**: The system must be able to continue operating with reduced functionality during partial failures. The grid protection function has absolute priority.
- **Economic Efficiency**: Low investment and operating costs.
- **Data Sovereignty and Privacy**: Local data processing in compliance with GDPR.
- **Scalability**: Support for a variable number of households in a neighborhood.
- **Simplicity**: Minimization of administrative and technical complexity.
- **Interoperability**: Compatibility with existing and future metering and control infrastructures.

### 5. Use-Case Diagrams (Mermaid)

#### Phase 1 — Data Collection

```mermaid
flowchart LR
    Participant["**Participant**\n(Household / Prosumer)"]:::actor
    DSO["**Grid Operator**\n(DSO)"]:::dso

    Participant --> UC01["Determine & broadcast\ngrid limit"]
    Participant --> UC02["Record consumption &\ngeneration data"]
    Participant --> UC05["Simple onboarding"]

    DSO --> UC01
```

#### Phase 2 — Control

```mermaid
flowchart LR
    Participant["**Participant**\n(Household / Prosumer)"]:::actor
    DSO["**Grid Operator**\n(DSO)"]:::dso

    Participant --> UC03["Offer / request\nflexibility"]
    Participant --> UC04["Local coordination &\nload shifting"]
    Participant --> UC06["Perform grid-serving\ncontrol"]

    DSO --> UC06

    classDef actor fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef dso fill:#e8f5e9,stroke:#388e3c,stroke-width:2px,color:#000
```

### 6. Detailed Use Cases

#### Phase 1 — Data Collection

- **UC-01 Determine & broadcast grid limit**: Periodic determination and distribution of the binding grid limit.
- **UC-02 Record consumption & generation data**: Provision of timely measurement values through suitable metering devices.
- **UC-05 Simple onboarding**: New participants can register via a self-service process with minimal configuration and are automatically recognized by the system.

#### Phase 2 — Control

- **UC-03 Offer / request flexibility**: Participants advertise available flexibility or signal demand to the local coordinator for load shifting.
- **UC-04 Local coordination & load shifting**: Coordination of flexibility to optimize self-consumption and grid-serving behavior.
- **UC-06 Grid-serving control**: Support for § 14a-compliant control of controllable consumption devices.

### 7. Sources

1. Gesetz über die Elektrizitäts- und Gasversorgung (Energiewirtschaftsgesetz - EnWG), § 14a – Netzorientierte Steuerung von steuerbaren Verbrauchseinrichtungen und steuerbaren Netzanschlüssen.  
   [https://www.gesetze-im-internet.de/enwg_2005/__14a.html](https://www.gesetze-im-internet.de/enwg_2005/__14a.html)

2. Bundesnetzagentur. Festlegungsverfahren zur Integration von steuerbaren Verbrauchseinrichtungen und steuerbaren Netzanschlüssen nach § 14a EnWG.  
   [https://www.bundesnetzagentur.de/enwg14a](https://www.bundesnetzagentur.de/enwg14a)

3. Gesetz über den Messstellenbetrieb und die Datenkommunikation in intelligenten Energienetzen (Messstellenbetriebsgesetz - MsbG).  
   [https://www.gesetze-im-internet.de/messbg/](https://www.gesetze-im-internet.de/messbg/)

4. Bundesnetzagentur. Informationen zur netzorientierten Steuerung und Netzentgeltreduzierung nach § 14a EnWG.  
   [https://www.bundesnetzagentur.de/DE/Vportal/Energie/SteuerbareVBE/start.html](https://www.bundesnetzagentur.de/DE/Vportal/Energie/SteuerbareVBE/start.html)

---

**Two-Pager: Decentralized LEM-Netz – Neighborhood Energy Management**

**Page 1 – System Description**

The decentralized LEM-Netz is a simple, robust system for local coordination of electricity generation and consumption in residential neighborhoods. It is based on the use of suitable metering devices, decentralized agents, and clear priority rules.

**Core functions**:
- Continuous monitoring and adherence to grid limits to protect local infrastructure.
- Coordinated use of generation surpluses and consumption flexibility.
- Support for grid-serving operation of controllable installations.

The system avoids complex billing mechanisms and focuses on practical, immediately usable benefits. It is designed to start with existing installations and be expanded incrementally.

**Why does implementation make sense?**  
The ongoing energy transition is leading to increasing decentralized generation and electrification of the heating and transport sectors. This significantly increases the load on low-voltage grids. Local coordination mechanisms can reduce grid congestion without costly grid expansion. At the same time, they enable households to realize direct economic benefits through optimized self-consumption and regulatory incentives such as § 14a EnWG.

**Page 2 – Advantages, Disadvantages, and Assessment**

**Advantages**:
- **Economic**: Utilization of grid fee reductions according to § 14a EnWG and increase in self-consumption share.
- **Technically robust**: High reliability through decentralized structure and graceful degradation.
- **Regulatory compliant**: No dependency on certified sharing metering systems; compatibility with current legal frameworks.
- **Practical**: Low entry barriers and use of existing metering infrastructure.
- **Future-proof**: Solid foundation for later regulatory developments.

**Disadvantages**:
- Limited monetary benefit when few controllable consumption devices are available.
- Dependence on the grid operator's willingness to cooperate for full § 14a benefits.
- Initial organizational effort within the neighborhood.

**Overall assessment**:  
The decentralized LEM-Netz represents a pragmatic and sensible approach. It addresses real physical and economic challenges of the energy transition at the neighborhood level, without requiring excessive complexity or high investment. In a phase of increasing grid load and grid fees, it offers a concrete contribution to local resilience, cost reduction, and efficient use of existing infrastructure.

**Recommendation**: Start with a small pilot project to validate the practical effectiveness of grid protection and flexibility coordination.

**Sources (Two-Pager)**  
See detailed sources in the requirements analysis (Section 7).
