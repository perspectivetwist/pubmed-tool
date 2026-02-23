# DEVELOPER.md — PubMed Recherche Tool

Technische Dokumentation für Engineers und zukünftige Claude-Sessions.

## Projektübersicht

Single-page HTML/JS App für einen Neurologen. Eine einzige Datei (`index.html`), kein Build-Step, kein Framework, kein Backend. Läuft im Browser, gehostet auf GitHub Pages.

**Was es tut:** Deutsche Freitext-Eingabe → PubMed-Suche mit MeSH-Validierung → Claude-Synthese → wissenschaftlich verwendbarer Output auf Deutsch.

## Dateistruktur

```
pubmed-tool/
├── index.html      ← Gesamte App (HTML + CSS + JS, ~1100 Zeilen)
├── README.md       ← Nutzer-Anleitung (für den Arzt)
├── DEVELOPER.md    ← Diese Datei (für Engineers)
├── CLAUDE.md       ← Kontext für Claude Code Sessions
└── WORK.md         ← Build-Dokumentation (wie das Projekt entstanden ist)
```

## Architektur

`index.html` ist in drei Blöcke unterteilt:

```
index.html (~1112 Zeilen)
├── CSS (~180 Zeilen)
│   ├── Design-System: CSS Custom Properties (--green, --purple, --orange, --base, --surface)
│   ├── Fonts: IBM Plex Mono (Headings/Code) + Inter (Body) via Google Fonts
│   ├── Layout: max-width 900px, responsive
│   └── Komponenten: Auth-Gate, Search, Filters, MeSH-Display, Output-Sections
│
├── HTML (~70 Zeilen)
│   ├── Auth-Gate (Passwort-Schutz)
│   └── App-Content
│       ├── API Key Setup/Status
│       ├── Suchformular
│       ├── Publikationstyp-Filter + Info-Popup
│       ├── Status + MeSH-Display
│       └── Output (Synthese | Fragen | Quellen)
│
└── JavaScript (~860 Zeilen)
    ├── Auth: SHA-256 Hash-Vergleich, localStorage
    ├── API Key: localStorage CRUD, maskierte Anzeige
    ├── Publikationstyp-Filter: load/save/getActive, localStorage-Persistenz
    ├── Wörterbuch: ~100 DE→EN medizinische Begriffe (MEDICAL_TERMS_DE_EN)
    ├── MeSH: lookupMeSH(), resolveMeSHTerms(), displayMeSHTerms()
    ├── Claude: optimizeQueryWithClaude(), synthesizeWithClaude()
    ├── PubMed: searchPubMed(), fetchArticles(), fetchWithTimeout()
    ├── Query: optimizeQuery() — Claude → MeSH → Filter → Fallback-Kaskade
    └── Suchflow: 4-stufige Strategie, Status-Updates, Rendering
```

## Suchflow

```
Nutzereingabe (deutsch)
  ↓
optimizeQueryWithClaude()     ← Freitext → englische PubMed-Query mit Synonymen
  ↓ (Fallback: optimizeQueryLocal() mit ~100 DE→EN Begriffen)
resolveMeSHTerms()            ← Parallel esearch + esummary pro Begriff gegen MeSH-DB
  ↓
Query-Aufbau: validierte → "Term"[MeSH], nicht-gefundene → term[tiab]
  ↓
getActivePubTypeFilter()      ← UI-Checkboxen → PubMed Pub-Type-Filter (OR-verknüpft)
  ↓
searchPubMed() — 4-stufige Fallback-Kaskade:
  1. MeSH-Query + Pub-Type-Filter
  2. MeSH-Query ohne Filter (bei <5 Treffer)
  3. Ungetaggte Base-Query (bei <5 Treffer)
  4. Lokales Wörterbuch (bei <5 Treffer)
  ↓
fetchArticles()               ← EFetch XML → DOMParser → JS-Objekte
  ↓
synthesizeWithClaude()        ← Abstracts als nummerierte Liste → Claude Synthese
  ↓ (Fallback: Top-30 Abstracts bei Token-Überschreitung)
Rendering: Synthese + Offene Fragen + Quellenliste mit PubMed-Links
```

## Konfigurierbare Parameter

| Parameter | Stelle im Code | Default | Erklärung |
|-----------|---------------|---------|-----------|
| Max. Artikel pro Suche | `searchPubMed()` → `retmax` | `50` | Balance Kosten vs. Vollständigkeit. PubMed liefert max. die relevantesten N. |
| Jahr-Filter | `searchPubMed()` → `mindate` | `2000` | Nur klinisch relevante Literatur ab 2000. |
| Claude Modell | `CLAUDE_MODEL` | `claude-sonnet-4-6` | Gutes Verhältnis Qualität/Kosten (~0.001$ pro Query). |
| Max Tokens Synthese | `synthesizeWithClaude()` → `max_tokens` | `4096` | Reicht für 3–5 Absätze Synthese + Fragen. |
| Max Tokens Query-Optimierung | `optimizeQueryWithClaude()` → `max_tokens` | `300` | Nur ein Suchstring, braucht wenig Tokens. |
| Token-Limit Input | `TOKEN_LIMIT` | `150000` | Sicherheitspuffer unter Claudes 200k Context-Limit. |
| Fallback-Artikelanzahl | `FALLBACK_ARTICLE_COUNT` | `30` | Bei Token-Überschreitung werden nur Top-30 Abstracts gesendet. |
| MeSH-Lookup Timeout | `lookupMeSH()` → `AbortController` | `5000` ms | Pro Begriff. Verhindert, dass ein langsamer MeSH-Lookup die gesamte Suche blockiert. |
| PubMed Timeout | `fetchWithTimeout()` | `30000` ms (ESearch), `60000` ms (EFetch) | EFetch braucht länger bei vielen Artikeln. |
| Passwort-Hash | `AUTH_HASH` | SHA-256 Hash | Im Code sichtbar — reicht nur gegen Zufallsbesucher. |
| Sort-Reihenfolge | `searchPubMed()` → `sort` | `relevance` | PubMed-Relevanz-Ranking. |

### localStorage Keys

| Key | Inhalt |
|-----|--------|
| `pubmed-tool-auth` | SHA-256 Hash des Passworts (Session-Persistenz) |
| `pubmed-tool-anthropic-key` | Anthropic API Key (Klartext) |
| `pubmed-tool-pub-filters` | JSON mit Checkbox-States der Publikationstyp-Filter |

## Externe APIs

| API | Endpunkt | Auth | CORS | Verwendung |
|-----|----------|------|------|------------|
| PubMed ESearch | `eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi` | Keine | Ja | Artikel-IDs finden (`db=pubmed`) + MeSH-Validierung (`db=mesh`) |
| PubMed EFetch | `eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi` | Keine | Ja | Abstracts laden (nur XML, kein JSON) |
| PubMed ESummary | `eutils.ncbi.nlm.nih.gov/entrez/eutils/esummary.fcgi` | Keine | Ja | Offiziellen MeSH-Term-Namen holen |
| Claude API | `api.anthropic.com/v1/messages` | API Key | Via `anthropic-dangerous-direct-browser-access: true` Header | Query-Übersetzung + Literatur-Synthese |

## Lokale Entwicklung

```bash
# Kein Build nötig — direkt im Browser öffnen:
open index.html

# Oder lokaler Server (empfohlen für konsistentes Verhalten):
python3 -m http.server 8080
# → http://localhost:8080
```

Voraussetzung: Ein Anthropic API Key (`sk-ant-...`). Wird beim ersten Start im Browser eingegeben und in localStorage gespeichert.

## Deployment

```bash
# Änderung committen → automatisch live via GitHub Pages (Legacy Build, main branch)
git add index.html
git commit -m "fix: beschreibung der änderung"
git push origin main
# Live in ~30 Sekunden: https://perspectivetwist.github.io/pubmed-tool
```

GitHub Pages ist als Legacy Build konfiguriert (main branch, root). Kein GitHub Actions Workflow nötig.

## Bekannte Limitierungen

- **API Key in localStorage:** Klartext, nur für Single-User geeignet. Für öffentliche Deployments bräuchte man ein Backend mit Key-Verwaltung.
- **Kein Rate-Limiting gegenüber PubMed:** Bei sehr schnellen aufeinanderfolgenden Suchen evtl. HTTP 429. PubMed empfiehlt max. 3 Requests/Sekunde ohne API Key.
- **Passwort-Schutz ist kein echter Schutz:** SHA-256 Hash ist im Quellcode sichtbar. Reicht gegen Zufallsbesucher, nicht gegen jemanden der die DevTools öffnet.
- **PubMed EFetch nur XML:** Kein JSON-Endpoint für Artikeldaten verfügbar. DOMParser im Browser parst zuverlässig, aber Structured Abstracts (mehrere `AbstractText`-Elemente mit Label-Attribut) müssen manuell zusammengefügt werden.
- **Token-Limit:** 50 Abstracts × ~300 Tokens ≈ 15k Tokens Input. Bei sehr langen Abstracts greift der Fallback auf Top-30.
- **Keine Export-Funktion:** Ergebnisse können nur per Copy-Paste aus dem Browser übernommen werden.
- **Keine Offline-Fähigkeit:** Braucht Internet für PubMed und Claude API.

## Code-Kommentar-Konvention

Jeder logische Block hat einen deutschen Kommentar als Einzeiler:

```javascript
// MeSH-Term Lookup: Validiert einen Suchbegriff gegen die PubMed MeSH-Datenbank
async function lookupMeSH(term) { ... }

// Claude-basierte Query-Optimierung: übersetzt deutsche Fragestellungen
// in optimale englische PubMed-Suchbegriffe mit MeSH-Terms
async function optimizeQueryWithClaude(rawInput) { ... }

// --- Publikationstyp-Filter ---
const PUB_TYPE_STORAGE = 'pubmed-tool-pub-filters';
```

Trenn-Kommentare (`// --- Abschnitt ---`) gruppieren zusammengehörige Funktionen.
