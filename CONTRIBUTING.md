# Mitwirken an SASD-Chess (CONTRIBUTING.md)

> **Version:** 0.1 · **Stand:** 27.10.2026 · **Status:** Projektrichtlinie im Entwurf

Vielen Dank für das Interesse an SASD-Chess. Das Projekt entwickelt ein **klassisches C++-Schachprogramm**, keine lernende Schach-KI. Besonders willkommen sind nachvollziehbare Verbesserungen am Regelkern, an der Testabdeckung, den UCI-Schnittstellen, der Benutzeroberfläche und der Dokumentation.

## 1. Vor einem Beitrag lesen

- [Lastenheft](docs/Lastenheft.md) – fachliche Anforderungen und Prioritäten.
- [Pflichtenheft](docs/Pflichtenheft.md) – technische Realisierung.
- [Architektur](docs/Architektur.md) – erlaubte Modulgrenzen und Zustandsverantwortung.
- [ROADMAP.md](ROADMAP.md) – aktuelle Meilensteine und deren Definition of Done.
- [AGENTS.md](AGENTS.md) – verbindliche Coding-/Qualitätsregeln für dieses Repository.
- [Testplan](docs/TESTPLAN.md) – Testmatrix, Prüfdaten und Release-Nachweise.

Alle diese Dokumente beschreiben teilweise **geplante** Funktionalität; nicht jede im Plan genannte Quellcodedatei existiert bereits.

## 2. Einen Beitrag vorbereiten

1. **Istzustand prüfen:** Branch, letzter Commit, existierende Komponenten und offene Änderungen sichten.
2. **Arbeitspaket wählen:** Idealerweise eines aus dem nächsten [Roadmap](ROADMAP.md)-Meilenstein und mit `LH-*`-Anforderung verknüpft.
3. **Problem und Lösung erläutern:** Bei nichttrivialen Änderungen gern vorab über Issue/PR diskutieren; Alternativen, Auswirkungen und Testplan benennen.
4. **Klein implementieren:** Eine Änderung soll möglichst nur eine zusammenhängende fachliche Aufgabe lösen.
5. **Automatisch testen:** Neue Tests ergänzen und vorhandene passende Tests ausführen.
6. **Dokumentieren:** Kommentare, Architektur und Nachweise aktualisieren.
7. **Review ermöglichen:** Klare Beschreibung, konkrete Testergebnisse und bekannte Einschränkungen liefern.

Bestehende Branch-Protection-, Commit- und PR-Regeln haben Vorrang. Diese Datei schreibt **keine** nicht bestätigte Main-only- oder Auto-Merge-Politik vor. Fremde Änderungen und nicht zugehörige Repositories dürfen nicht überschrieben werden.

## 3. Technische Konventionen

**Sprache und Toolchain:** C++20, CMake, CTest und portable Implementierung. Geplante erste Zielplattformen sind **Windows und Linux**. GUI- und OS-spezifischer Code gehört in Adapter, nicht in den Schachkern. Abhängigkeiten bleiben sparsam und müssen hinsichtlich Lizenz und Portabilität geprüft werden.

**Namensgebung:** Fachliche C++-Typen in `sasd::chess`; selbsterklärende englische Bezeichner. Kommentarsprache projektweit einheitlich festlegen. Öffentliche Funktionen/Klassen, Zustandsübergänge, Voraussetzungen, Besitz und Besonderheiten der Schachregeln **verständlich und ausführlich** dokumentieren.

**Design:** Eine autoritative Legality-Implementation, eine Partieverwaltung und ein unabhängiger Match-Schiedsrichter. Keine zweite Schachregellogik in GUI, Formaten, Suche oder Match. Keine grafische Bibliothek im Core. Klare Fehlerergebnisse statt stiller Korrektur fehlerhafter Daten.

**Performance:** Zuerst bekannte Perft- und UCI-/Game-Tests bestehen. Optimierung braucht nachvollziehbare Messwerte vor und nach der Änderung.

## 4. Lokaler Build und Tests

Die folgenden Befehle beschreiben den **vorgesehenen Ablauf nach M0**. Solange `CMakeLists.txt` und Tests noch nicht eingerichtet sind, sind sie **keine** Zusicherung eines funktionierenden Builds.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
ctest --test-dir build --output-on-failure
```

Bei MSVC mit Multi-Config-Generator:

```powershell
cmake -S . -B build
cmake --build build --config Debug
ctest --test-dir build --config Debug --output-on-failure
```

**Korrigieren statt kaschieren:** Compilerwarnungen, gescheiterte Tests oder abweichende Perft-Werte sind zu untersuchen. Ein Test darf nicht entfernt, ignoriert oder sein Sollwert verändert werden, nur um CI grün zu bekommen.

## 5. Tests je Änderungsbereich

- **Core:** Unit-/Invariantentests, Perft/Divide, Sonderzüge, vollständiges make/unmake.
- **Game:** Matt/Patt, Remis-Beanspruchung, Wiederholungen, 50/75-Züge-Regeln, Historie.
- **Engine:** legale Bestmoves, terminierende Suche, Stop, deterministische Tiefenläufe.
- **UCI/Prozesse:** `uci`/`isready`, Position, `go`/`stop`, Pipes, Timeouts, beschädigte Antworten.
- **Match:** zwei separate Prozesse, SASD gegen SASD, fremde UCI-Engine, gültige PGN-Datei, Prozessaufräumen.
- **GUI:** vollständige Partien, Umwandlung, Rücknahme, FEN-/PGN-Bedienung.
- **Laboratory:** `LAB-T-*` gemäß [Testplan](docs/TESTPLAN.md), erst beim jeweiligen Ausbau.

Kritische UCI-Fehlerfälle sollen mit eigenständigen kleinen **Mock-Engine-Prozessen** prüfbar sein, ohne beim regulären Build Drittanbieter-Engines herunterladen zu müssen.

## 6. Pull-Request-/Änderungsbeschreibung

Eine hilfreiche PR-Beschreibung enthält mindestens:

```text
## Ziel
Welches Problem wird gelöst? Welche LH-IDs/Meilensteine sind betroffen?

## Änderung
Welche Module/Dateien wurden fachlich geändert und warum?

## Architektur
Welche Abhängigkeiten, Daten-/Thread-Verträge oder ADRs sind betroffen?

## Tests
Welche Befehle wurden tatsächlich unter welchem OS/Compiler ausgeführt?
Was ist PASS/FAIL/NOT RUN? Perft-/UCI-/Match-Ergebnisse?

## Risiken
Bekannte Einschränkungen, Migrationen und offene Aufgaben.

## Dokumentation
Welche Anforderungen/Roadmap/Changelog/Anleitungen wurden aktualisiert?
```

Empfohlene Commitpräfixe: `feat(core):`, `feat(uci):`, `feat(match):`, `fix(game):`, `test(perft):`, `docs:`, `refactor:`. Das ist eine Empfehlung, solange kein anderes offizielles Schema beschlossen ist.

## 7. Architekturentscheidungen

Bei Einführung neuer Abhängigkeiten, GUI-Frameworks, großen Datenmodelländerungen, Suchrepräsentationen, UCI-Abweichungen oder Matchreglements eine [Architecture Decision Record](docs/adr/README.md) vorbereiten. Ein ADR nennt Kontext, Alternativen, Entscheidung, Konsequenzen, Test-/Migrationsfolgen und betroffene Anforderungs-IDs.

**Keine „Architektur auf Vorrat“:** Nur Abstraktionen anlegen, die einen aktuellen, nachvollziehbaren Bedarf erfüllen.

## 8. Sicherheit, Datenschutz, Rechte

Eine fremde Schach-Engine ist **ausführbarer lokaler Code**. Engine-Pfade nur bewusst aus vertrauenswürdigen Quellen wählen; nicht aus importierten Partien automatisch ausführen. Dateieingaben begrenzen; stderr/stdout-Verarbeitung so gestalten, dass sie auch unter Fehlerlast nicht hängenbleibt.

**Lizenzstatus:** Die endgültige Open-Source-Lizenz und die Rechte an Figurenassets sind im Projektstand vom 27.10.2026 **noch nicht festgelegt**. Das öffentliche GitHub-Repository allein erteilt **keine** pauschale Nutzungserlaubnis für fremde Codebeiträge. Vor der Übernahme externer Beiträge und spätestens vor Veröffentlichung von 1.0 müssen Nutzungs-/Weitergaberechte geklärt und eine gültige LICENSE aufgenommen werden. Keine ungeprüften fremden Engine-Implementierungen, Quelltexte oder Grafiken kopieren.

## 9. Definition of Done und Freigabe

Ein Beitrag ist erst dann **fertig**, wenn die fachliche Aufgabe erfüllt, Tests nachvollziehbar bestanden, Fehlerfälle behandelt und relevante Dokumente aktualisiert wurden. Nicht ausgeführte Tests explizit kennzeichnen. Die endgültige 1.0-Freigabe richtet sich nach **LH-A-01 bis LH-A-17** und nicht nach einem einzelnen erfolgreichen Build.

## 10. Dokumentenpflege

`README.md` erklärt das Projekt und verweist auf die kanonischen Details. `AGENTS.md` enthält die projektweiten Coding-Regeln. `ROADMAP.md` verfolgt geplante Etappen. `CHANGELOG.md` beschreibt **tatsächlich abgeschlossene** Änderungen. Im Konfliktfall sind Lastenheft/Pflichtenheft/Architektur und freigegebene ADRs maßgeblich.

---

| Version | Datum | Änderung |
| --- | --- | --- |
| 0.1 | 27.10.2026 | Beitragspraxis, Qualitätsanforderungen, Build-/Testvorlagen und Lizenzhinweise als deutschsprachigen Projektstandard ergänzt. |
