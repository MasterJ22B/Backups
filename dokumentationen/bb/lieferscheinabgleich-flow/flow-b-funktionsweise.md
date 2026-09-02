# Flow B — Bereinigung aus CSV (Shopify + Mollie)

> Stand: 24.08.2026 · 43 Nodes · Status: getestet, im Etappenbetrieb
> Vorgelagert: **Flow A — Mollie-Scan** (erzeugt die Kandidaten-CSV)

---

## Zweck

Flow B arbeitet den Rückstand ab: Altbestellungen, die in JTL-Wawi versendet
wurden, ohne dass Shopify oder Mollie davon erfahren haben.

Er wird **manuell gestartet** und verarbeitet eine hochgeladene CSV.
Kein Zeitfenster, kein Trigger — die Bestellungen kommen aus der Liste.

| | |
|---|---|
| **Eingang** | CSV aus Flow A: `bestellnummer;mollieId;…` |
| **Prüfung** | JTL: existiert ein versendeter Lieferschein? |
| **Ausgang 1** | Shopify: Fulfillment je Lieferschein |
| **Ausgang 2** | Mollie: Shipment je Lieferschein → Geldeingang |

---

## Grundprinzip

**1 Lieferschein = 1 physische Sendung = 1 Meldung je Zielsystem.**

Der Flow schließt niemals eine Bestellung ab. Er meldet Sendungen, und beide
Zielsysteme rechnen die Vollständigkeit selbst aus:

- Shopify zieht von `remainingQuantity` ab → `PARTIALLY_FULFILLED` → `FULFILLED`
- Mollie zieht von `shippableQuantity` ab → `shipping` → `completed`

Daraus folgt die **Idempotenz**: Was gemeldet ist, steht im Zielsystem auf 0.
Dieselbe CSV kann beliebig oft hochgeladen werden — bereits erledigte
Bestellungen laufen als Skip durch. Es gibt keine Ausschlussliste und keinen
Verbrauchsvermerk; die Zielsysteme sind das Gedächtnis.

---

## Ablauf in vier Phasen

```
CSV hochladen → CSV Auslesen → Config → Kandidaten aufbereiten
                                              ↓
                                    Kandidaten vorhanden?
                                              ↓
PHASE 1  JTL-Batches bilden → Loop JTL ─┐
              ┌─────────────────────────┘
              │  JTL-Query bauen → SQL → Zeilen filtern → Wait
              ↓ [done]
         Lieferscheine gruppieren
              ↓
PHASE 2  Batches bilden → Loop Over Batches ─┐
              ┌──────────────────────────────┘
              │  Shopify Token holen → Lookup → Zuordnung
              │  → fulfillmentCreate → Wait
              ↓ [done]
         Shopify-Debug-Tabelle
              ↓
PHASE 3  Mollie-Kandidaten bauen → URL bauen → Order lesen
              → Zuordnung + Plan → senden?
                    [true]  → Body → Shipment → Ergebnis prüfen ─┐
                    [false] ──────────────────────────────────────┴→
              ↓
         Mollie-Debug-Tabelle
              ↓
PHASE 4  Auswertung bauen → Report-Mail
         CSV bauen (parallel)
```

---

## Phase 0 — CSV einlesen

**`CSV hochladen`** (Form Trigger) nimmt die Datei über ein Webformular an.

**`CSV Auslesen`** liest sie als Text. Der Node sucht die erste vorhandene
Binärproperty, statt einen Namen zu erwarten — das Formular vergibt ihn je nach
Feldbezeichnung unterschiedlich.

> **Reihenfolge ist zwingend:** `CSV Auslesen` muss **direkt** hinter dem
> Trigger liegen. Code-Nodes geben ein neues Item zurück und verwerfen dabei
> Binärdaten. Deshalb steht `Config` erst danach.

**`Kandidaten aufbereiten`** parst die CSV selbst (RFC 4180, Anführungszeichen,
BOM, `;` oder `,`) und verwirft dabei:

| Grund | Warum |
|---|---|
| Bestellnummer nicht rein numerisch | geht später in eine SQL-IN-Liste |
| ID ohne `ord_`-Präfix | keine Orders-API-Referenz |
| doppelte Bestellnummer | Dublette |

Danach: Whitelist (`nurBestellnummern`), Sortierung nach **`restfristTage`
aufsteigend** — fällige zuerst — und Begrenzung über `testLimit`.

---

## Phase 1 — JTL-Prüfung (Sicherheitsschritt)

Die Bestellnummern werden zu Batches gebündelt (`jtlBatchGroesse`, Standard 200)
und je Batch eine SQL-Abfrage gebaut.

**Ohne Zeitfenster.** Stattdessen `WHERE b.cBestellNr IN (…)`. Dieselbe
CTE-Logik wie im fortlaufenden Flow:

| Filter | Bedeutung |
|---|---|
| `tVersand.dVersendet IS NOT NULL` | **Versandkriterium** |
| `kShop = 3`, `nStorno = 0`, `nPlatform = 2` | Abgrenzung Altbestand |
| `nType = 1` | nur echte Artikelpositionen |
| Versandart 108 ausgeschlossen | „Kein Versand" |

> **Ohne versendeten Lieferschein passiert nichts.** Das ist der eine Fehler,
> der teuer wird: ein Shipment ohne Lieferung zieht Geld für nicht gelieferte
> Ware ein. Im fortlaufenden Flow liegt der Lieferschein per Definition vor —
> hier muss er geprüft werden.

**`Lieferscheine gruppieren`** faltet die Positionszeilen zu einem Item je
Lieferschein zusammen und hängt die Mollie-Daten aus der CSV an. Sortierung
chronologisch — entscheidend für die Restmengen-Reihenfolge.

---

## Phase 2 — Shopify (Path A)

Bestellungen werden zu 25er-Batches gebündelt; je Batch ein GraphQL-Lookup.

**Der Token wird pro Batch neu geholt.** Bei mehreren hundert Bestellungen
läuft ein einmalig geholter Token ab.

Der Lookup holt drei Dinge: offene FulfillmentOrder-Positionen mit `sku` und
`remainingQuantity`, die bereits hinterlegten Sendungsnummern, und den
Fulfillment-Status.

### Matching — umschaltbar

`config.skuMatching` steuert das Verfahren:

| Wert | Verfahren |
|---|---|
| `false` (aktiv) | Titel exakt → Titel normalisiert |
| `true` | ausschließlich `sku` |

Die migrierten Bestellpositionen tragen **keine SKU** — der Matrixify-Import
hat sie ohne Variantenbezug angelegt. Deshalb läuft der Abgleich über
`cArtikelname` ⇄ `productTitle`. Trefferquote im Test: 98,5 %.

Stufe 2 (Diakritika weg, Sonderzeichen zu Leerzeichen) hat einen
**Kollisionsschutz**: fallen zwei Positionen derselben Bestellung auf denselben
Schlüssel, wird er gesperrt statt geraten.

### Idempotenz

Die Sendungsnummer `cIdentCode` wird als `trackingInfo.number` mitgeschrieben.
Vor jedem Schreiben werden die vorhandenen Fulfillments gelesen — ist die Nummer
dabei → `skipped_already_shipped`.

> `trackingSenden: false` würde die Idempotenz aushebeln. Die beiden Schalter
> hängen zusammen.

### Versanddienstleister

`cAmazonCarrierCode` liefert pauschal `DHL`; Shopify macht daraus „DHL Express"
und verlinkt falsch. Erkennung deshalb am Format der Sendungsnummer:

| Muster | Company |
|---|---|
| `^\d{20}$` | DHL Paket |
| `^[A-Za-z]{2}\d{9}[A-Za-z]{2}$` | Deutsche Post |

`url` wird explizit mitgegeben, damit der Link nicht von Shopifys
Interpretation abhängt.

### Ignorierte Positionen

JTL bucht Versandzuschläge teils als echten Artikel (`nType = 1`), z. B.
`VS-02-NST`. In Shopify ist Versand keine Position — der Abgleich kann nie
treffen. `ignorierteArtikelNr: ['VS-']` überspringt sie und unterdrückt die
Teilzuordnungs-Warnung.

---

## Phase 3 — Mollie (Path B)

**Ohne Schleife.** n8n-Nodes verarbeiten alle Items ihres Eingangs von selbst:
der HTTP-Node macht bei 200 Items 200 Calls, die Code-Nodes laufen im
Per-Item-Modus. Eine Schleife wäre nur für die Wartezeit zwischen den Calls
nötig gewesen — und war fehleranfällig.

> **Historie:** Die frühere Schleifenkonstruktion gab am `done`-Ausgang auch die
> unbearbeiteten Eingangs-Items zurück und verarbeitete pro Lauf nur ein Item.
> Symptome: Dubletten im Report, Einträge mit Status `id_gefunden`.
> Seit dem Umbau tritt das nicht mehr auf.

### Ablauf

1. **`Mollie-Kandidaten bauen`** — filtert auf Shopify-Ausgänge aus
   `mollieShopifyOk`. Bewusst inklusive der Skips: eine Bestellung kann in
   Shopify längst gemeldet, bei Mollie aber offen sein.
2. **`Mollie: URL bauen`** → `GET /v2/orders/{ord_id}`
3. **`Mollie: Order lesen`** — ein Call je Bestellung, keine Paginierung
4. **`Mollie-Zuordnung + Plan`** — Abgleich über `sku`, gedeckelt auf
   `shippableQuantity`
5. **`POST /v2/orders/{id}/shipments`** mit `lines[{id, quantity}]` und
   `tracking{carrier, code, url}`

### Matching über SKU

Mollie-Orderlines tragen eine `sku` im Format von `cArtNr` (z. B.
`M-TOG-P-GP`). **Path B braucht kein Titel-Matching** — anders als Path A.

### Statusprüfung vor dem Schreiben

| Mollie-Status | Ergebnis |
|---|---|
| `authorized` | offen → melden |
| `shipping` | teilweise erfasst → Rest melden |
| `paid` / `completed` | `skipped_bereits_erfasst` |
| `expired` | `skipped_verfallen` — nicht mehr einziehbar |
| `canceled` | `skipped_storniert` |

### Versandkosten

Mollie führt Versand als **eigene Orderline**. Wird sie nie gemeldet, bleibt der
Versandbetrag dauerhaft unerfasst. Steuerung über `mollieVersandkosten`:

| Wert | Verhalten |
|---|---|
| `bei_letzter_lieferung` (Standard) | sobald keine Produktzeile mehr offen ist |
| `immer` | bei jeder Meldung |
| `nie` | nie |

---

## Phase 4 — Auswertung

**`Mollie-Debug-Tabelle`** erzeugt eine flache Zeile je Lieferschein und
übersetzt Zwischenzustände (`kandidat`, `id_gefunden`, `lesen`) in
`nicht_verarbeitet` — sie dürfen im Endergebnis nicht auftauchen.

**`Auswertung bauen`** verdichtet und versendet den Report per Mail.
**`CSV bauen`** läuft parallel und liefert die Ergebnisdatei.

### Wichtige Spalten der Ergebnis-CSV

| Spalte | Bedeutung |
|---|---|
| `shopifyEntscheidung` | Ergebnis Phase 2 |
| `entscheidung` | Ergebnis Phase 3 |
| `shipmentId` | Mollie-Shipment, falls geschrieben |
| `mengeZugeordnet` | tatsächlich gemeldete Menge |
| `versandGemeldet` | Versandzeile mitgemeldet |
| `restfristTage` | Verfallsfrist |

Filter auf `entscheidung = done` ergibt die tatsächlich erfassten Vorgänge.

---

## Config-Parameter

| Parameter | Empfehlung | Zweck |
|---|---|---|
| `nurBestellnummern` | `[]` | Whitelist für Einzeltests |
| `testLimit` | `0` = alle | Etappengröße |
| `dryRun` | `false` | Shopify: `true` = kein Schreib-Call |
| `mollieDryRun` | `false` | Mollie: `true` = nur planen |
| `skuMatching` | `false` | Titel- vs. SKU-Abgleich |
| `notifyCustomer` | `false` | keine Kundenmails für Altbestellungen |
| `trackingSenden` | `true` | trägt die Shopify-Idempotenz |
| `jtlBatchGroesse` | `200` | Bestellnummern je SQL-Abfrage |
| `batchGroesse` | `25` | Bestellungen je GraphQL-Lookup |
| `mollieVersandkosten` | `bei_letzter_lieferung` | Versandzeile |
| `ignorierteArtikelNr` | `['VS-']` | Positionen ohne Shopify-Entsprechung |

---

## Betriebshinweise

**Etappen statt Vollstart.** Bei echtem Geldeinzug empfiehlt sich ein
stufenweises Vorgehen über `testLimit`: 1 → 10 → 100 → Rest. Dieselbe CSV bleibt
wiederverwendbar, Erledigtes läuft als Skip durch.

**Zwei Credentials erforderlich:**

| Node | Credential |
|---|---|
| `JTL: Versendete Lieferscheine` | MSSQL, `eazybusiness` |
| `Mollie: Order lesen` / `Shipment anlegen` | Bearer Auth, Mollie-Live-Key |
| `Report-Mail` | SMTP |

> Das Bearer-Credential aus dem LEAF-Abo-Flow ist **nicht** verwendbar — dort
> steht ein Node-Ausdruck im Token, der außerhalb jenes Flows nicht auflösbar
> ist. Ein eigenes Credential anlegen.

**Code-Node-Modi.** Sechs Nodes laufen zwingend im Modus
**„Run Once for Each Item"**. Im All-Items-Modus verarbeitet n8n nur das erste
Item eines Batches — **ohne Fehlermeldung**. Betroffen:
`⚠️ Query bauen (fulfillmentCreate)`, `⚠️ Fulfillment-Ergebnis pruefen`,
`Mollie: URL bauen`, `Mollie-Zuordnung + Plan`, `⚠️ Mollie: Body bauen`,
`⚠️ Mollie-Ergebnis pruefen`.

In diesem Modus muss die Rückgabe ein **Objekt** sein (`return { json: … }`),
kein Array — sonst: *„A 'json' property isn't an object"*.

**Mollie antwortet mit `application/hal+json`.** n8n parst das nicht
automatisch, sondern legt den Body als String in `$json.data` ab. Alle
Mollie-Auswerteknoten parsen selbst und haben eine Rückfallebene.

**Rate-Limit.** Seit dem Entfernen der Mollie-Schleife gibt es keine Wartezeit
zwischen den Calls. Der HTTP-Node hat `retryOnFail` mit drei Versuchen. Bei
großen Etappen beobachten.

---

## Was nicht rückholbar ist

| Vorgang | Rückweg |
|---|---|
| Shopify-Fulfillment | `fulfillmentCancel` mit der Fulfillment-ID |
| **Mollie-Shipment** | **keiner** — nur Rückerstattung als separater Vorgang |

Deshalb: Mollie erst scharf schalten, wenn der Dry-Run geprüft ist. Das Feld
`_lines` in der Debug-Tabelle zeigt den geplanten Request-Body.

---

## Offene Punkte

| # | Punkt |
|---|---|
| 1 | Status `skipped_no_match` ist irreführend, wenn die Ursache `kein_rest_versendbar` lautet — also bereits erfasst |
| 2 | Wie viele der 661 Kandidaten haben überhaupt einen versendeten Lieferschein? Prüfabfrage vorhanden |
| 3 | Mollie-Rate-Limit bei Etappen über 100 beobachten |
| 4 | `Loop Over Batches` im Shopify-Zweig hat dieselbe Konstruktion wie die entfernte Mollie-Schleife — bislang unauffällig, aber prüfenswert |
