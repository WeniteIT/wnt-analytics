# wnt-analytics – WENITE-Betriebsdoku

Umami 3.x (Fork von `umami-software/umami`) als eigene Webanalyse für **wenite.de** (später auch wa365.de).

- **Hosting:** Vercel, Funktionsregion `fra1` (Frankfurt), Domain `analytics.wenite.de`
- **Datenbank:** Supabase Postgres (Free), Region Frankfurt (`eu-central-1`)
- **Grundregel:** Am Umami-Quellcode wird **nichts** geändert. Alles WENITE-Spezifische liegt in diesem Ordner `wenite/` oder in den Vercel-Einstellungen. So bleiben Updates aus upstream konfliktfrei.

Dateien in diesem Ordner:
- `README.md` – diese Anleitung
- `example.env` – alle Umgebungsvariablen mit Erklärung (Vorlage, keine echten Werte)
- `docker-compose.yml` – lokales Umami + Postgres zum Testen

---

## 1. Lokal starten (Docker Desktop)

```powershell
cd C:\Repositories\Wenite\wnt-analytics
docker compose -f wenite/docker-compose.yml up -d
```

- Aufruf: http://localhost:3000 – Login `admin` / `umami` → Passwort sofort ändern
- Sprache steht auf Deutsch, Telemetrie ist aus
- Tracker-Script lokal: http://localhost:3000/wnt.js
- Stoppen: `docker compose -f wenite/docker-compose.yml down`
- Alles zurücksetzen (Daten weg): `docker compose -f wenite/docker-compose.yml down -v`
- Logs: `docker compose -f wenite/docker-compose.yml logs -f umami`

**Schnelltest Tracking:** In Umami eine Website `localhost` anlegen, Website-ID kopieren, dann eine Test-HTML-Datei öffnen mit:

```html
<script defer src="http://localhost:3000/wnt.js" data-website-id="WEBSITE-ID"></script>
<button data-umami-event="test_klick">Test</button>
```

Seitenaufruf und Klick müssen nach wenigen Sekunden im Dashboard unter „Echtzeit“ erscheinen.

---

## 2. Supabase einrichten

1. https://supabase.com → **New project**
   - Name: `wnt-analytics`
   - Region: **Central EU (Frankfurt)**
   - Datenbank-Passwort: lang und zufällig, im Passwort-Manager ablegen
2. **Connect** (oben) → Reiter „Connection string“:
   - **Transaction pooler** (Port **6543**) → wird `DATABASE_URL`
   - **Session pooler** (Port **5432**, gleicher Host) → wird `DIRECT_DATABASE_URL`
   - `[YOUR-PASSWORD]` in beiden URLs ersetzen
   - Die „Direct connection“ (`db.<ref>.supabase.co`) **nicht** nehmen: ohne IPv4-Add-on nur per IPv6 erreichbar, das klappt von Vercel aus nicht zuverlässig.
3. **AV-Vertrag:** Supabase-DPA unter https://supabase.com/legal/dpa akzeptieren und als PDF ablegen.
4. Kein Schema anlegen – Umami legt beim ersten Build alle Tabellen selbst an.

Warum zwei URLs? Umami läuft im Betrieb über den Transaction Pooler (viele kurze Verbindungen, passend zu Vercel). Die Datenbank-Migrationen beim Build laufen über `DIRECT_DATABASE_URL` (siehe `scripts/check-db.js`), weil Prisma-Migrationen eine stabile Sitzung brauchen.

---

## 3. Vercel-Projekt anlegen

1. Vercel → **Add New → Project** → Repo `WeniteIT/wnt-analytics` importieren
   - Team: das WENITE-Team (Pro) – der Hobby-Plan ist nicht für kommerzielle Nutzung
   - Framework: Next.js (wird erkannt)
   - Production Branch: `master`
   - Install Command kommt aus `vercel.json` (pnpm 12.3.4) – nichts ändern
   - Build/Output: Standard lassen
2. **Environment Variables** (Production + Preview): alle Werte aus `example.env`
   - `APP_SECRET` und `TWO_FACTOR_ENCRYPTION_KEY` je **eigenen** Wert erzeugen. PowerShell:
     ```powershell
     -join ((1..32) | % { '{0:x2}' -f (Get-Random -Max 256) })
     ```
3. **Deploy** klicken. Im Build-Log müssen diese Zeilen erscheinen:
   - `✓ Database connection successful.`
   - `✓ Database version check successful.`
   - `✓ Database is up to date.`
4. **Settings → Functions → Function Region:** `Frankfurt, Germany (fra1)` → danach einmal neu deployen
5. **Settings → Domains:** `analytics.wenite.de` hinzufügen, den angezeigten DNS-Eintrag (CNAME) beim Domain-Anbieter setzen
6. **AV-Vertrag Vercel:** DPA unter https://vercel.com/legal/dpa – als PDF ablegen
7. Optional: **Settings → Git → Ignored Build Step**, damit Commits, die nur `wenite/` ändern, keinen neuen Build auslösen:
   ```
   git diff --quiet HEAD^ HEAD -- . ':(exclude)wenite'
   ```

Hinweis Geo-Daten: Auf Vercel nimmt Umami Land/Region/Stadt aus den Vercel-Headern (`x-vercel-ip-country` …). Es wird keine GeoIP-Datenbank heruntergeladen.

---

## 4. Erster Login und Einrichtung

1. https://analytics.wenite.de → `admin` / `umami`
2. **Sofort:** Profil → Passwort ändern. Besser: eigenen Admin-Benutzer anlegen, `admin` löschen.
3. Zwei-Faktor-Anmeldung aktivieren (Profil)
4. **Einstellungen → Websites → Website hinzufügen**
   - Name: `wenite.de`, Domain: `wenite.de`
   - Website-ID kopieren → kommt in wnt-web-v2 als `NEXT_PUBLIC_UMAMI_WEBSITE_ID`
5. Eigene Besuche nicht zählen: auf wenite.de in der Browser-Konsole einmalig
   `localStorage.setItem('umami.disabled', 1)` ausführen (pro Browser), oder Büro-IP in `IGNORE_IP` eintragen

---

## 5. Updates aus upstream holen

Auf GitHub: Repo-Startseite → **Sync fork → Update branch**. Vercel deployt danach automatisch, die Migrationen laufen im Build mit.

Oder lokal:

```powershell
git fetch upstream
git checkout master
git merge upstream/master
git push
```

Vor großen Versionssprüngen (z. B. 3.x → 4.x) die Release Notes lesen: https://github.com/umami-software/umami/releases
Datensicherung vorher: Supabase Free hat **keine** automatischen Backups. Manuell sichern:

```powershell
pg_dump "<DIRECT_DATABASE_URL>" -Fc -f wnt-analytics-JJJJ-MM-TT.dump
```

---

## 6. Einbindung in wnt-web-v2 (nur Beschreibung – Umsetzung im Website-Repo)

**Variante A – direkt (zum Start empfohlen):**

```tsx
// app/layout.tsx
import Script from 'next/script';

{process.env.NEXT_PUBLIC_UMAMI_WEBSITE_ID && (
  <Script
    src="https://analytics.wenite.de/wnt.js"
    data-website-id={process.env.NEXT_PUBLIC_UMAMI_WEBSITE_ID}
    data-domains="wenite.de,www.wenite.de"
    data-exclude-hash="true"
    strategy="afterInteractive"
  />
)}
```

**Variante B – über wenite.de geproxyt (weniger Adblocker-Verluste):**

```ts
// next.config – rewrites
{ source: '/wnt.js', destination: 'https://analytics.wenite.de/wnt.js' },
{ source: '/api/wnt', destination: 'https://analytics.wenite.de/api/wnt' },
```

Script dann mit `src="/wnt.js"`. Der Tracker sendet automatisch an den Host, von dem das Script kommt.
**Vorher testen:** Bei der Proxy-Variante kann die Herkunft (Land/Stadt) falsch erkannt werden, weil die Anfrage über Vercel weitergereicht wird. Nach dem Umstellen im Dashboard prüfen, ob die Länder plausibel sind. Wenn nicht: bei Variante A bleiben.

**Events:** `window.umami?.track('kontakt_gesendet', { quelle: 'kontaktseite' })` oder per HTML-Attribut `data-umami-event="termin_klick"`.

**Nicht vergessen:** GA4 entfernen, Google-Ads-Tag + Consent Mode behalten, Datenschutzerklärung anpassen.

---

## 7. Entscheidungen und offene Punkte

| Thema | Stand |
|---|---|
| Salt-Rotation | `daily` gesetzt: keine Wiedererkennung über Tage, datenschutzfreundlicher. Wenn tagesübergreifende Besucher-Journeys wichtiger sind: `monthly` (Umami-Standard). |
| Backups | Supabase Free ohne Backups → monatlich `pg_dump` oder später Neon Launch / Supabase Pro |
| Speicher | Supabase Free: 500 MB. Füllstand unter Supabase → Reports prüfen, ab ~400 MB handeln |
| Rechtliches | AV-Verträge Vercel + Supabase, Datenschutzerklärung (Umami statt GA4) |
| Build hier ungetestet | Die Konfiguration wurde gegen den Umami-Code geprüft, ein echter Vercel-Build mit Supabase steht noch aus |
