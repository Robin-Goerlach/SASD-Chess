# AGENTS.md – Entwicklungsanweisungen für SASD-Chess

> **Dokument:** DEV-AGENTS-001 · **Version:** 0.1 · **Stand:** 27.10.2026  
> **Geltungsbereich:** Gesamtes Repository `Robin-Goerlach/SASD-Chess` einschließlich Unterverzeichnissen.  
> **Status:** Projektweite Leitlinien für menschliche und automatisierte Entwicklungsbeiträge.  
> **Hinweis:** Die Datei **AGENTS.md** ist die kanonische Agentenanweisung. [AGENT.md](AGENT.md) dient ausschließlich als Verweis.

## 1. Auftrag und unveränderliche Projektgrenze

Entwickle **SASD-Chess** als eigenständiges, klassisches Schachprogramm in modernem **C++**. Der verbindliche Umfang umfasst:

- Vollständiges Regelwerk für **orthodoxes Standardschach**, legale Zugerzeugung und korrekte Partieende-Regeln.
- Spielverwaltung, FEN, SAN und PGN.
- Eigene klassische Strategie-Engine auf Basis handprogrammierter Bewertungsfunktionen und Suche (Negamax/Alpha-Beta).
- Standardisierte **UCI-Engine-Schnittstelle**.
- **Unabhängigen UCI-Match-Controller**, der zwei **separate Prozesse** gegeneinander spielen lässt – auch zweimal dieselbe eigene Engine.
- Desktop-Anwendung für die vorgesehenen menschlichen Spielmodi.
- Regeltests, Perft/Divide, CI sowie später schachbezogene Analyse- und Laboratory-Funktionen.

**Nicht entwickeln:** trainierte/neuronale Schachsysteme, allgemeine KI-Workflows, Cloud-Plattform, Multiplayer-Server oder eine generische Spiele-Engine. Ein offenes Schachprotokoll ist kein Anlass, ein anderes Produkt oder Repository in diesen Projektumfang einzubeziehen.

**Arbeitsprinzip:** Funktionierende, gut kommentierte und korrekt getestete Lösungen sind wichtiger als frühzeitige Mikrooptimierung.

## 2. Normative Projektquellen und Rangfolge

1. [Lastenheft](docs/Lastenheft.md): **Was** ist fachlich gefordert? Die `LH-*`-IDs, Prioritäten und Abnahmekriterien sind hier verankert.
2. [Pflichtenheft](docs/Pflichtenheft.md): **Wie** wird die Anforderung technisch umgesetzt?
3. [Architektur](docs/Architektur.md): Modulgrenzen, Zustände, Kontrollfluss, Sicherheits-/Fehlerkonzept.
4. [Roadmap](ROADMAP.md): Reihenfolge, Abhängigkeiten, Arbeitspakete und Meilenstein-Gates.
5. [Testplan](docs/TESTPLAN.md): Konkrete Nachweise, Testkategorien und Testdatenregeln.
6. [ADR-Register](docs/adr/README.md): ausdrücklich getroffene technische Entscheidungen.
7. Diese Datei und [CONTRIBUTING.md](CONTRIBUTING.md): Arbeitsregeln.

Bei Widersprüchen: **nicht stillschweigend eine eigene Interpretation wählen**. Den Konflikt benennen, betroffene `LH-*`-IDs dokumentieren und mit einer entsprechenden Änderung/ADR behandeln. Anforderungen dürfen nicht durch Codekommentare, spontane README-Änderungen oder Tests allein herabgestuft werden.

## 3. Vor jeder Änderung: Istzustand feststellen

1. Repository, aktiven Branch, `HEAD`, Arbeitsbaum und relevante Projektdokumente lesen.
2. Vorhandene Implementierung und CMake-Targets **prüfen**, nicht von der Zielstruktur in der Architektur auf ihre Existenz schließen.
3. Die kleinste abgrenzbare Aufgabe des aktuellen Meilensteins wählen; bisherige Arbeit nicht überschreiben.
4. Relevante Testfälle und notwendige Grenzfälle vor der Änderung notieren.
5. Nutzeränderungen, uncommittete Dateien, Branch-Konventionen und Repository-Schutzregeln respektieren.
6. **Keine Änderungen an anderen Repositories** unter dem Vorwand gemeinsamer Schachfunktionalität.

Keine falschen Aussagen über vorhandene Funktionen, Testläufe, CI-Status oder fertiggestellte Meilensteine machen. Wenn Tests mangels Toolchain/Plattform nicht ausführbar waren, den fehlenden Nachweis konkret benennen.

## 4. Architekturregeln – verbindlich

| Modul / Verantwortung | Darf | Darf ausdrücklich nicht |
| --- | --- | --- |
| **Chess Core** | Zustände, Angriffsfelder, Zuglegalität, make/unmake, Perft/Divide | GUI, UCI, Dateisystem, Netz, OS-Prozesse oder Suche kennen |
| **Game** | Partiehistorie, Wiederholung, Remisansprüche, Ergebnisse, Snapshots | Eigene alternative Zuglegalität implementieren |
| **Formats** | FEN/SAN/PGN/UCI-Zugnotation parsen/serialisieren | Core-Regeln duplizieren oder ungültige Inhalte still reparieren |
| **Search** | Klassische Bewertung, Negamax/Alpha-Beta, Limits, Abbruch | Laufende Game-Instanz oder UI-Widgets direkt ändern |
| **Diagnostics** | Optionale Metriken, begrenzte Ereignisse, Bewertungsterme | Abweichende Stellungsbewertungsformel schaffen |
| **UCI Host** | Eigenen Engine-Prozess auf stdin/stdout steuern | Debugtext außerhalb UCI auf stdout schreiben |
| **UCI Client / Process** | Fremdengine starten, Antworten lesen, Timeouts behandeln | Einen empfangenen Zug ohne Game/Core-Prüfung übernehmen |
| **Match Controller** | Zwei unabhängige Engines, autoritatives Game, Uhren, Fehler, PGN | Engines direkt miteinander verbinden oder Schachregeln verdoppeln |
| **Desktop** | Darstellung, Benutzerbefehle, asynchrone Status-Snapshots | Regel-/Suchalgorithmus selbst implementieren |
| **Laboratory** | Schachbezogene Experimente über Core/Engine/Match | Eigene zweite Engine oder allgemeines Statistikframework voraussetzen |

Die fachliche Baseline ist eine einfache **64-Felder-Repräsentation**, sofern eine konkrete ADR nicht anders entscheidet. Öffentliche Schnittstellen dürfen nicht unnötig von einer speziellen Optimierungsrepräsentation abhängen.

**Besitz und Parallelität:** Engine-Suchen arbeiten auf eigenen Positions-/Historienkopien; eine autoritative Game-Instanz wird nur von ihrer steuernden Komponente mutiert. UI erhält kopierte/immutable Snapshots. Späte Suchresultate müssen gegen Such-ID und Spielgeneration geprüft werden.

**UCI:** Je Engine-Sitzung höchstens **eine** aktive Suche. `stop` absetzen, auf `bestmove` oder kontrolliertes Fehlende warten, erst danach den nächsten Auftrag zulassen. stdout nur UCI; stderr für Diagnostik. Der Match-Schiedsrichter überprüft jeden Bestmove eigenständig.

**C++20 beachten:** Beispielsweise ist `std::expected` erst C++23; bei C++20 einen kleinen dokumentierten Fehler-/Result-Typ oder passende bestehende Alternative verwenden.

## 5. Implementierungs- und Kommentierungsstil

- **C++20**, dokumentierte CMake-Targets, idiomatische Standardbibliothek, möglichst geringe Drittanbieterabhängigkeiten.
- `sasd::chess::...` als Namensraumfamilie, aussagekräftige englische API-/Variablennamen. Kommentare bleiben konsistent in einer durch ADR/Projektkonvention festgelegten Sprache; ohne Festlegung **gut verständliche englische API-Kommentare** bevorzugen.
- **Ausführliche, begründende Kommentare** für öffentliche APIs und komplexe Vorgänge: Vor-/Nachbedingungen, Invarianten, Fehlerszenarien, Besitz, Threadannahmen und Besonderheiten von Schachregeln.
- Nicht ausschließlich `// increment counter` bei einem selbsterklärenden Inkrement schreiben; das **Warum** eines Verfahrens erklären.
- Reine Werttypen und klar beschränkte Zuständigkeiten bevorzugen; RAII, `const`-Korrektheit, sichere Wertebereiche und explizite Error-Paths.
- Keine versteckten veränderbaren globalen Singletons, unbeschränkten Puffersammlungen, unkontrollierten Prozessstarts oder pauschales Ausnahme-Verschlucken.
- Fehlerhafte Eingaben dürfen **keine partielle Zustandsänderung** verursachen.
- Nachvollziehbarer Code vor Tricks; Optimierung nur auf Basis einer Korrektheits- und Messbaseline.
- Kleine verständliche Änderungen bevorzugen. Große Umstrukturierungen nur mit konkreter Begründung, Tests und aktualisierter Architektur.

## 6. Schachliche Pflichtinvarianten

- Der **eigene König** bleibt nach jedem legalen Zug sicher; der gegnerische König wird nicht „geschlagen“.
- Rochade erfordert sämtliche Rechte, passende Turm-/Königstellung und nicht angegriffene Königswegfelder.
- En passant wird einschließlich Aufdeckung eigener Königslinien korrekt behandelt.
- Promotion unterstützt **Dame, Turm, Läufer, Springer**.
- `unmake(make(position, legal_move))` stellt **alle** relevanten Felder, Rechte, Zähler und – soweit verwendet – Hashes wieder her.
- Angreifbare Felder und legale Züge sind unterschiedliche Begriffe; Fesselungen und Schachabwehr werden berücksichtigt.
- Die vollständige Historie wird für Stellungswiederholungen benötigt; **FEN allein** beweist keine dreifache Wiederholung.
- Dreifache Wiederholung und 50-Züge-Regel sind beanspruchbar, fünffache Wiederholung und 75-Züge-Regel automatisch; Matt-/Remisprioritäten korrekt.
- Für **tote Stellungen** nur nachweislich gültige automatische Remisurteile. Unvollständige Heuristiken sind als Grenze zu dokumentieren; volle Regeltreue erfordert nachgewiesene Abdeckung.
- Ein technischer Engine-Timeout ist **nicht automatisch** ein regulärer Schachuhrverlust.
- Schiedsrichterliche Remisentscheidungen für UCI-Matches müssen als explizite Match-Policy beschrieben sein; UCI enthält keinen allgemeinen `claim_draw`-Befehl.

## 7. Tests – bei jeder betroffenen Änderung

| Betroffener Bereich | Erforderlicher Test |
| --- | --- |
| Zugerzeugung / Core | Unit- und Spezialregeltests, Perft/Divide, make/unmake |
| Game / Remis | Historie, Wiederholungen, Anspruch/automatisch, Matt/Patt, Undo |
| FEN/SAN/PGN | Positive/negative Parserfälle, Roundtrips, unabhängige PGN-Lesbarkeit |
| Suche | Nur legale Bestmoves, deterministische feste Tiefe, Stop, terminale Stellung |
| UCI Host | `uci`, `isready`, `position`, `go`, `stop`, `quit`, stdout-Reinheit |
| UCI Client | Handshake, Protokollfehler, lange Zeilen, stderr-Flut, Zeitüberschreitung |
| Match | Zwei separate Prozesse, SASD×SASD, SASD×Fremdengine, Ergebnis/PGN/Shutdown |
| GUI | Mensch/Mensch, Mensch/Engine, Undo, Promotion, Status, kein UI-Hänger |
| Laboratory | Nach `LAB-T-01` bis `LAB-T-10`; nicht implementierte Zukunftsfunktionen blockieren Release 1.0 nicht |

**Perft-Anfangsstellung:** Tiefe 1–5: `20, 400, 8902, 197281, 4865609`; Tiefe 6 (`119060324`) als optionaler längerer Lauf. In Divide muss Summe der Erstzug-Zweige gleich Gesamt-Perft sein. Zusätzlich Referenz-FEN mit Quelle dokumentieren.

**Bauen/Prüfen nach Existenz der M0-Projektdateien:**

~~~sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
ctest --test-dir build --output-on-failure
~~~

Auf Multi-Config-Generatoren, z. B. MSVC, stattdessen zusätzlich `--config Debug` beim Build und CTest setzen. Diese Befehle sind **Soll-Workflow nach M0**, keine Behauptung, dass das Repository bereits heute vollständig baubar ist. Windows/Linux separat nachweisen; Plattformen niemals nur aus einer fremden CI-Vermutung als getestet markieren.

## 8. Arbeitsreihenfolge und Definition of Done

Die Meilensteine folgen [ROADMAP.md](ROADMAP.md):

1. **M0:** CMake, CTest, Windows/Linux-CI, Architektur-/Dokumentationsbasis.
2. **M1:** Core, legale Züge, Undo, Perft/Divide.
3. **M2:** Game, FEN/SAN/PGN, korrekte Partien/Remis.
4. **M3:** Eigene klassische Engine, UCI-Host, Stop/Limits.
5. **M4:** Fremdengine-UCI-Client, zwei getrennte Prozesse, Match/PGN.
6. **M5:** Desktop-Oberfläche und verpflichtende Spielmodi.
7. **M6:** Release-1.0-Abnahme mit `LH-A-01–LH-A-17`.
8. **L1–L3:** Spätere schachbezogene Analyse- und Laboratory-Funktionen, ohne 1.0 zu blockieren.

Eine Aufgabe ist erst abgeschlossen, wenn **Code, Tests, Dokumentation und nachvollziehbarer Commit** vorhanden sind. Keine neue Plattform/GUI-Technologie ohne kontrollierten Architekturentscheid; keine falschen grünen Tests aus reiner Quelltextbetrachtung ableiten.

## 9. Git und Zusammenarbeit

- Vor Git-Änderungen Branch/Commit/Arbeitsbaum prüfen. Bestehende Nutzeränderungen niemals blind überschreiben.
- Bestehende Repository-Branchregeln und Schutzmaßnahmen haben Vorrang. **Nicht automatisch fremde Branches löschen oder PRs mergen**, außer dies wurde ausdrücklich erlaubt und alle Regeln sind erfüllt.
- Commits möglichst klein und thematisch klar halten, z. B. `feat(core): ...`, `test(perft): ...`, `fix(uci): ...`, `docs: ...`.
- Wenn Tests laufen können: sie vor Übergabe ausführen; wenn nicht: die **konkrete Ursache** nennen.
- In Abschlussberichten kurz und überprüfbar angeben: veränderte Dateien, implementierte Funktionen, Testbefehle/Ergebnisse, Git-Commit, verbleibende Probleme und nächsten kleinsten Schritt.
- Dokumentationsversionen und tatsächlich vorhandene Features müssen zueinander passen.
- Eine Datei nur deshalb anzulegen, weil sie im Architektur-Zielbaum erwähnt ist, ist **kein** Entwicklungsfortschritt.

## 10. Sicherheit und externe Engines

Die eigene Engine ist ein normaler lokaler Prozess. **Fremde UCI-Engines sind ausführbarer Code** und daher nur aus bewusst gewählten, vertrauenswürdigen Quellen zu starten; Prozessisolation ersetzt keine Sandbox. Pfade und Argumente ohne Shell-Injektion verarbeiten. stdout/stderr kontinuierlich lesen, Puffer begrenzen und hängende Prozesse kontrolliert terminieren.

Keine Geheimnisse, persönlichen Pfade oder sensiblen Systeminformationen unnötig in Logs, Matchdaten oder Forschungsberichte schreiben. Keine Online- oder Telemetriepflicht schaffen.

## 11. Wissenschaftliches Laboratory und ehrliche Messmethodik

Perft beweist nicht Spielstärke, Nodes/s beweist nicht Zugqualität, einzelne Matchsiege beweisen keine Überlegenheit. Vergleiche dokumentieren Engine-/Git-Versionen, Konfiguration, FEN-Korpus, Compiler, Hardware und methodisch geeignete Wiederholungen. **Statistische Unsicherheit** ist offen auszuweisen. Tracing bleibt begrenzt und kann Performance-Messungen verzerren.

Das Laboratory **benutzt** den geprüften Schachkern und den Match-Schiedsrichter; es führt keine zweite Bewertungs- oder Legalitätsimplementierung ein. Erweiterungen werden anhand der priorisierten `LH-F-042–056` und der `LAB-T-*`-Tests aus dem Pflichtenheft umgesetzt.

## 12. Entscheidungen, die noch nicht getroffen sind

Folgende Themen sind **offen** und dürfen nicht als beschlossen dargestellt werden:

- Konkretes GUI-Framework (SASD-UI-Toolkit ist eine mögliche, **nicht verpflichtende** Option).
- Endgültiges C++-Testframework.
- Open-Source-Lizenz des Projekts und der Figurenassets.
- Präzise erste Match-CLI-/Konfigurationssyntax und Turnier-/Uhren-Policy.
- Zusätzliche Zielplattformen jenseits Windows/Linux.
- Einheitliche Kommentarsprache, soweit noch nicht separat beschlossen.

Vor weitreichenden Entscheidungen [ADR](docs/adr/README.md) anlegen und Alternativen/Tests/Risiken erläutern. Nicht sämtliche Optionen bereits während M0 in ein übergroßes Framework einbauen.

---

**Leitfrage vor jedem Commit:** Macht diese Änderung **SASD-Chess** nachweislich korrekter, verständlicher, robuster, testbarer oder nutzbarer – und bleiben der klassische Schachkern und seine Architekturgrenzen gewahrt?
