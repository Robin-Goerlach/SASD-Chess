# SASD-Chess – Lastenheft

> **Dokumentenstatus:** Entwurf zur fachlichen Abstimmung · **Version:** 0.2 · **Datum:** 27.10.2026  
> **Projekt:** SASD-Chess · **Repository:** https://github.com/Robin-Goerlach/SASD-Chess  
> **Sprache:** Deutsch · **Geplanter Produkttyp:** frei verfügbare, lokal nutzbare Schachanwendung mit eigener klassischer Engine

## 1. Zweck und Geltungsbereich

Das Lastenheft beschreibt aus Sicht des Auftraggebers, **was** SASD-Chess leisten soll und unter welchen Bedingungen die Ergebnisse abgenommen werden. Es beschreibt bewusst keine verbindliche Implementierung einzelner Klassen oder Algorithmen; diese wird im [Pflichtenheft](Pflichtenheft.md) konkretisiert.

**Status der Festlegungen:** Die Anforderungen sind ein aus den bisherigen Projektzielen abgeleiteter, fachlich konsistenter **Vorschlag**. Entscheidungen zu Lizenz, konkretem GUI-Framework und finaler Plattformmatrix bedürfen noch der Bestätigung. In diesem Dokument bedeutet **M** „Muss für Version 1.0“, **S** „Soll, sofern in 1.0 realisierbar“, **K** „Kann“ und **Z** „spätere Ausbaustufe“. Die Kennzeichnungen sind keine bereits erteilte Abnahme.

## 2. Ausgangssituation und Motivation

SASD-Chess soll ein nachvollziehbar aufgebautes C++-Schachprogramm werden, das eine vollständige Schachpartie zwischen Menschen sowie gegen eine **selbst entwickelte klassische Strategie-Engine** ermöglicht. Die Kernfunktionen sollen ohne Cloud, Benutzerkonto, Training einer KI oder Internetverbindung funktionieren. Eine spätere Erweiterung durch lernende Bewertungsverfahren soll möglich sein, ohne das Schachregelwerk neu zu entwickeln.

Die Lösung soll zugleich ein langfristig wartbares Lehr-, Forschungs- und Entwicklungsprojekt sein. **Korrektheit, klare Architektur, aussagekräftige Tests und ausführliche Codekommentare haben zunächst Vorrang vor maximaler Performance**.

## 3. Ziele und Erfolgsmaßstäbe

| ID | Ziel | Erfolgskriterium |
| --- | --- | --- |
| LH-Z-01 | Regelkonformes Standardschach | Alle üblichen Spiel- und Sonderregeln inklusive Partieende werden korrekt behandelt. |
| LH-Z-02 | Gut bedienbare Oberfläche | Zwei Personen können am selben Rechner ohne Kommandozeile eine Partie vollständig spielen. |
| LH-Z-03 | Eigenständiger Computergegner | Eine eigene, deterministisch reproduzierbare klassische Engine wählt legale Züge und erkennt einfache taktische Situationen. |
| LH-Z-04 | Trennung von Zuständigkeiten | Regeln, Partieablauf, Engine und UI sind getrennt test- und austauschbar. |
| LH-Z-05 | Portabilität | Grundmodule und Engine lassen sich mindestens unter Linux und Windows bauen und testen. |
| LH-Z-06 | Offene Weiterentwicklung | Architektur und Dokumentation erlauben spätere zusätzliche Oberflächen, Analysefunktionen und KI-Experimente. |
| LH-Z-07 | Interoperable Engine-Matches | SASD-Chess kann über UCI sowohl gegen externe Programme als auch gegen eine zweite Instanz der eigenen Engine spielen; ein unabhängiger Match-Controller führt und protokolliert die Partien. |

**Nicht-Ziel für Version 1.0:** eine Elo-Garantie, Spielstärke auf Stockfish-Niveau oder ein selbstlernendes neuronales Schachsystem.

## 4. Beteiligte und Zielgruppen

- **Auftraggeber/Projektverantwortung:** legt Funktionsumfang, Qualitätsmaßstäbe und Freigaben fest.
- **Entwicklung und Test:** realisiert die Komponenten, dokumentiert Entscheidungen und führt automatisierte sowie manuelle Prüfungen aus.
- **Gelegenheitsspieler:** möchte legal und bequem Schach spielen, optional gegen den Computer.
- **Technisch interessierte Nutzer:** möchte Stellungen importieren, Züge nachvollziehen und die Engine separat nutzen.
- **Künftige Mitwirkende:** benötigen klare Schnittstellen, verständliche Dokumentation, reproduzierbare Builds und eine offene Beitragspraxis.

## 5. Produktumfang und Priorisierung

### 5.1 Funktionale Anforderungen – Schachregeln und Partie

| ID | Prio | Anforderung und überprüfbares Ergebnis |
| --- | :--: | --- |
| LH-F-001 | M | Das System stellt eine reguläre 8×8-Anfangsstellung her; Weiß beginnt. |
| LH-F-002 | M | Alle Figuren bewegen und schlagen ausschließlich entsprechend den Regeln des **orthodoxen Standardschachs**. |
| LH-F-003 | M | Illegale Züge werden abgewiesen, insbesondere Züge, nach denen der eigene König im Schach verbliebe. |
| LH-F-004 | M | Bauern-Doppelschritt, En passant und Umwandlung in Dame, Turm, Läufer oder Springer funktionieren in legalen Situationen. |
| LH-F-005 | M | Kurze und lange Rochade berücksichtigen Zugrechte, freie Felder, Königsschach sowie angegriffene Durchgangs- und Zielfelder. |
| LH-F-006 | M | Schach, Schachmatt und Patt werden zuverlässig erkannt; nach Partieende werden keine Züge mehr angenommen. |
| LH-F-007 | M | Remis durch tote Stellung (keine mögliche Mattfolge), fünffache Stellungswiederholung und 75-Züge-Regel wird gemäß Regelwerk erkannt; beanspruchbares Remis nach dreifacher Wiederholung oder 50 Zügen wird dem Spieler angeboten. Vorrangige Mattregeln bleiben gewahrt. |
| LH-F-008 | M | Der vollständige Partieablauf mit Zugrecht, Ergebnis, Rochaderechten, En-passant-Status und Halb-/Vollzugzählern ist konsistent. |
| LH-F-009 | M | Eine neue Partie kann gestartet und die laufende Partie zurückgesetzt werden. |
| LH-F-010 | M | Der Spieler kann im lokalen Spiel Züge zurücknehmen; Zustand und Zugliste werden konsistent wiederhergestellt. |

### 5.2 Funktionale Anforderungen – Oberfläche und Bedienung

| ID | Prio | Anforderung und überprüfbares Ergebnis |
| --- | :--: | --- |
| LH-F-011 | M | Eine Desktop-Oberfläche zeigt Brett, Figuren, Zugrecht und aktuellen Spielstatus eindeutig an. |
| LH-F-012 | M | Figuren lassen sich per Auswahl eines Start- und Zielfeldes bewegen; legale Zielfelder sind auf Wunsch erkennbar. |
| LH-F-013 | M | Die Oberfläche ermöglicht die Auswahl einer Umwandlungsfigur und verhindert unbeabsichtigte illegale Züge. |
| LH-F-014 | M | Eine fortlaufende, für Menschen lesbare Zugliste wird dargestellt. |
| LH-F-015 | M | Mindestens die Modi „Mensch gegen Mensch (lokal)“ und „Mensch gegen Computer“ sind verfügbar; die menschliche Farbe ist wählbar. |
| LH-F-016 | M | Regelrelevante Ereignisse (Schach, Matt, Patt, Remis, Engine denkt) werden klar kommuniziert. |
| LH-F-017 | S | Brettdrehung, Feldkoordinaten, letzter Zug und Tastaturbedienung werden unterstützt. |
| LH-F-018 | S | Drag-and-drop ist zusätzlich zur Bedienung durch zwei Auswahlen möglich, nicht als einzige Eingabemethode. |
| LH-F-019 | S | Eine optionale Schachuhr mit üblichen Grundparametern ist verfügbar. |
| LH-F-020 | K | Visuelle Themes, alternative Figurensätze, Animationen und weitere Bedienhilfen. |

### 5.3 Funktionale Anforderungen – Engine, Daten und Schnittstellen

| ID | Prio | Anforderung und überprüfbares Ergebnis |
| --- | :--: | --- |
| LH-F-021 | M | Eine eigene **klassische Engine ohne Lernpflicht** berechnet aus einer legalen Stellung einen legalen Zug. |
| LH-F-022 | M | Die Engine berücksichtigt mindestens Material und einfache Positionsmerkmale sowie mehrzügige Antwortvarianten. |
| LH-F-023 | M | Eine Suche lässt sich über Suchtiefe oder zeitliches Budget begrenzen; ein Suchabbruch ist möglich. |
| LH-F-024 | M | Die Engine stellt eine minimale, interoperable **UCI-Schnittstelle** zur Verfügung, unabhängig von der GUI. |
| LH-F-025 | M | Positionen können im **FEN-Format** eingelesen und ausgegeben werden; ungültige Eingaben werden nachvollziehbar zurückgewiesen. |
| LH-F-026 | M | Partieverläufe lassen sich als **PGN** exportieren; Züge werden in gültiger algebraischer Notation (SAN) ausgegeben. |
| LH-F-027 | S | PGN-Import und Fortsetzung einer geladenen Partie. |
| LH-F-028 | S | Einstellbare Engine-Spielstärke, etwa durch Suchzeit, Tiefe oder bewusst begrenzte Suchleistung. |
| LH-F-029 | S | Engine-Diagnoseinformationen wie Suchtiefe, untersuchte Knoten, Hauptvariante und Bewertungswert. |
| LH-F-030 | K | Stellungsanalyse, Variantenbaum, Eröffnungsbuch, Endspieltabellen und Engine-gegen-Engine-Turniermodus. |
| LH-F-031 | Z | Austauschbare lernende Bewertungsmodelle, trainierbare Engine-Komponenten und systematisches Selbstspiel. |
| LH-F-032 | Z | Weitere Brettspiele mittels eines gesonderten generischen Spielkerns; **keine** Voraussetzung für Schach 1.0. |
| LH-F-033 | M | Eine externe **UCI-konforme** Schach-Engine kann als Gegner über einen gesonderten Prozess eingebunden werden; standardmäßige Ein-/Ausgabe (stdin/stdout) dient als Kommunikationskanal. |
| LH-F-034 | M | Ein **Match-Controller** (Schiedsrichter) kann zwei voneinander unabhängige UCI-Engine-Prozesse starten und eine Partie zwischen ihnen vollständig leiten. |
| LH-F-035 | M | Der Match-Controller unterstützt ausdrücklich zwei Instanzen **derselben** SASD-Chess-Binärdatei sowie zwei unterschiedliche Engines. Prozesse haben getrennten Zustand; Spielerfarben und Startparameter sind konfigurierbar. |
| LH-F-036 | M | Der Match-Controller verwaltet einen autoritativen Brettzustand und prüft **jeden** gemeldeten Zug auf Legalität, bevor er ihn übernimmt und der Gegenseite die aktualisierte Stellung übergibt. |
| LH-F-037 | M | Der Match-Controller kennt feste Suchtiefe und definierte Zeitkontrollen, beendet Spiele korrekt und erkennt Zeitüberschreitung, ungültige Züge, Verbindungs-/Prozessabbruch und Protokollfehler. |
| LH-F-038 | M | Jede beendete Partie kann mit Engine-Kennungen, Farben, Ergebnis, Zeitkontrolle und vollständiger Zugfolge als PGN protokolliert werden. |
| LH-F-039 | S | Automatisierte Serien von Engine-Partien mit Farbwechsel, Resultatübersicht (Sieg/Remis/Niederlage) und reproduzierbarer Konfiguration sind möglich. |
| LH-F-040 | S | Ein laufendes Engine-Match kann in der Desktop-Oberfläche beobachtet, gestoppt und mit Zugliste angezeigt werden. |
| LH-F-041 | K | Späterer Netzwerk-Transport bzw. XBoard/CECP-Adapter ohne Veränderung der Kern- und Matchlogik. |

## 6. Nichtfunktionale Anforderungen

| ID | Prio | Anforderung / Mess- oder Prüfansatz |
| --- | :--: | --- |
| LH-N-001 | M | **Korrektheit:** Der Regelkern besteht definierte Perft-Referenztests und gezielte Sonderregeltests. |
| LH-N-002 | M | **Stabilität:** Fehlerhafte FEN-, UCI- oder GUI-Eingaben dürfen weder zu unbeabsichtigten Zügen noch zu einem Absturz führen. |
| LH-N-003 | M | **Wartbarkeit:** Öffentliche Verantwortlichkeiten sind dokumentiert; Module können unabhängig getestet werden. |
| LH-N-004 | M | **Nachvollziehbarkeit:** Nicht offensichtlicher Code, Invarianten und Algorithmen werden ausführlich und verständlich kommentiert; Benennungen sind konsistent. |
| LH-N-005 | M | **Portabilität:** Mindestens Windows und Linux werden in automatisierten Builds geprüft; zusätzliche POSIX-/BSD-/macOS-Ziele werden architektonisch nicht ausgeschlossen. |
| LH-N-006 | M | **Offline-Fähigkeit:** Grundfunktionen benötigen keine Registrierung, Telemetrie, Cloud oder Internetverbindung. |
| LH-N-007 | M | **Determinismus:** Bei identischer Stellung, Engine-Konfiguration und festem Suchbudget nach Tiefe sind wiederholbare Ergebnisse möglich (soweit keine Parallelisierung Zufall einführt). |
| LH-N-008 | M | **Testbarkeit:** Regelwerk und Engine laufen headless in automatisierten Tests; die Benutzeroberfläche ist nicht Voraussetzung für Engine-Tests. |
| LH-N-009 | M | **Ressourcenkontrolle:** Engine-Suche besitzt eine begrenzbare Laufzeit und Speichernutzung; ein Stop-Signal beendet die Berechnung kontrolliert. |
| LH-N-010 | S | **Bedienbarkeit:** Kernaktionen sind ohne Drag-and-drop und möglichst auch per Tastatur erreichbar; Statusinformationen sind nicht nur farblich vermittelt. |
| LH-N-011 | S | **Performance:** Definierte Benchmarks melden Suchknoten, Laufzeit und Speichervolumen; konkrete Grenzwerte werden erst anhand dokumentierter Referenzhardware festgelegt. |
| LH-N-012 | M | **Sicherheit/Datenschutz:** Lokale Partien werden nicht ohne ausdrückliche Aktion übertragen; Formateingaben sind begrenzt und robust zu verarbeiten. |
| LH-N-013 | M | **Build-Reproduzierbarkeit:** Bauen und Testen erfolgen über dokumentierte, weitgehend automatisierbare Schritte. |
| LH-N-014 | S | **Open Source:** Ein öffentlich nutzbares Lizenzmodell, Contributor-Regeln und Drittanbieter-Lizenzprüfung werden vor der ersten öffentlichen Version verbindlich festgelegt. |
| LH-N-015 | M | **Prozessisolation:** Zwei gleichzeitig gestartete Engines teilen weder globale Partiestände noch Engine-Speicher; fehlgeschlagene Prozesse blockieren den Match-Controller nicht dauerhaft. |

## 7. Typische Nutzungsszenarien

### LH-UC-01 – Zwei Menschen spielen lokal

**Vorbedingung:** Die Anwendung wurde gestartet.  
**Ablauf:** Neue Partie auswählen → Weiß zieht → Oberfläche zeigt neuen Zustand → Schwarz zieht → Partie wird bis zu einer regelkonformen Beendigung fortgeführt.  
**Ausnahmen:** Illegale Züge werden ohne Zustandsänderung zurückgewiesen; bei Bauernumwandlung wird die Figur gewählt.  
**Ergebnis:** Zugliste, Brett und Partieergebnis stimmen überein.

### LH-UC-02 – Mensch spielt gegen die Engine

**Vorbedingung:** Engine ist verfügbar.  
**Ablauf:** Mensch wählt Farbe und optional Suchgrenze → führt legalen Zug aus → Engine antwortet innerhalb des Budgets → Status und Zugliste werden aktualisiert.  
**Ausnahmen:** Bei Abbruch, fehlenden legalen Zügen oder Programmende wird kein veralteter Engine-Zug ausgeführt.  
**Ergebnis:** Eine vollständige Partie gegen den Computer ist möglich.

### LH-UC-03 – Position laden und weiterführen

**Vorbedingung:** Eine formal und schachlich gültige FEN-Stellung liegt vor.  
**Ablauf:** FEN laden → Stellung einschließlich Zugrecht und Spezialrechten anzeigen → legal weiterziehen.  
**Ausnahmen:** Ungültige Zeichenfolgen oder widersprüchliche Zustände werden mit konkretem Fehler abgelehnt.  
**Ergebnis:** Interner Zustand und FEN-Ausgabe sind konsistent.

### LH-UC-04 – Partie exportieren

**Ablauf:** Partie spielen oder starten → PGN exportieren → Datei in einem unabhängigen PGN-Leser prüfen.  
**Ergebnis:** Die Zugfolge und das Ergebnis sind in Standardnotation nachvollziehbar.

### LH-UC-05 – Externe Schachoberfläche nutzt die Engine

**Ablauf:** Externe Oberfläche startet UCI-Prozess → sendet Identifikation, Stellung und Suchauftrag → Engine liefert einen legalen `bestmove` oder einen regelkonformen Ergebnisfall.  
**Ergebnis:** Basisinteroperabilität ohne eigene Desktop-GUI.

### LH-UC-06 – Externe Engine gegen SASD-Chess

**Vorbedingung:** Zwei ausführbare, UCI-fähige Engine-Programme sind erreichbar.  
**Ablauf:** Match-Controller mit Pfaden, Farben, Zeitbudget oder Suchtiefe konfigurieren → für jede Engine einen eigenständigen Prozess starten → beide per UCI initialisieren → abwechselnd die autoritative Stellung senden, die jeweilige Engine rechnen lassen und den empfangenen `bestmove` vor der Übernahme validieren.  
**Ausnahmen:** Verbotener Zug, Prozessabbruch, Protokollfehler und Überschreitung des zulässigen Budgets werden protokolliert und nach definiertem Regelwerk als Fehler bzw. Partieergebnis behandelt.  
**Ergebnis:** Eine vollständige, reproduzierbar dokumentierte Partie einschließlich Ergebnis und PGN.

### LH-UC-07 – SASD-Chess gegen sich selbst

**Vorbedingung:** Das Engine-Programm wurde erfolgreich gebaut.  
**Ablauf:** Der Match-Controller startet die gleiche Engine-Binärdatei zweimal mit getrennten Prozesskanälen; optional unterschiedliche Einstellungen, ansonsten gleiche Konfiguration. Die Prozesse spielen Weiß und Schwarz und dürfen nicht direkt auf gemeinsame Partiezustände zugreifen.  
**Ergebnis:** Beide Instanzen spielen eine vollständige legale Partie. Ein Abbruch einer Instanz beendet die übrigen Prozesse kontrolliert; die bislang gespielten Züge bleiben für die Diagnose erhalten.

## 8. Ein- und Ausgaben / Daten

- **Eingaben:** Feld-/Figurenauswahl, Modi, Suchparameter, FEN-Stellung, UCI-Kommandos; später optional PGN-Dateien.
- **Ausgaben:** Brettzustand, Statusmeldung, SAN-Zugliste, FEN, PGN, UCI-Antworten, optional Suchstatistik.
- **Datenhaltung:** Version 1.0 benötigt keine Datenbank und keine Benutzerkonten. Dateispeicherung erfolgt explizit durch den Nutzer.
- **Sprachkonzept:** Die Projektdokumentation und Benutzerdokumentation starten auf Deutsch; Bezeichner, Protokolle und Schnittstellen folgen den jeweiligen technischen Konventionen.
- **Brett- und Figurendarstellung:** Die Darstellung darf keine bestehende proprietäre Bildersammlung ohne entsprechende Nutzungsrechte voraussetzen.

## 9. Fachliche Randbedingungen und Abgrenzung

1. Regelgrundlage ist das **orthodoxe Standardschach**; Schach960, Märchenschach und abweichende Turnierregeln gehören nicht zu 1.0.
2. Die Engine wird zunächst **ohne neuronales Netz oder Training** realisiert; spätere KI-Experimente dürfen die saubere Trennung nicht aufheben.
3. Die Desktop-Oberfläche darf ein separates UI-Toolkit verwenden, die Spielregeln müssen jedoch unabhängig davon bleiben.
4. Nicht geplant für 1.0: Online-Multiplayer, zentrale Nutzerverwaltung, Cloud-Synchronisierung, Bezahlfunktionen, Anti-Cheat-Plattform, mobile Apps, offizielle Turnierzertifizierung und ELO-Zusagen.
5. Ein **generischer Brettspielbaukasten** ist ausdrücklich keine Voraussetzung für 1.0, kann aber langfristig separat untersucht werden.
6. Die Auswahl von Lizenz und konkreter GUI-Technologie ist eine **offene Entscheidung**, keine bereits erfüllte Anforderung.

## 10. Abnahmekriterien für Version 1.0

| ID | Prüffall | Erwartetes Abnahmeergebnis |
| --- | --- | --- |
| LH-A-01 | Perft für die Standard-Anfangsstellung mit Tiefen 1 bis 5 | Exakt 20 / 400 / 8.902 / 197.281 / 4.865.609 Blätter; keine abweichenden Ergebnisse. |
| LH-A-02 | Zusätzliche Referenzstellungen mit Rochade, En passant, Schach und Umwandlung | Alle dokumentierten Referenzergebnisse stimmen; Sonderfälle besitzen Regressionstests. |
| LH-A-03 | Manuelle vollständige Partie in der Desktop-Oberfläche | Legale Züge ausführbar, illegale Züge abgewiesen; Schachmatt/Patt/Remis korrekt. |
| LH-A-04 | Bauernumwandlung in jede erlaubte Figur | Die gewählte Figur erscheint und wird beim Weiterspielen korrekt behandelt. |
| LH-A-05 | FEN-Ein-/Ausgabe | Definierte Teststellungen werden korrekt geladen und round-trip-fähig ausgegeben; fehlerhafte Eingaben sicher abgewiesen. |
| LH-A-06 | PGN-Ausgabe | Gültiges PGN mit korrekter Zugnotation und Resultat ist extern lesbar. |
| LH-A-07 | Mensch gegen Engine | Engine liefert ausschließlich legale Züge und respektiert Suchabbruch sowie definierte Zeit-/Tiefengrenzen. |
| LH-A-08 | UCI-Basistest | Handshake, Stellungsübergabe, Suche, Stop und Quit funktionieren ohne GUI. |
| LH-A-09 | Zustandskonsistenz | Zug/Rücknahme rekonstruiert die vorherige Stellung samt Sonderrechten und Zählern. |
| LH-A-10 | CI/Portabilität | Dokumentierter Build und automatisierte Core-/Engine-Tests sind unter Windows und Linux erfolgreich. |
| LH-A-11 | Offline/Datenschutz | Spielen, Laden, Export und Engine-Suche funktionieren ohne Netzwerk. |
| LH-A-12 | Dokumentationsprüfung | Installations-/Buildanleitung, Architekturübersicht, Tests und bekannte Einschränkungen sind vorhanden. |
| LH-A-13 | Match gegen fremde UCI-Engine | Zwei unabhängige Engine-Prozesse werden erfolgreich initialisiert; ein vollständiges legales Spiel zwischen SASD-Chess und externer UCI-Engine wird durchgeführt und als PGN protokolliert. |
| LH-A-14 | Selbstspiel mit zwei Prozessen | Zweimal dieselbe SASD-Chess-Engine-Binärdatei starten; beide Prozesse spielen getrennt, ohne gegenseitige Zustandskorruption, bis zum dokumentierten Partieende. |
| LH-A-15 | Fehler- und Zeitkontrollfälle | Der Match-Controller meldet illegale `bestmove`-Ausgabe, unerwartetes Prozessende, ausbleibende UCI-Antwort und Zeitüberschreitung korrekt; alle gestarteten Prozesse werden kontrolliert bereinigt. |
| LH-A-16 | Synchronisation | Nach jeder legalen Antwort beider Engines stimmen autoritativer Brettzustand, UCI-Zugfolge und protokollierte Partie überein. |

**Abnahmeregel:** Ein Pflichtkriterium gilt nur bei reproduzierbarem Nachweis als erfüllt. Nicht abgeschlossene S-/K-Funktionen dürfen als offen dokumentiert werden; nicht erfüllte M-Kriterien verhindern die Freigabe von 1.0.

## 11. Vorschlag für Lieferstufen

| Stufe | Fachliches Ergebnis | Voraussetzung für nächste Stufe |
| --- | --- | --- |
| M0 – Projektbasis | Repository-Struktur, CMake, Tests, Dokumentationskonventionen | Sauberer, reproduzierbarer Minimalbuild |
| M1 – Schachkern | Brett, Regeln, Zustandsverwaltung, Perft | Alle Kern- und Spezialregeltests grün |
| M2 – Partie und Formate | Partieablauf, SAN, FEN, PGN-Export, Rücknahme | Reproduzierbare vollständige Partien |
| M3 – Klassische Engine | Bewertung, Suche, UCI, Suchabbruch; UCI-Match-Controller für Fremdengine und Selbstspiel | Legalitäts-, Protokoll- und Engine-Match-Tests erfolgreich |
| M4 – Desktop-Oberfläche | Spielbare Mensch/Mensch- und Mensch/Engine-Modi | End-to-End-Abnahmetests erfolgreich |
| M5 – 1.0-Stabilisierung | Dokumentation, Plattformmatrix, Performance-Baseline, Releases | LH-A-01 bis LH-A-16 erfüllt |

Diese Stufen definieren **Abhängigkeiten**, keine zugesagten Kalendertage.

## 12. Risiken, Annahmen und offene Entscheidungen

| ID | Punkt | Behandlung |
| --- | --- | --- |
| LH-O-01 | GUI-Technologie (z. B. SASD-UI-Toolkit oder anderer Adapter) | Nach einem kleinen lauffähigen GUI-Prototypen entscheiden; kein Zwang, unreife Komponenten in den Regelkern einzubauen. |
| LH-O-02 | Open-Source-Lizenz | Vor erster Veröffentlichung einer 1.0-Fassung verbindlich auswählen und Drittanbieter-Lizenzen prüfen. |
| LH-O-03 | Zielplattformen jenseits Windows/Linux | Nach vorhandenen CI-Ressourcen und Toolkit-Reife priorisieren. |
| LH-O-04 | Schachuhr und Zeitkontrollen | Als Soll-Funktion eingeplant; Regelwerk der Partiebeendigung ohne Uhr muss vorher korrekt sein. |
| LH-O-05 | Erreichbare Engine-Spielstärke | Über Vergleichspartien und reproduzierbare Benchmarks **messen**, nicht durch unbelegte Elo-Werte versprechen. |
| LH-O-06 | FIDE-Regeländerungen | Regelwerksbezug und daraus resultierende Anpassungen bei Bedarf versionieren. |
| LH-O-07 | Ressourcen und Deadlocks bei Engine-Suche | Abbruch, Timeouts, Thread- und Speichergrenzen früh testen. |
| LH-O-08 | Anforderungen an Barrierefreiheit | Vor GUI-Freigabe Bedienung ohne Drag-and-drop sowie Kontrast/Statusdarstellung prüfen. |
| LH-O-09 | Externe Engines mit UCI-Abweichungen | UCI-konformes Minimum garantieren; Erweiterungen und inkompatible Engines über klar getrennte Adapter und Fehlermeldungen behandeln. |

## 13. Änderungs- und Freigabeverfahren

Jede fachliche Änderung erhält eine dokumentierte Begründung, betroffene **LH-IDs**, Priorität, Auswirkungen auf Architektur/Tests sowie eine Entscheidung. Der Versionsstand des Lastenhefts wird gemeinsam mit dem Pflichtenheft angepasst. Als **Baselines** gelten ausschließlich ausdrücklich freigegebene Versionen.

**Nächster Entscheidungsschritt:** Den als Entwurf vorliegenden 1.0-Umfang samt Muss/Soll-Zuordnung prüfen und anschließend als fachliche Baseline freigeben.
