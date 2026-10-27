# Änderungsverlauf – SASD-Chess (CHANGELOG)

> **Stand:** 27.10.2026 · **Status:** Vorbereitungs- und Spezifikationsphase  
> **Format:** Orientierung an „Keep a Changelog“; die unten stehenden Einträge beschreiben **nachweisbar dokumentierte** Änderungen, keine fertiggestellten Programmfunktionen.  
> **Repository:** https://github.com/Robin-Goerlach/SASD-Chess

## Grundregeln

- Einträge berichten über **tatsächlich abgeschlossene** Änderungen. Zukunftswünsche gehören in die [Roadmap](ROADMAP.md).
- Arten: **Added / Added (Docs)**, **Changed**, **Fixed**, **Removed**, **Security**, **Deprecated**.
- Ein Projekt-Release bekommt eine eigene Version und ein Datum erst bei tatsächlicher Veröffentlichung.
- Keine erfundenen Releases, nicht existente CI-Prüfungen oder unbestätigte Engine-Funktionen dokumentieren.
- Bei Änderungen mit fachlicher Wirkung `LH-*`-Anforderungen, Testnachweise und gegebenenfalls ADRs referenzieren.
- Möglichst nur für Nutzer/Entwickler sichtbare Veränderungen hier aufführen; detaillierte Einzelcommits bleiben in Git.

## [Unreleased]

### Added (Docs)

- Ausführliches deutsches [Lastenheft](docs/Lastenheft.md), [Pflichtenheft](docs/Pflichtenheft.md) und [Architekturdokument](docs/Architektur.md) für SASD-Chess als **klassisches C++-Schachprogramm**.
- UCI-Engine- und UCI-Client-Rolle sowie unabhängiger Zwei-Prozess-Match-Schiedsrichter einschließlich **Selbstspiel** als Architektur- und Muss-Anforderung beschrieben.
- Wissenschaftliche schachbezogene Analyse- und Laboratory-Funktionen als gestufte Zukunftsplanung ergänzt, mit **Perft/Divide** als Qualitätssicherungs-Pflicht für die 1.0.
- [ROADMAP.md](ROADMAP.md) mit M0–M6, Abnahmekriterien, L1–L3 und kritischem Pfad.
- [AGENTS.md](AGENTS.md) als zentrale projektweite Coding-/Architekturanweisung sowie [AGENT.md](AGENT.md) als Verweis.
- [CONTRIBUTING.md](CONTRIBUTING.md) für Beitragspraxis, Reviews und Testnachweise.
- [Test- und Abnahmeplan](docs/TESTPLAN.md) mit Perft-, UCI-, Match- und Laboratory-Prüfungen.
- [ADR-Regeln](docs/adr/README.md) und [Entwicklungsumgebung](docs/DEVELOPMENT.md) als vorbereitete Projektprozesse, soweit diese Dokumente im Repository vorhanden sind.

### Project status

- **Dokumentierte Planung, keine abgeschlossenen Implementierungsmeilensteine.**
- C++-Schachkern, eigene Strategie-Engine, Desktop-Anwendung und Match-Controller werden erst nach Implementierung und erfolgreich nachgewiesenen Tests als fertig aufgeführt.
- Lizenz, endgültiges Testframework, GUI-Framework und weitere Details stehen noch zur Entscheidung.

---

## Vorgehen bei der nächsten Änderung

Neue abgeschlossene Arbeiten werden unter `[Unreleased]` mit fachlich präzisem Text eingetragen, z. B. erst **nach** tatsächlich erfolgreichem M0:

```text
### Added
- CMake/CTest-Grundlage mit nachgewiesenem Windows-/Linux-Build (M0; <Commit/Testnachweis>).
```

Der Beispieltext ist **keine bereits durchgeführte Arbeit**.

## Versionshistorie dieses Dokuments

| Version | Datum | Änderung |
| --- | --- | --- |
| 0.1 | 27.10.2026 | Changelog für die Dokumentations-/Planungsphase angelegt und Statusregeln festgelegt. |
