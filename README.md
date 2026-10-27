# SASD-Chess

**SASD-Chess** ist ein eigenständiges Open-Source-Projekt zur Entwicklung eines klassischen Schachprogramms in **C++**. Es umfasst das vollständige Schachregelwerk, eine grafische Oberfläche und eine **selbst entwickelte Strategie-Engine** mit klassischer Spielbaumsuche und heuristischer Stellungsbewertung.

Der Schwerpunkt liegt auf **Korrektheit, sauberer Architektur, dokumentierten Algorithmen, automatisierten Tests und Portabilität**. Eine Internetverbindung oder ein Benutzerkonto sind für die vorgesehenen Kernfunktionen nicht erforderlich.

> **Projektstatus (27.10.2026): Planung und Spezifikation.** Die nachfolgenden Eigenschaften beschreiben das Entwicklungsziel. Es liegt noch keine fertige, spielbare Programmversion vor.

## Anforderungen und technische Planung

Die ausführlichen, deutschsprachigen Projektdokumente sind:

- **[Lastenheft](docs/Lastenheft.md)** – fachlicher Bedarf, Zielgruppen, Schachregeln, Funktionen, Qualitätsanforderungen, Muss-/Soll-Priorisierung, Nutzungsszenarien und messbare Abnahmekriterien.
- **[Pflichtenheft](docs/Pflichtenheft.md)** – technische Umsetzung, C++-Architektur, Datenmodell, Zugerzeugung, klassische Spielstrategie, UCI-Protokoll, Prozesssteuerung, Testmatrix und Meilensteine.
- **[Architekturdokument](docs/Architektur.md)** – zusammenhängende Zielarchitektur, Modulgrenzen, Daten- und Kontrollflüsse, Lebenszyklen, Schnittstellen, Laboratory und technische Entscheidungen.
- **[Roadmap](ROADMAP.md)** – M0–M6 mit Arbeitspaketen und verbindlichen Abnahme-Gates sowie der gestufte Laboratory-Ausbau L1–L3.
- **[Agentenanweisungen](AGENTS.md)** – projektweite Entwicklungsregeln für Menschen und Coding Agents; [AGENT.md](AGENT.md) verweist darauf.
- **[Test- und Abnahmeplan](docs/TESTPLAN.md)** – überprüfbare Anforderungen und Testfälle für Chess Core, UCI, Selbstspiel, GUI und Laboratory.
- **[Entwicklungs- und Buildhandbuch](docs/DEVELOPMENT.md)** – vorgesehene CMake-/CTest-Abläufe, Windows/Linux und Fehlersuche.
- **[Mitwirkungsrichtlinien](CONTRIBUTING.md)** – Code-/Review-/Testregeln und Hinweise zum noch offenen Lizenzstatus.
- **[Änderungsverlauf](CHANGELOG.md)** – tatsächlich dokumentierte Projektänderungen statt ungeprüfter Funktionszusagen.
- **[ADR-Register](docs/adr/README.md)** – Vorlage und Status für nachprüfbare Architekturentscheidungen.

Lasten- und Pflichtenheft liegen als Entwürfe in **Version 0.4** vor; das vertiefende Architekturdokument als **Version 0.2**. Nicht bestätigte technische Entscheidungen sind ausdrücklich gekennzeichnet. Die Dokumente beschreiben eine **Zielarchitektur** und keinen bereits implementierten Softwarestand.

## Geplante Funktionen

| Bereich | Funktionsumfang |
| --- | --- |
| **Schachkern** | Standardschach, legale Züge, Rochade, En passant, Umwandlung, Matt, Patt, Remis und Rücknahme |
| **Spielstrategie** | Eigene klassische Engine mit Negamax/Alpha-Beta, Stellungsbewertung, Suchtiefe und Zeitkontrolle |
| **Benutzeroberfläche** | Schachbrett, Zugliste, Spielstände, Mensch/Mensch und Mensch/Computer |
| **Formate** | FEN-Ein-/Ausgabe, SAN-Notation, PGN-Export |
| **UCI-Engine** | Nutzung von SASD-Chess durch UCI-kompatible externe Schachoberflächen |
| **UCI-Client** | Start und Steuerung einer fremden UCI-Engine über eigene Prozesskanäle |
| **Match-Controller** | **Automatische Partien zwischen zwei UCI-Prozessen**, auch zwei Instanzen derselben SASD-Chess-Binärdatei |
| **Tests** | Perft, Unit-/Integrations- und UCI-Protokolltests, Debug-/Release-Builds unter Windows und Linux |

### Schach-Engine gegen Schach-Engine

Eine Engine kommuniziert nicht direkt mit einer zweiten Engine. Ein **separater Match-Controller** startet zwei unabhängige Programme, sendet die Stellungen über UCI, validiert jeden gemeldeten Zug über den gemeinsamen Schachkern und dokumentiert den Verlauf als PGN.

Die Schnittstelle erlaubt insbesondere **SASD-Chess gegen fremde UCI-Engines** und **SASD-Chess gegen SASD-Chess** (zwei gleichzeitig gestartete Prozesse). Der Match-Controller ist auch ohne grafische Oberfläche als Kommandozeilenprogramm vorgesehen.

## Science-Beitrag: SASD-Chess Laboratory (geplant)

Das Projekt soll sich zusätzlich als **schachbezogenes Experimentierwerkzeug für angewandte Informatik** eignen. Ziel ist nicht nur eine spielbare klassische Schach-Engine, sondern die nachvollziehbare Untersuchung von Softwarekorrektheit, Suchalgorithmen, Bewertungsfunktionen und Rechenleistung.

Geplante Bereiche sind **Spielen**, **Analysieren** und **Laboratory**. Technische Funktionen: Perft/Divide, transparente Stellungsbewertung, reproduzierbare Benchmarks, UCI-Vergleichspartien mit statistischer Auswertung, begrenzte Suchtraces, Stellungslabor, Experimentdefinitionen und Forschungsberichte.

**Priorisierung:** Perft/Divide gehört zur 1.0-Qualitätssicherung. Die anderen Analyse- und Forschungsfunktionen sind Soll- oder spätere Ausbaustufen und dürfen die erste spielbare Schachversion nicht blockieren. **Reproduzierbarkeit und transparente Unsicherheiten** haben Vorrang vor unbelegten Aussagen über Suchleistung oder Spielstärke.

Die vollständigen Anforderungen stehen im [Lastenheft, Abschnitt 5.4–5.5](docs/Lastenheft.md), die technische Umsetzung im [Pflichtenheft, Abschnitt 19](docs/Pflichtenheft.md).

## Entwicklungsschritte

Die Meilensteine und ihr tatsächlicher Status werden in **[ROADMAP.md](ROADMAP.md)** gepflegt. Die folgende Aufzählung ist eine Kurzfassung des geplanten Ablaufs:

1. **M0 – Projektbasis:** CMake, CTest, CI, Coding- und Dokumentationsregeln.
2. **M1 – Schachkern:** Stellungen, legale Züge, Sonderregeln und Perft-Referenztests.
3. **M2 – Spielverwaltung:** Zugliste, Remisregeln, FEN, SAN und PGN.
4. **M3 – Strategie-Engine:** Klassische Suche, Bewertung und UCI-Engine-Prozess.
5. **M4 – Engine-Matches:** UCI-Client, Match-Controller, Fremdengine-Partien und Selbstspiel.
6. **M5 – Desktop:** Spielbare Benutzeroberfläche für die Pflichtmodi.
7. **M6 – Veröffentlichung:** Regressionstests, Dokumentation, Plattformprüfung und Release 1.0.

Das Projekt strebt **C++20** und **CMake** an. Die Auswahl des GUI-Frameworks und der konkreten Open-Source-Lizenz ist noch offen. Die Schachlogik soll vollständig unabhängig von der gewählten Benutzeroberfläche sein.

## Geltungsbereich

Dieses Repository behandelt ausschließlich **SASD-Chess als klassisches Schachprogramm**. Der fachliche und technische Rahmen wird durch Lastenheft, Pflichtenheft und die zugehörigen Architektur- und Entwicklungsdokumente definiert. **Geplante Eigenschaften sind nicht automatisch bereits implementiert oder getestet.**
