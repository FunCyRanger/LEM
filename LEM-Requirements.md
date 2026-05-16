**Anforderungsanalyse und Use-Case-Dokumentation: Dezentrales LEM-Netz (vereinfachte Version)**

**Status:** Entwurf  

### 1. Einleitung und Zweck

Das dezentrale Lokale Energiemanagement-System (LEM-Netz) ermöglicht Quartieren eine robuste und kostengünstige Koordination von Erzeugung und Verbrauch. Es priorisiert den Schutz der Netzinfrastruktur und realisiert wirtschaftliche Vorteile primär über § 14a EnWG sowie optimierten Eigenverbrauch. Formale bilanzielle Energy-Sharing-Abrechnung wird bewusst vermieden, um Komplexität und zusätzliche regulatorische Hürden zu minimieren.

### 2. Ziele

- Höchste Priorität: Sicherstellung der Netzinfrastruktursicherheit (Transformatoren und Leitungen).
- Steigerung des lokalen Eigenverbrauchs und der Lastflexibilität.
- Nutzung bestehender und zukünftiger regulatorischer Anreize (§ 14a EnWG).
- Hohe Robustheit und Selbstverwaltungsfähigkeit des Systems.
- Niedrige Einstiegshürden und einfache Erweiterbarkeit.
- Gewährleistung der Datensouveränität der Teilnehmer.

### 3. Funktionale Anforderungen (FR)

**FR-01 Netzinfrastrukturschutz (höchste Priorität)**  
Das System muss periodisch das maximale zulässige Netto-Export-/Import-Limit für das Quartier oder den betroffenen Netzstrang ermitteln und verbindlich an alle Teilnehmer verteilen.

**FR-02 Messdatenerfassung**  
Bereitstellung von zeitlich aufgelösten Verbrauchs- und Erzeugungsdaten durch ein geeignetes, geeichtes Messmittel. Als Privatperson muss der Zugriff auf diese Messdaten möglich sein.

**FR-03 Dezentrale Agenten**  
Jeder Teilnehmer betreibt einen autonomen Agenten, der lokale Messdaten verarbeitet, Flexibilität anbietet oder nachfragt und eigene Anlagen optimiert.

**FR-04 Lokale Koordination**  
Unterstützung der Abstimmung von lokalem Überschuss und Bedarf innerhalb der geltenden Netzlimits zur Steigerung des Eigenverbrauchs und der Netzdienlichkeit (ohne bilanzielle Abrechnung).

**FR-05 Einfacher Beitritt**  
Neue Teilnehmer müssen sich ohne umfangreichen administrativen Aufwand in das System integrieren können.

**FR-06 Unterstützung netzdienlicher Steuerung**  
Bereitstellung von Mechanismen zur netzorientierten Anpassung steuerbarer Verbrauchseinrichtungen gemäß § 14a EnWG.

### 4. Nicht-funktionale Anforderungen

- **Robustheit**: Das System muss bei Teilausfällen mit reduzierter Funktionalität weiterbetrieben werden können. Die Netzschutzfunktion hat absolute Priorität.
- **Wirtschaftlichkeit**: Niedrige Investitions- und Betriebskosten.
- **Datensouveränität und Datenschutz**: Lokale Verarbeitung der Daten unter Einhaltung der DSGVO.
- **Skalierbarkeit**: Unterstützung einer variablen Anzahl von Haushalten in einem Quartier.
- **Einfachheit**: Minimierung administrativer und technischer Komplexität.
- **Interoperabilität**: Kompatibilität mit bestehenden und zukünftigen Mess- und Steuerungsinfrastrukturen.

### 5. Use-Case-Diagramm (Mermaid)

```mermaid
flowchart TD
    subgraph "Dezentrales LEM-System"
        direction TB
        UC01["Netzlimit ermitteln\n& broadcasten"]
        UC02["Verbrauchs- und Erzeugungsdaten erfassen"]
        UC03["Flexibilität anbieten / nachfragen"]
        UC04["Lokale Koordination & Lastverschiebung"]
        UC05["Einfacher Beitritt"]
        UC06["Netzdienliche Steuerung durchführen"]
    end

    Teilnehmer["**Teilnehmer**\n(Haushalt / Prosumer)"]:::actor
    DSO["**Netzbetreiber**\n(DSO)"]:::dso

    Teilnehmer --> UC01
    Teilnehmer --> UC02
    Teilnehmer --> UC03
    Teilnehmer --> UC04
    Teilnehmer --> UC05
    Teilnehmer --> UC06

    DSO --> UC01
    DSO --> UC06

    UC03 -.->|<<include>>| UC04
    UC01 -.->|<<constraint>>\n(höchste Priorität)| UC03
    UC01 -.->|<<constraint>>| UC04
    UC01 -.->|<<constraint>>| UC06

    classDef actor fill:#e3f2fd,stroke:#1976d2,stroke-width:2px,color:#000
    classDef dso fill:#e8f5e9,stroke:#388e3c,stroke-width:2px,color:#000
```

### 6. Detaillierte Use Cases (Kern)

- **UC-01 Netzlimit ermitteln & broadcasten**: Periodische Bestimmung und Verteilung des verbindlichen Netzlimits.
- **UC-02 Verbrauchs- und Erzeugungsdaten erfassen**: Bereitstellung zeitnaher Messwerte durch geeignete Messmittel.
- **UC-04 Lokale Koordination & Lastverschiebung**: Abstimmung von Flexibilität zur Optimierung von Eigenverbrauch und Netzdienlichkeit.
- **UC-06 Netzdienliche Steuerung**: Unterstützung § 14a-konformer Steuerung steuerbarer Verbrauchseinrichtungen.

### 7. Quellenverzeichnis

1. Gesetz über die Elektrizitäts- und Gasversorgung (Energiewirtschaftsgesetz - EnWG), § 14a – Netzorientierte Steuerung von steuerbaren Verbrauchseinrichtungen und steuerbaren Netzanschlüssen.  
   [https://www.gesetze-im-internet.de/enwg_2005/__14a.html](https://www.gesetze-im-internet.de/enwg_2005/__14a.html)

2. Bundesnetzagentur. Festlegungsverfahren zur Integration von steuerbaren Verbrauchseinrichtungen und steuerbaren Netzanschlüssen nach § 14a EnWG.  
   [https://www.bundesnetzagentur.de/enwg14a](https://www.bundesnetzagentur.de/enwg14a)

3. Gesetz über den Messstellenbetrieb und die Datenkommunikation in intelligenten Energienetzen (Messstellenbetriebsgesetz - MsbG).  
   [https://www.gesetze-im-internet.de/messbg/](https://www.gesetze-im-internet.de/messbg/)

4. Bundesnetzagentur. Informationen zur netzorientierten Steuerung und Netzentgeltreduzierung nach § 14a EnWG.  
   [https://www.bundesnetzagentur.de/DE/Vportal/Energie/SteuerbareVBE/start.html](https://www.bundesnetzagentur.de/DE/Vportal/Energie/SteuerbareVBE/start.html)

---

**Two-Pager: Dezentrales LEM-Netz – Quartiersenergiemanagement**

**Seite 1 – Systembeschreibung**

Das dezentrale LEM-Netz ist ein einfaches, robustes System zur lokalen Koordination von Stromerzeugung und -verbrauch in Wohnquartieren. Es basiert auf der Nutzung geeigneter Messmittel, dezentraler Agenten und klarer Prioritätenregelungen.

**Kernfunktionen**:
- Kontinuierliche Überwachung und Einhaltung von Netzlimits zum Schutz der lokalen Infrastruktur.
- Koordinierte Nutzung von Erzeugungsüberschüssen und Verbrauchsflexibilität.
- Unterstützung netzdienlicher Betriebsweisen steuerbarer Anlagen.

Das System vermeidet komplexe Abrechnungsmechanismen und konzentriert sich auf praktische, sofort nutzbare Vorteile. Es ist so gestaltet, dass es mit vorhandenen Installationen startet und schrittweise erweitert werden kann.

**Warum ist die Umsetzung sinnvoll?**  
Die fortschreitende Energiewende führt zu steigender dezentraler Erzeugung und Elektrifizierung des Wärme- und Verkehrssektors. Dies erhöht die Belastung der Niederspannungsnetze erheblich. Lokale Koordinationsmechanismen können Netzüberlastungen reduzieren, ohne kostspielige Netzausbauten. Gleichzeitig ermöglichen sie Haushalten direkte wirtschaftliche Vorteile durch optimierten Eigenverbrauch und regulatorische Anreize wie § 14a EnWG.

**Seite 2 – Vorteile, Nachteile und Bewertung**

**Vorteile**:
- **Wirtschaftlich**: Nutzung von Netzentgeltreduktionen nach § 14a EnWG und Steigerung des Eigenverbrauchsanteils.
- **Technisch robust**: Hohe Ausfallsicherheit durch dezentrale Struktur und Graceful Degradation.
- **Regulatorisch sicher**: Keine Abhängigkeit von zertifizierten Sharing-Messsystemen; Kompatibilität mit aktuellen gesetzlichen Rahmenbedingungen.
- **Praktikabel**: Niedrige Einstiegshürden und Nutzung vorhandener Messinfrastruktur.
- **Zukunftssicher**: Gute Grundlage für spätere regulatorische Entwicklungen.

**Nachteile**:
- Begrenzter monetärer Nutzen, wenn wenige steuerbare Verbrauchseinrichtungen vorhanden sind.
- Abhängigkeit von der Kooperationsbereitschaft des Netzbetreibers für volle § 14a-Vorteile.
- Initialer Organisationsaufwand im Quartier.

**Gesamtbewertung**:  
Das dezentrale LEM-Netz stellt einen pragmatischen und sinnvollen Ansatz dar. Es adressiert reale physikalische und wirtschaftliche Herausforderungen der Energiewende auf Quartiersebene, ohne übermäßige Komplexität oder hohe Investitionen zu erfordern. In einer Phase steigender Netzbelastung und Netzentgelte bietet es einen konkreten Beitrag zur lokalen Resilienz, Kostensenkung und effizienten Nutzung bestehender Infrastruktur.

**Empfehlung**: Beginnen Sie mit einem kleinen Pilotprojekt, um die praktische Wirksamkeit des Netzschutzes und der Flexibilitätskoordination zu validieren.

**Quellenverzeichnis (Two-Pager)**  
Siehe detailliertes Quellenverzeichnis in der Anforderungsanalyse (Abschnitt 7).
