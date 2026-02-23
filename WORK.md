# Work Document — PubMed Recherche Tool

## Zusammenfassung

Browserbasiertes Single-Page-Tool (eine `index.html`, kein Backend) für einen Neurologen.
Freitext-Eingabe auf Deutsch → PubMed-Suche → Claude-Synthese → wissenschaftlich verwendbarer Output.

- **Stack:** HTML/CSS/JS, PubMed E-utilities API, Claude API (claude-sonnet-4-6), GitHub Pages
- **Repo:** [perspectivetwist/pubmed-tool](https://github.com/perspectivetwist/pubmed-tool)
- **Live:** https://perspectivetwist.github.io/pubmed-tool
- **Zeitraum:** 23. Februar 2026, 14:38–20:18 Uhr (ca. 5.5h effektiv)
- **Ergebnis:** 1112 Zeilen, 18 Commits, alle 18 Tasks Done

---

## Projektplan-Abarbeitung

### Phase 1: GitHub Setup (14:38–14:56)

| Task | Commit | Was passiert ist |
|------|--------|-----------------|
| 1.1 Repo anlegen | `b1c4d6f` | `gh repo create` mit `--source .` und `--push`. Placeholder `index.html` + `README.md` + `CLAUDE.md` angelegt. |
| 1.2 GitHub Pages | `cb83de5` | Legacy build aktiviert (main branch, root). Erster Build manuell via `POST /pages/builds` getriggert. HTTPS enforced. |

**Entscheidung:** Legacy build statt GitHub Actions Workflow — einfacher für ein statisches Single-File-Projekt.

### Phase 2: PubMed API (15:00–15:12)

| Task | Commit | Was passiert ist |
|------|--------|-----------------|
| 2.1 ESearch | `8e62a6c` | `esearch.fcgi` mit `db=pubmed`, `retmax=50`, `mindate=2000`, `sort=relevance`, `retmode=json`. CORS funktioniert out-of-the-box. |
| 2.2 Query-Optimierung | `d0ae88c` | Deutsch→Englisch Wörterbuch mit ~30 neurologischen Begriffen. Longest-match-first für Mehrwort-Terme. Review-Artikel-Filter `(query) AND (review[pt] OR systematic review[pt] OR meta-analysis[pt])`. |
| 2.3 EFetch | `28beef8` | `efetch.fcgi` mit `retmode=xml`. DOMParser für XML. Abstracts mit mehreren `AbstractText`-Elementen (Label-Attribut) zusammengefügt. |
| 2.4 Fehlerbehandlung | `0b8ec32` | `AbortController` mit 30s Timeout. HTTP 429/5xx/Offline-Erkennung. Alle Fehlermeldungen auf Deutsch. |

**Entscheidung:** PubMed EFetch liefert nur XML (kein JSON für Artikeldaten) — DOMParser im Browser parst das zuverlässig.

### Phase 3: Claude API (15:13–15:20)

| Task | Commit | Was passiert ist |
|------|--------|-----------------|
| 3.1 API Key | `20e6891` | `localStorage` für Key-Persistenz. Prefix-Validierung (`sk-ant-`). Maskierte Anzeige. |
| 3.2 Synthese | `6c64b13` | `anthropic-dangerous-direct-browser-access: true` Header für Browser-CORS. Abstracts als nummerierte Liste `[1]...[n]` für referenzierbare Quellenverweise. |
| 3.3 System Prompt | `e9bc4db` | Prompt fordert: Deutsch, Quellenverweise `[Erstautor Jahr]`, konkrete Zahlen (NNT, OR, CI, p-Werte), thematische Gliederung, Widersprüche darstellen. Output: 3–5 Absätze Synthese + 3–6 offene Fragen. |
| 3.4 Token-Limit | `acbd784` | Doppelte Absicherung: Pre-Check mit `estimateTokens()` (~4 chars/token, Limit 150k) + Retry bei HTTP 400. Fallback: Top-30 Abstracts mit Transparenz-Hinweis. |

**Entscheidung:** `claude-sonnet-4-6` statt Opus — gutes Verhältnis Qualität/Kosten für Synthesen (~0.001$ pro Query).

### Phase 4: UI (15:25–15:32)

| Task | Commit | Was passiert ist |
|------|--------|-----------------|
| 4.1 Layout | `4e65623` | "Psychedelic Noir" Design: Dark base (#1A1C1E), drei Akzentfarben — Grün (Synthese), Purple (Fragen), Orange (Quellen). CSS Custom Properties. IBM Plex Mono + Inter. Glow-Effekte auf Hover. |
| 4.2 Ladeindikator | `ae529f5` | `setStatus()` Helper. CSS-Spinner (14px, border-Animation). Pulse-Glow-Animation für laufende Status. Phasen: "Suche in PubMed..." → "Abstracts werden geladen..." → "Analysiere mit KI..." |
| 4.3 Quellenliste | — | Bereits in 4.1 mit implementiert (klickbare PubMed-Links, Autoren/Journal/Jahr). |

**Entscheidung:** Task 4.3 war in 4.1 bereits miterledigt — kommt bei sequentieller Planung vor, kurze Prüfung reicht.

### Phase 5: Deploy & Test (15:33–15:35)

| Task | Commit | Was passiert ist |
|------|--------|-----------------|
| 5.1 Deployment | — | GitHub Pages war seit 1.2 aktiv. Push auf main → automatisches Deployment. |
| 5.2 Live-Test | — | 3 Test-Queries: "Migräne Prophylaxe Kinder", "Autoimmune Enzephalitis", "Deep Brain Stimulation Parkinson". Alle liefern 50+ Reviews. |
| 5.3 README | `40817a5` | 3-Schritte-Anleitung: Link öffnen → API Key → Suchen. |

### Phase 6: Iteration (16:00–20:18)

| Task | Commit(s) | Was passiert ist |
|------|-----------|-----------------|
| Claude Query-Optimierung | `5d979de` | Problem: Deutsche Wörter wie "Differentialdiagnose" blieben in der Query (nicht im Wörterbuch). Lösung: Claude übersetzt die komplette Fragestellung in eine breite englische PubMed-Query. System-Prompt instruiert Synonyme mit OR. Fallback auf lokales Wörterbuch wenn API-Call scheitert. |
| Erweitertes Wörterbuch | `5d979de` | ~100 neue Begriffe (autoimmun, antikörper, liquor, MRT, immuntherapie, biomarker...). 3-stufige Suchstrategie: Reviews → Alle Typen → Lokales Wörterbuch. |
| Passwort-Schutz | `b0cd7f2` | SHA-256 gehashtes Passwort im Code. Auth-Gate versteckt App-Content. localStorage für Session-Persistenz. Kein echter Schutz (Hash sichtbar), reicht für Zufallsbesucher. |
| MeSH-Term Optimierung | `178fc6c` | Neue Funktionen: `lookupMeSH()` (esearch + esummary, 5s Timeout), `resolveMeSHTerms()` (parallel via Promise.allSettled), `displayMeSHTerms()`. Query-Begriffe werden gegen PubMed MeSH-DB validiert: gefundene → `"Term"[MeSH]`, nicht-gefundene → `term[tiab]`. Suchstrategie jetzt 4-stufig. UI: Grüne MeSH-Tags + graue Freitext-Anzeige unter Suchfeld. |
| Publikationstyp-Filter | `178fc6c`, `2f1d7b2`, `ce07f32` | 5 Checkboxen (Syst. Reviews, Meta-Analysen, RCTs, Fallberichte, Editorials) mit localStorage-Persistenz. Ersetzen hardcoded Review-Filter. Auto-Fallback bei <5 Treffern. Info-Popup mit Evidenzhierarchie-Erklärung (Purple-Akzent, Neon-Green Labels). |

---

## Architektur (finale Version)

```
index.html (1112 Zeilen, eine einzige Datei)
├── CSS (~180 Zeilen)
│   ├── Design-System: CSS Custom Properties (--green, --purple, --orange, --base, --surface...)
│   ├── Layout: max-width 900px, responsive
│   ├── Komponenten: Search, Filters, Status, MeSH-Display, 3 Output-Sections, Auth-Gate
│   └── Animationen: Spinner, Pulse-Glow, Hover-Transitions
│
├── HTML (~70 Zeilen)
│   ├── Auth-Gate (Passwort-Schutz)
│   ├── App-Content
│   │   ├── API Key Setup/Status
│   │   ├── Suchformular
│   │   ├── Publikationstyp-Filter + Info-Popup
│   │   ├── Status + MeSH-Display
│   │   └── Output (Synthese | Fragen | Quellen)
│   └── Keine externen Abhängigkeiten (nur Google Fonts)
│
└── JavaScript (~860 Zeilen)
    ├── Auth: SHA-256 Hash-Vergleich, localStorage
    ├── API Key: localStorage CRUD, maskierte Anzeige
    ├── Wörterbuch: ~100 DE→EN medizinische Begriffe
    ├── MeSH: lookupMeSH(), resolveMeSHTerms(), displayMeSHTerms()
    ├── Claude: optimizeQueryWithClaude(), synthesizeWithClaude()
    ├── PubMed: searchPubMed(), fetchArticles(), fetchWithTimeout()
    ├── Query: optimizeQuery() (Claude → MeSH → Filter → Fallback-Kaskade)
    ├── Filter: load/save/getActive PubTypeFilters, Info-Toggle
    └── Suchflow: 4-stufige Strategie, Status-Updates, Rendering
```

## Suchflow (finale Version)

```
Nutzereingabe (deutsch)
  ↓
Claude-Übersetzung → englische Query mit MeSH-Terms + Synonymen
  ↓ (Fallback: lokales Wörterbuch mit ~100 Begriffen)
MeSH-Validierung (parallel esearch + esummary pro Begriff)
  ↓
Query-Aufbau: validierte MeSH-Terms [MeSH] + Freitext [tiab]
  ↓
Publikationstyp-Filter aus UI-Checkboxen anhängen (OR-verknüpft)
  ↓
PubMed ESearch — 4-stufige Fallback-Kaskade:
  1. MeSH-Query + Pub-Type-Filter
  2. MeSH-Query ohne Filter (bei <5 Treffer)
  3. Ungetaggte Query (bei <5 Treffer)
  4. Lokales Wörterbuch (bei <5 Treffer)
  ↓
EFetch: XML → Titel, Autoren, Journal, Jahr, Abstract
  ↓
Claude Synthese (claude-sonnet-4-6, max 4096 Tokens)
  ↓ (Fallback: Top-30 Abstracts bei Token-Überschreitung)
Output: Literatursynthese + Offene Fragen + Quellenliste
```

## Externe APIs

| API | Endpunkt | Auth | CORS | Verwendung |
|-----|----------|------|------|------------|
| PubMed ESearch | `eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi` | Keine | Ja | Artikel-IDs finden |
| PubMed EFetch | `eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi` | Keine | Ja | Abstracts laden (XML) |
| PubMed ESummary | `eutils.ncbi.nlm.nih.gov/entrez/eutils/esummary.fcgi` | Keine | Ja | MeSH-Term-Namen |
| Claude API | `api.anthropic.com/v1/messages` | API Key | Via Header | Übersetzung + Synthese |

## Key Decisions & Lessons Learned

### Architektur
- **Single-File statt SPA-Framework:** Bewusste Entscheidung für eine `index.html`. Kein Build-Step, kein Node, kein Framework. Direkt auf GitHub Pages deploybar. Für ein Single-User-Tool mit einer Seite ist das die einfachste Lösung.
- **Kein Backend:** PubMed CORS funktioniert, Claude API mit `anthropic-dangerous-direct-browser-access` Header auch. API Key bleibt im Browser (localStorage). Für ein Tool mit einem Nutzer akzeptabel.

### PubMed API
- **EFetch nur XML:** PubMed liefert Artikeldaten nur als XML — `DOMParser` im Browser parst das zuverlässig.
- **AbstractText mit Labels:** Structured Abstracts haben mehrere `AbstractText`-Elemente mit Label-Attribut (AIMS, METHODS, RESULTS) — alle zusammenfügen.
- **MeSH esummary:** Offizieller Term-Name unter `ds_meshterms[0]` oder `title` — beides abfangen.
- **systematic[sb] vs systematic review[pt]:** Der Subset-Filter `[sb]` ist breiter und fängt auch Cochrane Reviews etc. ab.

### Claude API
- **Browser-CORS:** `anthropic-dangerous-direct-browser-access: true` Header ermöglicht direkten Browser-Zugriff.
- **Token-Management:** Doppelte Absicherung (Pre-Check + Retry bei 400) mit Fallback auf Top-30 Abstracts.
- **Prompt-Design:** Strukturvorgaben + explizite Forderung nach konkreten Zahlen erhöht die Qualität massiv.

### Suchqualität
- **Claude-Übersetzung:** Statisches Wörterbuch skaliert nicht für medizinische Fachsprache. LLM-Übersetzung ist robuster (~0.001$ pro Query).
- **MeSH-Validierung:** Begriffe gegen echte MeSH-DB validieren statt blind `[MeSH]`-Tags setzen.
- **Fallback-Kaskade:** 4 Stufen stellen sicher, dass auch bei Edge Cases Treffer gefunden werden.
- **UI-gesteuerte Filter:** Flexibler als hardcoded Review-Filter. Der Nutzer kontrolliert die Evidenzstufe.

### UI/UX
- **Tooltips auf Deutsch:** Wichtig für den nicht-technischen Endnutzer.
- **Evidenzhierarchie-Popup:** Erklärt dem Nutzer den Impact der Filterauswahl auf die Datenqualität.
- **MeSH-Anzeige:** Transparenz darüber, welche Begriffe als offizielle MeSH-Terms erkannt wurden.

## Git-Statistik

```
18 Commits, 1 Datei (index.html), 1112 Zeilen
Zeitraum: 23.02.2026, 14:38–20:18 Uhr

Commit-Verteilung:
  Phase 1 (Setup):     2 Commits
  Phase 2 (PubMed):    4 Commits
  Phase 3 (Claude):    4 Commits
  Phase 4 (UI):        2 Commits
  Phase 5 (Deploy):    1 Commit
  Phase 6 (Iteration): 5 Commits
```

## Notion Projektplan

Alle 18 Tasks in 6 Phasen auf "Done". Jeder Task hat:
- **DoD** (Definition of Done) — messbare Kriterien
- **Notizen** — technische Details der Implementierung
- **Lessons Learned** — was man beim nächsten Mal anders machen würde
- **Abhängigkeiten** — welche Tasks vorher fertig sein mussten

Projektplan-DB: https://www.notion.so/53a2f554f86847b38fb91911f05b45c5
