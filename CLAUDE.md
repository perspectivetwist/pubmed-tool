# PubMed Recherche Tool – CLAUDE.md

## Projekt
Browserbasiertes Single-Page-Tool für einen Neurologen (kein Tech-Background).
Freitext-Eingabe → PubMed sucht → Claude fasst zusammen → Ergebnis lesbar.

## Stack
- index.html (eine einzige Datei, kein Backend)
- PubMed E-utilities API (kostenlos, CORS erlaubt seit 2009)
- Claude API claude-sonnet-4-6 (Analyse + Synthese)
- GitHub Pages Hosting (kostenlos, HTTPS)
- API Key in localStorage (einmalige Eingabe, Single-User-Tool)

## GitHub
- Account: perspectivetwist
- Repo-Name: pubmed-tool
- Pages URL: https://perspectivetwist.github.io/pubmed-tool

## Notion
- Projektplan-DB: https://www.notion.so/53a2f554f86847b38fb91911f05b45c5
- Hauptseite: https://www.notion.so/310a9eccf6e38193b904e90a69a894d8

## Output (immer Deutsch)
1. Literatursynthese (3–5 Absätze, Quellenverweise [Erstautor Jahr])
2. Offene Fragen & Kontroversen
3. Quellenliste (Titel, Autoren, Journal, Jahr, klickbarer PubMed-Link)

## Technische Entscheidungen
- 50 Artikel pro Suche, nur ab Jahr 2000, Review-Artikel bevorzugt
- Fallback: Top-30 Abstracts wenn Token-Limit überschritten
- Keine Export-Funktion in V1

## Arbeitsweise
- Tasks sequentiell: 1.1 → 1.2 → 2.1 → 2.2 → 2.3 → 2.4 → 3.1 → 3.2 → 3.3 → 3.4 → 4.1 → 4.2 → 4.3 → 5.1 → 5.2 → 5.3
- Nach jedem Task: Status in Notion → "Done", Lessons Learned eintragen
- Nur fragen wenn wirklich blockiert (z.B. API Key fehlt)
