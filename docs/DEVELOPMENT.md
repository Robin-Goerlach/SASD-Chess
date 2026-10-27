# SASD-Chess – Entwicklungs- und Buildhandbuch

> **Dokument:** DEV-001 · **Version:** 0.1 · **Stand:** 27.10.2026  
> **Status:** Zielworkflow **ab Meilenstein M0**. Solange CMake- und Quelldateien fehlen, sind die Befehle noch **nicht** ausführbar.

## 1. Zweck und aktueller Stand

Dieses Handbuch beschreibt die vorgesehene lokale Entwicklungsumgebung, die Build-/Testabläufe und ein nachvollziehbares Vorgehen beim Debugging des klassischen C++-Schachprogramms **SASD-Chess**.

Die fachlichen und technischen Grundlagen sind [Lastenheft](Lastenheft.md), [Pflichtenheft](Pflichtenheft.md), [Architektur](Architektur.md), [Roadmap](../ROADMAP.md), [AGENTS.md](../AGENTS.md) und [Testplan](TESTPLAN.md).

**Wichtig:** Die vollständige Entwicklungsinfrastruktur wird **erst im Meilenstein M0** implementiert. Dokumentierte Kommandos sind deshalb zunächst Zielkonventionen. Keine der nachstehenden Ausgaben ist ein behaupteter erfolgreicher Projektbuild.

## 2. Vorgesehene Werkzeuge

| Werkzeug | Vorgabe / Ziel |
| --- | --- |
| C++-Compiler | GCC/Clang unter Linux und MSVC unter Windows |
| C++-Sprachversion | **C++20**, keine C++23-Pflicht |
| Buildsystem | **CMake**, Version wird in M0 getestet und anschließend als Minimum dokumentiert |
| Testausführung | **CTest**, Testframework-Auswahl per ADR |
| Git | Nachvollziehbare Branches/Commits und vollständige Commit-Kennung in Testprotokollen |
| Debugger | GDB/LLDB unter POSIX, Visual Studio Debugger unter Windows |
| CI | GitHub Actions, mindestens Windows/Linux |
| Format-/Lint-Werkzeuge | Konkrete Version und Regeln später per ADR entscheiden |

Die konkrete Versionenmatrix wird bei der ersten erfolgreichen CI-Baseline ergänzt; keine noch ungeprüften Mindestversionen erfinden.

## 3. Geplanter lokaler Build – Linux (ab M0)

```sh
git clone https://github.com/Robin-Goerlach/SASD-Chess.git
cd SASD-Chess

cmake -S . -B build/debug -DCMAKE_BUILD_TYPE=Debug
cmake --build build/debug
ctest --test-dir build/debug --output-on-failure
```

Für Release:

```sh
cmake -S . -B build/release -DCMAKE_BUILD_TYPE=Release
cmake --build build/release
ctest --test-dir build/release --output-on-failure
```

Bei Bedarf Compiler vor CMake-Konfiguration wählen und einen **neuen** Buildordner verwenden. Clang/GCC-Builds dürfen nicht blind denselben CMake-Cache weiterbenutzen.

## 4. Geplanter lokaler Build – Windows (ab M0)

In einer geeigneten Visual-Studio-Developer-PowerShell mit installiertem MSVC und CMake:

```powershell
git clone https://github.com/Robin-Goerlach/SASD-Chess.git
cd SASD-Chess

cmake -S . -B build\vs
cmake --build build\vs --config Debug
ctest --test-dir build\vs --config Debug --output-on-failure
```

Alternativ Release:

```powershell
cmake --build build\vs --config Release
ctest --test-dir build\vs --config Release --output-on-failure
```

Die Befehle können je nach Generator leicht variieren. **Unter Multi-Config-Generatoren ist `CMAKE_BUILD_TYPE` alleine nicht ausreichend**; `--config` bestimmt den Test-/Buildmodus.

## 5. Trennung zwischen Core, Engine, Match und GUI

Die Zielstruktur folgt [Architektur §14](Architektur.md); die tatsächlichen CMake-Targetnamen werden in M0 verbindlich festgelegt.

- **Core/Game:** Keine GUI-Abhängigkeit, keine OS-Prozessoperationen, keine UCI-Ausgaben.
- **Formats:** FEN/SAN/PGN, robustes Parsen und Serialisieren.
- **Search:** Klassische Strategie-Engine, Suchlimits, Unterbrechung und optionale Diagnosedaten.
- **UCI Host:** Eigenständig ausführbare Engine mit stdin/stdout-Protokoll.
- **UCI Client und Match:** Externe Programme über getrennte Kindprozesse steuern; Schiedsrichter besitzt den autoritativen Spielstand.
- **Desktop:** Eigenständiger Präsentationsadapter; konkretes GUI-Framework bleibt bis zum Spike offen.
- **Laboratory:** Bei späterem Ausbau nur bestehende Module wiederverwenden; keine eigene Zuglegalität.

Ein Build **ohne Desktop** muss möglich bleiben. Geplante Buildoptionen wie `SASD_CHESS_BUILD_DESKTOP` sind erst nach Implementierung als tatsächlich verfügbar auszugeben.

## 6. Geplanter Debugging-Ablauf

### 6.1 Falsche Zugerzeugung

1. Eine minimale **legale FEN** reproduzieren.
2. Tatsächliche und bekannte Perft-Blattzahlen bei der kleinsten fehlerhaften Tiefe vergleichen.
3. Perft **Divide** aktivieren; betroffenen ersten Zug isolieren.
4. Bei Bedarf den betroffenen Zweig weiter vertiefen.
5. Sonderrechte, `make/unmake`, Königssicherheit und Angriffsfelder untersuchen.
6. Regressionstest mit erklärender Referenz und erneutem Perft/Divide anlegen.

**Niemals:** Ein bekannt falsches Perft-Ergebnis als neuen Referenzwert übernehmen, nur um Tests grün zu machen.

### 6.2 Fehler bei Partieende/Remis

Die Partiehistorie überprüfen, nicht nur die momentane FEN. Dreifache Wiederholung/50-Züge-Regel sind Anspruchsbedingungen; fünffache Wiederholung/75-Züge-Regel automatische Endzustände. Regelprioritäten sowie FIDE-Fälle von **toter Stellung** sorgfältig prüfen. Regelbewertung und technische Match-Abbrüche strikt auseinanderhalten.

### 6.3 Suchalgorithmus liefert schlechten oder illegalen Zug

Illegale Züge sind ein **Korrektheitsdefekt**. Ein strategisch schlechter, aber legaler Zug ist zunächst eine Bewertungs-/Suchfrage. Suchstand mit konkreter FEN, Engine-/Commit-ID, Tiefe, Zeitlimit und Diagnoseprofil festhalten. Prüfen, dass terminale Stellungen, Vorzeichen der Bewertung, Mattwerte und Abbruchfallback korrekt sind.

### 6.4 UCI-Match hängt

Jeden Prozess einzeln betrachten:

1. UCI-Handshake `uci/uciok` und `isready/readyok`?
2. Bekam **nur** die am Zug befindliche Engine `position` und `go`?
3. Wird stdout **und** stderr laufend gelesen?
4. Ist `bestmove` vollständig angekommen und legal?
5. Ist `stop` verarbeitet worden, ohne einen alten Bestmove einem neuen Auftrag zuzuordnen?
6. Stoppt/terminiert jeder Kindprozess innerhalb definierter Fristen?
7. Ist der technische Timeout korrekt von einem Schachuhrverlust getrennt?

Logs benötigen je Prozess/Suche eine eindeutige Kennung; UCI-stdout darf **nicht** durch Debugausgaben verschmutzt werden.

## 7. Tests und CI-Gates

Der verbindliche Nachweis steht im [Testplan](TESTPLAN.md). Er umfasst Unit-/Invarianten-/Perft-/Game-/Search-/UCI-/Match-/Desktop-Tests und nach Umsetzung Laboratorystufen.

**Kleinster geplanter CI-Lauf:** CMake-Konfiguration → Compiler-Build → kurze CTest-Suite. Perft-Referenzen sowie Mock-UCI-Prozess-Tests folgen nach Implementierung der betreffenden Meilensteine.

**Langläufe:** Perft Tiefe 6, viele Engine-Vergleichspartien und Benchmarks sollen in getrennten, reproduzierbar parametrierten Testgruppen laufen.

In einem Testbericht muss klar zwischen `PASS`, `FAIL`, `BLOCKED` und `NOT RUN` unterschieden werden.

## 8. Forschungs- und Benchmarkdaten

Sobald die Funktionen vorhanden sind, sollte ein Experiment folgende Angaben protokollieren:

- Engine-Version und Git-Commit.
- Ausgewählte FEN-Stellungen/Testkorpus.
- Suchlimits, Bewertungs- und Algorithmusparameter.
- Compiler/Buildmodus/Plattform/CPU.
- Ausgeführte Wiederholungen und Rohmessungen.
- Gegebenenfalls PGN, Farbwechsel, Eröffnungsplan und statistische Unsicherheit.

**Fachliche Einschränkung:** Zeitmessungen sind nicht zwingend bitgenau reproduzierbar. Nodes/s ist kein Spielstärkemaß. Trace-Logging kann die Messung beeinflussen und muss begrenzt werden.

## 9. Sicherheits- und Lizenzhinweise

Fremde UCI-Engine-Programme sind lokaler ausführbarer Code mit den Berechtigungen des Benutzers. Nur bewusst ausgewählte und vertrauenswürdige Programme starten. Nicht automatisch Dateiinhalte als Shell-Befehle ausführen. Pipes, Zeilen und Zeit-/Speicherbudgets begrenzen.

Vor Veröffentlichung von 1.0 müssen die Open-Source-Lizenz und die Rechte an Figurenassets geklärt werden. **Bis eine LICENSE vorhanden ist, keine Lizenz behaupten.**

## 10. Pflege

Nach Abschluss von M0 sollen hier **nachgewiesene** Mindestversionen von CMake/Compilern, tatsächliche CMake-Targets, Testgruppen und gegebenenfalls `CMakePresets.json` dokumentiert werden. Dieser Plan ist kein Ersatz für reale Build-/CI-Ergebnisse.

| Version | Datum | Änderung |
| --- | --- | --- |
| 0.1 | 27.10.2026 | Entwicklungs- und Debugging-Workflow für Linux/Windows, Build-/Testvorschläge und Integrationsgrenzen beschrieben. |
