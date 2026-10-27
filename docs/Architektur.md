# SASD-Chess – Softwarearchitektur

> **Dokument:** ARCH-001 · **Version:** 0.2 · **Stand:** 27.10.2026  
> **Status:** Architekturentwurf zur technischen und fachlichen Freigabe; keine Aussage über bereits vorhandenen Implementierungsstand  
> **Repository:** https://github.com/Robin-Goerlach/SASD-Chess  
> **Grundlagen:** [Lastenheft v0.4](Lastenheft.md) · [Pflichtenheft v0.4](Pflichtenheft.md)  
> **Geltungsbereich:** Klassisches Schachprogramm in C++, Standardschach, Desktop-Oberfläche, eigene klassische Such-Engine, UCI, Engine-Matches, schachbezogenes Laboratory.  
> **Ausdrücklich ausgeschlossen:** trainierte Schachmodelle und der Aufbau einer allgemeinen Forschungs-/Brettspielplattform.

## 1. Zweck, Status und Architekturgrundsätze

Dieses Dokument definiert eine **umsetzbare Zielarchitektur** für SASD-Chess. Es verbindet die fachlichen Anforderungen mit technischen Modulgrenzen, Schnittstellen, Zustandsregeln, Parallelitäts- und Fehlerkonzepten, Qualitätssicherung und einer schrittweisen Realisierungsstrategie. Es unterscheidet bewusst zwischen **verbindlichen Architekturprinzipien** und **noch zu bestätigenden Detailentscheidungen**.

**Leitprinzip:** Schachregeln sind die gemeinsame Wahrheit des Programms. Oberfläche, Suche, UCI, Partieverwaltung, Match-Controller und wissenschaftliche Messwerkzeuge dürfen keine abweichenden eigenen Regelimplementierungen schaffen.

### 1.1 Architekturprinzipien

1. **Korrektheit vor Performance.** Zuerst eine einfache, verifizierbare Regelimplementierung; optimierte Implementierungen müssen exakt denselben fachlichen Tests genügen.
2. **Eine Verantwortung pro Komponente.** Brett-/Zuglogik, Partie, Suche, Textprotokolle, Prozesssteuerung, UI und Messdaten bleiben getrennt.
3. **Abhängigkeiten nur nach innen.** Der Chess Core kennt weder GUI noch Betriebssystem, Dateisystem, UCI, Schachuhr, Forschungsläufe oder Engine.
4. **Gemeinsame fachliche Identität.** Alle Nutzer der Schachregeln verwenden dieselben Typen und Legalitätsentscheidungen. Ein Engine-„Bestmove“ wird an einer Systemgrenze erneut überprüft.
5. **Explizite Zustände und Lebenszyklen.** Suchaufträge, Prozesse, Partien, Uhren und Dateitransaktionen besitzen klar definierte Zustände und Übergänge.
6. **Headless first für den Kern.** Core, Engine, UCI, Match-Controller, Perft und Benchmarks lassen sich ohne grafische Oberfläche bauen und ausführen.
7. **Portabilität durch Adapter.** Betriebssystemabhängige Prozess-, Zeit- und UI-Funktionen werden gekapselt; Windows und Linux sind die verpflichtenden Zielplattformen.
8. **Nachvollziehbarkeit.** Öffentliche APIs sind dokumentiert, komplexe Algorithmen begründet kommentiert, Annahmen explizit und Entscheidungen in ADRs dokumentiert.
9. **Verlässliche Interoperabilität.** Standardisierte Formate (FEN, SAN, PGN, UCI) bilden externe Grenzen; interne C++-Objekte werden nicht als Dateiformat missbraucht.
10. **Beobachtbarkeit ohne Verfälschung.** Wissenschaftliche Diagnose ist optional, messbar begrenzt und darf die Normalfunktion nicht verändern.
11. **Inkrementelle Lieferbarkeit.** Nach jedem Meilenstein entsteht ein baubarer, getesteter und fachlich abgrenzbarer Stand; geplante Laboratory-Extras blockieren die Schachversion 1.0 nicht.

### 1.2 Qualitätsziele in Prioritätsreihenfolge

| Rang | Attribut | Architekturantwort | Nachweis |
| --- | --- | --- | --- |
| 1 | Regelkorrektheit | Ein Chess Core, unveränderliche Verträge, reversible Züge | Unit-Tests, Perft/Divide, Sonderregeltests |
| 2 | Wartbarkeit | Kleine Bibliotheken, eine Abhängigkeitsrichtung, ADRs | Review der Modulgrenzen und CMake-Targets |
| 3 | Robustheit | Transaktionale Zustandsübernahme, typisierte Fehler, Timeouts | Negativtests, Prozessabbruch-/UCI-Tests |
| 4 | Testbarkeit | Headless-Dienste, Test-Doubles, deterministische Tiefenlimits | CTest und automatisierte CI |
| 5 | Portabilität | C++20, CMake, OS-Adapter | Windows-/Linux-Buildmatrix |
| 6 | Interoperabilität | Eigenes UCI-Engine-Programm und separater UCI-Client | UCI-Konformitäts- und Matchtests |
| 7 | Bedienbarkeit | UI als Präsentationsadapter, klare Spielaktionen | End-to-End-Tests |
| 8 | Messbarkeit | Versionierte Benchmarks, deaktivierbare Diagnostik | Reproduzierbare Testprotokolle |
| 9 | Performance | Optimieren anhand von Profilen unter gleichbleibenden Verträgen | Vorher/Nachher-Benchmarks |

**Nicht-Leistungszusage:** Keine konkrete Elo-Zahl oder Mindestzahl durchsuchter Knoten wird vor Implementierung und Messung versprochen.

## 2. Systemkontext und externe Schnittstellen

~~~text
                           Menschen
                              |
                 +------------v------------+
                 | SASD-Chess Desktop      |
                 | Brett, Menüs, Analyse   |
                 +------------+------------+
                              |
                 +------------v------------+
                 | SASD-Chess Anwendung    |
                 | Spiel / Engine / Match  |
                 | Analyse / Laboratory    |
                 +------+-----------+------+
                        |           |
       Dateien: FEN, PGN, JSON/CSV |      externe UCI-Oberfläche
                        |           |               |
                  +-----v----+ +----v------+        |
                  | Formate  | | Match     |        |
                  +----------+ | UCI-Client|        |
                               +-----+-----+        |
                                     |              |
                         zwei unabhängige Prozesse  |
                              Engine A / B          |
                                                    |
                               SASD-Chess UCI-Engine+
~~~

**Systemgrenzen:**

- **Desktop:** Anzeige und menschliche Eingaben; kennt keine eigene Zuglegalitätslogik.
- **Eigenes UCI-Programm:** Wird von fremden UCI-Clients gestartet und berechnet Züge ohne GUI.
- **Match-Controller:** Ist selbst UCI-Client für zwei *separate Kindprozesse* und deren Schiedsrichter; auch zwei identische SASD-Chess-Binärdateien sind möglich.
- **Datei- und Forschungsformate:** Import/Export wird an expliziten Parser-/Serializer-Grenzen durchgeführt.
- **Externe Engines:** Sind ausführbarer Fremdcode; die Prozessgrenze ist keine Sicherheits-Sandbox.

**Kein Netzwerkdienst für V1.0.** Lokale Pipe-Kommunikation und lokale Dateien sind ausreichend. Spätere Netzwerkprotokolle sind kein Grund, die Fachkomponenten heute zu verallgemeinern.

## 3. Bausteinsicht und Dependency-Regeln

### 3.1 Logische Module

| Modul / vorgeschlagenes CMake-Target | Verantwortung | Erlaubte direkte Abhängigkeiten |
| --- | --- | --- |
| **chess-core** | Brett, Stellung, Figuren, Züge, Angriffserkennung, Legalität, reversible Zustandsänderung, Perft | C++-Standardbibliothek |
| **chess-game** | Partiehistorie, Wiederholungen, Remisansprüche, Spielende, Ergebnis, Zuglisten | chess-core |
| **chess-formats** | FEN, SAN, PGN, UCI-Zugnotation und Validierung | chess-core, chess-game |
| **chess-search** | Klassische Stellungsbewertung, Negamax/Alpha-Beta, Suchgrenzen und Suchresultat | chess-core, chess-game, optional chess-diagnostics-API |
| **chess-diagnostics** | Schmale Diagnoseverträge, optionale Zähler und begrenzte Traces | Standardbibliothek; keine Abhängigkeit von chess-search |
| **chess-uci-protocol** | UCI-Zeilenparser, Kommandos, Antworttypen und Formatierung | möglichst nur Standardbibliothek, chess-formats sofern erforderlich |
| **chess-uci-host** | Eigenen UCI-Prozess betreiben, Suchcontroller und stdout/stderr koordinieren | chess-uci-protocol, chess-search, chess-game, chess-formats |
| **chess-process** | OS-unabhängige Kindprozesse, Pipes, Exit-/Timeout-Behandlung | Standardbibliothek, OS-Adapter |
| **chess-uci-client** | Fremdengine-Handshake, Konfiguration, Suchanfrage, Bestmove-Empfang | chess-process, chess-uci-protocol |
| **chess-match** | Unabhängiger Schiedsrichter: Spielstand, Uhr, Engines, Fehler, Ergebnisse | chess-game, chess-formats, chess-uci-client |
| **chess-application** | Anwendungsfälle, Kommandos, Spielmodi, Engine-Sitzungen, UI-Snapshots | chess-game, chess-formats, chess-search, chess-match |
| **chess-benchmark** | Perft-/Such-Benchmarks, Datensatzläufe, strukturierte Messung | chess-core, chess-search, chess-diagnostics |
| **chess-laboratory** (später) | Experimente, Vergleiche, Berichte und aggregierte Ergebnisse | chess-benchmark, chess-match, chess-formats |
| **chess-desktop** | Konkretes Desktop-Frontend und Plattformbindung | chess-application, optionale Analyse-/Lab-Anwendungsfälle |

**Unterschied zwischen fachlichem Kern und Adapter:** Die konkrete Aufteilung innerhalb eines Moduls darf geändert werden, solange die Verantwortlichkeiten, die Abhängigkeitsrichtung und die nachprüfbaren Schnittstellen erhalten bleiben.

### 3.2 Abhängigkeitsgraph

~~~text
C++ Standardbibliothek
         |
     chess-core
       |   |
       |   +--> chess-game
       |             |
       +-------------+------> chess-search ---> chess-uci-host ---> UCI-Engine-EXE
       |             |               |
       +-------------+----> chess-formats
       |                            |
       +--> chess-diagnostics ------+----> chess-benchmark
                                               |
  chess-process --> chess-uci-client           +--> chess-laboratory
                        |                           ^   |
                        +--> chess-match -----------+   |
                                  |                     |
                                  +--> chess-application<+
                                            |
                                        chess-desktop
~~~

*Das Diagramm zeigt eine mögliche Richtung; die Abhängigkeitspfeile werden durch CMake-Zielverknüpfungen überprüft. Eine abstrakte Vertragsbibliothek darf nur eingeführt werden, wenn eine reale zyklische oder anderweitig unauflösbare Abhängigkeit entsteht. Nicht jedes Interface braucht ein eigenes Projekt.*

### 3.3 Architekturregeln für Reviews

- **Verboten:** includes aus Desktop/OS/UCI in chess-core, versteckte globale Singleton-Partie, alternative Regelprüfung im Match-Controller, direkter Dateizugriff aus Suche, Netzwerkzugriff aus Core.
- **Verboten:** ein Engine-Objekt, dessen interner Zustand direkt und ungeprüft den Match-Zustand überschreibt.
- **Erlaubt:** kleiner interner Suchzustand, optimierte Brettrepräsentation hinter einem stabilen Fachvertrag, Verwendung derselben Core-Routinen in Tests und Match.
- **Für V1.0 bevorzugt:** Bibliotheken in einem CMake-Monorepo; keine Plug-in-ABI, kein RPC-Framework und keine künstlich aufgeteilten Services.


### 3.4 Verbindliche technische Verträge an Modulgrenzen

**Einheitliches Zustandsrecht:** Der `chess-core` entscheidet, welche Einzelzüge legal sind; `chess-game` entscheidet unter Einbeziehung der Historie über beanspruchbare und automatische Remisfolgen sowie den Partieabschluss. `chess-match` entscheidet nur über die **Ausführung des Wettbewerbs** (Spielerwechsel, Uhren, technische Fehler, Partieergebnis gemäß Reglement), nicht über neue Schachregeln. Dadurch existiert keine zweite, sich abweichend entwickelnde Zuggenerierung.

| Grenze | Eingaben | Ergebnis | Verbote/Invarianten |
| --- | --- | --- | --- |
| Core: Legalität | Gültige `Position`, vorgeschlagener `Move` | Akzeptierter legaler Zug oder typisierter Fehler | Bei Fehler keine Zustandsänderung |
| Core: Suchoperation | Mutable *lokale* `Position` + nachweislich legaler Zug | `UndoState`, danach vollständige Wiederherstellung | Keine fremd geteilte mutable Stellung |
| Game: Spieleraktion | `Game`, legaler Zug oder Remisanspruch | Neuer `GameSnapshot`, Ereignisse/Ergebnis | Historie und aktueller Zustand sind atomar konsistent |
| Search: Suche | Positions-/Historien-Snapshot, Limits, Stop-Token, Diagnoseprofil | `SearchResult` mit legalem Zug oder terminalem Zustand | Darf autoritatives `Game` nicht mutieren |
| Format: FEN/SAN/PGN | Fremdtext bzw. Domänensnapshots | Validiertes Domänenobjekt oder Serialisierung | Kein partielles Überschreiben laufender Spiele |
| UCI Host: Protokoll | Zeilenstrom + Sitzung | Serialisierte UCI-Antworten | Kein unformatierter stdout-Text |
| UCI Client: Engine | Prozesspfad/Argumente + UCI-Commands | Asynchrone Antworten oder technischer Fehler | Kein Vertrauen in Zuglegalität des Gegners |
| Match: Zugtransaktion | Aktuelle autoritative Partie + `bestmove` | Akzeptierter Zug/Ergebnis/technischer Abbruch | Kein Zug ohne erneute Legalitätsprüfung |
| Laboratory: Versuch | Unveränderliche Versuchskonfiguration + Datenkorpus | Versionierte Rohdaten und Metadaten | Kein eigener Regelkern; keine stillschweigend geänderten Suchparameter |

**Fehlervertrag (C++20):** An öffentlichen Domänen- und Formatgrenzen werden explizite Ergebnistypen (z. B. `Result<T, Error>` als projektspezifisch kleiner Typ, `std::variant` oder gleichwertig) verwendet; `std::expected` darf **nicht** ungeprüft als C++20-Standardfeature vorausgesetzt werden. Ausnahmen können in Implementierungsdetails zulässig sein, dürfen aber keine unkontrollierten Prozess-/UI-Grenzen überqueren.

**Besitzvertrag:** `Game` besitzt seine aktuellen Positionen und Historiendaten; `SearchTask` besitzt eine unabhängige Stellungskopie samt benötigtem History-Kontext; ein `MatchController` besitzt zwei `UciSession`-Objekte und deren Kindprozess-Ressourcen. GUI- und Laboratory-Zugriffe nutzen Snapshots oder ausdrücklich synchronisierte read-only-Sichten, niemals rohe mutable Referenzen.

**Abhängigkeitsregel:** Die statischen Diagnosetypen werden in einem kleinen, eigenständig baubaren Vertrag bereitgestellt. `chess-search` darf diesen Vertrag aufrufen; `chess-diagnostics` darf dagegen **nicht** die konkrete Suche kennen. Eine optionale Diagnoseimplementierung wird bei der Komposition angeschlossen. Ein CMake-Zyklus zwischen `chess-search` und `chess-diagnostics` ist unzulässig.

## 4. Schachdomäne: Zustände und Invarianten

### 4.1 Drei strikt getrennte Zustandsbegriffe

1. **Position** beschreibt *eine* Stellung: 64 Felder, am Zug befindliche Seite, Rochaderechte, En-passant-Status, Halbzugzähler, Vollzugnummer. Sie enthält noch **keine** vollständige Partiehistorie.
2. **Game** beschreibt eine Partie: Ausgangsposition, aktuelle Position, Folge gespielter Züge, historische Zustände, Wiederholungsinformation, Ergebnis und beanspruchbare Remisrechte.
3. **SearchState** beschreibt einen *temporären* Engine-Suchzweig, der Position und für Suchentscheidungen notwendige historische Merkmale enthält; er darf eine laufende Game-Instanz nicht verändern.

Dieser Unterschied ist wesentlich: Aus einer isolierten FEN-Stellung lässt sich eine dreifache Wiederholung ohne Historie nicht zuverlässig ableiten. Suchbäume benötigen zudem einen eigenen Undo-/History-Stack.

### 4.2 Fachliche Typen (konzeptionell)

~~~cpp
enum class Color { White, Black };
enum class PieceType { Pawn, Knight, Bishop, Rook, Queen, King };
struct Square;       // gültige Felder 0..63, ungültige Werte werden abgewiesen
struct Piece;        // Farbe + Figurentyp bzw. leeres Feld
struct Move;         // Start, Ziel, optionale Umwandlung, interne Move-Art
struct Position;     // Brett und alle stellungsbezogenen Rechte/Zähler
struct UndoState;    // Information zur vollständigen Rücknahme eines Zuges
struct GameResult;   // laufend, Sieg/Remis, fachlicher Grund
struct GameSnapshot; // kopierbare, für UI/Adapter sichere Sicht
~~~

Die Skizze legt keine öffentlichen Header oder ABI fest; sie erläutert die benötigten Verträge.

### 4.3 Unveränderliche Invarianten

- Jeder erlaubte Zug ist legal für die jeweils aktive Farbe und lässt den eigenen König nicht im Schach zurück.
- Die öffentliche Spielaktion prüft vor der Zustandsänderung und verändert bei Fehlern **nichts**.
- Für gültige Ausgangszustände gilt: **unmake(make(Position, Move)) = ursprüngliche Position**, einschließlich Zählern, Rechten und gegebenenfalls Stellungs-Hash.
- Ein Schachbrett hat niemals „geschlagene Könige“; bei Matt wird die Partie beendet.
- Bei Bauernumwandlung sind **Dame, Turm, Läufer, Springer** möglich; nicht der König oder ein neuer Bauer.
- Engine, GUI, UCI-Client und Schiedsrichter nutzen denselben Generator legaler Züge.
- Formatparser können ungültige Eingaben abweisen, ohne bereits bestehenden Spielzustand zu beschädigen.

### 4.4 Brettrepräsentation und spätere Optimierung

**Baseline:** 64-Felder-Array mit klaren Typen. Dadurch sind Regeln, Debugging und Perft zunächst leicht nachvollziehbar. Ein optimiertes Bitboard-Modell darf später intern ersetzt oder alternativ implementiert werden; *öffentliche* Anwendungs- und UCI-Schnittstellen dürfen dadurch nicht geändert werden müssen.

**Bewusst keine Premature Optimization:** Magic Bitboards, SIMD, spezielle Cache-Layouts oder handoptimierte Lookup-Tabellen erst nach Korrektheits- und Benchmark-Baseline.

### 4.5 Zuggenerator und Reversibilität

~~~text
Position + aktive Farbe
        |
  Pseudo-legaler Zugvorrat
        |
  Kandidat probeweise make()
        |
  König der ziehenden Seite danach angegriffen?
       / \
      Ja  Nein
      |     |
   verwerfen  legal übernehmen
        \   /
        unmake()  (exakt vollständig)
~~~

Sonderfälle: kurze/lange Rochade einschließlich angegriffener Durchgangsfelder, En passant auch bei entstehender Linienöffnung, Doppel-/Fesselungsschach, Bauernumwandlung, Matt-/Patt-Erkennung, Abweisung eines Königsschlags. Das **Angriffsfeld-Konzept** ist von legalen Zügen zu unterscheiden: Ein Feld kann angegriffen sein, selbst wenn eine Figur wegen eigener Königsfesselung nicht entsprechend ziehen darf.

**Perft/Divide:** Perft zählt rekursiv legale Zugfolgen, Divide die legalen Ausgangszüge mit deren Teilzahlen. Tiefe 0 ergibt 1 Blatt. Für die Startstellung ergeben sich bei Tiefe 1–6 die Werte 20 / 400 / 8.902 / 197.281 / 4.865.609 / 119.060.324. Sonderstellungen werden versioniert. Divide-Teilsummen müssen Perft ergeben.

### 4.6 Validierung externer Stellungen

Zwei Eingangsmodi sind sinnvoll:

- **Spielbare Stellung (strict):** FEN-Syntax und regelrelevante Konsistenz werden geprüft; Position darf in Game/Engine verwendet werden.
- **Stellungseditor-Entwurf:** Unvollständige Anordnung darf vorläufig sichtbar sein, aber nicht als legale Partie gestartet oder an die Engine übergeben werden.

FEN-Parser und Validator werden getrennt. Ein standardkonformes En-passant-Feld kann nach einem Doppelschritt auch dann angegeben werden, wenn kein EP-Schlag legal ist; die *Wiederholungsäquivalenz* berücksichtigt dagegen nur tatsächlich verfügbare Möglichkeiten. Widersprüche werden eindeutig als Fehler erklärt; ein großzügiger Importmodus darf niemals stillschweigend eine ungültige Stellung „reparieren“.

## 5. Spielverwaltung, Zuggeschichte, Regeln und Uhr

### 5.1 Game als alleiniger autoritativer Spielstand

Eine Game-Instanz verwaltet die Folge *akzeptierter* Züge. Eine Spielaktion wird atomar abgewickelt:

~~~text
Benutzereingabe / Engine-UCI-Zug
     -> Parsen / syntaktische Validierung
     -> Legalität an aktueller Game-Position
     -> Make + Historie + Wiederholungsschlüssel
     -> Ergebnis-/Remisprüfung
     -> unveränderlichen Snapshot veröffentlichen
     -> erst danach UI/Messung/PGN informieren
~~~

**Ereignisreihenfolge ist entscheidend:** Bei einem Fehler darf keine UI-Animation, kein PGN-Eintrag und keine Uhrumschaltung einen nicht gespielten Zug als gültig erscheinen lassen. Technische Matchfehler bekommen einen separaten Ergebnisgrund.

### 5.2 Rücknahme und Verlauf

Die Partiehistorie enthält mindestens vorherige Zustandsmerkmale, die genaue Zugfolge sowie Wiederholungsinformationen. Ein Benutzer-Undo stellt den Zustand inklusive Bauern-/Schlagzähler, Rochaderechten und En-passant-Möglichkeit wieder her. Der Controller stoppt vorher eventuell laufende Berechnungen und verwirft Ergebnisse veralteter Stellungen.

PGN ist ein **Export-/Austauschformat**, nicht der alleinige interne Undo-Stack.

### 5.3 Remis und Spielende

Fachliche Resultate: Schachmatt, Patt, tote Stellung, automatische fünffache Wiederholung, automatische 75-Züge-Regel (Matt am letzten Zug hat Vorrang), korrekt beanspruchbare dreifache Wiederholung und 50-Züge-Regel sowie Aufgabe/Remisvereinbarung als Spielereignisse. Je nach Zeitmodus kommen Flaggenfälle hinzu.

**Besondere Architekturentscheidung:** Anspruchsberechtigung (*kann Remis beanspruchen*) und automatisches Partieende (*ist Remis*) sind unterschiedliche Zustände. Die Engine darf eine für den Spieler nur beanspruchbare Remissituation nicht eigenmächtig als offiziell beendete Partie melden. In der Suche kann sie die Option des Beanspruchens gesondert bewerten.

**Tote Stellung:** Die FIDE-Definition ist stärker als eine bloße Materialheuristik. Die erste Implementierung soll ausschließlich **nachweislich sichere** Muster erkennen; Fälle außerhalb dieser Muster dürfen nicht fälschlich als Remis gewertet werden. Bevor vollständige FIDE-Korrektheit behauptet wird, sind Grenzfälle und die verbleibenden Erkennungsgrenzen fachlich freizugeben.

**Wiederholungsschlüssel:** Figuren, Zugrecht, Rochaderechte und tatsächlich mögliche En-passant-Schläge. Halb-/Vollzugzähler sind dafür nicht Teil der Identität. Ein Zobrist-Hash darf *nur als Index* genutzt werden; bei Bedarf Kollisionen absichern, nicht als mathematisch unfehlbaren Gleichheitsbeweis behandeln.

### 5.4 Schachuhren

Zeitkontrolle ist **nicht** im Chess Core und nicht Teil des Positionsobjekts. Der Match-/Game-Service besitzt eine Uhr je Seite und nutzt eine monotone Zeitquelle; Echtzeit-Uhrzeiten dienen nur der Protokollierung.

- **V1.0 Match-Minimum:** feste Tiefe oder feste Suchzeit je Zug und definierte maximale Ausführungszeit.
- **Zeitkontrollen mit Inkrement:** separate Regel für Start, Ende, Overhead, Inkrement und Flaggenfall; erst dann als vollständig unterstützt deklarieren.
- **Wichtige Trennung:** Ein technischer UCI-Timeout ist nicht immer identisch mit einem regelkonformen Verlust durch Zeitüberschreitung.
- **Testbarkeit:** Zeitquelle injizieren; mit Fake Clock Grenzfälle ohne Warten simulieren.


### 5.5 Remisansprüche, Regelprioritäten und die Grenze des FEN-Formats

Nach den [FIDE-Schachregeln, Artikel 9.2 und 9.3](https://handbook.fide.com/chapter/E012023), kann eine spielberechtigte Seite ein Remis wegen dreifacher Wiederholung oder der 50-Züge-Regel **auch unter Angabe eines beabsichtigten legalen Zuges** beanspruchen, wenn erst dieser Zug den Anspruch begründen würde. Die Architektur benötigt deshalb zwei unterschiedliche fachliche Operationen:

- `claim_draw_now(reason)`: prüft die aktuelle Partie **mit vollständiger Historie**; verändert bei zurückgewiesenem Anspruch weder Figurenstellung noch Historie.
- `claim_draw_after(intended_move, reason)`: prüft den vorgeschlagenen legalen Zug **transaktional** in einer temporären Stellung und dem projizierten Historienzustand. Bei berechtigtem Anspruch wird das Remis festgestellt; der Zug wird nicht irrtümlich als bereits ausgeführter Partiehalbzug protokolliert. Die genaue Behandlung von formalen Fehlansprüchen und Zeitstrafen ist von einem expliziten Turnierreglement abhängig.

Ein `GameResult` sollte mindestens **Ergebnis** und **Ursache** voneinander trennen (z. B. `Checkmate`, `Stalemate`, `DeadPosition`, `FivefoldRepetition`, `SeventyFiveMoves`, `ValidClaim`, `Resignation`, `Agreement`, `TimeForfeit`, `TechnicalAbort`). Nicht alle Gründe entstehen im Core; `TechnicalAbort` wird insbesondere vom Match-Schiedsrichter erzeugt.

**Wichtige Grenze von UCI:** Das Standardprotokoll ist primär eine Zug- und Suchschnittstelle; es bietet keinen allgemeinen standardisierten `claim_draw`-Befehl einer Engine an den Match-Schiedsrichter. Für reine UCI-Engine-Partien wird deshalb eine **separat dokumentierte und konfigurierbare Schiedsrichterpolitik** benötigt (beispielsweise automatische Anerkennung eines sonst beanspruchbaren Remis durch den Match-Controller). Eine solche Entscheidung ist als **Matchreglement** zu protokollieren und darf nicht fälschlich als automatisch eintretende FIDE-Regel dargestellt werden.

**Schwieriger Pflichtpunkt „tote Stellung“ (LH-F-007):** Eine Materialheuristik ist nicht mit der vollständigen FIDE-Definition gleichzusetzen. Für eine Freigabe mit behaupteter vollständiger Regeltreue muss die Implementierung ihre Abdeckung nachweisen oder die fachliche Muss-Anforderung **vorher ausdrücklich** abändern. Eine unvollständige Prüfung darf nicht als erfülltes V1.0-Abnahmekriterium verbucht werden.

**Uhrregeln:** Bei Zeitüberschreitung ist nach [FIDE Artikel 6.9](https://handbook.fide.com/chapter/E012023) zu beachten, ob die Gegenseite durch *irgendeine legale Zugfolge* überhaupt Matt erzielen könnte; andernfalls ist die Partie remis. Vereinfachte Materialheuristiken dürfen eine falsche Verlustwertung nicht verursachen. Der Match-Controller trennt darüber hinaus einen **technischen Antworttimeout** von einer tatsächlichen Schachuhr-Flagge.

**FEN besitzt keine Zuggeschichte**: Ein isolierter FEN-Import darf keinen bereits früher entstandenen Wiederholungsanspruch erfinden. Die Vollzug-/Halbzugzähler sind für bestimmte Regeln ausreichend, aber nicht für den Verlauf mehrfach aufgetretener Stellungen. Partieimporte rekonstruieren die dafür notwendige Historie aus legal nachvollzogenen Zügen.

## 6. Klassische Strategie-Engine

### 6.1 Internes Konzept

~~~text
EngineSession / SearchController
   +-- SearchLimits (Tiefe / Zeit / Knoten / Stop)
   +-- SearchState (isolierte temporäre Position)
   +-- SearchAlgorithm (Negamax + Alpha-Beta)
   |     +-- MoveOrdering
   |     +-- optional QuiescenceSearch
   |     +-- optional TranspositionTable
   +-- Evaluation (Material + Positionsmerkmale)
   +-- Diagnostics (aus / Statistik / begrenzter Trace)
   +-- SearchResult (Zug, Score, PV, Tiefe, Laufdaten)
~~~

**Startalgorithmus:** Negamax mit Alpha-Beta-Pruning, stabiler Bewertungs-Perspektive, terminaler Matt-/Remisbehandlung und begrenzter Suchressource. Danach iterative Vertiefung. Zusätzliche Heuristiken werden separat eingeführt und vermessen.

**Bewertung:** Statische Teile (Material, Zentrum, Figurenaktivität, Königssicherheit, Bauernstruktur) haben dokumentierte Vorzeichen und Einheiten (z. B. Zentibauern). Die optionale Bewertungsaufschlüsselung muss aus **denselben Termen** entstehen, nicht aus einer zweiten nur für die GUI berechneten Formel.

**Wichtige Differenzierung:** Ein statischer Stellungswert ist nicht identisch mit dem Ergebnis einer mehrzügigen Suche. Die GUI bezeichnet die Werte entsprechend.

### 6.2 Suchauftrag und Lebenszyklus

~~~text
Idle --start(position snapshot, limits, generation)--> Running
Running --fertig-------------------------------------> Completed
Running --stop / Zeitlimit---------------------------> Stopping
Stopping --letztes vollständiges Resultat-----------> Completed
Completed --Ergebnis abgeholt------------------------> Idle
beliebiger Zustand --quit/shutdown-------------------> Closed
~~~

Ein **SearchTask** besitzt ID, Positionssnapshot, Konfiguration, Stop-Token und nur ihm zugeordnete Ausgabekanäle. Eine einzelne Engine-Sitzung darf nicht zwei Suchaufträge gleichzeitig als denselben aktiven Auftrag betrachten. Späte Resultate werden anhand von Generation und Such-ID verworfen.

**Abbruchregel:** Iterative Deepening stellt, soweit bereits ein legaler Zug bekannt ist, den letzten vollständig abgeschlossenen Suchstand bereit; bei sehr frühem Abbruch bleibt ein vorher legal generierter Fallback-Zug. In einer terminalen Stellung wird UCI-konform **bestmove 0000** ausgegeben. Das UCI-Protokoll verlangt zu jeder gestarteten Suche genau einen Bestmove-Abschluss; die Host-Schicht setzt dies mit einem kontrollierten Lebenszyklus durch.

### 6.3 Bewertungs- und Suchstabilität

- Mattbewertungen außerhalb des normalen Positionswertebereichs; Mattdistanz eindeutig.
- Keine unkontrollierten Ganzzahlüberläufe beim Negieren von Scores oder beim Rechnen mit Unendlichkeits-Schranken.
- Vollständige Behandlung von legalen Null-Zug-Situationen und Remis-/History-Kontext.
- Zufall ist optional und bei Tests deaktiviert oder mit festem Seed versehen; bei gleicher Konfiguration gelten dokumentierte deterministische Bedingungen.
- Suchoptimierungen dürfen die Legalität und die Regeln des Game nicht verändern.

### 6.4 Diagnose ohne Eingriff in die Strategie

**Off:** keine aufwendige Ereignisaufzeichnung.  
**Stats:** aggregierte Zähler (Knoten, Tiefe, Cutoffs, Hash-Treffer, Zeit) mit dokumentierter Definition.  
**Trace:** begrenzte Anzahl ausgewählter Suchereignisse mit Maximalgröße und Maximaldauer; niemals unkontrollierte Speicher-/I/O-Last.

Tracing und Stats werden als optionaler **Sink/Observer** angesprochen; die Engine funktioniert ohne Diagnostik identisch. Instrumentierung kann Laufzeiten beeinflussen und ist deshalb in wissenschaftlichen Ergebnissen sichtbar zu machen.

## 7. UCI: eigene Engine und fremde Engines

### 7.1 Zwei unterschiedliche UCI-Rollen

**UCI-Engine-Host:** SASD-Chess beantwortet Kommandos einer externen GUI/eines Schiedsrichters.  
**UCI-Client:** SASD-Chess startet und kontrolliert einen externen Engine-Prozess.  

Die Rollen teilen Protokolltypen und Parser, aber **nicht** dieselbe Zustandsmaschine.

Die Grundlage ist die öffentlich dokumentierte UCI-Spezifikation von 2006 (siehe Referenzen). Ihre Kommandos und erwarteten Antwortfolgen sind das Interoperabilitätskriterium.

### 7.2 Engine-Host: UCI-Verhalten

Minimal: **uci**, **isready**, **ucinewgame**, **position startpos/fen ... moves**, **go depth**, **go movetime**, **go wtime/btime/winc/binc**, **stop**, **quit**, gültiger **bestmove**. Optionen über **setoption** nur veröffentlichen, wenn tatsächlich unterstützt. Laufende Suche verhindert nicht die Verarbeitung von **isready** oder **stop**.

**Protokollschutz:**

- stdout enthält **ausschließlich** UCI-konforme Nachrichten; Debugmeldungen an stderr.
- Eingaben sind zeilen-/token-/wertbegrenzt; unbekannte Kommandos tolerant und ohne Seiteneffekte behandeln.
- Ein Antwortschreiber serialisiert sämtliche UCI-Antworten; mehrere Threads dürfen UCI-Zeilen nicht unkontrolliert ineinander schreiben.
- Jeder Suchauftrag hat exakt einen Abschluss; **info**-Meldungen dürfen dazwischen auftreten.
- Kommandos, die während einer Suche den Spielstand ersetzen oder eine zweite Suche starten würden, werden nach einem definierten Protokollzustandsmodell **deferiert oder abgewiesen**, nie ungeprüft parallel ausgeführt.

### 7.3 UCI-Client: Prozess und Protokoll

~~~text
Configured -> ProcessStarted -> UciHandshaking -> Initializing
        -> Ready -> Searching -> Ready -> Shutdown -> Exited
                                   \-> Failed / Timeout -> Cleanup
~~~

Der Client startet eine explizit gewählte Binärdatei **ohne Shell-Interpretation**, hat ein eigenes stdin, stdout und stderr und durchläuft uci/uciok, optional setoption, isready/readyok sowie ucinewgame. Protokollzeilen werden in begrenzte Puffer eingelesen; stdout und stderr müssen kontinuierlich abgearbeitet werden, damit die Pipes nicht blockieren.

**Prozessadapter:** POSIX-Prozesse (z. B. fork/exec oder passende sichere Spawn-API) und Windows-Prozesse (CreateProcess-basierter Adapter) erfüllen denselben technischen Vertrag: Start, schreiben, lesen, Stop/Exit, Wartezeitbegrenzung, Prozess-/Pipe-Aufräumen. Pfad und Argumente werden als getrennte Daten behandelt. Ein unbekanntes Engine-Binary besitzt die Rechte des gestarteten Benutzers; SASD-Chess garantiert keine OS-Sandbox.


### 7.4 UCI-Protokollsitzung: Serialisierung, Korrelation und Reihenfolge

UCI besitzt im Drahtprotokoll keine standardisierten Request-IDs zur Zuordnung beliebig überlappender Suchantworten. Darum wird **pro Engine-Sitzung höchstens ein aktiver Suchauftrag** zugelassen. Interne Such-IDs dienen nur dem eigenen Lebenszyklus und der Benutzeroberfläche; sie werden fremden Engines **nicht** als UCI-Erweiterung aufgezwungen.

- Vor dem ersten Suchauftrag: `uci` → `uciok`, Optionen setzen, `isready` → `readyok`. Bei Partiewechsel `ucinewgame`, dann `isready` → `readyok`.
- Je Halbzug `position startpos moves ...` oder `position fen ... moves ...` aus einem zentralen, vollständig legal rekonstruierten Verlauf; erst danach genau ein `go`.
- Während `go` sind `info`-Zeilen erlaubt; eine Suchantwort wird erst mit `bestmove` abgeschlossen. Ein `stop` beendet den Auftrag **nicht unmittelbar**, sondern löst ein begrenztes Warten auf dessen Abschluss aus.
- Eine neue Position oder ein neues `go` wird erst an dieselbe Sitzung geschickt, wenn die vorherige Suche eindeutig abgeschlossen oder die Sitzung explizit als fehlerhaft beendet wurde.
- Bei parallelen Ereignissen (neue GUI-Partie, Undo, Stopp) wird das Suchergebnis aufgrund einer **lokalen Generation** verworfen; ein verspätetes `bestmove` darf nicht einem neuen Auftrag zugeschrieben werden.
- UCI-Host stdout bekommt **einen** serialisierenden Schreiber, UCI-Client stdout/stdin einen getrennten asynchronen Reader/Writer je Prozess. `info` wird typisiert verarbeitet oder begrenzt ignoriert, jedoch nicht als `bestmove` fehlinterpretiert.

Die Interpretation von unbekannten `option`-Werten, zusätzlichen `info`-Feldern, leeren Ausgaben und Fehlerfällen wird mit realen UCI-Kompatibilitätstests sowie einem absichtlich fehlerhaften Mock-Prozess getestet. Ein UCI-Host darf ohne eigene Benutzeroberfläche gestartet werden.

## 8. Match-Controller als eigenständiger Schiedsrichter

### 8.1 Drei Prozesse, zwei Engine-Sitzungen

~~~text
              SASD-Chess Match Controller (eigener Prozess)
                               |
                       Autoritative Game
                         + MatchClock
                         + MoveHistory
                         + Result
                         + PGN Writer
                               |
               +---------------+---------------+
               |                               |
           UCI-Client A                    UCI-Client B
               |                               |
          stdin/stdout A                  stdin/stdout B
               |                               |
          Engineprozess A                 Engineprozess B
~~~

Die Engine-Prozesse können unterschiedliche Programme oder **zweimal dieselbe** SASD-Chess-Binärdatei sein. Sie besitzen eigenständigen Prozesszustand und kommunizieren **nicht direkt** miteinander.

### 8.2 Kontrollierte Zugtransaktion

1. Match-Konfiguration validieren und beide Prozesse getrennt starten.
2. UCI-Handshake/Readiness mit beiden durchführen.
3. Partie initialisieren; Controller ist alleiniger Eigentümer des autoritativen Game.
4. Aktueller Engine eine Stellung und **vollständige legale Zugfolge** via position übermitteln; danach go senden.
5. **bestmove** fristgerecht abwarten, Format parsen, **unabhängig mit chess-core auf Legalität prüfen**.
6. Bei gültiger Antwort: Game und Uhr atomar aktualisieren, Status/Resultat ermitteln, PGN/Events fortschreiben.
7. Bei Partieende: finale PGN-/Statusdaten schreiben und Prozesse geordnet schließen.
8. Bei technischem Fehler: den konkreten Abbruchgrund protokollieren; keine erfundenen Schachzüge oder Ergebnisursachen.

**Wichtig:** Ein in einem externen Prozess berechneter Zug gilt niemals allein aufgrund des Protokolls als legal.

### 8.3 Fehlerpolitik

| Ereignis | Behandlung |
| --- | --- |
| uciok/readyok bleibt aus | Fehler in Initialisierung; kein Matchstart; alle Prozesse bereinigen |
| bestmove syntaktisch unzulässig | Keine Zustandsänderung; technischer Fehler/regelbasierte Verlustwertung entsprechend Konfiguration |
| bestmove ist regelwidrig | Keine Zustandsänderung; protokollierte technische Niederlage oder Abbruch |
| bestmove 0000 trotz legaler Züge | Protokollfehler |
| Keine Antwort innerhalb des Budgets | stop, definierte Grace-Period, dann beenden; Zeit-/Technikursache getrennt |
| Prozess beendet unerwartet | Fehlergrund und unvollständige PGN/Diagnose erhalten |
| Benutzer bricht ab | Stopp, Ressourcenaufräumen, PGN-Ergebnis "*" bei unbekanntem Ergebnis |
| Stderr-Flut / sehr lange UCI-Zeile | Begrenzte Diagnoseaufnahme, Protokollgrenze, keine blockierten Pipes |

Jeder Matchlauf besitzt Match-ID, getrennte Spielerkennungen und definierte Beendigungsgründe. Ein PGN-Ergebnis **1-0 / 0-1 / 1/2-1/2** wird nur ausgegeben, wenn ein solcher Ausgang gemäß Matchreglement tatsächlich feststeht; sonst **\*** mit passender Abbruchinformation.

### 8.4 Spielmodi und Wiederverwendung

- Mensch/Mensch: Game + Application, kein fremder Prozess.
- Mensch/Engine: Game + Engine-Sitzung, optional eigener UCI-Prozess.
- Engine/Engine: MatchController + zwei UCI-Prozesse, headless nutzbar.
- Spätere Matchserien: wiederverwenden des MatchControllers mit Farbwechsel, gepaarten Eröffnungen und unabhängigen Ergebnissen.

**Nicht empfohlen:** eigene Engine direkt mit internen Methoden gegen eine externe Engine spielen lassen und dadurch einen zweiten Matchregelkern erzeugen.


### 8.5 Zwei Uhren, Fehlerpriorität und Ergebniszustand

Ein Match wird durch eine **einzige** Ablaufsteuerung serialisiert; Prozesse existieren gleichzeitig, die beauftragten Züge folgen jedoch dem regulären Zugrecht. Mindestens diese Ereignisse werden fachlich getrennt:

| Ereignis | Quelle | Folge |
| --- | --- | --- |
| Legal empfangener Zug | UCI + Core-Prüfung | Uhr aktualisieren, Game-Zug anwenden, Endstatus prüfen |
| Matt/Patt/automatisches Remis | Game | Partie beenden; keinen weiteren `go`-Auftrag auslösen |
| Korrekt beanspruchbares Remis | Game + festgelegte Matchpolitik | Eigenständige Entscheidung samt Reglement-Ursache protokollieren |
| Schachuhr abgelaufen | Monotone Match-Uhr + Spielregeln | Flaggenfall gemäß vorher gewähltem Reglement (inkl. möglichem Remis) |
| Antwort-/Protokolltimeout | UCI-Client/Watchdog | Technischer Fehler, **nicht automatisch** ein Schach-Uhrenverlust |
| Prozessabsturz / illegaler Bestmove | Process-/Core-Grenze | Keine Zugannahme; technischer Abbruch oder regelbasierte Wertung gemäß Matchkonfiguration |

**Messgrenzen:** Für Schachuhren wird der Zeitbereich vom Abschicken von `go` bis zum vollständigen Empfang der `bestmove`-Zeile auf der Controller-Seite gemessen; ein gegebenenfalls gewünschter Kommunikations-Overhead ist offen, symmetrisch und als eigene Policy zu definieren. Bei fester Suchtiefe gilt ein **technischer Sicherheitstimeout**, der nicht als gewöhnliche Schachuhr missverstanden werden darf. Das Zeitmodell wird durch injizierbare Clock/Transport-Testdoubles geprüft.

**Shutdown-/Fehlerreihenfolge:** Bei Abbruch keine weiteren `go`-Aufträge senden → aktiven Suchauftrag kooperativ stoppen → ausstehende Ausgabe-/Diagnosekanäle begrenzt abarbeiten → `quit` senden → angemessen begrenzt auf Prozessende warten → gegebenenfalls OS-seitig beenden → Ressourcen und Kindprozesse vollständig freigeben → PGN-Ergebnis und technische Ursache korrekt ablegen. Unbekanntes Ergebnis wird als `*` protokolliert und nicht zum Remis umgedeutet.

**Fremde Executables:** OS-Prozessgrenzen verhindern versehentliche gemeinsame Partiestände, sind aber **keine Sicherheits-Sandbox**. Plattformspezifische Prozessadapter sollen auch bei Fehlern zuverlässig aufräumen (unter Windows kann ein Job-Objekt, unter POSIX eine Prozessgruppe ein geeigneter Baustein sein; Details per ADR), ohne unnötige Rechte anzufordern.

## 9. Oberflächenarchitektur und Anwendungsfälle

### 9.1 Adapter statt UI-gekoppelter Spielregeln

Die Desktop-Oberfläche stellt visuelle Elemente bereit; die Application-Schicht führt Befehle aus. Beispielhafte **Anwendungsfälle**:

- neue Partie anlegen, Spielmodus/Farbe festlegen;
- Figur von A nach B ziehen, Umwandlung auswählen;
- Zug rückgängig machen, Partie beenden;
- FEN-Stellung laden, PGN exportieren;
- Engine-Sitzung starten/stoppen, UCI-Prozess konfigurieren;
- Engine-gegen-Engine-Match beginnen/stoppen;
- Analyse/Bewertungsaufschlüsselung und später Laboratory-Läufe darstellen.

Die Anwendung liefert **immutable Snapshots** bzw. kopierte Read-Modelle an das Frontend und akzeptiert Befehle als Eingaben. Das Frontend darf den intern von Game gehaltenen Position-Speicher nicht mutieren.

### 9.2 UI-Technologie ist austauschbar

Das konkrete GUI-Framework bleibt bis zu einem eigenständigen Spike offen. Das separate **SASD-UI-Toolkit** ist ein möglicher Adapter, aber **keine Voraussetzung** für Core oder Engine. Keine Architekturentscheidung koppelt die Schachlogik an eine nicht bestätigte GUI-Technologie.

Das Desktop-MVP zeigt Brett, Figuren, Zugliste, Status, lokale Mensch/Mensch- und Mensch/Engine-Partien, Umwandlung und FEN/PGN-Aktionen. Laboratory-Anzeigen folgen erst in der jeweiligen Ausbauphase.

### 9.3 Asynchroner Kontrollfluss

Die grafische Ereignisschleife darf nicht auf Engine-Suche, blockierendes UCI-Lesen, Dateiexport oder Matchserien warten. Längere Aktionen laufen in einem Worker/Task, liefern Statusereignisse und besitzen explizite Abbruchmöglichkeiten. UI-Aktualisierungen erfolgen ausschließlich über den UI-eigenen Ereignis-/Dispatch-Mechanismus; keine direkten Widgetzugriffe aus Worker-Threads.

**Generation-Prinzip:** Bei neuer Partie, Undo oder geladener FEN wird die Spielgeneration erhöht. Ergebnisse früherer Berechnungen werden verworfen, auch wenn sie erst nach der Zustandsänderung eintreffen.

## 10. Dateien, Persistenz und Austauschformate

### 10.1 FEN

FEN repräsentiert *eine* Stellung, keine vollständige Historie. Die sechs Standardfelder werden geparst und serialisiert. Parserfehler liefern Ort/Kategorie/Nachricht, nicht Programmabbruch. Externe Stellungstexte werden erst nach erfolgreichem Parse/Validieren übernommen.

### 10.2 SAN und PGN

SAN entsteht aus dem aktuellen Legalzugvorrat und der Stellung (inklusive eindeutiger Zugdisambiguierung, Schach/Matt, Umwandlung, Rochade). PGN speichert Partie-Tags, SAN-Folge, Ergebnis und bei Bedarf eine FEN-Startstellung. Für 1.0 ist Export Muss, Import Soll. Ein Import rekonstruiert die Partie durch **Zug für Zug erfolgende Legalitätsprüfung** und darf nicht nur SAN-Texte übernehmen.

### 10.3 Forschungsdaten

PGN für Schachpartien, FEN für Positionen, **versioniertes JSON** für Experimente/Metadaten, CSV für tabellarische Messwerte, Markdown/HTML für spätere Berichte. Die Format- und Persistenzkomponenten schreiben atomar über temporäre Dateien und Umbenennen, sofern die Zielplattform/Dateisystemsemantik dies erlaubt; Fehler müssen gemeldet werden, statt vorherige gültige Daten zu beschädigen.

**Keine Pflichtdatenbank.** Kleine Konfigurationen können als versioniertes JSON gespeichert werden; eine spätere Datenbank kann über einen eigenen Repository-Adapter eingebunden werden, wenn ein konkreter Bedarf nachgewiesen wird.

### 10.4 Sichere Eingaben

Dateigrößen, Tokenlängen und Rekursionstiefe (insbesondere PGN mit Varianten) begrenzen. Keine Exe-Pfade aus PGN/FEN-Daten ausführen. Engine-Startpfade bedürfen einer ausdrücklich gewählten Konfiguration. Fremdtexte für GUI und Logs kontextgerecht darstellen.

## 11. SASD-Chess Laboratory – schachbezogene Experimentalarchitektur

### 11.1 Abgrenzung und Stufen

**Spielen**, **Analysieren** und **Laboratory** sind logische UI-Arbeitsbereiche, keine eigenen Schachregel-Implementierungen. Geplante Funktionen (LH-F-042–056): Stellungslabor, statische Bewertungsaufschlüsselung, Suchtraces, Algorithmusvergleich, Benchmarks, Engine-Serien, Experimentverwaltung, Berichte, Perft/Divide, Partieanalyse, Taktikregressionen, Brettrepräsentationsvergleich, Diagnoseprofile, strukturierte Exporte.

**Verbindlicher V1.0-Baustein:** Perft/Divide als Regelprüfung (LH-F-051). Weitere Funktionen sind Soll bzw. nach 1.0 einzuplanen. Wer einen Laboratory-Zusatz implementiert, muss dessen zugeordneten Test nachweisen; nicht umgesetzte Zusatzfunktionen blockieren V1.0 nicht.

### 11.2 Messpipeline

~~~text
ExperimentDefinition / BenchmarkRequest
                |
       Validieren + festlegen
                |
 Testkorpus/FEN + Engine-Version + Konfiguration
                |
           Runner (headless)
                |
  Core / Search / Match / Diagnostics
                |
     rohe Einzelmessungen / PGN
                |
    Ergebnis-Aggregation / Statistik
                |
      JSON/CSV + später Bericht
                |
         GUI als Betrachter
~~~

**ExperimentDefinition:** Kennung, Frage, Hypothese, Teststellungs-IDs, Engine-Binaries/Commits, Einstellungen, Tiefen-/Zeitgrenzen, Wiederholungen, Seed/Eröffnungsplan, gewählte Messmethodik.

**ExperimentResult:** Schema-Version, Einzelmessungen, Programm-/Hardware-/Buildinformationen, Fehlversuche, Ergebnisdateien, Prüfsummen und aggregierte Kennzahlen. Keine Ergebnisdatei ohne passende Konfigurationsreferenz.

### 11.3 Methodisch korrekte Messungen

- **Perft-Korrektheit** nicht mit Engine-Spielstärke verwechseln.
- Bei **identischer exakter Negamax-/Alpha-Beta-Suche** und gleicher Blattbewertung muss die Wurzelbewertung für eine festgelegte Tiefe übereinstimmen; ein Quiescence-Unterschied verändert dagegen die Suchsemantik.
- **NPS** zeigt Knoten pro Zeit, nicht „Qualität pro Knoten“.
- **Engine-Vergleichspartien** benötigen möglichst gleiche Regeln, Farbwechsel, gepaarte Eröffnungen und ausreichend Daten; statistische Unsicherheit ausweisen.
- **Tracing** verändert möglicherweise die Laufzeit: Referenz-Benchmarks ohne Trace messen, Zusatzoverhead gesondert dokumentieren.
- **Reproduzierbarkeit** in Kategorien ausweisen: deterministische Zugfolge bei fester Tiefe versus schwankende Laufzeit-/Uhr-Ergebnisse.
- Compiler, Plattform, CPU, Build-Flags, Suchlimits und Messumfang speichern.
- Beobachtete Korrelationen und kleine Stichproben nicht als allgemeine Kausalität oder wissenschaftlichen Beweis darstellen.

### 11.4 Erweiterbare Laboratory-Verträge

- **PositionEditorController:** erlaubt unvollständige Editierstellungen, exportiert nur validierte Spielstellungen.
- **EvaluationBreakdown:** liefert berechenbare, summierbare statische Beiträge; keine zweite Bewertungsformel.
- **BenchmarkRunner:** nimmt festen Korpus und Messparameter an; liefert immutable Run-Resultate.
- **MatchSeriesRunner:** ruft bestehendes Schiedsrichter-/UCI-Modul mehrfach auf; erzeugt gepaarte Partien und Ergebnisdaten.
- **ExperimentRepository:** persistiert Konfiguration und Einzelergebnisse mit Schema-Version/Validierung.
- **ReportBuilder:** erzeugt aus gespeicherten Daten methodisch begrenzte Markdown-/HTML-Berichte.
- **TraceCollector:** zeichnet nur ausgewählte Ereignisse unter expliziten Speicher-/Ereignislimits auf.

Die Schnittstellen sind **fachliche Verträge**, keine Anweisung, vor dem ersten funktionierenden Schachprogramm leere Frameworkklassen zu bauen.

## 12. Parallelität, Isolation und Ressourcen

### 12.1 Thread- und Prozessbesitz

| Ausführungseinheit | Darf besitzen | Darf nicht |
| --- | --- | --- |
| GUI-Thread | Widgets, sichtbare Modelle, UI-Dispatch | Lange Such-/UCI-Arbeit blockierend durchführen |
| Engine-Worker | Kopierter Suchzustand, lokale Heuristiken und Stack | Live-Game/GUI direkt ändern |
| UCI-Eingabethread | Zeilenparser, Host-Protokollzustand | Direkt rekursive Suche blockierend durchführen |
| UCI-Schreibkoordination | Exklusive stdout-Nachrichtenwarteschlange | Diagnosen außerhalb UCI auf stdout schreiben |
| MatchController | Autoritative Game, Clock, Prozesssessions, Matchresultat | Fremdengine-internen Zustand als Wahrheit behandeln |
| Prozess-I/O-Worker | Begrenzte Pipe-Puffer und Timeoutzustände | Unkontrolliert unbegrenzt Nachrichten puffern |
| Laboratory-Runner | Experiment-Snapshot und Rohmessungen | Normalpartie bei Ablauf ändern |

**Erst V1.0:** Ein Engine-Suchworker je Prozess, keine verpflichtende parallele interne Suche. Zwei Match-Engines sind gerade deshalb zwei **unabhängige OS-Prozesse**.

### 12.2 Abbruch und Shutdown

Stopp ist kooperativ: SearchController setzt ein Stop-Token, Suche prüft es periodisch und gibt eine konsistente Antwort zurück. Ein Matchprozess erhält zuerst **stop**, danach bei Bedarf **quit**; bei Überschreitung definierter Fristen wird der Kindprozess auf OS-Ebene beendet und vollständig aufgeräumt. Ressourcenobjekte verwenden RAII und expliziten Besitz.

Beim Beenden werden neue Kommandos gesperrt, aktive Suchläufe gestoppt, Dateien soweit möglich abgeschlossen und Kindprozesse freigegeben. Kein Thread darf ein bereits zerstörtes Game-, Logger- oder UI-Objekt dereferenzieren.

### 12.3 Beobachtbarer Fehler statt versteckter Fallback

Technische Fehler sind von Partieniederlagen getrennt zu codieren. Eine nicht erreichbare Engine erzeugt keinen synthetischen legalen Zug. Ein fehlgeschlagener Dateiexport darf nicht als erfolgreich angezeigt werden. Die GUI liefert verständliche Fehlertexte; interne Logs behalten maschinenlesbare Fehlercodes und Kontext.

## 13. Fehler-, Sicherheits- und Loggingkonzept

**Fehlerfamilien:**

- **InvalidPosition / IllegalMove / InvalidGameAction:** fachliche Eingabefehler.
- **FormatError:** FEN/PGN/SAN ungültig, mit Stelle und Fehlergrund.
- **ProtocolError / ProtocolTimeout:** UCI-Verletzung oder keine Antwort.
- **ProcessStartError / ProcessExitError:** externer Prozess nicht gestartet oder unerwartet beendet.
- **SearchInterrupted / ResourceLimit:** abgebrochene bzw. begrenzte Berechnung.
- **StorageError:** Lese-/Schreib-/Serialisierungsfehler.
- **UnsupportedFeature / ConfigurationError:** ungültige oder noch nicht implementierte Konfiguration.

Fehler werden an Modulgrenzen als strukturierte Ergebnisse behandelt; Exceptions dürfen innerhalb enger Grenzen verwendet werden, aber niemals ungeprüft über Prozess- oder UI-Grenzen wandern.

**Sicherheit:** Keine unkontrollierte Ausführung fremder Dateien, keine implizite Shell, keine obligatorische Netzverbindung, Eingabegrößen begrenzen, Logs von Geheimnissen und unnötigen lokalen Pfaden freihalten. Eine externe UCI-Engine ist ausführbarer Code mit Benutzerrechten; die Anwendung muss die Vertrauensgrenze deutlich kommunizieren, denn Prozesse allein stellen keine vollständige Sicherheitsisolation dar.

**Logging-Kanäle:** UCI stdout ausschließlich Protokoll, stderr für Diagnostik; lokale Anwendung strukturierte, begrenzte Logs mit Level, Zeitstempel, Match-/Such-ID und Fehlercode. Technische Logs sind **keine** PGN-Partien; Forschungs-Messdaten sind **keine** Fehlerlogs.

## 14. Build, Quellbaum und Entwicklungsregeln

### 14.1 Vorgeschlagene Ordner

~~~text
SASD-Chess/
├── CMakeLists.txt
├── CMakePresets.json                # optional nach Setup
├── README.md
├── LICENSE                          # Lizenz vor Release festlegen
├── cmake/
├── include/sasd/chess/
│   ├── core/
│   ├── game/
│   ├── formats/
│   ├── search/
│   ├── diagnostics/
│   ├── uci/
│   ├── process/
│   ├── match/
│   ├── application/
│   └── laboratory/                  # erst im Ausbau
├── src/
│   ├── core/
│   ├── game/
│   ├── formats/
│   ├── search/
│   ├── diagnostics/
│   ├── uci/
│   ├── process/
│   ├── match/
│   ├── application/
│   ├── desktop/
│   └── laboratory/                  # erst im Ausbau
├── apps/
│   ├── chess-uci-main.cpp
│   ├── chess-match-main.cpp
│   ├── chess-desktop-main.cpp
│   └── chess-bench-main.cpp         # später
├── tests/
│   ├── core/
│   ├── game/
│   ├── formats/
│   ├── search/
│   ├── uci/
│   ├── process/
│   ├── match/
│   └── laboratory/                  # nach Bedarf
├── benchmarks/                      # kleine, versionierte FEN-Korpora
├── research/                        # erst bei Laboratory-Ausbau
│   ├── experiments/
│   ├── datasets/
│   ├── results/
│   └── reports/
├── docs/
│   ├── Lastenheft.md
│   ├── Pflichtenheft.md
│   ├── Architektur.md
│   ├── adr/                          # Architekturentscheidungen
│   └── testing/
└── .github/workflows/
~~~

Die Struktur ist **Zielstruktur**, kein Anspruch, dass alle Ordner jetzt schon existieren. Unbenutzte leere Bibliotheken werden **nicht vorsorglich** angelegt. Modularität ergibt sich aus stabilen Zuständigkeiten und Tests, nicht allein aus vielen Verzeichnissen.

### 14.2 Buildregeln

- C++20, CMake, CTest; Debug und Release.
- Core und alle headless Targets ohne GUI-Bibliotheken baubar.
- Linux mindestens GCC/Clang, Windows MSVC; weitere Plattformen als zusätzliche CI-Läufe sobald verfügbar.
- Keine Plattformspezifika in öffentlichen Chess-Core-Headern.
- Compilerwarnungen ernst nehmen, Sanitizer soweit Toolchain verfügbar.
- Datenkorrektheit in Debug/Release identisch; Zeitmesswerte nur in sauber dokumentierter Konfiguration vergleichen.
- Third-Party-Abhängigkeiten bewusst knapp halten und Lizenz vor 1.0 prüfen.

### 14.3 Coding- und Kommentierungsregeln

Öffentliche Klassen/Funktionen bekommen erklärende Header-/API-Kommentare mit Zweck, Parameterinvarianten, Rückgabewerten, Fehlern, Ownership und Thread-Annahmen. Nicht offensichtliche Algorithmen kommentieren vor allem **warum** ein bestimmter Ansatz korrekt ist (z. B. En passant, Rochade, Wiederholung, UCI-Zustände, Uhr). Englischsprachige Codebezeichner sind empfohlen; die gewünschte Kommentarsprache wird projektweit einheitlich festgelegt, nicht pro Datei gemischt. Die Dokumentation ist deutschsprachig.

**Testcode ist Produktcode.** Jeder Bugfix für ein Regel-/Protokollproblem enthält nach Möglichkeit zuerst einen reproduzierbaren Regressionstest.

## 15. Testarchitektur und Definition of Done

### 15.1 Testpyramide

| Testebene | Gegenstand | Mussnachweis |
| --- | --- | --- |
| Unit | Geometrie, Angriffe, Rochade, En passant, Umwandlung, Undo | Korrekte Grenzen und Invarianten |
| Perft/Divide | Referenz-FEN, Zugzahlen bis definierte Tiefe | Summen und bekannte Referenzzahlen stimmen |
| Property/Mutation | make/unmake, FEN-Roundtrip, keine ungültigen Zustandswechsel | Ausgangszustand exakt rekonstruiert |
| Game | Matt, Patt, Remis/Beanspruchung, Ergebnis-Historie | Fachlich korrekte Endzustände |
| Search | Legalität, feste Tiefen, Stop, Mattbewertung | Kein illegaler bestmove, kein Such-Hänger |
| UCI Host | Handshake, Position, Suche, Stop, Quit, Protokollordnung | UCI-korrekte Zeilen, genau ein Bestmove |
| UCI Client/Process | Antwortausfall, kaputter Text, Absturz, stderr-Flut | Timeouts ohne Deadlock/Prozessleck |
| Match | Eigene Engine ×2, externe Engine, Uhr, PGN | Autoritativer Zustand, Ergebnis korrekt |
| Desktop | Mensch/Mensch, Mensch/Engine, Promotion, Undo | Vollständige Partien ohne inkonsistente UI |
| Laboratory (nach Ausbau) | Metadaten, Vergleiche, Stats, Trace-Limits | Reproduzierbarkeit und begrenzte Ressourcen |

**CI-Grundregel:** Externe Engines sind keine Voraussetzung für jeden Build. Kritische UCI-Fehlerfälle werden durch kleine **Mock-Engine-Prozesse** simuliert; reale Interoperabilitätstests mit bekannter Fremdengine werden separat dokumentiert.

### 15.2 Architektur-Gates pro Meilenstein

| Etappe | Definition of Done |
| --- | --- |
| **M0 – Fundament** | CMake-Ziele, Core-Hülle, Test-Harness, Windows-/Linux-Build, Doku/ADR-Basis |
| **M1 – Regelkern** | alle regulären/sonderrechtlichen Züge, Undo, Perft/Divide mit Referenzfällen |
| **M2 – Game und Formate** | Ergebnis/Remis, Historie, FEN, SAN, PGN-Export, Zustands-/Roundtrip-Tests |
| **M3 – Engine/UCI Host** | Negamax/Alpha-Beta, Suche mit Limits/Stop, korrekter UCI-Prozess |
| **M4 – UCI Client/Match** | zwei unabhängige Prozesse, Self-play und Fremdengine-Match, Legalität/Protokoll/Fehler getestet |
| **M5 – Desktop** | menschliche Spielmodi, Engine-Einbindung, brauchbare Benutzeroberfläche, End-to-End-Tests |
| **M6 – Release 1.0** | Pflichtabnahmen einschließlich LH-A-17, Windows/Linux CI, Benutzer-/Builddoku, Lizenz |
| **L1 – Analysebasis** | Stellungslabor, Bewertungen, Diagnoseprofile und Grund-Benchmarks |
| **L2 – Vergleichslabor** | Engine-Serien, reproduzierbare Ergebnisse, PGN-Analyse und Datenexport |
| **L3 – Vertieftes Laboratory** | Experimentverwaltung, begrenzte Suchtraces, Algorithmus-/Datenstrukturvergleiche und Berichte |

**Abnahmekriterium:** Ein Arbeitspaket ist erst abgeschlossen, wenn es kompiliert, automatisiert getestet, kommentiert, dokumentiert und reproduzierbar integriert ist. Ein Pull Request oder eine Datei allein genügt nicht.

### 15.3 Requirements-Traceability

- **LH-F-001–010 / LH-A-01–05, 09, 17:** chess-core und chess-game, Perft/Divide, Undo, Status.
- **LH-F-011–020:** chess-application und chess-desktop; manuelle/automatisierte UI-Prüfung.
- **LH-F-021–024, 028–029:** chess-search und chess-uci-host; Strategie-/Such-/Protokollprüfungen.
- **LH-F-025–027:** chess-formats; FEN-/SAN-/PGN-Prüfungen.
- **LH-F-033–040 / LH-A-13–16:** chess-uci-client, chess-process und chess-match; zwei-Prozess-Matches.
- **LH-F-042–056 / LAB-T-01–10:** Diagnostik, Benchmark und später Laboratory entsprechend Priorität.
- **LH-N-001–020:** Qualität, Portabilität, Sicherheit, Messung und Betrieb; Review- und CI-Nachweise.

*LH-F-031 wurde im Lastenheft zurückgezogen und darf nicht für einen anderen Zweck wiederverwendet werden.*

## 16. Architekturentscheidungen und offene Punkte

Die folgenden Kennungen sind **Architekturentscheidungs-Kandidaten**. **„Vorschlag“** bedeutet fachlich/technisch motiviert, aber noch nicht als eigenständiger ADR freigegeben.

| Kennung | Gegenstand | Richtung | Status |
| --- | --- | --- | --- |
| ADR-001 | Programmiersprache/Build | C++20 + CMake/CTest | Vorschlag im Pflichtenheft |
| ADR-002 | Fachliche Wahrheit | Ein Schachkern, keine parallele Legalitätsimplementierung | Kernprinzip |
| ADR-003 | Brettrepräsentation | Zunächst 64-Felder-Array, später messbare Optimierung | Vorschlag |
| ADR-004 | Trennung Position / Game / SearchState | Unabhängige Repräsentation fachlicher Zustände | Kernprinzip |
| ADR-005 | Suche | Negamax/Alpha-Beta + iterative Vertiefung | Vorschlag |
| ADR-006 | UCI | Separates Engine-Binary + UCI-Client | Kernprinzip |
| ADR-007 | Matches | Eigenständiger Zwei-Prozess-Schiedsrichter | Kernprinzip |
| ADR-008 | GUI | Konkretes Toolkit per Spike auswählen | Offen |
| ADR-009 | Betriebssystemadapter | Prozess/Pipes/Uhr plattformabhängig kapseln | Kernprinzip |
| ADR-010 | Persistenz | FEN/PGN; JSON/CSV für spätere Experimente; keine Pflichtdatenbank | Vorschlag |
| ADR-011 | Wissenschaftliche Instrumentierung | Off/Stats/Trace, begrenzte Diagnose | Vorschlag |
| ADR-012 | Testframework | CTest plus konkretes kleines Framework | Offen |
| ADR-013 | Open-Source-Lizenz und Figurenassets | Vor Veröffentlichung verbindlich auswählen | Offen |
| ADR-014 | Remis-/Zeit-/Fehlerreglement | FIDE-nahe Fachlogik, technische Matchfehler getrennt | Detailentscheidung offen |
| ADR-015 | GUI-Kommentarsprache / API-Konventionen | Einheitlich dokumentierte Projektkonvention | Offen |

### 16.1 Entscheidungsprotokoll für neue ADRs

Für jede wesentliche Entscheidung: **Kontext, Problem, Alternativen, Entscheidung, Konsequenzen, Gegenargumente, Datum, Status und betroffene LH-IDs**. Keine großflächige Architekturänderung ohne dokumentierten Grund und Regressionstests.

### 16.2 Bewusst vermiedene Komplexität

Für den Stand 1.0 werden **nicht** eingeführt: Microservices, Netzwerkruntime, Cloud-Account, Datenbankpflicht, Plug-in-Binär-ABI, generische Spieleengine, unbeschränkte Trace-Erfassung oder ein eigener Ersatz für etablierte UCI-/PGN-Protokolle.

**Evolutionsstrategie:** Neue Abstraktion erst nach mindestens einem realen zweiten Nutzer oder einer nachweislichen Kopplung; nicht „für alle denkbaren Fälle“ vorbauen.

## 17. Risikoregister und Gegenmaßnahmen

| Risiko | Auswirkung | Gegenmaßnahme |
| --- | --- | --- |
| Sonderregelfehler (EP/Rochade/Remis) | Falsche Partien / illegale Engine-Züge | Perft/Divide, Randfall-FEN, History-/Undo-Tests |
| Wiederholung falsch erkannt | Unberechtigtes Remis oder ignorierter Anspruch | Historie plus korrekt kanonisierter Positionsschlüssel |
| Tiefe Suche blockiert UCI | Verlorener Timeout/Protokollfehler | Separater Suchworker, Stopp- und Ready-Tests |
| Prozess-/Pipe-Deadlock | Hängender Matchlauf | Asynchrones Lesen beider Kanäle, Limits, Watchdog |
| Veraltete Suchantwort | Zug in falscher Partie | Generation/Such-ID und Snapshot-Prüfung |
| Unbekannte fremde EXE | Schadsoftware-/Datenschutzrisiko | Explizite lokale Pfadauswahl, Vertrauenswarnung, keine Shell |
| Zu frühe Bitboard-Optimierung | Schwer nachvollziehbare Regeldefekte | Erst Array-/Perft-Baseline und Tests |
| Gemischte Format-/Regellogik | Doppelte Zustandsregeln | FEN/SAN/PGN nur als Adapter zur Domäne |
| Laboratory-Tracing verzerrt Benchmarks | Wissenschaftlich ungültiger Vergleich | Diagnoseprofile, getrennte Referenzmessung |
| Zu viele Module/Frameworks zu Beginn | Langsamer Projektfortschritt | Stufenweise Targets und Schnittstellen schaffen |
| Unklare Lizenz von Figurenassets | Veröffentlichung blockiert | Vor Release prüfen, freie/selbst erstellte Assets |
| GUI-Toolkit unreif | Verzögerung der Engine-Arbeit | Headless Build, spätere Austauschbarkeit |

## 18. Quellen, Glossar und Dokumentenpflege

### 18.1 Referenzdokumente

- [Lastenheft v0.4](Lastenheft.md), fachliche Anforderungen und Abnahme.
- [Pflichtenheft v0.4](Pflichtenheft.md), technische Lösungsanforderungen.
- [FIDE Laws of Chess (FIDE Handbook, Stand 01.01.2023)](https://handbook.fide.com/chapter/e012023), normativer Ausgangspunkt für Standardschach; vor 1.0 auf Aktualität prüfen.
- [Universal Chess Interface, dokumentierte Version April 2006](https://backscattering.de/chess/uci/), UCI-Mindestinteroperabilität.
- [PGN Standard Specification (1994)](https://github.com/fsmosca/PGN-Standard/blob/master/docs/pgn_standard/starting_doc/PGN-Standard.txt), PGN/SAN-Austausch.
- Dokumentierte Perft-Teststellungen werden nach M1 mit Herkunft, FEN und Referenzwerten unter docs/testing/ versioniert.

### 18.2 Glossar

| Begriff | Bedeutung |
| --- | --- |
| **Core** | UI-freie Bibliothek für Zustände, legale Züge und schachliche Regeln |
| **Position** | Momentane Figurenstellung und Sonderrechte/Zähler, ohne vollständige Partiehistorie |
| **Game** | Zugverlauf, Historie, Ergebnis und anspruchsabhängige Regeln |
| **Ply** | Einzelner Halbzug |
| **Perft/Divide** | Zählen legaler Suchbaumblätter / Aufschlüsseln nach erstem Zug |
| **UCI Host/Client** | Engine-seitiger Empfänger bzw. steuernde Seite des UCI-Protokolls |
| **Match-Controller** | Unabhängiger Schiedsrichter für zwei Engine-Prozesse |
| **PV** | Principal Variation, beste bekannte Suchvariante |
| **Ablation** | Deaktivieren eines Algorithmusteils, um seinen Beitrag kontrolliert zu messen |
| **Laboratory** | Schachbezogene, nach V1.0 ausbaubare Analyse-/Experimentfunktionen |

### 18.3 Änderungsverlauf

| Version | Datum | Änderung |
| --- | --- | --- |
| **0.1** | **27.10.2026** | Erstes zusammenhängendes deutsches Architekturkonzept mit Schachdomäne, Bibliotheksgrenzen, Such- und UCI-Lebenszyklen, Match-Schiedsrichter, UI, Laboratory, Qualitätsgates und ADR-Vorbereitung. |

**Freigabepunkt vor Implementierungsbeginn:** Die Kernprinzipien und der M0/M1-Schnitt werden fachlich/technisch geprüft. Toolchain-/Testframework-/GUI-/Lizenzentscheidungen erfolgen in abgegrenzten ADRs, nicht durch unbemerkte Festlegungen in Implementierungsdetails.
