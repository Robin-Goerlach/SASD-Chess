# SASD-Chess – Test- und Abnahmeplan

> **Dokument:** QA-001 · **Version:** 0.1 · **Stand:** 27.10.2026  
> **Status:** Geplanter Testprozess, **keine Aussage über bereits bestandene Prüfungen**  
> **Bezug:** [Lastenheft v0.4](Lastenheft.md) · [Pflichtenheft v0.4](Pflichtenheft.md) · [Architektur v0.2](Architektur.md) · [Roadmap](../ROADMAP.md)

## 1. Zweck, Geltungsbereich und Nachweisprinzip

Dieses Dokument macht fachliche Anforderungen und Architekturversprechen für die Entwicklung von **SASD-Chess** überprüfbar. Es behandelt klassische Schachregeln, Spielverwaltung, eigene Such-Engine, UCI in beiden Rollen, Prozess-Matches, Desktop-Anwendung und schachbezogenes Laboratory.

Ein grüner Compilerlauf **beweist keine Schachkorrektheit**. Perft belegt die Anzahl legaler Zugfolgen für geprüfte Positionen, **nicht** allein vollständige FIDE-Regeltreue oder strategische Spielstärke. Eine absolvierte Schachpartie belegt nicht alle Protokoll- und Fehlerfälle.

Für jeden abgenommenen Test sind **Test-ID, Git-Commit, Umgebung/Compiler/Buildmodus, Eingabe, Sollwert/Erwartung, Ergebnis, relevante Log-/PGN-Datei und Datum** nachvollziehbar zu dokumentieren. Fehlende oder nicht ausgeführte Tests bleiben offen und gelten nicht als bestanden.

## 2. Testkategorien

| Kategorie | Gegenstand | Standardhäufigkeit |
| --- | --- | --- |
| **Unit** | Werttypen, Zuggeometrie, Königssicherheit, Parser, Evaluationsfunktionen | Bei jeder betroffenen Änderung |
| **Invariant/Property** | `make/unmake`, Snapshot-Gleichheit, keine Zustandsänderung bei Fehlern | Bei jeder Core-/Game-Änderung |
| **Perft/Divide** | Referenz-FENs, Sonderzüge, Blattzahlen und Teilzweigsummen | Bei jeder Core-Änderung |
| **Game-Regeln** | Matt/Patt/Remis, Historie, Anspruch, Uhr-/Resultatsemantik | Bei jeder Game-/Match-Änderung |
| **Format** | FEN/SAN/PGN, fehlerhafte Eingaben, Serialisierung und unabhängiges Nachspielen | Bei jeder Formatänderung |
| **Search** | Legalität, Terminalzustände, Abbruch, Tiefen-/Zeitlimits | Bei jeder Suchänderung |
| **UCI Host** | Handshake, Steuerbefehle, stdout-Reinheit, parallele Stop-/Ready-Signale | Bei jeder UCI-Host-Änderung |
| **UCI Client/Process** | Kindprozesse, Timeouts, Pipes, beschädigte Antworten und Cleanup | Bei jeder Process-/Client-Änderung |
| **Match E2E** | Selbstspiel, Fremdengine, Zeit-/Fehlerfälle, PGN | Vor M4-/M6-Freigabe |
| **Desktop E2E** | Mensch/Mensch, Mensch/Engine, Promotion, Undo, Dateiaktionen | Vor M5-/M6-Freigabe |
| **Benchmarks** | Knoten, Laufzeit, Speicher, Daten-/Konfigurations-Metadaten | Vor/nach Optimierung |
| **Laboratory** | Diagnostik, Vergleiche, Trace-Limits, statistische Auswertung, Reports | Erst bei Implementierung der jeweiligen Zusatzfunktion |

Kurze deterministische Tests gehören zu regulärem CTest; teure Langläufe werden separat etikettiert. Referenz-FENs erhalten eine Quellenangabe; eigens konstruierte Stellungen werden als solche markiert und unabhängig geprüft.

## 3. Regelkern: Referenzdaten und Perft

### 3.1 Verbindliche Anfangsstellung

```text
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1
```

| Tiefe | Erwartete Perft-Blätter | Status zu Beginn |
| ---: | ---: | --- |
| 0 | 1 | Geplant |
| 1 | 20 | Geplant |
| 2 | 400 | Geplant |
| 3 | 8.902 | Geplant |
| 4 | 197.281 | Geplant |
| 5 | 4.865.609 | Geplant |
| 6 (separater Langtest) | 119.060.324 | Geplant |

**Divide:** Bei `depth >= 1` muss die Summe der Teilzahlen aller legalen ersten Züge exakt dem Gesamtergebnis `perft(position, depth)` entsprechen. Für `depth = 0` ist `perft = 1`, ein Divide-Ausgabeformat wird explizit festgelegt (z. B. leere Liste ohne Summenbehauptung).

**Grenzfälle:** Rochade/angegriffene Durchgangsfelder, aufgedecktes Schach nach En passant, Bauernpromotion unter Schach, doppelte Schachbedrohung, unterbrochene Linienfiguren, Zug mit Matt, nicht erlaubter Königsschlag. Eine zusätzliche, extern belegte Referenz-FEN-Sammlung ist vor Abschluss von M1 Pflicht.

### 3.2 Verlustfreie Zugrücknahme

Für jede legale Teststellung und jeden legalen Kandidaten:

1. Originalstellung mit **allen** Statusfeldern kopieren.
2. Zug ausführen, danach vollständig zurücknehmen.
3. Alle Felder, Zugrecht, En-passant-Feld, Rochaderechte, Halb-/Vollzugzähler sowie definierte Hashes vergleichen.
4. Zusätzlich mehrere legale Züge verschachtelt ausführen und in umgekehrter Reihenfolge zurücknehmen.

Falsche oder unvollständige Undo-Daten gelten als **Core-Defekt**, auch wenn einzelne Perft-Werte zufällig korrekt sind.

### 3.3 Schach- und Remissemantik

Tests müssen mindestens unterscheiden:

- **Matt:** keine legalen Züge, eigener König angegriffen.
- **Patt:** keine legalen Züge, König nicht angegriffen.
- **Dreifache Wiederholung:** korrekt beanspruchbarer Zustand, **nicht** stillschweigend als automatisch beendet.
- **Fünffache Wiederholung:** automatisches Remis.
- **50-Züge-Regel:** korrekt beanspruchbar, auch bei regelgerecht angekündigtem beabsichtigten Zug.
- **75-Züge-Regel:** automatisches Remis, soweit kein vorrangiges Matt des letzten Zugs.
- **Tote Stellung:** ausschließlich fachlich belegte Fälle; Beispiele, in denen eine naive Materialheuristik falsch liegt.
- **FEN ohne Historie:** erzeugt keinen erfundenen Wiederholungsanspruch.
- **Uhr-Flagge:** fehlende Mattmöglichkeit der Gegenseite ist gemäß Regelwerk zu berücksichtigen.
- **Unberechtigter Remisanspruch:** keine ungewollte partielle Zustandsänderung.

Die exakte FIDE-Regelreferenz und die Abnahme fachlich schwieriger Randfälle sind vor 1.0 zu dokumentieren.

## 4. FEN, SAN, PGN

**FEN:** gültige Start- und Spezialstellungen, fehlende Felder, fehlerhafte Zahlen, falsche Königsanzahl, widersprüchliche Rechte, ungültige Zeichen und übergroße Eingaben. Der Parser muss präzise Fehler liefern. Die Serialisierung muss die Stellung korrekt reproduzieren, ohne zu behaupten, dass ein FEN vollständige Historie enthält.

**SAN:** Rochade `O-O`/`O-O-O`, Schlagfälle, Zweideutigkeitsauflösung, Promotions, Schach/Schachmatt. SAN-Ausgabe aus bekannten legalen Zugfolgen mit unabhängigem Parser vergleichen.

**PGN:** Tags, Zugnummern, korrektes Ergebnis, optional FEN-Startposition; externe Lesbarkeit und erneutes Nachspielen verifizieren. PGN-Import ist Soll und wird, sobald vorhanden, gesondert mit Varianten, beschädigten Dateien und großen Eingaben geprüft. Keine unvalidierte PGN-Partie direkt zum autoritativen Zustand machen.

## 5. Klassische Suche und Bewertungslogik

| Test | Erwartung |
| --- | --- |
| Legale Stellung | Gelieferter Bestmove ist legal |
| Kein legaler Zug | Terminaler Zustand; UCI-Host kann `bestmove 0000` ausgeben |
| Matt-/Remisbewertung | Keine gewöhnliche Materialheuristik statt terminalem Wert |
| Feste Tiefe | Reproduzierbare Ergebnisse bei identischer Implementierung und deterministischer Konfiguration |
| Sehr kurze Suche / Stop | Kontrollierte Rückkehr, legaler bisher bekannter Kandidat |
| `quit` während Suche | Threads/Prozesse terminieren kontrolliert |
| Ungültiges Budget | Validierter Fehler oder dokumentierte Begrenzung |
| Bewertung aufgeschlüsselt (L1) | Summierte Terme stimmen mit statischer Bewertung überein |
| Suchoptimierung | Regel-/Perft-/Legalitätstests bleiben unverändert grün |

**Nicht verwechseln:** Eine höhere `nps`-Rate bedeutet nicht zwingend höhere Spielstärke. Ein strategischer Taktiktest darf nicht jeden alternativen legalen Zug automatisch als Core-Fehler werten.

## 6. UCI-Verifikation

### 6.1 Eigener UCI-Host

Mindestens:

- `uci` liefert Identifikation und `uciok`, `isready` liefert `readyok`.
- `ucinewgame` setzt partienbezogenen Suchzustand zurück; `position startpos` und `position fen ... moves` gelten als vollständige Positionsanweisungen.
- `go depth`, `go movetime` und Uhrparameter werden im dokumentierten Umfang verarbeitet.
- `stop` darf während der Suche nicht blockieren und beendet den aktiven Auftrag korrekt.
- Genau **eine** `bestmove`-Antwort je abgeschlossener aktiver Suche; erlaubte `info`-Zeilen werden korrekt serialisiert.
- Nichtprotokollgerechte Debugausgaben fehlen auf `stdout`, stehen bei Bedarf auf `stderr`.
- `quit` beendet die Anwendung auch bei laufender Suche sauber.
- Unbekannte/ungültige Eingaben beschädigen keinen Such- oder Partiezustand.

### 6.2 UCI-Client und Prozesse

Mit **kleinen Mock-UCI-Programmen** automatisiert prüfen: fehlendes `uciok`/`readyok`, ungültige `bestmove`-Syntax, `bestmove 0000` bei vorhandenen legalen Zügen, stderr-Flut, sehr lange Zeile, EOF/Crash, verzögerter alter Suchabschluss, blockierende Ausgabekanäle, Timeout und fehlende Ausführungsberechtigung.

Testen, dass der Matchcontroller fehlerhafte Prozesse nicht unbegrenzt am Leben erhält. **Ein externer UCI-Prozess ist nicht als Sandbox zu betrachten.**

## 7. Match-Controller und Selbstspiel

| Test-ID | Szenario | Erfolgskriterium |
| --- | --- | --- |
| MT-01 | SASD gegen unabhängige UCI-Engine | Vollständig legale, protokollierte Partie |
| MT-02 | **Dieselbe SASD-Binärdatei zweimal** | Zwei getrennte Kindprozesse; gültiges Ergebnis |
| MT-03 | Andere Farben/Startstellung | Spielstand, Zugrecht und Ergebnisse korrekt zugeordnet |
| MT-04 | Illegaler Bestmove | Keine Zustandsänderung; klarer Fehler-/Reglementstatus |
| MT-05 | Timeout bei Handshake | Kein Hängen/Prozessleck; verständliche Diagnose |
| MT-06 | Absturz während Suche | Partie-/Logstand erhalten; kontrollierte Bereinigung |
| MT-07 | Schachuhr vs. technischer Timeout | Unterschiedliche Ursachen und Wertungen |
| MT-08 | Positionssynchronisation | `Game`, UCI-Zugfolge und PGN stimmen überein |
| MT-09 | Reproduzierbarer Tiefentest | Festes Suchlimit mit dokumentierter Konfiguration |
| MT-10 | PGN-Interop | Unabhängiger PGN-Leser bestätigt Zugfolge/Resultat |
| MT-11 | Abbruch/Shutdown | Keine aktiven Leichenprozesse oder blockierten Pipes |
| MT-12 | Spezialzüge via UCI | Promotion, En passant und Rochade korrekt |

**Reale Interoperabilität:** Gegen eine tatsächliche fremde Engine testen und dabei Version/Quelle und Installationsbedingungen festhalten. Diese externe Engine muss **nicht** in jedem Pull-Request-CI-Lauf installiert sein; Mocks sichern die reguläre CI.

## 8. Desktop und Anwenderabnahme

Manuelle End-to-End-Szenarien mit nachvollziehbarer Schrittfolge:

1. Mensch/Mensch – neue Partie, normale Züge, Schachmatt/Remis, Zurücksetzen.
2. Mensch/Engine – als Weiß und Schwarz, Start/Stop ohne UI-Blockade.
3. Sonderzug – Rochade, En passant, Umwandlung in alle vier Figuren.
4. Rücknahme – alle Sonderrechte und Zugliste korrekt wiederherstellen.
5. Import/Export – FEN laden, PGN exportieren und mit externem Werkzeug lesen.
6. Negativeingabe – illegaler Zug, kaputtes FEN, Abbruch einer Engine.
7. Bedienung – Aktionen auch ohne verpflichtendes Drag-and-drop, erkennbarer Fokus und Status.

Die konkreten Schritte und Testergebnisse werden mit dem ersten lauffähigen Desktop-Build versioniert.

## 9. Abnahme-Matrix Lastenheft 1.0

| Lastenheft-Abnahme | Verantwortlicher Test-/Meilensteinbereich |
| --- | --- |
| LH-A-01 | Standard-Perft, M1 |
| LH-A-02 | Referenz-Sonderstellungen, M1/M2 |
| LH-A-03 | Vollständige Desktop-Partie, M5 |
| LH-A-04 | Promotionsauswahl, M1/M5 |
| LH-A-05 | FEN-Ein-/Ausgabe und fehlerhafte FEN, M2 |
| LH-A-06 | PGN/SAN-Export und unabhängiger Leser, M2 |
| LH-A-07 | Such-Legalität, Abbruch, Limits, M3 |
| LH-A-08 | UCI-Konformität, M3 |
| LH-A-09 | Undo-/Game-Konsistenz, M1/M2 |
| LH-A-10 | Build/Tests Windows/Linux, M0/M6 |
| LH-A-11 | Vollständiger Offline-Betrieb, M5/M6 |
| LH-A-12 | Vollständigkeit der Benutzer-/Entwicklungsdokumentation, M6 |
| LH-A-13 | Fremdengine-Match, M4 |
| LH-A-14 | **Zwei-Prozess-Selbstspiel**, M4 |
| LH-A-15 | Prozess-/UCI-/Zeitfehlerfälle, M4 |
| LH-A-16 | UCI-/Game-/PGN-Synchronisation, M4 |
| LH-A-17 | **Perft Divide**, M1 |

**Abnahmeregel:** 1.0 darf erst als erfüllt dargestellt werden, wenn sämtliche Muss-Anforderungen inklusive ihrer Nachweise erfüllt sind. Bestehen einzelne Regelgrenzen, ist die fachliche Abnahme neu zu entscheiden und nicht stillschweigend als erfolgreich zu markieren.

## 10. Laboratory-Verifikation nach Umsetzung

| Test-ID | Anforderung | Prüfansatz |
| --- | --- | --- |
| LAB-T-01 | LH-F-051 | Perft/Divide-Summen, **Pflicht in M1** |
| LAB-T-02 | LH-F-044 | Bewertungsaufschlüsselung konsistent |
| LAB-T-03 | LH-F-055 | Off/Stats/Trace: fachlich konsistente Ergebnisse, Instrumentierungs-Overhead dokumentiert |
| LAB-T-04 | LH-F-047 | Lauf-/Build-/Korpusmetadaten vollständig und nachvollziehbar |
| LAB-T-05 | LH-F-046 | Optimierungsschalter/Ablation nachvollziehbar; identische Suchsemantik bei Vergleich vorausgesetzt |
| LAB-T-06 | LH-F-048 | Farbwechsel, gepaarte Startstellungen, Ergebnis-/Unsicherheitsauswertung |
| LAB-T-07 | LH-F-049/LH-F-056 | Export-/Import-Roundtrip und Schema-Versionen |
| LAB-T-08 | LH-F-045 | Trace-Grenzen bzgl. Ereigniszahl, Laufzeit und Speicher |
| LAB-T-09 | LH-F-052 | PGN-Stellungsfolge/Engine-Annotationen korrekt |
| LAB-T-10 | LH-F-050 | Bericht verweist auf Messdaten, Konfiguration und Einschränkungen |

**Release-Gate:** LAB-T-01 ist V1.0-pflichtig. LAB-T-02 bis LAB-T-10 sind erst beim jeweiligen Laboratory-Ausbau verbindlich, nicht automatisch für das erste Release.

## 11. CI, Testdauer und Plattformmatrix

- **Linux:** GCC, Clang (soweit im Runner eingerichtet); Debug/Release-Builds.
- **Windows:** MSVC; Debug/Release.
- **Zusätzliche Systeme:** BSD/macOS nach technischer Verfügbarkeit und dokumentierter Entscheidung.
- **Reguläre CI:** Konfiguration, Build, CTest-Unit, Kern-/Sonderfälle, kleine Perft-Tiefen, Format- und Mock-UCI-Protokolltests.
- **Langläufe:** Perft Tiefe 6, längere Engine-Turniere, umfangreiche Benchmarks und Sanitizer-Profile bei Bedarf separat.
- **Benchmark-Fairness:** Zeit-/Hardwarevergleiche nicht ohne Kontrolle von Compilerflags, CPU/Last, Messwiederholungen und Diagnoseeinstellungen publizieren.

**Beispielbefehle erst ab erfolgreichem M0-Setup:**

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
ctest --test-dir build --output-on-failure
```

MSVC mit einem Multi-Config-Generator benötigt zusätzlich `--config Debug` für Build und CTest. Noch nicht existierende CMake-Targets oder Flags dürfen nicht als bereits lauffähig behauptet werden.

## 12. Testbericht – Vorlage

```text
Testlauf-ID:
Datum:
Repository/Branch/Commit:
Meilenstein / LH-Anforderung:
Betriebssystem/Version:
Compiler und Compiler-Version:
Buildmodus, Optionen und Abhängigkeiten:
Testbefehl:
Testdatensatz / FEN / PGN / Matchkonfiguration:
Erwartetes Ergebnis:
Tatsächliches Ergebnis:
PASS / FAIL / BLOCKED / NOT RUN:
Dauer (falls gemessen):
Abweichungen / Issues:
Verweise auf Logs/Artefakte:
```

**Wichtig:** Ein Testbericht ist nur ein Nachweis, wenn er auf tatsächlich ausgeführten Prüfungen basiert. Vorlagen oder beispielhafte Logs sind nicht als „PASS“ zu markieren.

## 13. Pflege und Versionshistorie

Wenn Anforderungen, Testfälle oder Meilensteine geändert werden, sind ihre `LH-*`, `MT-*` und `LAB-T-*`-Bezüge sowie der Testplan zu prüfen. Vor der 1.0 wird der Testkorpus zur betreffenden Release-Baseline versioniert.

| Version | Datum | Änderung |
| --- | --- | --- |
| 0.1 | 27.10.2026 | Umfassender Test- und Abnahmeplan mit Core-/Game-/UCI-/Match-/Desktop- und Laboratory-Nachweisen. |
