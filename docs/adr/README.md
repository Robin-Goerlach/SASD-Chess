# SASD-Chess – Architecture Decision Records (ADR)

> **Version:** 0.1 · **Stand:** 27.10.2026 · **Status:** Entscheidungsprozess definiert; **nicht** alle technischen Entscheidungen bereits getroffen.

## 1. Warum ADRs?

SASD-Chess soll langfristig eine **saubere, überprüfbare C++-Architektur** behalten. Für weitreichende Entscheidungen reicht eine beiläufige Implementierung nicht. Ein Architecture Decision Record hält **Kontext, Alternativen, die ausgewählte Entscheidung und deren Konsequenzen** nachvollziehbar fest.

ADRs ersetzen weder das [Lastenheft](../Lastenheft.md) noch das [Pflichtenheft](../Pflichtenheft.md) oder die [Architektur](../Architektur.md). Bei fachlichen Änderungen werden zuerst die betroffenen `LH-*`-Anforderungen überprüft.

## 2. Wann ist ein ADR erforderlich?

Beispiele:

- GUI-Framework (einschließlich möglicher Eignung des SASD-UI-Toolkits).
- Festlegung von Testframework, Bibliotheksabhängigkeiten, Compiler-/Plattformunterstützung.
- Wechsel zwischen 64-Felder-Array, Bitboards oder grundlegende Änderungen an der Core-API.
- UCI-Prozessmodell, Timeout-/Uhrenreglement, Behandlung technischer Matchabbrüche.
- Abgrenzung von Game-Historie, Remisansprüchen, Transposition-Hashing.
- Forschungs-/Diagnoseschnittstellen mit spürbaren Performance-/API-Folgen.
- Einführung persistenter Datenbanken, Plugin-ABI oder externer Dienste (**nicht** voreilig einführen).
- Lizenz- und Assetentscheidungen bei Veröffentlichung.

Kleine rein lokale Refactorings ohne wesentliche Auswirkungen benötigen gewöhnlich keinen ADR.

## 3. Dateinamen und Status

**Dateinamenkonvention:** `NNNN-kurzer-titel.md`, beispielsweise `0001-cpp-build-toolchain.md`. Nummern werden **fortlaufend** vergeben, ohne vorhandene Nummern neu zu verwenden. Die `ADR-001` bis `ADR-015` in [Architektur §16](../Architektur.md) sind vorerst **Themen-/Kandidatenkennungen**, keine bereits bestätigten ADR-Dateien. Beim Übergang zu echten ADRs ist die Zuordnung explizit zu dokumentieren.

Mögliche Statuswerte:

- **Vorgeschlagen** – begründete Entscheidungsvorlage, noch nicht akzeptiert.
- **Akzeptiert** – als technische Baseline freigegeben.
- **Abgelehnt** – bewusst verworfene Alternative, mit Begründung.
- **Ersetzt** – durch neueren ADR abgelöst, mit gegenseitigem Verweis.
- **Zurückgezogen** – Entscheidung/Entwurf ohne Geltung.

Ein akzeptierter ADR wird nicht rückwirkend umgeschrieben, um seine Geschichte zu verschleiern. Eine geänderte Entscheidung wird durch einen neuen ADR dokumentiert.

## 4. ADR-Vorlage

Neue Datei `docs/adr/NNNN-titel.md`:

```markdown
# ADR-NNNN: Kurzer Entscheidungstitel

- **Datum:** JJJJ-MM-TT
- **Status:** Vorgeschlagen
- **Projekt:** SASD-Chess
- **Bezug:** LH-F-..., LH-N-..., Roadmap M...
- **Autor/Verantwortung:** ...
- **Ersetzt/ersetzt durch:** -

## Kontext und Problem
Welche konkrete fachliche oder technische Situation liegt vor?
Welche überprüfbaren Qualitätsziele sind betroffen?

## Anforderungen und Entscheidungskriterien
Welche Muss-Kriterien und nichtfunktionalen Grenzen gelten?

## Untersuchte Alternativen
### Alternative A
Vorteile, Nachteile, Risiken, Aufwand.
### Alternative B
Vorteile, Nachteile, Risiken, Aufwand.

## Entscheidung
Welche Alternative wird **nach Freigabe** gewählt und warum?

## Konsequenzen
Auswirkungen auf Module, APIs, Build, OS-Portabilität und Wartung.

## Migration und Tests
Konkrete Test-/Benchmark-Gates; Datenmigration, falls erforderlich.

## Offene Fragen
Was ist noch ungeklärt?

## Quellen/Referenzen
Lastenheft, Pflichtenheft, Architektur, externe Spezifikationen.
```

**Wichtig:** Der Vorlagentext ist eine **Arbeitsvorlage**, kein genehmigter ADR.

## 5. Aktuelle Entscheidungsthemen (keine Freigabe)

| Architekturreferenz | Thema | Derzeitiger Status |
| --- | --- | --- |
| ADR-001 | C++20, CMake/CTest | Geplanter Technikstandard |
| ADR-002 | Ein gemeinsamer Schachkern | Architektur-Leitprinzip |
| ADR-003 | Start mit 64-Felder-Array | Begründeter Vorschlag |
| ADR-004 | Getrennte Position/Game/SearchState | Architektur-Leitprinzip |
| ADR-005 | Klassische Negamax-/Alpha-Beta-Suche | Begründeter Vorschlag |
| ADR-006 | UCI Host und UCI Client | Architektur-Leitprinzip |
| ADR-007 | Zwei-Prozess-Match-Controller | Architektur-Leitprinzip |
| ADR-008 | GUI-Technologie | **Offen** |
| ADR-009 | OS-Prozess-/Uhradapter | Architektur-Leitprinzip; Detail offen |
| ADR-010 | FEN/PGN, spätere JSON/CSV-Ergebnisse | Vorschlag |
| ADR-011 | Diagnostics Off/Stats/Trace | Vorschlag |
| ADR-012 | Konkretes Testframework | **Offen** |
| ADR-013 | Lizenz und Figurenassets | **Offen** |
| ADR-014 | Remis-/Zeit-/Fehlerreglement | Detailentscheidung **offen** |
| ADR-015 | Einheiten für APIs/Codekommentare | **Offen** |

Aus einem „Architektur-Leitprinzip“ folgt nicht automatisch, dass ein separat signierter ADR existiert. Bei Bedarf werden die jeweiligen Dokumente mit Status nachgezogen.

## 6. Pflege und Prüfschritte

1. Problem anhand eines konkreten Arbeitspakets/Tests benennen.
2. Mindestens zwei realistische Alternativen, Nachteile und Zielkonflikte dokumentieren.
3. Mögliche Auswirkungen auf korrekte Schachregeln, UCI, Portabilität und Testbarkeit prüfen.
4. Entscheidung nach fachlicher/technischer Freigabe als `Akzeptiert` kennzeichnen.
5. Betroffene Architecture-, Pflichtenheft-, Roadmap-, Agent- und Testpassagen synchronisieren.
6. Keine „Entscheidung“ alleine dadurch festschreiben, dass provisorischer Code bereits existiert.

Eine vorhandene Zielarchitektur soll **nicht** dazu führen, frühzeitig leere Interface-/Framework-Projekte zu erzeugen.

---

| Version | Datum | Änderung |
| --- | --- | --- |
| 0.1 | 27.10.2026 | ADR-Vorgehen, Vorlage, Entscheidungsstatus und Zuordnung der aktuellen ADR-Kandidaten beschrieben. |
