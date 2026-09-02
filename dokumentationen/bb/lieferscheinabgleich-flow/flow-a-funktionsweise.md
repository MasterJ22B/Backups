# Flow A — Mollie-Scan → Kandidaten-CSV

> Stand: 24.08.2026 · 10 Nodes · rein lesend
> Nachgelagert: **Flow B — Bereinigung aus CSV**

---

## Zweck

Flow A ermittelt, welche Mollie-Orders noch offen sind, und schreibt sie in eine
CSV. Er schreibt **nichts** in Zielsysteme — ausschließlich `GET`-Requests.

Die CSV ist die Schnittstelle zu Flow B.

### Warum ein Scan überhaupt nötig ist

Zwei Randbedingungen erzwingen ihn:

**Die Mollie Orders API kennt keine Suche.** `GET /v2/orders` akzeptiert nur
`from`, `limit`, `sort` und `profileId` — kein Filter nach `orderNumber`,
`metadata`, Status oder Datum
([Referenz](https://docs.mollie.com/reference/list-orders)). Eine Order lässt
sich nur über ihre `ord_`-ID direkt abrufen.

**JTL kennt die Mollie-ID nicht.** `tZahlung.cExternalTransactionId` ist für die
Fälle ab 03/2026 leer. Die Verbindung existiert nur in der Gegenrichtung, im
Mollie-Objekt unter `metadata.order_number`.

Ergebnis: Es muss geblättert und lokal gefiltert werden. Der Scan liefert dabei
`ord_`-ID **und** Bestellnummer gemeinsam — deshalb braucht Flow B keine
Zuordnungstabelle.

---

## Ablauf

```
Manuell starten → Config → Scan: Startpunkt
                                  ↓
              ┌──────→ Mollie: Orders-Seite lesen
              │              ↓
        Wait (Scan)   Scan: Seite auswerten
              │              ↓
              └────── Weitere Seite?
                             ↓ [nein]
                    Kandidaten aufbereiten
                             ↓
                        CSV schreiben
```

Eine echte Schleife — im Gegensatz zu Flow B ist sie hier **unvermeidbar**: Die
URL der nächsten Seite steht erst in der Antwort der vorherigen (`_links.next`).

### Abbruchbedingungen

| Bedingung | `abbruchGrund` |
|---|---|
| kein `_links.next` mehr | `letzte Seite erreicht` |
| `seite >= scanMaxSeiten` | `maxSeiten erreicht` → **unvollständig** |
| Seite enthält nur Orders vor `scanAbbruchAelterAls` | `nur noch Orders vor …` |
| Parse- oder HTTP-Fehler | `Fehler`, Zwischenstand bleibt erhalten |

Bei `maxSeiten erreicht` liefert das Ergebnis eine `scanFortsetzenAb`-URL. Die
wird in `scanStartUrl` eingetragen, und der nächste Lauf setzt dort auf.

> `from` ist ein **Cursor auf eine Order-ID**, kein Offset. Deshalb funktioniert
> das Fortsetzen exakt und man kann faktisch in Monatsetappen arbeiten.

---

## Speicheroptimierung

Das war der aufwändigste Teil. Die ersten Versuche haben mehrfach die
Server-Verbindung gerissen. Vier Maßnahmen, nach Wirkung sortiert.

### 1. Antwort als Binärdatei statt JSON — der entscheidende Hebel

Am Node `Mollie: Orders-Seite lesen`:

```json
"options": { "response": { "response": {
    "neverError": true,
    "responseFormat": "file",
    "outputPropertyName": "seite"
}}}
```

**Warum das wirkt:** Eine Seite mit 250 Orders ist rund **1,1 MB**. n8n hält
Ein- und Ausgabe jedes Nodes für die Ausführungsansicht vor — bei 120 Seiten
wären das über 200 MB im Arbeitsspeicher.

Binärdaten legt n8n dagegen im **Dateisystem** ab (`binaryMode: filesystem`).
Der Auswerteknoten liest sie gezielt ein:

```js
const puffer = await this.helpers.getBinaryDataBuffer(0, 'seite');
resp = JSON.parse(puffer.toString('utf8'));
```

Eine Rückfallebene über `$json.data` ist eingebaut, falls doch JSON ankommt.

### 2. Sammler im Workflow-Speicher, nicht im Item

`Scan: Startpunkt` setzt zurück, `Scan: Seite auswerten` schreibt hinein:

```js
const speicher = $getWorkflowStaticData('global');
speicher.scanTreffer = [];   // im Startpunkt
speicher.scanTreffer.push({ … });  // je Treffer
```

**Warum:** Lag die Trefferliste im Item, wurde sie bei **jeder** Iteration
komplett kopiert — und n8n hielt Ein- und Ausgabe jeder Iteration vor. Das ergibt
quadratischen Verbrauch: nach 40 Seiten rund 7,6 MB nur für den Zustand.

Im Workflow-Speicher bleibt das Item konstant klein (`naechsteUrl`, `seite`,
Zähler). Der Reset im Startpunkt ist zwingend — bei aktiven Workflows überlebt
der Speicher die Ausführung, sonst summieren sich Treffer über mehrere Läufe.

### 3. Trefferobjekte ausgedünnt

Von 9 auf 7 Felder, 284 → 174 Byte je Treffer:

```js
{ mollieId, bestellnummer, status, betrag, erfasst, expiresAt, versendbar }
```

Weggelassen: `method`, `authorizedAt`, `waehrung`, `createdAt`, `isCancelable`,
`email`, `orderNumber`. Bei ~1.000 Treffern und vielen Iterationen zählt jedes
Feld.

### 4. Ausführungsdaten nicht persistieren

```json
"settings": {
  "saveExecutionProgress":    false,
  "saveDataSuccessExecution": "none",
  "saveDataErrorExecution":   "all",
  "saveManualExecutions":     false,
  "binaryMode":               "filesystem"
}
```

Erfolgreiche Läufe werden nicht aufbewahrt, Fehler bleiben nachvollziehbar.

### Wirkung

| Seiten | ursprünglich | nur Speicher-Fix | **+ Datei-Variante** |
|---:|---:|---:|---:|
| 30 | 57 MB | 53 MB | **0,1 MB** |
| 50 | 100 MB | 88 MB | **0,2 MB** |
| 200 | 537 MB | 352 MB | **0,6 MB** |

### Was nicht optimierbar ist

**Der manuelle Start.** n8n streamt bei manueller Ausführung jede Iteration über
WebSocket an den Browser — unabhängig von allen Einstellungen. Das war die
Ursache der ersten Abstürze.

Für große Läufe: Manual Trigger gegen **Schedule Trigger** tauschen, Workflow
aktivieren, Ergebnis per Mail. Dann bleibt alles serverseitig.

Zum Testen mit `scanMaxSeiten: 3` ist manuell unproblematisch.

---

## Config-Parameter

| Parameter | Wert | Zweck |
|---|---|---|
| `scanProSeite` | `250` | Maximum der Orders API |
| `scanMaxSeiten` | `30` | Deckel gegen Endlosschleife |
| `scanSortierung` | `desc` | neueste zuerst → Datums-Abbruch greift |
| `scanAbbruchAelterAls` | `'2026-03-01'` | Stichtag |
| `scanStartUrl` | `''` | Fortsetzung einer Etappe |
| `scanStatus` | `['authorized','shipping']` | die offenen Zustände |
| `nurBestellnummern` | `[]` | Whitelist |
| `testLimit` | `0` | Begrenzung der Ausgabe |

**Wichtig zum Stichtag:** Er muss **vor** der neuesten Order liegen. Die Orders
API enthält nur Altbestand, die jüngste Order ist vom **16.06.2026**. Ein
Stichtag danach — etwa `2026-08-01` — lässt den Scan sofort abbrechen, weil
bereits Seite 1 komplett „zu alt" ist.

---

## Ausgabe

### CSV — Schnittstelle zu Flow B

```
bestellnummer;mollieId;status;betrag;erfasst;expiresAt;restfristTage;versendbar
```

Semikolon, Dezimalkomma, BOM für Excel. **Das Format ist die Schnittstelle** —
Änderungen hier müssen in Flow B nachgezogen werden. Zwingend sind dort nur
`bestellnummer` und `mollieId`.

Die Kandidaten sind nach `expiresAt` aufsteigend sortiert: **fällige zuerst.**

### Kennzahlen im JSON

| Feld | Bedeutung |
|---|---|
| `zeilen` | Anzahl Kandidaten |
| `summeEUR` | offener Gesamtbetrag |
| `scanSeiten` / `scanGesehen` | Umfang des Scans |
| `scanAbbruch` | Abbruchgrund |
| `scanFortsetzenAb` | URL für die nächste Etappe, sonst `null` |
| `kuerzesteFrist` | dringendster Fall in Tagen |
| `bereitsAbgelaufen` | Anzahl verfallener Autorisierungen |

---

## Ergebnis des Referenzlaufs (24.08.2026)

| Kennzahl | Wert |
|---|---|
| Kandidaten | **661** |
| Summe | **27.629,15 €** |
| Status | 660 × `authorized`, 1 × `shipping` |
| Zeitraum | Autorisierung 27.03. – 15.06.2026 |
| Kürzeste Restfrist | 30 Tage |
| Bereits abgelaufen | 0 |

**Gegenprüfung:** Ein Dashboard-Export mit 990 Zeilen enthielt unsere 661 als
exakte Teilmenge. Die Differenz von 329 waren `cancelled` (317) und `paidout`
(12) — bei beiden gibt es nichts einzuziehen. Deren Zeitraum reicht bis 2021
zurück.

---

## Betriebshinweise

**Credential:** Bearer Auth mit dem Mollie-Live-Key. Das Credential aus dem
LEAF-Abo-Flow ist nicht verwendbar — dort steht ein Node-Ausdruck im Token, der
außerhalb jenes Flows nicht auflösbar ist.

**Mollie antwortet mit `application/hal+json`.** n8n parst das nicht
automatisch — daher das eigene Parsen im Auswerteknoten.

**Etappenbetrieb** ist der empfohlene Weg: `scanMaxSeiten` klein halten,
`scanFortsetzenAb` übernehmen. Jede Etappe erzeugt eine eigene CSV, die
einzeln geprüft und in Flow B eingespielt werden kann.

---

## Offene Punkte

| # | Punkt |
|---|---|
| 1 | Manual Trigger gegen Schedule tauschen für unbeaufsichtigte Vollläufe |
| 2 | Mail-Node mit CSV-Anhang — dann steht `scanFortsetzenAb` direkt in der Mail |
| 3 | Parameter `profileId` prüfen: bei mehreren Mollie-Profilen ließe sich die Blättermenge eingrenzen |
| 4 | Die Orders API ist von Mollie als `deprecated` markiert. Für den Altbestand irrelevant — er besteht aus Order-Objekten |
