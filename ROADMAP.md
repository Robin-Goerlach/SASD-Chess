# SASD-Chess – Entwicklungsroadmap

> **Dokument:** ROADMAP-001 · **Version:** 0.1 · **Stand:** 27.10.2026  
> **Status:** Planungsentwurf, keine abgeschlossenen Implementierungsmeilensteine zugesichert  
> **Repository:** https://github.com/Robin-Goerlach/SASD-Chess  
> **Grundlagen:** [Lastenheft v0.4](docs/Lastenheft.md) · [Pflichtenheft v0.4](docs/Pflichtenheft.md) · [Architektur v0.2](docs/Architektur.md)

## 1. Projektauftrag und Grenzen

SASD-Chess wird ein eigenständiges, offline nutzbares **klassisches Schachprogramm in C++20**, das die Regeln des Standardschachs vollständig abbildet, über eine Desktop-Oberfläche bedient wird und eine **selbst entwickelte, auf Spielbaumsuche und expliziten heuristischen Bewertungen beruhende Strategie-Engine** mitbringt.

Zur 1.0 gehört sowohl eine **eigene UCI-Engine** als auch ein **UCI-Client mit unabhängigem Match-Controller**. Zwei getrennt gestartete Engine-Prozesse müssen automatisch gegeneinander spielen können, auch zwei Instanzen derselben SASD-Chess-Binärdatei. Der Match-Controller bleibt der autoritative Schiedsrichter.

Wissenschaftlicher Mehrwert entsteht durch **Perft/Divide, nachvollziehbare Diagnostik, reproduzierbare Messungen und spätere schachbezogene Laboratory-Funktionen**. Das Laboratory ist kein zweiter Schachkern und kein allgemeines Forschungsprogramm.

**Nicht Bestandteil dieses Projekts:** lernende Schachsysteme, trainierte Modelle, Cloud-/Onlinepflicht, Multiplayer-Server, generische Brettspiel-Engine, garantierte Elo-Zahl. Das Projekt bleibt vollständig bei **klassischen** Algorithmen.

## 2. Umgang mit Planung, Status und Prioritäten

- **Geplant:** noch nicht vollständig implementiert oder nicht nachweisbar abgenommen.
- **In Arbeit:** nachweisbarer Code/Commits liegen vor; Akzeptanzkriterien sind noch offen.
- **Blockiert:** eine konkrete externe Entscheidung oder ein nachgewiesenes technisches Hindernis verhindert Fortschritt.
- **Abgenommen:** alle ausdrücklich genannten Nachweise/Tests und Dokumentationsanforderungen sind erbracht.
- Der Status darf **nicht** nur aus vorhandenen Dateien, optimistischen Aussagen oder einer grünen Build-Ausgabe abgeleitet werden.
- Die nachstehenden M0–M6 sind **sequenzielle Qualitäts-Gates**, kein Terminplan. Einzelne vorbereitende Arbeiten dürfen parallel verlaufen, aber ein Meilenstein wird nur mit erfüllten Gates freigegeben.
- **Muss (M)** aus dem Lastenheft blockiert Release 1.0; **Soll (S)** und **Zukunft (Z)** blockieren ihn nicht, sofern nicht später ausdrücklich umpriorisiert.
- Die zurückgezogene Kennung **LH-F-031** wird nicht neu vergeben.
- Offen gebliebene Architekturentscheidungen werden in einem [ADR](docs/adr/README.md) behandelt und **nicht stillschweigend** getroffen.

### Statusübersicht (Dokumentationsstand 27.10.2026)

| Phase | Gegenstand | Dokumentierter Status |
| --- | --- | --- |
| Vorbereitung | Lastenheft, Pflichtenheft, Architektur | Entwürfe vorhanden; formale Freigabe offen |
| **M0** | Build-/Testfundament | **Geplant** |
| **M1** | Regelkern, Legalität, Perft/Divide | **Geplant** |
| **M2** | Spielverwaltung, Ergebnisse, FEN/SAN/PGN | **Geplant** |
| **M3** | Klassische Engine und UCI-Engine-Host | **Geplant** |
| **M4** | UCI-Client und unabhängiger Match-Schiedsrichter | **Geplant** |
| **M5** | Spielbare Desktop-Anwendung | **Geplant** |
| **M6** | Vollständige 1.0-Abnahme und Release | **Geplant** |
| **L1–L3** | SASD-Chess Laboratory – Erweiterungen | **Nach 1.0 geplant**, soweit nicht anders priorisiert |

*Bei Änderungen ist dieser Status auf Basis des tatsächlichen Repository-/CI-Standes zu aktualisieren.*

## 3. Gemeinsame Definition of Done (DoD)

Für **jedes** abgeschlossene Arbeitspaket gelten:

1. Der Code kompiliert auf den zum Meilenstein unterstützten Plattformen in den vorgesehenen Buildmodi.
2. Neue und bestehende fachlich relevante automatisierte Tests sind erfolgreich; Fehlerfälle besitzen Regressionstests, soweit reproduzierbar.
3. Modulgrenzen des Architekturdokuments werden eingehalten; die Änderung führt keine verdeckte zweite Regelimplementierung ein.
4. Öffentliche C++-APIs sowie schwer nachvollziehbare Algorithmen/Invarianten sind **ausführlich und begründend kommentiert**.
5. Fehlerszenarien, Grenzen, Ownership, Nebenläufigkeit und Ressourcenverbrauch sind bedacht.
6. Dokumentation und Requirements-/Test-Traceability werden gepflegt; der Meilensteinnachweis verweist auf Commit, Testlauf und bekannte Einschränkungen.
7. Keine optimierungsbedingte Abweichung bei Perft, legalen Zügen oder Format-/UCI-Konformität.
8. Nicht erfüllte Pflichtpunkte werden sichtbar als **offen** geführt, nicht durch stilles Herabstufen kaschiert.

## 4. M0 – Technisches Fundament

**Ziel:** Eine kleine, baubare, GUI-unabhängige C++20-Basis, die unter Linux und Windows getestet werden kann.

**Arbeitspakete:**

| ID | Paket | Abgrenzbares Ergebnis |
| --- | --- | --- |
| M0-01 | CMake-Projekt und Targets | Ein Core-Target, ein kleines Testtarget, CTest-Integration; keine leeren Bibliotheken ohne konkreten Nutzen |
| M0-02 | Toolchain-Konventionen | Dokumentierte C++20-Compiler, Warnungs-/Debug-/Release-Einstellungen, portable Dateipfade |
| M0-03 | GitHub Actions | Linux (GCC/Clang soweit verfügbar), Windows (MSVC); Build + CTest |
| M0-04 | Dokumentations-/Codekonventionen | `AGENTS.md`, `CONTRIBUTING.md`, Dokumentation und ADR-Vorgehen abgestimmt |
| M0-05 | Architektur-Skelett | `sasd::chess`-Namensraum, kleine fachliche Zielstruktur, keine UI-Kopplung |
| M0-06 | Entscheide/Abhängigkeiten | Testframework und erste Toolchain dokumentiert; offene GUI-/Lizenzfragen markiert |

**Abnahme-Gate:** Reproduzierbarer CMake-Build und mindestens ein ausführbarer CTest-Test unter Linux **und** Windows. Core-Header sind frei von GUI- und OS-spezifischen Includes. Alle noch offenen Entscheidungen sind benannt.

**Nicht erforderlich:** Spielbare Partie, fertige UI, Engine.

## 5. M1 – Schachkern, legale Züge und Perft/Divide

**Ziel:** Fachlich korrekte und unabhängig testbare Zugerzeugung für Standardschach.

| ID | Paket | Abgrenzbares Ergebnis |
| --- | --- | --- |
| M1-01 | Werttypen | `Color`, `PieceType`, `Piece`, `Square`, `Move`, geprüfte Konvertierungen |
| M1-02 | `Position` | Brett, Zugrecht, Rochaderechte, En-passant-Feld, Halb-/Vollzugzähler |
| M1-03 | Angriffs-/Zuggeometrie | Angriffsfelder getrennt von Zuglegalität, pseudo-legale Züge |
| M1-04 | Vollständig legale Züge | Königssicherheit, Fesselungen, Schachabwehr, Verbot des Königsschlags |
| M1-05 | Sonderzüge | Kurze/lange Rochade, En passant, vier mögliche Bauernumwandlungen |
| M1-06 | Reversibilität | `make/unmake` stellt **alle** Zustandsfelder exakt wieder her |
| M1-07 | Perft/Divide | Rekursive Blattzahlen, Zerlegung nach erstem Zug, CLI-/Test-API |
| M1-08 | Referenzfälle | Anfangsstellung und zusätzliche Spezial-FENs mit nachvollziehbarer Quelle und Erwartung |

**Abnahme-Gate:** Anfangsstellung Perft-Tiefen 1–5 liefert exakt **20 / 400 / 8.902 / 197.281 / 4.865.609**. Divide-Summe = Perft-Summe. Dokumentierte Tests für Rochade, En passant (auch Linienöffnung), Fesselung, Schach und Umwandlung grün; make/unmake-Identität geprüft.

**Vorbehalt:** Endgültige Partiehistorie/Remisbehandlung wird erst in M2 vollständig ergänzt. Perft allein ist **kein** Vollbeweis aller Schachregeln.

## 6. M2 – Partie, Historie und Austauschformate

**Ziel:** Vollständige, konsistente Partieabläufe und standardisierte Dateiformate.

| ID | Paket | Abgrenzbares Ergebnis |
| --- | --- | --- |
| M2-01 | `Game` und Verlauf | Autoritativer Spielstand, Zugfolge, Rücknahme und Snapshot-API |
| M2-02 | Status/Ergebnis | Matt, Patt, Remis, Beanspruchungen, FIDE-Regelprioritäten |
| M2-03 | Historische Wiederholungen | Stellungsschlüssel berücksichtigen Spielrecht, Rochade und zulässiges En passant |
| M2-04 | FEN | Strukturell und fachlich validierter Import/Export mit Roundtrip-Tests |
| M2-05 | SAN | Eindeutige Notation inkl. Schach, Matt, Promotion und Rochade |
| M2-06 | PGN-Export | Spielbare/verifizierbare Partie mit Ergebnis, Zugfolge, Tags und optionaler FEN-Startstellung |
| M2-07 | Fehlertransaktionen | Ungültige FEN oder illegale Züge verändern die laufende Partie nicht |
| M2-08 | PGN-Import (Soll) | Bei vorhandener Kapazität; nicht für die 1.0-Muss-Abnahme voraussetzen |

**Abnahme-Gate:** Mehrere vollständige Referenzpartien einschließlich Sonderfälle; korrekte Wiederholungs-/50-/75-Züge-Unterscheidung und geprüfte Remisansprüche; gültige FEN-/SAN-/PGN-Ausgaben und externe PGN-Lesbarkeit.

**Fachlicher Risikopunkt:** Eine unvollständige Heuristik für die FIDE-*tote Stellung* darf nicht als vollständige Regeltreue ausgegeben werden. Das in Architektur §5.5 dokumentierte Thema benötigt nachgewiesene Abdeckung oder eine ausdrücklich fachlich genehmigte Anpassung **vor Release 1.0**.

## 7. M3 – Klassische Such-Engine und UCI-Host

**Ziel:** Eigene klassische Strategie, extern über UCI nutzbar.

| ID | Paket | Abgrenzbares Ergebnis |
| --- | --- | --- |
| M3-01 | Statische Bewertung | Material und einfache Positionsmerkmale mit konsistenter Perspektive |
| M3-02 | Negamax/Alpha-Beta | Korrekte terminale Werte, Mattabstand, begrenzte Suche |
| M3-03 | Iterative Vertiefung | Verwertbarer Bestmove nach vollständig abgeschlossenen Tiefen |
| M3-04 | SearchController | Unabhängiger Suchzustand, Stop-Token, Laufzeit-/Tiefenlimit |
| M3-05 | UCI-Host | `uci`, `isready`, `ucinewgame`, `position`, `go`, `stop`, `quit`, `bestmove` |
| M3-06 | Protokollschutz | stdout nur UCI, stderr-Diagnose, genau ein Bestmove je aktiver Suche |
| M3-07 | Diagnosedaten (Grundlage) | Knoten/Zeit/Tiefe nach Möglichkeit ohne zwingende Trace-Infrastruktur |

**Abnahme-Gate:** Die Engine produziert nur legale Züge, kann ohne GUI gestartet werden, stoppt kontrolliert, antwortet UCI-konform und besteht Protokoll-/Negativtests. Ein externes UCI-Programm kann sie erfolgreich steuern.

**Nicht als Voraussetzung:** Stockfish-ähnliche Spielstärke, Multi-PV, Mehrthread-Suche, Opening Book, umfangreiches Laboratory.

## 8. M4 – UCI-Client, Zwei-Prozess-Match und Selbstspiel

**Ziel:** Vollautomatische, unabhängig geprüfte Matches mit zwei getrennten UCI-Prozessen.

| ID | Paket | Abgrenzbares Ergebnis |
| --- | --- | --- |
| M4-01 | Plattform-Prozessadapter | Kindprozesse, Pipes, Exit, Stop, Timeouts, Windows/Linux |
| M4-02 | UCI-Client | Handshake, Readiness, Position, Go, Info, Bestmove, Stop |
| M4-03 | Schiedsrichter | Autoritative `Game`-Instanz, erneute Zuglegalität, Spielerwechsel |
| M4-04 | Matchlimits | Feste Tiefe/feste Zeit, technischer Watchdog, ggf. Schachuhr gemäß explizitem Reglement |
| M4-05 | Fehlersicherheit | Illegaler Bestmove, unerwartetes Ende, Protokollfehler, ausbleibende Antworten |
| M4-06 | Selbstspiel | Zweimal dieselbe ausführbare SASD-Chess-Binärdatei in getrennten Prozessen |
| M4-07 | Fremdengine-Partie | SASD-Chess gegen mindestens eine unabhängige UCI-konforme Engine |
| M4-08 | Ergebnis/PGN | Vollständige Partie, Kennungen, Zeitkontrolle, korrekter Ergebniseintrag |
| M4-09 | Headless CLI | Kein Desktop für Engine-/Match-Tests erforderlich |

**Abnahme-Gate:** Zwei unabhängige Engines spielen bis zu einem dokumentierten regulären Partieende. Selbstspiel und Fremdengine-Match werden nachgewiesen; unzulässige Bestmoves werden **nie** übernommen; abgestürzte/hängende Engines lassen keine Prozesse hängen. Unterschiede zwischen Schachuhr-Flagge und technischem Timeout sind getestet. Die vollständige Partie wird als PGN exportiert.

**Soll-Ausbau:** Mehrteilige Vergleichsserien, Farbwechsel und Ergebnisstatistik; kein Blocker des ersten einzelnen Engine-Matches, außer explizit umpriorisiert.

## 9. M5 – Desktop-Anwendung und Spielbarkeit

**Ziel:** Das Schachprogramm ist durch Menschen bedienbar, ohne technische Interna zu verstehen.

| ID | Paket | Abgrenzbares Ergebnis |
| --- | --- | --- |
| M5-01 | GUI-Entscheidung | Dokumentierter Spike/ADR zur konkreten Oberfläche, keine Rückwirkung auf Core |
| M5-02 | Brettbedienung | 8×8-Brett, Figuren, Start-/Zielfeld, legale Hinweise, Umwandlung |
| M5-03 | Spielmodi | Mensch/Mensch und Mensch/klassische Engine |
| M5-04 | Verlauf und Ergebnis | Zugliste, Zugrecht, Schach/Matt/Patt/Remis, neue Partie, Rücknahme |
| M5-05 | Dateien | FEN-Laden/-Ausgabe und PGN-Export |
| M5-06 | Nichtblockierende Engine | SearchWorker, Generation-Prüfung, kein UI-Freeze |
| M5-07 | Grundbedienbarkeit | Kein Drag-and-drop-Zwang, klarer Fokus und verständliche Statusmeldungen |

**Abnahme-Gate:** Unter Windows und Linux dokumentierte vollständige Partien in allen Pflicht-Spielmodi, inklusive Promotion, Undo und Spielende. Suchabbrüche und Dateifehler erzeugen keine falschen Spielstände.

## 10. M6 – Stabilisierung und Version 1.0

**Ziel:** Nachgewiesenes, dokumentiertes, benutzbares Release.

| ID | Paket | Abgrenzbares Ergebnis |
| --- | --- | --- |
| M6-01 | Abnahmekatalog | **LH-A-01 bis LH-A-17** mit reproduzierbaren Nachweisen |
| M6-02 | CI/Portabilität | Windows/Linux, Debug/Release, automatisierte Tests |
| M6-03 | Anwender-/Builddokumentation | Voraussetzungen, Start, Formate, UCI, Match-CLI, bekannte Grenzen |
| M6-04 | Release- und Lizenzprüfung | Verbindliche Open-Source-Lizenz; Rechtmäßigkeit von Code/Assets/Drittbibliotheken |
| M6-05 | Fehler- und Sicherheitsprüfung | Untrusted FEN/PGN/UCI-Eingaben, Kindprozesse, Ressourcen und Shutdown |
| M6-06 | Performance-Baseline | Dokumentierte reproduzierbare Knoten-/Zeit-/Speicherwerte, ohne Elo-Versprechen |
| M6-07 | Release-Artefakte | Konsistente Versionierung, Changelog, Quell-/Binärpakete nach Plattformmatrix |

**Abnahme-Gate:** Alle Muss-Kriterien des Lastenhefts sowie alle zugeordneten Tests erfüllt. Nicht fertiggestellte Soll-Funktionen bleiben sichtbar als offen dokumentiert. Ohne formelle fachliche Freigabe des Umfangs wird der Entwurf nicht als 1.0-Baseline ausgegeben.

## 11. Laboratory-Ausbaustufen nach Version 1.0

SASD-Chess bleibt eine **klassische Schachanwendung**. Die folgenden schachbezogenen Funktionen werden als klar abgegrenzte Erweiterungen entwickelt und **dürfen den Release 1.0 nicht blockieren**.

| Stufe | Funktionen | Test- und Ergebnisgate |
| --- | --- | --- |
| **L1 – Analysieren** | Stellungslabor, Angriffsfelder, Bewertungsaufschlüsselung, Diagnoseprofile Off/Stats, erste Benchmarks | Korrekte FEN-/Bewertungs- und Diagnose-Roundtrips, Statistiken vollständig |
| **L2 – Vergleichen** | Farbwechsel-/Eröffnungs-gepaarte Engine-Serien, Ergebnisstatistik, JSON/CSV-Messdaten, PGN-Partieanalyse nach Import, erweiterte Taktiktests | Kein falscher Spielstärkenachweis; Daten und Ergebnisse reproduzierbar dokumentiert |
| **L3 – Experimentieren** | Experimentdefinitionen, Algorithmus-/Brettrepräsentationsvergleiche, begrenzte Suchbaumtraces, Markdown-/HTML-Berichte | Ablationsmethodik, Trace-Ressourcengrenzen, reproduzierbare Rohdatenverweise und Berichte |

**Ausnahme:** Perft/Divide (LH-F-051) ist bereits Pflicht für M1/M6. Optionale Grundlagen für Diagnostik dürfen früher eingeführt werden, solange sie keine Pflichtmodule von L1–L3 voraussetzen.

## 12. Abhängigkeiten und kritischer Pfad

~~~text
M0 Build/Test
  └─> M1 Schachregeln + Perft/Divide
       └─> M2 Partie + Formate
            ├─> M3 klassische Suche + UCI Host
            │    └─> M4 UCI Client + Schiedsrichter
            │         └─> M5 Desktop-Integration
            └─> M5 Brett + menschliche Spielsteuerung
                       └─> M6 Release-1.0-Gate
                            └─> L1 → L2 → L3
~~~

Die GUI-technische Evaluierung kann parallel zu M1/M2 geschehen. Eine externe Engine kann erst sinnvoll getestet werden, nachdem die Domänen- und UCI-Verträge stabil sind. M4 darf keine zweite Zuglegalität einführen.

## 13. Risiken und offene Entscheidungen

| Risiko / Entscheidung | Betroffen | Gegenmaßnahme |
| --- | --- | --- |
| Vollständige korrekte Remis-/Dead-Position-Regeln | M2/M6 | Referenzfälle, fachlich korrekte Anspruchssemantik, dokumentierte Regelgrenzen |
| GUI-Framework noch offen | M5 | Schlanker GUI-Spike, ADR; Kern/Engine vollständig headless |
| Lizenz/Assets noch offen | M6 | Frühzeitig diskutieren, vor erstem 1.0-Release verbindlich lösen |
| Betriebssystemabhängige Pipes/Shutdown | M4 | Portabler Prozessvertrag, Windows/POSIX-Implementierung, Hänger-/Crash-Tests |
| UCI-Protokoll/Antwortzuordnung | M3/M4 | Pro Sitzung ein aktiver Suchauftrag, Timeout, Mock-Engine, reale UCI-Tests |
| Verfrühte Suchoptimierung | M1–M3 | Korrektheit und Baseline zuerst; Optimierung separat messen |
| Laboratory-Scope wächst unkontrolliert | M6/L1–L3 | Abgeschlossene Zusatzphasen mit ausdrücklicher Nicht-Blockade der 1.0 |
| Spielstärke durch kleine Samples überschätzt | L2 | Farbwechsel, gepaarte Eröffnungen, statistische Unsicherheit |
| Keine belastbaren Release-Nachweise | Alle | Testmatrix, CI, Changelog, Anforderungen IDs und reale Testprotokolle |

## 14. Änderung der Roadmap und Fortschrittsberichte

Für jeden Meilenstein sollten im Repository mindestens **Status, verantwortliche Änderung, Commit/PR, Tests, bekannte Einschränkungen und nächste konkrete Aufgabe** nachvollziehbar sein. Die Roadmap ist kein Ersatz für GitHub-Issues; Issues dürfen einzelne Arbeitspakete enthalten.

Reihenfolge/Umfang der Muss-Anforderungen wird nur bei dokumentierter fachlicher Entscheidung geändert. In der Änderungsbeschreibung sind betroffene `LH-F-*`-/`LH-A-*`-Kennungen sowie Konsequenzen für [Pflichtenheft](docs/Pflichtenheft.md), [Architektur](docs/Architektur.md) und [Testplan](docs/TESTPLAN.md) zu nennen.

**Nächster Entwicklungsauftrag:** **M0-01 bis M0-03**: minimaler C++20-/CMake-/CTest-Build mit Linux-/Windows-CI. Erst wenn diese Grundlage überprüft ist, M1-Zuglogik beginnen.

## 15. Versionshistorie

| Version | Datum | Änderung |
| --- | --- | --- |
| 0.1 | 27.10.2026 | Vollständige, überprüfbare M0–M6-Roadmap, Gates, Risiken, Requirements-Verknüpfungen und gestufter Laboratory-Ausbau L1–L3. |
