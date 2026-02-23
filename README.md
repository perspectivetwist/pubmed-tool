# PubMed Recherche Tool

Browserbasiertes Single-Page-Tool für neurologische Literaturrecherche.

Freitext-Eingabe → PubMed-Suche → Claude-Analyse → lesbare Zusammenfassung auf Deutsch.

## Anleitung

1. **Link öffnen:** [perspectivetwist.github.io/pubmed-tool](https://perspectivetwist.github.io/pubmed-tool)
2. **API Key eintragen:** Anthropic API Key eingeben (wird nur lokal im Browser gespeichert)
3. **Suchen:** Fragestellung auf Deutsch eingeben, z.B. *„Migräne Behandlung bei Kindern"*

Das Tool übersetzt automatisch ins Englische, sucht in PubMed nach Review-Artikeln und erstellt eine wissenschaftliche Synthese mit Quellenverweisen.

## Ergebnis

- **Literatursynthese** — 3–5 Absätze mit Quellenverweisen [Erstautor Jahr]
- **Offene Fragen & Kontroversen** — direkt aus der Literatur abgeleitet
- **Quellenliste** — klickbare Links zu PubMed-Artikeln

## Technisch

- Einzelne HTML-Datei, kein Backend
- PubMed E-utilities API (kostenlos, offen)
- Claude API für Analyse (eigener API Key erforderlich)
- Gehostet auf GitHub Pages (HTTPS)
