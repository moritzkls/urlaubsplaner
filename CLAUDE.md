# Urlaubsplaner

Privater Urlaubsplaner für Vanessa & Moritz. Ein-Datei-Web-App mit
Realtime-Sync zwischen Geräten.

---

## Wer plant was

- **Vanessa** ('v', pink `#F0659A`) – 36 Urlaubstage/Jahr
- **Moritz** ('m', blau `#4A90D9`) – 27 Urlaubstage/Jahr
- Bundesland: **Hessen** (Default, im UI änderbar)
- Feiertage werden automatisch berechnet, Wochenenden + Feiertage zählen nicht ins Kontingent
- Beide können zusätzlich **Vorjahresurlaub** und **Gleittage** (Überstundenausgleich) konfigurieren

---

## Stack & Architektur

- **`index.html`** – Eine einzige Datei. Kein Build-Step, kein Bundler.
- Vanilla JavaScript, inline CSS, kein Framework.
- **Firebase Realtime Database** (compat SDK via CDN) für Sync.
- **localStorage** als Offline-Cache und Default-Speicher.
- Hosting: **GitHub Pages** über das Public Repo `moritzkls/urlaubsplaner`.
- Live-URL: <https://moritzkls.github.io/urlaubsplaner/>
- Deployment: `git push` zu `main` reicht – GitHub Pages baut in 1–2 Minuten.

---

## Datenmodell (im Memory + Cloud)

```
marks[dateKey]       = Set('v', 'm', ...)   // Urlaubstage
gleit[dateKey]       = Set('v', 'm', ...)   // Gleittage (gegenseitig exklusiv mit marks)
labels[dateKey]      = "Italien"            // gleiche Beschriftung für alle Tage im Block
carryover[year][p]   = Number               // Vorjahresübertrag
gleitDays[year][p]   = Number               // Gleittage-Kontingent
state                = 'HE'                 // Bundesland
```

`dateKey` = `'YYYY-MM-DD'`. `p` = `'v'` oder `'m'`.

**Firebase-Pfad:** `/{coupleId}/...` mit `coupleId` = 24-Zeichen-Random-String
aus `[a-z2-9]`. Der Couple-Code liegt **nur in localStorage**, nie im Code.
Database Rules verlangen `$coupleId.length > 20` – der Pfad selbst ist das
„Passwort".

---

## Interaktion

### Maus (Desktop)
- **Klick** auf Tag → markieren / abwählen
- **Drag** → mehrere Tage auf einmal
- **Shift+Klick** → Bereich (Anchor zum aktuellen Klick)
- **Rechtsklick** auf markierten Tag → Label-Editor (`prompt()`)
- **Klick auf sichtbares Label** → Label bearbeiten

### Touch (iPhone/iPad)
- **Tap** → toggle (wird erst auf `touchend` ausgeführt, nicht bei `touchstart`)
- **Wischen** → drag (`preventDefault` verhindert Page-Scroll)
- **Long-Press (450 ms)** auf markierten Tag → Label-Editor + Vibration

### Globale Toggles im Header
- Jahr-Navigation (`‹ 2026 ›`)
- View-Toggle: **▦ Raster** (4×3 Monatskarten) vs **▥ Tabelle** (12 Spalten × 31 Zeilen)
- Person-Toggle (Vanessa / Moritz) – beeinflusst was neue Klicks markieren
- Mode-Toggle (☀️ Urlaub / ⏱ Gleittag) – pro Person + Tag exklusiv
- Bundesland-Dropdown
- ☁ Sync-Button + Panel

---

## Visuelle Konventionen

- **Solid Farbe** = Urlaub
- **Diagonale Streifen** = Gleittag (gleiche Farbe wie Urlaub, aber gestreift)
- **Horizontale Teilung** (pink oben, blau unten) = beide haben Urlaub am selben Tag
- **Horizontale Teilung + Streifen** = beide Gleittag
- **Oranges Dreieck unten rechts** = Feiertag
- **Schwarzer Rahmen** = heute
- Aufeinanderfolgende Tage formen einen **zusammenhängenden Block**:
  - Bridge-Element (`.block-bridge`) füllt den 2-px-Gap
  - Border-Radius nur am Block-Anfang / -Ende
  - Über Wochengrenzen / Monatsgrenzen bricht der Block visuell auf
  - Labels werden nur auf `.block-start` angezeigt (oder Single-Day)

---

## Cloud-Sync-Mechanik

1. `initCloud()` läuft am Ende von Init, holt/generiert `coupleId`,
   abonniert `firebase.database().ref(coupleId)` mit `.on('value', ...)`.
2. Jedes lokale `save()` ruft `schedulePush()` (debounced 400 ms).
3. `cloudPush()` macht `cloudRef.update({...})` mit **nur dem aktuellen Jahr** +
   globalen Feldern (bundesland, carryover[year], gleitDays[year], _meta).
   → Andere Jahre werden nie überschrieben → keine Race Conditions zwischen Jahren.
4. Anti-Pingpong: `_meta.writer = myInstanceId`. Beim Empfang prüfen wir das
   und ignorieren eigene Echos.
5. Während eines aktiven Drags (`dragging || touchStartKey`) wird ein
   eingehendes Update in `pendingCloudData` gepuffert und nach Drag-Ende
   via `flushPendingCloudData()` angewendet.
6. Beim Anwenden von Cloud-Daten ist `suppressCloudPush = true`, damit das
   Re-Rendern keinen Push triggert.
7. Status-Indikator (`.sync-status-dot`): grün=verbunden, orange=syncing,
   grau=offline, rot=error.

**Wichtig**: Daten aus der Cloud werden **gemergt** (für `carryover`/`gleitDays`)
oder **nur angewendet wenn vorhanden** (für `years[year]`). So gehen lokale
Daten nicht verloren wenn die Cloud sie noch nicht kennt.

---

## Firebase-Setup (zur Erinnerung)

- Projekt: `urlaubsplanungvamo`
- Region: `europe-west1`
- Auth: keine (Pfad ist das Passwort)
- Rules (im Repo zur Doku):
  ```json
  {
    "rules": {
      "$coupleId": {
        ".read":  "$coupleId.length > 20",
        ".write": "$coupleId.length > 20"
      }
    }
  }
  ```
- Firebase API-Key ist im Client-Code – das ist by design OK
  (siehe Firebase-Doku zu Web-API-Keys). Sicherheit kommt von den Rules.

---

## Persistierungs-Keys in localStorage

```
up_coupleId             - Firebase-Pfad (NICHT teilen mit anderen Couples)
up_year, up_person      - UI-State (lokal, NICHT gesynct)
up_view, up_mode        - UI-State (lokal, NICHT gesynct)
up_state                - Bundesland (gesynct)
up_carryover            - JSON, alle Jahre
up_gleitdays            - JSON, alle Jahre
up_marks_{YEAR}         - JSON pro Jahr
up_gleit_{YEAR}         - JSON pro Jahr
up_labels_{YEAR}        - JSON pro Jahr
```

---

## Wichtige Funktionen (zum schnellen Auffinden)

| Funktion | Aufgabe |
|---|---|
| `getMarkClass(key)` | Liefert CSS-Klasse: `v-only`/`m-only`/`both`/`v-gleit`/`m-gleit`/`both-gleit`/`''` |
| `getBlockClass(key)` | Liefert `block-start`/`-mid`/`-end`/`''` basierend auf Nachbarn |
| `getBlockKeys(key)` | Alle Tage eines Blocks (für Label-Propagation) |
| `applyToKey(key, action)` | Markieren/abwählen, beachtet `markMode` und Mutual-Exclusion |
| `refreshDayEl(key)` | Lightweight DOM-Update für eine Zelle |
| `refreshDayWithNeighbors(key)` | Self + Vorgänger + Nachfolger (für Block-Übergänge) |
| `render()` | Full Re-Render – wechselt zwischen `renderGrid` / `renderLinear` |
| `buildHolidays(y, st)` | Feiertage berechnen (Easter-Algorithm + Bundesland-Tabelle) |
| `isWorkday(key)` | Mo-Fr UND nicht Feiertag |
| `save()` | localStorage + `schedulePush()` |
| `editLabel(key)` | Prompt → schreibt Label für alle Tage im Block |

---

## Offen / Geplant: Mobile UX (Stand 2026-05-24)

Die App ist auf Desktop sehr poliert, auf iPhone aber noch verbesserbar:

- [ ] **Sync-Panel mobil-optimieren** – aktuell ist es 340 px breit und fest oben rechts. Auf kleinen Screens sollte es als Bottom-Sheet oder zentriert mit Backdrop kommen.
- [ ] **Stats-Bar kompakter machen** – auf Mobile stapelt sie vertikal und wird sehr hoch. Idee: collapsible / nur Hauptzahl, Details auf Tap.
- [ ] **Label-Eingabe als Custom-Modal** statt `prompt()` – das native iOS-prompt ist hässlich und cuttet bei langen Beschriftungen.
- [ ] **Carryover/Gleittage-Eingabe** ebenfalls als Modal statt `prompt()`.
- [ ] **Toast „Gespeichert ✓"** ist auf Mobile evtl. überflüssig – jedes Antippen flasht.
- [ ] **Reset-Button** rechts unten kollidiert mit der hohen Stats-Bar.
- [ ] **Year-Nav** evtl. als Swipe-Geste statt Pfeil-Buttons.
- [ ] **Touch-Selektion** prüfen: während Drag scrollt Safari manchmal trotzdem mit; `passive: false` checken.
- [ ] **Sync-Code teilen** über native Share-API (`navigator.share`) statt nur „Kopieren".
- [ ] **Initial-Screen** wenn Vanessa erstmals öffnet: kurz erklären „Code von Moritz eingeben oder eigenen starten".

---

## Conventions

- Deutsch in UI-Texten, Variablen/Code auf Englisch.
- Keine externen CSS/JS-Dependencies außer Firebase SDK von gstatic.com.
- Keine Build-Tools – `index.html` muss alleine lauffähig sein
  (auch als `file://` URL für lokales Testen).
- Commit-Messages auf Deutsch, mit kurzer Bullet-Liste der Änderungen.
- Vor `git push`: kurz im Browser testen (Cmd+R auf der Live-URL).

---

## Lokales Testen

```bash
# Einfach öffnen
open /Users/moritz/Desktop/Urlaub/index.html

# Oder als lokaler Server (falls Firebase Probleme macht mit file://)
cd /Users/moritz/Desktop/Urlaub
python3 -m http.server 8080
# → http://localhost:8080
```

---

## Quick-Reference für neue Sessions

Wenn du (Claude) das hier liest und der User sagt „weiter machen":

1. Falls noch nicht gemacht: **`index.html` einlesen** – die Datei ist
   selbst-dokumentierend mit Kommentaren in den Sections.
2. Nicht alles neu erfinden – die Architektur steht. Punktuell ändern.
3. Bei UI-Änderungen die **Mobile-Media-Query** (`@media (max-width: 820px)`)
   und die **PWA-Standalone-Regel** mitbedenken.
4. Bei Daten-Änderungen: localStorage-Keys nicht brechen (Migration einbauen
   wenn nötig).
5. `git push` ist deployment. Public Repo – keine Secrets in den Code.
