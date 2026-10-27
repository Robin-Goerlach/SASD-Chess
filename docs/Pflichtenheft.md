# SASD-Chess – Pflichtenheft

> **Version:** 0.3 · **Status:** technischer Entwurf zur Freigabe · **Datum:** 27.10.2026  
> **Zuordnung:** [Lastenheft](Lastenheft.md) · **Repository:** https://github.com/Robin-Goerlach/SASD-Chess  
> **Geltungsbereich:** Standardschach, ausschließlich klassische Such- und Bewertungsalgorithmen, grafische Anwendung, UCI-Schnittstellen und Engine-gegen-Engine-Modus.

## 1. Zweck, Status und verbindliche Begriffe

Dieses Dokument beschreibt, **wie** die Anforderungen des Lastenhefts technisch umgesetzt und nachgewiesen werden sollen. Es ist eine **geplante Architektur, keine Aussage über bereits implementierte Funktionalität**. Änderungen am Lastenheft haben Vorrang vor technischen Details dieses Entwurfs. **SASD-Chess entwickelt ausschließlich ein klassisches Schachprogramm mit deterministischen Regeln, handentworfener Bewertung und algorithmischer Spielbaumsuche**. Eine Ausweitung auf andere Produkte oder Verfahren wird in diesem Repository nicht verfolgt.

- **Muss (M):** erforderlich für Version 1.0.
- **Soll (S):** angestrebt, aber kein Blocker für eine abgestimmte 1.0-Abnahme.
- **Kann (K):** optionale Verbesserung.
- **Später (Z):** außerhalb Version 1.0.
- **Engine:** Komponente, die aus einer Stellung einen Zug berechnet.
- **UCI-Client/Controller:** Komponente, die eine UCI-Engine startet und steuert (als `GUI` im UCI-Protokoll bezeichnet).
- **Match-Controller / Schiedsrichter:** Komponente, die zwei Spieler/Engine-Prozesse verwaltet, Zugrechte durchsetzt, Partien protokolliert und Ergebnisse festlegt.

**Wesentliche Systementscheidung:** Eine UCI-Engine spricht **nicht direkt** mit einer zweiten UCI-Engine. Beide werden von einem **Match-Controller** gesteuert. Deshalb realisiert SASD-Chess **beide Seiten** des Protokolls: einen UCI-Engine-Server (als Prozess) und einen UCI-Client in der Match-Verwaltung. Nur so funktionieren sowohl Fremdengine-Partien als auch zwei gleichzeitig laufende Instanzen von SASD-Chess ohne Sonderprotokoll.

## 2. Technische Zielarchitektur

```text
                         +----------------------+
                         | Desktop UI / CLI     |
                         | Start, Anzeige, Log  |
                         +-----------+----------+
                                     |
                         +-----------v----------+
                         | Game / Match Control |
                         | authoritative board  |
                         | rules, clocks, PGN   |
                         +-----------+----------+
                                     |
                    +----------------+----------------+
                    |                                 |
          +---------v----------+            +---------v----------+
          | UCI Process Client |            | UCI Process Client |
          | Engine A (white)   |            | Engine B (black)   |
          +---------+----------+            +---------+----------+
                    | stdin/stdout                     | stdin/stdout
          +---------v----------+            +---------v----------+
          | SASD / external    |            | SASD / external    |
          | UCI engine process |            | UCI engine process |
          +--------------------+            +--------------------+

  Other UCI GUI ---------------------------------> SASD-Chess --uci
                                                     |
                                             +-------v-------+
                                             | Core + Search |
                                             +---------------+
```

Der **Schachkern** ist in beiden Betriebsarten derselbe. Die Match-Verwaltung kann die Engine über einen separaten Prozess starten, statt Sonderzugriffe auf Engine-Interna zu verwenden. Eine spätere In-Process-Optimierung ist optional, darf aber nicht die einzige Schnittstelle sein.

### 2.1 Komponenten und Verantwortlichkeiten

| Komponente | Zuständigkeit | Darf nicht |
| --- | --- | --- |
| `chess-core` | Brett, Züge, Regeln, Ergebnis, Stellung, Hashes | GUI-/UCI-Ausgaben direkt erzeugen oder Prozesse starten |
| `chess-formats` | FEN, SAN, PGN, UCI-Zugnotation | Legale Züge unabhängig vom Core behaupten |
| `chess-engine` | Stellungsbewertung, Suche, Limits, Abbruch | Partiestand einer fremden Engine verändern |
| `chess-uci-engine` | UCI-Engine-Prozess: stdin lesen, stdout schreiben, Suchsteuerung | Fortschritt als beliebigen Text auf stdout ausgeben |
| `chess-uci-client` | UCI-Prozess starten, initialisieren, Befehle senden und Antworten lesen | Bestmoves ohne Legalitätsprüfung ungeprüft übernehmen |
| `chess-match` | Zwei Spieler zuweisen, Partieabwicklung, Uhren, Ergebnis, Abbruch | Schachregeln duplizieren |
| `chess-ui` | Brett, Menüs, Zugliste, Partie- und Matchsteuerung | Direkte Regel-/Engine-Implementierung |
| `chess-cli` | Headless-Optionen für Engine, Tests und automatisierte Matches | GUI zur Laufzeit zwingend benötigen |

Abhängigkeiten verlaufen von UI/CLI, Match und Engine **zum Core**, niemals umgekehrt. Engine und Match werden über klar definierte Schnittstellen und Datentransferobjekte gekoppelt. Keine RTTI- oder konkrete GUI-Abhängigkeit im Core erforderlich.

## 3. Datenmodell des Schachkerns

### 3.1 Fachliche Typen

**Vorschlag für C++-Typen:** `Color`, `PieceType`, `Square`, `Move`, `Position`, `GameState`, `UndoState`, `GameResult`, `SearchLimits`, `SearchResult` und `EngineOptions`.

- `Square`: 64 reguläre Felder; nachvollziehbare Konvertierung zu `a1` bis `h8`.
- `Move`: Start/Ziel, Umwandlung und ggf. zusätzliche Zugart; der Zug allein ist **keine** komplette Stellung.
- `Position`: Figuren, Zugrecht, Rochaderechte, En-passant-Ziel, Halbzug-/Vollzugzähler.
- `GameState`: Position **plus** Zug-/Stellungshistorie, Remisansprüche und Ergebnis; wichtig für Wiederholung.
- `UndoState`: alle irreversiblen Informationen für präzise Rücknahme.
- `GameResult`: läuft / Weiß gewinnt / Schwarz gewinnt / remis plus normierter Grund.
- `SearchLimits`: Tiefe, Knotenlimit, Zeitbudget und Abbruchsignal; wohldefinierte Präzedenz.

### 3.2 Repräsentation

Zum Start kann ein einfaches Array mit 64 Feldern verwendet werden. Es muss nicht vorab die endgültige performante Darstellung (Bitboards o. Ä.) gewählt werden. **Zuerst korrekt und testbar; später messen und gezielt optimieren.** Interne Datenstrukturen dürfen nicht als dauerhaftes Dateiformat verwendet werden.

### 3.3 Invarianten

Eine gültige Standardstellung besitzt genau einen König jeder Farbe, konsistente Zugrechte und eine wohldefinierte Belegung. Illegaler und inkonsistenter Zustand ist von einem schachlich ungewöhnlichen, aber gültigen Zustand zu unterscheiden. Parser liefern strukturierte Fehler statt Abstürze.

## 4. Regeln, Zugerzeugung und Rücknahme

### 4.1 Verarbeitung

1. Pseudo-legale Züge für die am Zug befindliche Farbe erzeugen.
2. Besonderheiten von Bauer, En passant, Umwandlung und Rochade prüfen.
3. Kandidaten probeweise anwenden, Königssicherheit testen und wieder zurücknehmen.
4. Ausschließlich legale Züge bereitstellen und ausführen.
5. Partieende, Wiederholungen und Remisansprüche im `GameState` aktualisieren.

Für Suche und Perft werden `makeMove` und `unmakeMove` bzw. äquivalente verlustfreie Zustandsübergänge benötigt. UI und UCI-Protokoll führen keine eigene Legalitätslogik.

### 4.2 Fachliche Sonderfälle

- Rochade nur mit vorhandenen Rechten, passendem Turm und bei freien/nicht angegriffenen Königsfeldern.
- En passant nur unmittelbar, wenn die Voraussetzungen vorliegen, und ohne Selbstschach.
- Umwandlung zu Dame, Turm, Läufer oder Springer, einschließlich Schlagen bei Umwandlung.
- Matt vor remisbedingten Zählregeln prüfen, wenn Regelkonflikte auftreten.
- Dreifache Stellungswiederholung und 50-Züge-Regel sind **beanspruchbar**; fünffache Wiederholung und 75-Züge-Regel **automatisch** (unter Beachtung des Vorrangs von Matt).
- Eine Stellung gilt nur dann als tot, wenn **keine mögliche Folge legaler Züge** einer Seite zum Matt führen kann. Vereinfachte Materialtests dürfen keine falschen automatischen Remis auslösen.
- Nur für die Wiederholung relevante Rochade- und En-passant-Möglichkeiten in Stellungsschlüssel aufnehmen; zu beachten sind auch historische Berechtigungen.

### 4.3 Testverfahren

Perft-Werte für die Standardanfangsstellung: Tiefe 1: 20, 2: 400, 3: 8.902, 4: 197.281, 5: 4.865.609. Zusätzliche dokumentierte Teststellungen prüfen Rochaden, Fesselungen, en-passant-bedingtes Selbstschach, Promotions und Schach-Evasion. Ergebnisse stets gegen unabhängige Referenzen vergleichen. Property-Tests: `unmakeMove(makeMove(position)) == position`, keine illegalen Züge, konsistente FEN-Serialisierung.

## 5. Engine und Suche

### 5.1 Erste Implementierung

Die eigene klassische Engine nutzt `Negamax` mit `Alpha-Beta-Pruning`, iterative Suchtiefen, eine anfängliche Material-/Positionsbewertung und eine endliche Suchgrenze. Evaluationskomponenten erhalten feste, dokumentierte Konventionen zur Vorzeichenrichtung und Maßeinheit (üblicherweise Zentibauern). Mattwerte liegen außerhalb des normalen Bewertungsbereichs und berücksichtigen die Mattdistanz.

### 5.2 Vorgesehene Erweiterungen

- Zugordnung (z. B. Hauptvariantenzug, Schläge, Killer-/History-Heuristik).
- Zobrist-Schlüssel und Transposition Tables mit Tiefe, Bound-Typ, Bestmove und generationsspezifischem Ersatz.
- Quiescence Search für taktische Ruhe an Blattknoten.
- Abbruch nach Zeit, Tiefen-/Knotenlimit und externer Anweisung.
- Strukturierte Suchstatistik: Zeit, Knoten, Suchtiefe, Hauptvariante, Bewertungswert.
- Später: Multi-PV, Eröffnungsbücher, Syzygy-Endspieltabellen, verbesserte klassische Positionsbewertung und reproduzierbare, statistische Abstimmung heuristischer Parameter.

Wichtig: Die Engine garantiert **legalen Bestmove**, nicht eine vorab behauptete Elo-Stärke. In einer terminalen Stellung ist der UCI-konforme Nullzug `0000` möglich.

## 6. UCI-Engine-Schnittstelle – Empfängerseite

**Abnahmerelevante Anforderungen:** LH-F-024 und LH-F-033, LH-A-08 sowie LH-A-13 bis LH-A-16.

### 6.1 Betriebsform

Ein eigenständig startbares Kommandozeilenprogramm (Beispiel `sasd-chess-engine --uci`) liest **zeilenweise von stdin**, schreibt **ausschließlich UCI-konforme Zeilen nach stdout** und schreibt Diagnosen bei Bedarf nach stderr. Der Prozess kann ohne Desktop-UI laufen und behält den UCI-Empfang auch während der Suche funktionsfähig.

### 6.2 Unterstützte Grundbefehle für 1.0

| Eingang (Controller → Engine) | Antwort/Verhalten |
| --- | --- |
| `uci` | `id name SASD-Chess …`, `id author …`, ggf. `option …`, danach `uciok`. |
| `isready` | Zeitnah `readyok`, auch während einer Berechnung. |
| `ucinewgame` | Partienspezifische Suchdaten zurücksetzen; danach auf `isready` antworten. |
| `position startpos [moves …]` | Startposition herstellen und Zugfolge legal nachvollziehen. |
| `position fen <FEN> [moves …]` | Validierte Stellung samt optionaler Zugfolge herstellen. |
| `go depth N` | Suche bis zur gewünschten Tiefe. |
| `go movetime M` | Suche mit Zeitbudget in Millisekunden. |
| `go wtime W btime B [winc I] [binc J]` | Uhrabhängige Suche mit verbleibender Zeit und Inkrement (ms). |
| `stop` | Laufende Suche kontrolliert abbrechen und zugehöriges `bestmove` liefern. |
| `quit` | Laufende Arbeiten aufräumen und Prozess geordnet beenden. |
| `setoption name … value …` | Unterstützte Optionen nachvollziehbar anwenden; unbekannte Optionen robust behandeln. |

`go nodes N`, `go infinite`, `ponder` und `ponderhit` werden als mögliche Ausbaupunkte separat geprüft. Für V1.0 müssen angebotene UCI-Optionen **wirklich unterstützt** werden; optionale Befehle dürfen nicht fälschlich zugesichert werden.

### 6.3 Antwortformat und Zustandsautomat

```text
START -> WAIT_FOR_UCI -> READY -> SEARCHING -> READY -> TERMINATED
                               ^      |  ^        |
                               |      +--+ stop   |
                               +-------- bestmove-+
```

- Nach `uci` folgt abschließend `uciok`; nach jedem `isready` folgt `readyok`.
- Ein `go` liefert genau **einen** zugehörigen `bestmove <uci-zug>`-Abschluss; `info …`-Zeilen sind davor erlaubt.
- Züge sind Koordinatenzüge wie `e2e4`, `e7e8q`, **nicht SAN**. Nach `position` validiert der Kern die intern gesetzte Stellung.
- Eingaben sind zeilen- und längenbegrenzt; fehlerhafte, zusätzliche oder unbekannte Zeilen führen nicht zu unkontrollierten Zuständen.
- Während der Suche darf `isready` nicht bis nach Suchende blockieren; `stop` und `quit` müssen verarbeitet werden können.
- Teilweise Ausgabezeilen, Debugging-Ausgaben auf stdout oder unzulässiges Vermischen paralleler Antworten sind untersagt.
- Ein nach Abbruch veralteter Suchlauf darf keinen zweiten Bestmove für einen neueren Suchauftrag ausgeben.

### 6.4 Beispielhafter Protokolldialog

```text
Controller -> Engine: uci
Engine     -> Controller: id name SASD-Chess 0.1
Engine     -> Controller: id author SASD-Chess contributors
Engine     -> Controller: uciok

Controller -> Engine: isready
Engine     -> Controller: readyok
Controller -> Engine: ucinewgame
Controller -> Engine: isready
Engine     -> Controller: readyok
Controller -> Engine: position startpos moves e2e4 e7e5
Controller -> Engine: go depth 6
Engine     -> Controller: info depth 1 score cp 10 nodes 20 time 1
Engine     -> Controller: ... weitere optionale info-Zeilen ...
Engine     -> Controller: bestmove g1f3
Controller -> Engine: quit
```

**Hinweis:** Der Dialog ist **illustrativ**, keine Vorhersage des tatsächlichen Bestmoves oder der gemeldeten Werte.

## 7. UCI-Client / externe Engine-Prozesse

**Abnahmerelevante Anforderungen:** LH-F-033 bis LH-F-040.

### 7.1 Prozessstart und Handshake

Die Match-Verwaltung startet die vom Nutzer konfigurierten Engine-Programme als **getrennte Kindprozesse**, verbindet stdin/stdout, sammelt begrenzte stderr-Diagnostik und führt den UCI-Handshake mit Timeout durch:

1. Engine-Pfad und zulässige Argumente aus einer lokalen Konfiguration lesen.
2. Prozess ohne implizite Shell-Interpretation starten (keine Kommandoinjektion).
3. `uci` senden, `id`/`option` lesen, bis `uciok` kommt.
4. Optional konfigurierbare `setoption`-Werte setzen.
5. `isready` senden und `readyok` abwarten.
6. Zum Partiestart `ucinewgame`, anschließend `isready`/`readyok`.
7. Züge ausschließlich nach legalitätsgeprüftem `bestmove` akzeptieren.
8. Zum Programmende `quit`; bei Bedarf nach Grace-Period beenden und Prozessressourcen freigeben.

Jeder Engine-Prozess besitzt **eigenen** Puffer, Handshake-Status, Such-ID, Zeitbudget, Fehlerstatus und Logkontext. Es gibt keine geteilte stdin/stdout-Leitung zwischen den zwei Engines.

### 7.2 Sicherheit und Fehlerbehandlung

- Ein externer Engine-Prozess ist **lokal ausgeführter Fremdcode**. Die Anwendung muss den Nutzer vor dem Start unbekannter Programme informieren; Pfade dürfen nicht automatisch aus untrusted PGN/FEN/Config-Dateien als Befehl ausgeführt werden.
- Keine Shell, sofern nicht explizit später als unsichere Expertenfunktion freigegeben.
- Größe und Anzahl von UCI-Textzeilen werden begrenzt; stdout **und stderr** werden abgearbeitet, um Pipe-Deadlocks zu vermeiden.
- Kein endloses Warten: Handshake, Antwort und Abschaltung haben konfigurierbare Timeouts.
- Ungültige `bestmove`-Antworten (Syntax, falsches Zugrecht, illegaler Zug, veraltete Antwort) werden abgewiesen, als Engine-/Partiefehler protokolliert und niemals ins Brett übernommen.
- Absturz, Hängenbleiben, Protokollfehler und fehlende Dateiberechtigung werden für den Nutzer verständlich angezeigt.
- Protokollmitschnitt ist optional und ohne private Nutzerdaten oder unkontrollierte Mengen an Text zu speichern.
- Ein Engine-Timeout ist nicht automatisch immer ein durch die Schachregeln gewonnener Zeitverlust. Die Match-Richtlinie definiert **gesondert** Uhrüberschreitung, technisches Timeout und Protokollfehler.

## 8. Engine gegen Engine und Selbstspiel (verbindlicher V1.0-Umfang)

Das ist eine eigenständige **Muss-Funktion**; reine UCI-Engine-Unterstützung genügt nicht.

### 8.1 Spielmodi

| Modus | Prozess A | Prozess B | Ergebnis |
| --- | --- | --- | --- |
| SASD gegen Fremdengine | SASD-Chess | externe UCI-Engine | automatisch gespielte Partie |
| SASD gegen SASD | SASD-Chess Instanz 1 | SASD-Chess Instanz 2 | **vollautomatisches Selbstspiel** |
| Fremdengine gegen Fremdengine | externe UCI-Engine 1 | externe UCI-Engine 2 | generischer UCI-Match-Betrieb |
| Mensch gegen Engine | lokale Eingabe | SASD- oder Fremdengine | manuell/automatisch gemischte Partie |

Der Match-Controller behandelt alle UCI-Prozesse technisch gleich. Zwei eigene Engines dürfen identische Programme mit **unterschiedlichen** Einstellungen/Versionen sein; die Prozesse laufen **parallel**, die eigentlichen Züge werden abwechselnd ausgeführt.

### 8.2 Ablauf einer Engine-Partie

1. Zwei Engine-Konfigurationen auswählen; Weiß/Schwarz zuordnen.
2. Beide Engines separat starten, UCI-Handshake durchführen, Partie initialisieren.
3. Zentrale `GameState`-Instanz und eine **autoritative** Uhr für jede Seite anlegen.
4. Für die aktive Engine die **vollständige aktuelle Zugfolge** mit `position startpos moves …` (oder FEN samt Zügen) synchronisieren.
5. Suchlimits mit `go depth N`, `go movetime M` oder `go wtime … btime …` übergeben.
6. Antwort lesen, anhand des **zentralen Regelkerns** validieren; nur einen legalen, fristgerecht gelieferten Zug ausführen.
7. Uhr, Hash, Zugliste, Brettanzeige und optional PGN/Zeitprotokoll aktualisieren.
8. Anderen Prozess aktivieren, bis ein Endzustand, ein Benutzerabbruch oder ein technischer Abbruch eintritt.
9. Ergebnis und eventuellen technischen Abbruchgrund eindeutig dokumentieren; Prozesse geordnet beenden oder für eine neue Partie initialisieren.

In einer neuen Partie müssen z. B. für die Stellungswiederholung relevante Informationen zentral geführt werden; eine UCI-Engine darf nicht als alleinige Wahrheitsquelle für Partieende dienen.

### 8.3 Konfigurierbarkeit

**Muss V1.0:**
- Engine A / Engine B über ausführbare Datei + optionale Startargumente auswählbar.
- Farbe und je Engine mindestens ein Tiefen- oder festes Zeitlimit konfigurierbar.
- Automatische Partie bis zum Ende **ohne menschliches Klicken pro Zug**.
- Start, Abbruch und sichtbarer Status über eine **ohne Desktop-GUI nutzbare Kommandozeilenschnittstelle**; vollständige Zugliste und PGN-Export.
- Zwei gleichzeitig laufende Kopien **derselben** SASD-Engine sind zulässig.
- Laufzeit-/Protokollfehler führen zu einem nachvollziehbar beendeten oder abgebrochenen Match.

**Soll (LH-F-039/LH-F-040):** Serien mit N Partien, Farbwechsel pro Partie, Zufalls-Seed/Eröffnungsvorgaben, Inkrement-Schachuhr, CSV-/JSON-Zusammenfassung (Siege/Remis/Niederlagen, mittlere Zugzahl, Zeitverbrauch), optionale grafische Live-Ansicht und pausierbare Wiedergabe.

**Kann:** Parallele Multi-Match-Ausführung mit Ressourcenbudget, Elo/SPRT-Auswertung, engineübergreifende Leistungsdiagramme.

### 8.4 Beispielhafter CLI-Aufruf (Entwurf, noch nicht implementiert)

```shell
# Zwei getrennte Prozesse derselben Schach-Engine:
sasd-chess-match --white ./sasd-chess-engine --black ./sasd-chess-engine --depth 6 --pgn selfplay.pgn

# Gegen eine andere UCI-konforme Schach-Engine:
sasd-chess-match --white ./sasd-chess-engine --black ./other-uci-engine --movetime 1000 --pgn vs-other.pgn
```

Das sind **geplante**, noch nicht verfügbare Optionen. Die Match-CLI steuert zwei Prozesse und prüft die Zugfolge zentral; die Engines kommunizieren nicht direkt miteinander. Pfade werden ausdrücklich als ausführbare Dateien behandelt und nicht per Shell interpretiert.

### 8.5 Beispielkonfiguration (Entwurf, noch keine implementierte Syntax)

```yaml
match:
  games: 2
  swap_colors: true
  limit:
    type: movetime
    milliseconds: 1000
  white:
    executable: ./sasd-chess-engine
    arguments: ["--uci"]
  black:
    executable: ./sasd-chess-engine
    arguments: ["--uci"]
  output:
    pgn: results/selfplay.pgn
```

Die konkrete Syntax und der Kommandozeilen-Aufruf werden erst bei Implementierung verbindlich festgelegt. Diese Beispielkonfiguration ist **keine derzeit verfügbare Funktion**.

### 8.6 Testmatrix für beide UCI-Rollen

| Test-ID | Versuch | Erwartung |
| --- | --- | --- |
| PH-UCI-01 | SASD als UCI-Engine an unabhängige GUI koppeln | Handshake und mindestens eine vollständige legale Partie |
| PH-UCI-02 | Fremdengine als Schwarz gegen SASD starten | UCI-Handshake, automatische Zugfolge, valides PGN |
| PH-UCI-03 | Zwei SASD-Instanzen gegeneinander | Zwei getrennte Prozesse, abwechselnde legale Züge ohne Nutzerinteraktion |
| PH-UCI-04 | Ein identisches Engine-Binary für Weiß und Schwarz | Kein gemeinsam genutzter veränderbarer Prozesszustand |
| PH-UCI-05 | Engine meldet illegalen `bestmove` | Zug nicht übernommen; technischer Fehler festgehalten |
| PH-UCI-06 | Engine antwortet nicht auf `uci`/`isready` | Timeout, klare Meldung, Kindprozess bereinigt |
| PH-UCI-07 | Engine hängt während `go` | Stop/Eskalation/Abbruch nach Richtlinie; Anwendung bleibt bedienbar |
| PH-UCI-08 | Match wird manuell abgebrochen | Kein weiterer Zug, Prozesse bereinigt, Teilergebnis gekennzeichnet |
| PH-UCI-09 | Schachmatt/Patt/Remis tritt ein | Keine weitere Suchanfrage; korrekter PGN-Ergebniseintrag |
| PH-UCI-10 | Zwei identische Selbstspiel-Starts mit Tiefenlimit | Bei gleicher Version, Konfiguration und deterministischem Modus reproduzierbare Folge |
| PH-UCI-11 | `stop` und `quit` während Suche | Reaktionsfähiger Prozess und kein veralteter Bestmove |
| PH-UCI-12 | Viele `info`-/stderr-Zeilen | Keine Blockade, begrenzte Logspeicherung |
| PH-UCI-13 | `position fen … moves …` mit Spezialzügen | Synchronisation mit zentralem Regelkern, keine Abweichung |
| PH-UCI-14 | Suchzeit mit realer Schachuhr und Inkrement | Uhr des Match-Controllers ist autoritativ; Zeitmessung korrekt |

## 9. Match-Uhr, Ergebnisse und Reproduzierbarkeit

Für V1.0 muss mindestens ein **festes Suchbudget je Zug** und ein Tiefenlimit unterstützt werden. Sobald ein Modus mit echter Schachuhr implementiert wird, gelten zusätzlich:

- Match-Controller misst monotone verstrichene Zeit; die Engine liefert keine maßgebliche eigene Uhr.
- Je Engine sind Restzeit, Inkrement und optionale Periodenregeln konsistent zu verarbeiten.
- Technische Timeout-Politik (Handshake und Netzwerk/Prozessfehler) ist strikt von Schachuhr-Flagfall getrennt.
- Ergebnisse beinhalten **Schachresultat** (1-0 / 0-1 / 1/2-1/2 / *) und ggf. einen separaten **technischen Ergebnisgrund**.
- Für reproduzierbare Tests werden Engine-Version, Kommandozeile, Optionen, Eingangsstellung, Suchtiefe/Zeiten, Plattform und Hardwareangaben optional protokolliert.
- Im Selbstspiel mit Zeitlimits ist deterministisches Verhalten wegen Laufzeitschwankungen **nicht** garantiert; für feste Reproduktion Tiefen-/Knotenlimits verwenden.

## 10. Formate und Datenflüsse

### 10.1 FEN

Einlesen und Ausgeben von Stellung, aktivem Spieler, Rochaderechten, En-passant-Ziel, Halbzug-/Vollzugzählern. Parser prüft Felder, Zeichenvorrat, Figurenkonstellation und logische Konsistenz. Round-trip-Tests mit Referenzstellungen.

### 10.2 SAN und PGN

Züge werden im Kern legal erzeugt und daraus **SAN** für Menschen bzw. **UCI-Koordinaten** für das Protokoll abgeleitet. PGN-Export umfasst Ergebnis und zweckmäßige Standard-Tags, etwa Event, Site, Date, Round, White, Black, Result; bei ungewöhnlicher Startstellung entsprechende FEN-/SetUp-Tags. Kein unbereinigtes Übernehmen von Engine-`info`-Text in PGN.

### 10.3 Protokoll und Logging

UCI über stdin/stdout, kein eigener proprietärer Zugaustausch als Voraussetzung. Diagnose-Logs gehen an stderr oder in separat konfigurierte Dateien. Datenmengenbegrenzung und UTF-8-Verarbeitung müssen getestet werden.

## 11. Benutzeroberfläche

Die UI hält einen beobachtbaren, **nicht eigenständig veränderbaren** Blick auf den Spielzustand. Sie zeigt 64 Felder, Figuren, Auswahl/Zielfelder, Zugliste, Spielmodus, aktuelle Seite, Schach/Matt/Remis, optional Uhren und Engine-Suchfortschritt.

### 11.1 Bedienabläufe

- Mensch gegen Mensch: Brettaktion → Kommando an Game Controller → Legalitätsprüfung → sichtbare Aktualisierung.
- Mensch gegen Engine: Mensch zieht → Match/Game Controller fordert Engine-Zug an → UI bleibt bedienbar → legale Antwort wird angezeigt.
- Engine gegen Engine: Beide Engine-Prozesse werden konfiguriert → Start → **keine Benutzereingabe pro Zug nötig** → Livebrett und Zugliste laufen automatisch → Ergebnis wird angezeigt.
- Programmabbrüche oder ungültige Engine-Ausgaben werden als technische Fehler sichtbar und nicht als „Schachmatt“ dargestellt.
- Handbedienung mit Quell- und Zielfeld muss auch ohne Drag-and-drop möglich sein.

**GUI-Framework:** vorerst offen. Ein Adapter zum separaten SASD-UI-Toolkit kann geprüft werden, ist keine Voraussetzung für die Core-Entwicklung. Windows/Linux sind die ersten verbindlich vorgesehenen Plattformen.

## 12. Build- und Verzeichnisstruktur

Geplante CMake-Struktur, nicht als bereits vorhandene Dateien zu verstehen:

```text
SASD-Chess/
├── CMakeLists.txt
├── CMakePresets.json                    # bei Bedarf
├── include/sasd/chess/
│   ├── core/
│   ├── formats/
│   ├── engine/
│   ├── uci/
│   └── match/
├── src/
│   ├── core/
│   ├── formats/
│   ├── engine/
│   ├── uci/
│   ├── match/
│   └── ui/
├── apps/
│   ├── engine-main.cpp                  # separater UCI-Prozess
│   ├── match-main.cpp                   # Match-Controller (CLI)
│   └── desktop-main.cpp                 # GUI
├── tests/
│   ├── core/
│   ├── perft/
│   ├── formats/
│   ├── engine/
│   ├── uci/
│   └── match/
├── docs/
│   ├── Lastenheft.md
│   ├── Pflichtenheft.md
│   ├── Architektur.md                  # später
│   └── Prüfprotokoll.md                # später
└── .github/workflows/
```

Die Bibliotheken `chess-core`, `chess-formats`, `chess-engine`, `chess-uci-engine`, `chess-uci-client` und `chess-match` sollen unabhängig von der Desktop-UI baubar und testbar sein. Über CMake/CTest werden Unit-, Integrations-, Perft- und Protokolltests ausgeführt. CI-Matrix zunächst mindestens Linux und Windows, ggf. mit GCC/Clang/MSVC.

## 13. Qualitätsstrategie und Testpyramide

**Ebene A – Unit-Tests:** Brettoperationen, Parser, Bewegungsregeln, Zugrücknahme, Stellungsgleichheit, Bewertungsfunktion, UCI-Parser.

**Ebene B – Regeltests:** Perft; FEN-Spezialstellungen; Matt, Patt, tote Stellung, Wiederholung, 50-/75-Züge-Regel; Round-trip von Zug/Rücknahme, FEN und PGN.

**Ebene C – Prozess-/Protokolltests:** Test-Engines simulieren Verzögerungen, fehlerhafte Ausgaben, Prozessabsturz, stdout-/stderr-Fluten und verweigerte Handshakes. UCI-Engine mit eigenem CLI-Tester prüfen.

**Ebene D – End-to-End-Tests:** Match-Controller startet zwei UCI-Prozesse, spielt Partien bis Ende, kontrolliert Legalität, Ergebnis, PGN, Aufräumverhalten und reproduzierbare Suchgrenzen.

**Ebene E – Performance/Spielstärke:** Perft- und Suchtiefe/Knoten-Benchmarks getrennt führen. Spielstärke nur über dokumentierte Vergleichspartien unter konstanten Einstellungen schätzen. Keine Tests durch Optimierungsmaßnahmen umgehen.

### 13.1 Abnahmematrix

| Lastenheft | Umsetzung/Nachweis |
| --- | --- |
| LH-F-001 bis LH-F-010 | Core + GameState, Unit-/Regel-/Perft-Tests |
| LH-F-011 bis LH-F-020 | UI, manuelle Szenario- und Bedienungstests |
| LH-F-021 bis LH-F-023 | Engine, feste Teststellungen, Suchlimit-/Abort-Tests |
| LH-F-024 | UCI-Engine-Prozess, PH-UCI-01 und PH-UCI-11 |
| LH-F-025 bis LH-F-027 | FEN/SAN/PGN-Parser/Exporter, Round-trip-/Kompatibilitätstests |
| LH-F-033 bis LH-F-040 | UCI-Client + Match-Controller, PH-UCI-01 bis PH-UCI-14 |
| LH-N-001 bis LH-N-015 | CTest/CI, Architekturprüfung, Offline-, Robustheits-, Ressourcen- und Sicherheitsprüfungen |
| LH-A-01 bis LH-A-16 | Testprotokoll mit exakter Version, Datum, Umgebung und Ergebnis |

## 14. Entwicklungsetappen und Definition of Done

| Meilenstein | Liefergegenstand | Definition of Done |
| --- | --- | --- |
| M0 | Projektbasis, CMake, CTest, CI-Start | Linux-/Windows-Build erfolgreich, Test-Harness lauffähig |
| M1 | Core, Zugerzeugung, Rücknahme, Partieende | Perft und Sonderregeltests grün |
| M2 | FEN, SAN, PGN, Game Controller | Vollständige Partie mit konsistentem Import/Export |
| M3 | Klassische Engine, UCI-Engine-Prozess | Legalität, Suchabbruch und UCI-Engine-Tests grün |
| M4 | UCI-Client, Match-Controller, Selbstspiel | **Zwei SASD-Instanzen** + Fremdengine automatisch getestet |
| M5 | Desktop-Oberfläche und Bedienung | Mensch/Mensch und Mensch/Engine in der GUI; Engine/Engine mindestens per Match-Controller bedienbar |
| M6 | Release-Stabilisierung | Pflichtanforderungen nachgewiesen, Doku/CI/Binär-Releases geprüft |

Bei jeder Etappe müssen alle betroffenen Tests grün, Dokumentation und Architekturänderungen nachvollziehbar sowie neue Schnittstellen ausreichend kommentiert sein. **Ein Meilenstein gilt nicht durch bloße Codeexistenz als abgeschlossen.**

## 15. Risiken und technische Entscheidungen

| Risiko / Entscheidung | Handhabung |
| --- | --- |
| Fehler in Zugerzeugung oder Rücknahme | Core zuerst mit Perft und Spezialstellungen absichern; GameState zentral halten |
| Deadlock zwischen zwei Engine-Prozessen | Unabhängige asynchrone Reader und bounded queues, beide Ausgabekanäle abarbeiten |
| UCI-Abweichungen externer Engines | Definierter Mindestumfang, toleranter Parser, strikte Zuglegalität |
| Hintergrundsuche blockiert UI/Protokoll | Getrennter Suchthread/-task, atomarer Abbruch, klarer Lifecycle |
| Gleiche Engine zweimal im Prozess | **Nicht** über Singleton teilen: getrennte Prozesse und Konfigurationen |
| Unterschiedliche Zeitkontrollen verfälschen Matches | Controller ist Zeitquelle; feste Tiefenlimits für reproduzierbare Tests |
| Optimierung gefährdet Nachvollziehbarkeit | Erst saubere Baseline und Tests, danach messbare Optimierung |
| Lizenz/Drittanbieter-Grafiken | Lizenzentscheidung und Prüfung vor öffentlichem 1.0-Release |
| Zusätzliche Plattformen | Portabler Core; UI und Prozessstart als adaptergebundene Bausteine |

## 16. Referenzen und dokumentierte Schnittstellen

- Fachliche Anforderungen: [Lastenheft](Lastenheft.md).
- UCI-Protokoll (historische Normfassung): https://backscattering.de/chess/uci/
- Internationale Schachregeln: FIDE Laws of Chess (maßgeblicher Regelstand vor Freigabe der 1.0 dokumentieren).
- FEN/PGN/SAN: übliche Standards und unabhängig überprüfte Referenzbeispiele.
- Referenz-Engines und Match-Tools können **zum Testen** verwendet werden; ihre Implementierungen werden nicht ungeprüft übernommen.

## 17. Noch zu treffende Entscheidungen

1. Konkretes GUI-Framework und unterstützte Betriebssystemversionen.
2. Open-Source-Lizenz des Projekts.
3. Erste verbindliche CLI-Syntax des Match-Controllers und dessen Konfigurationsdateiformat.
4. Umfang von Inkrement- und Turnierzeitkontrollen für 1.0 oder später.
5. Ressourcengrenzen/Default-Timeouts für Fremdprozesse und Testhardware.
6. Dokumentations- und Namenskonventionen (beispielsweise deutschsprachige Handbücher und englische C++-Bezeichner/Kommentare).

**Freigabevermerk:** Dieses Pflichtenheft ist ein abgestimmter technischer **Vorschlag** und wird erst nach expliziter Bestätigung des Umfangs zu einer verbindlichen Entwicklungsbaseline.


## 18. Versionshistorie und Änderungssteuerung

| Version | Datum | Änderung |
| --- | --- | --- |
| 0.1 | 27.10.2026 | Technischer Lösungsansatz: Schachkern, Game-Controller, klassische Suche, FEN/SAN/PGN und UCI. |
| 0.2 | 27.10.2026 | UCI-Client, zwei unabhängige Engine-Prozesse, Match-Controller, Selbstspiel, Fehler-/Zeitkontrollen und Tests konkretisiert. |
| 0.3 | 27.10.2026 | Geltungsbereich auf klassische Schachsoftware präzisiert; nicht zugehörige technische Ausbaustufen aus dem Lösungsentwurf entfernt. |

Jede künftige Änderung an funktionalen Anforderungen erhält eine referenzierbare `LH-F-*`-Kennung beziehungsweise verweist auf bestehende Kennungen, dokumentiert ihre Abnahmewirkung und nennt den technischen Nachweis. Zurückgezogene Kennungen werden nicht anderweitig neu vergeben.
