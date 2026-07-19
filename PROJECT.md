# Portfolio Terminal

Eine private, browserbasierte Vermögensübersicht ("Terminal"-Look) für Krypto- und ETF-Positionen, die alle Werte automatisch in CHF konsolidiert. Läuft als einzelne `index.html` ohne Build-Schritt und wird über GitHub Pages ausgeliefert.

## Was die App macht

- **Zentrales Portfolio-Dashboard** mit drei Tabs: Übersicht, Krypto, ETFs.
- **Positionen** werden in CHF normalisiert dargestellt, unabhängig davon in welcher Original-Währung/Broker sie gehalten werden (EUR, USD, GBP/GBX, CHF).
- **Live-Kurse** werden aus mehreren Quellen parallel geladen und aktualisieren die gespeicherten Positionen:
  - Binance (Krypto-Preise)
  - CoinGecko (Kaspa, direkt in CHF)
  - Yahoo Finance, via eigenem Cloudflare Worker als Proxy (ETF-Kurse, inkl. London-Notierungen in GBp/Pence)
- **Tageswechselkurse** (CHF pro Fremdwährung) via `open.er-api.com`, mit 1h-TTL-Cache in `localStorage`, damit Tab-Wechsel keine falschen Neu-Umrechnungen auslösen.
- **KI-gestützte Depot-Erfassung**: Screenshots oder kopierter Text von Broker-Apps (z. B. Swissquote, deutsche ETF-Broker) werden an Claude (via Cloudflare Worker) geschickt; die KI erkennt Quelle, Positionen und Währung und liefert strukturierte JSON-Daten zurück, die automatisch ins Portfolio übernommen werden.
- **ETF-Holdings-Deduplizierung**: Mehrfach erfasste Holdings (z. B. aus wiederholten Uploads) werden beim Speichern und bei der Migration anhand des normalisierten Tickers zusammengeführt (Wert summiert, P/L gewichtet gemittelt).
- **Persistenz**: Daten werden primär in Cloudflare KV gespeichert (`/api/save`, `/api/load` über den Worker) und zusätzlich lokal in `localStorage` gecacht/als Fallback verwendet, falls der Worker nicht erreichbar ist.
- **Conviction-Score** pro Position (0–100), Einstandspreis, P/L%, Plattform/Broker-Zuordnung.

## Tabs / Struktur

| Tab | Inhalt |
|---|---|
| Übersicht | Allokation nach Kategorie (Krypto / ETF / Cash) als Pie-Chart, Gesamtwert, Stat-Cards, Positionstabelle mit aufklappbaren ETF-Holdings |
| Krypto | Einzelpositionen BTC, ETH, SOL, KAS, RNDR (Plattformen u. a. BitBox02, Swissquote, Kraken) |
| ETFs | Depots je Broker (z. B. Swissquote: V3AA; Deutsches ETF-Depot: iShares-Holdings wie `ISHSIII-M.W.S.C.U.ETF DLA`) |

Ein Upload-Modal erlaubt die Zuordnung neuer Screenshots/Texte zu einer Quelle (`swissquote`, `etfDE`, …) vor dem KI-Parsing.

## Tech-Stack

Reines Frontend, kein Build-Tool, keine `node_modules` — alles läuft direkt im Browser über CDN-Skripte in `index.html`:

- **React 18** (UMD build) + **Babel Standalone** (JSX wird zur Laufzeit im Browser transpiliert, `<script type="text/babel">`)
- **Tailwind CSS** via CDN (`cdn.tailwindcss.com`)
- **Recharts** für Pie-/Area-/Line-Charts
- Inline-SVG-Icons statt Icon-Library
- **Cloudflare Worker** (`portfolio.y68xps2sdk.workers.dev`) als serverseitiger Proxy für:
  - Claude API (hält den API-Key serverseitig, verarbeitet Bild+Text-Uploads)
  - Yahoo-Finance-Kursabfragen (`/api/yahoo`)
  - Cloudflare KV Persistenz (`/api/save`, `/api/load`)

## Architektur-Hinweise (in `index.html`)

Die Datei ist in klar kommentierte Abschnitte gegliedert:

1. Icons
2. Seed-Daten (initiale Beispielpositionen)
3. Storage Layer (`localStorage`, inkl. ETF-Holdings-Dedup beim Speichern/Migrieren)
4. Claude API via Cloudflare Worker (Vision + Text)
5. FX-Kurse (Wechselkurs-Fetch + Cache)
6. Live-Preise (Binance/CoinGecko/Yahoo, Ticker-Mapping `ETF_HOLDING_TICKERS`)
7. Cloudflare KV Storage (Save/Load)
8. Utilities & UI-Komponenten
9. Setup-Screen (API-Key/Erststart)
10. Upload-Modal
11. Main Dashboard (`App()`, Tab-Routing)
12. Overview-/Crypto-/ETFs-Tab-Komponenten

## Lokal ausführen

Kein Build nötig — einfach `index.html` in einem lokalen Server öffnen (wegen `fetch`-Aufrufen empfiehlt sich ein simpler Static-Server statt `file://`):

```bash
npx serve .
# oder
python3 -m http.server
```

## Deployment

Die `index.html` wird direkt aus dem Repository ausgeliefert (z. B. GitHub Pages). Es gibt keinen Build-/CI-Schritt.

## Offene/bekannte Punkte

- Seed-Daten in Abschnitt "SEED DATA" sind Beispielwerte (Stand Mai 2026) und dienen als Fallback vor dem ersten KV-Load.
- `ETF_HOLDING_TICKERS` muss manuell gepflegt werden, wenn neue ETF-Holdings hinzukommen (Ticker-Zuordnung ISIN → Yahoo-Symbol).
- Der Cloudflare-Worker-Code selbst liegt nicht in diesem Repo.
