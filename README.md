# SASD-Chess
SASD-Chess is an open-source C++ chess project with a modular architecture separating chess rules, legal move generation, game management, user interface, and a classical search engine. Designed for portability, clear code, automated tests, and UCI compatibility, it also provides a flexible foundation for future AI and machine learning experiments.

## Projektplanung / Project planning

SASD-Chess befindet sich in der Planungsphase. Die folgenden Dokumente beschreiben den **geplanten Umfang**, nicht bereits fertig implementierte Funktionen:

- [Lastenheft (Deutsch)](docs/Lastenheft.md) – fachliche Ziele, Anforderungen und Abnahmekriterien.
- [Pflichtenheft (Deutsch)](docs/Pflichtenheft.md) – geplante C++-Architektur, Umsetzung und Tests.

### Engine-Interoperabilität und Selbstspiel

SASD-Chess soll sowohl als **UCI-Engine** für externe Schachoberflächen als auch als **UCI-Match-Controller** arbeiten können. Der Controller soll zwei unabhängige Engine-Prozesse starten und automatisch gegeneinander spielen lassen – ausdrücklich auch zwei parallele Instanzen von SASD-Chess. Alle Züge werden zentral auf Legalität überprüft und können als PGN ausgegeben werden.

**Status:** Die UCI-Match-Funktionen sind spezifiziert, aber noch nicht implementiert. Für die erste Version sind C++ und eine entkoppelte, testbare Architektur vorgesehen.
