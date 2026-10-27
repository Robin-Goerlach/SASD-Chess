# SASD-Chess

SASD-Chess is an open-source C++ chess project with a modular architecture separating chess rules, legal move generation, game management, user interface, and a classical search engine. Designed for portability, clear code, automated tests, and UCI compatibility, it also provides a flexible foundation for future AI and machine learning experiments.

## Project status

**Requirements and technical design phase.** The following documents describe planned capabilities, **not** already completed features.

## Dokumentation (Deutsch)

- [Lastenheft](docs/Lastenheft.md) – fachliche Ziele, Anforderungen und Abnahmekriterien.
- [Pflichtenheft](docs/Pflichtenheft.md) – technische Architektur, Algorithmen, Schnittstellen und Prüfverfahren.

## Interoperabilität und Selbstspiel (geplant für Version 1.0)

SASD-Chess soll eine standardkonforme **UCI-Engine** bereitstellen, die mit anderen Schachoberflächen zusammenarbeitet. Ein separater, auch **ohne GUI** nutzbarer **Engine-Match-Controller** übernimmt die Rolle des Schiedsrichters: Er startet zwei voneinander unabhängige UCI-Prozesse, empfängt deren Züge, prüft jeden Zug mit dem eigenen Schachregelwerk, verwaltet die Partie und speichert das Ergebnis als PGN.

Damit sind zwei wichtige Einsatzfälle möglich:

- **SASD-Chess gegen eine fremde UCI-Engine**.
- **SASD-Chess gegen SASD-Chess:** Zwei parallele Prozesse desselben Programms spielen automatisch gegeneinander.

Erweiterte Turniere, Serienstatistiken und KI-Trainingsverfahren können später auf derselben Schnittstelle aufbauen.

> **Noch kein ausführbarer Match-Controller:** Kommandozeilenbeispiele im Pflichtenheft sind Entwürfe und noch nicht implementiert.
