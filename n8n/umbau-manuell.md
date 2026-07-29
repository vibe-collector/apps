# Umbau `sendemaifromexecute` — manuell in der n8n-Oberfläche

Für den Fall, dass die schreibenden MCP-Aufrufe gesperrt bleiben.
**Nach Wichtigkeit sortiert.** Schritt 1 dauert zwei Minuten und behebt den
gefährlichsten Fehler. Der Rest kann warten.

Node-Einstellungen findest du jeweils, indem du den Node per Doppelklick
öffnest und oben auf den Reiter **Settings** wechselst.

---

## Schritt 1 — Die Lüge abstellen (2 Minuten, höchste Priorität)

Betrifft **`Send email info`** und **`Send contact`**.

Diese beiden melden einen SMTP-Fehler als Erfolg, woraufhin Notion die Zeile
auf „Executed" setzt, obwohl nie eine Mail rausging.

Pro Node im Reiter **Settings**:

| Einstellung | von | auf |
|---|---|---|
| Always Output Data | an | **aus** |
| On Error | Continue (using regular output) | **Continue (using error output)** |

Danach schlägt ein Fehlschlag sichtbar fehl, statt sich als Erfolg zu tarnen.
Der Fehlerausgang hängt zunächst in der Luft — das ist in Ordnung, die Zeile
bleibt dann einfach offen statt fälschlich abgehakt. Schritt 4 hängt ihn an.

---

## Schritt 2 — Retry und Fehlerausgang für alle Versand-Nodes

Betrifft alle sieben: `Send email info`, `Send contact`, `Send support GAD`,
`Send shop GAD`, `Send office schwack`, `Send coach photocoach`,
`Send a message`.

Pro Node im Reiter **Settings**:

| Einstellung | Wert |
|---|---|
| Retry On Fail | **an** |
| Max. Tries | **3** |
| Wait Between Tries (ms) | **5000** |
| On Error | **Continue (using error output)** |
| Always Output Data | **aus** |

---

## Schritt 3 — Gmail-Empfänger korrigieren

Node **`Send a message`**, Reiter Parameters:

| Feld | von | auf |
|---|---|---|
| To | `{{ $json.to }}` | `{{ $json.toemail }}` |

`to` enthält den Anzeigenamen („Harald"), nicht die Adresse. Diese Route konnte
bisher nie zustellen.

---

## Schritt 4 — Fehlerpfad anlegen

### 4a) Notion-Node „Execute zuruecksetzen"

Neuer Node, Typ **Notion**:

| Feld | Wert |
|---|---|
| Credential | dasselbe wie in `Update a database page` |
| Resource | Database Page |
| Operation | Update |
| Page ID | Modus **By ID**, Wert `{{ $json.pageId }}` |
| Properties | eine Eigenschaft hinzufügen: **Execute** (Checkbox), Haken **leer lassen** |

Wichtig: **kein** `Executed`-Datum setzen. Genau das unterscheidet den
Fehlerpfad vom Erfolgspfad — die Zeile bleibt offen und sichtbar.

### 4b) Telegram-Node „Alert Fehler"

Neuer Node, Typ **Telegram**:

| Feld | Wert |
|---|---|
| Credential | Telegram account |
| Chat ID | `622619977` |
| Text | siehe unten |
| Additional Fields → Parse Mode | HTML |
| Additional Fields → Append Attribution | aus |

```
<b>⚠️ Versand fehlgeschlagen</b>

Absender: {{ $('Build Signature').item.json.emailfrom || '(leer)' }}
Empfänger: {{ $('Build Signature').item.json.toemail || '(leer)' }}
Betreff: {{ $('Build Signature').item.json.subject || '(leer)' }}
Typ: {{ $('Build Signature').item.json.type || '(leer)' }}

Fehler: {{ $('Build Signature').item.json.error?.message || $('Build Signature').item.json.error || 'kein Routing-Treffer' }}

Haken wurde entfernt, die Zeile bleibt offen.
```

### 4c) Verkabeln

* Von **jedem** der sieben Versand-Nodes den **roten Fehlerausgang** (der
  zweite, untere Ausgang) auf `Execute zuruecksetzen` ziehen.
* `Execute zuruecksetzen` → `Alert Fehler`.

---

## Schritt 5 — Fallback-Ausgänge

Verhindert, dass Zeilen lautlos verschwinden und dann alle 10 Minuten
wiederkehren.

**Node `Switch`** (der nach Type) → Options → **Add Option** → **Fallback
Output** → auf **Extra Output** setzen.
Grund: `Briefing` existiert in Notion als Auswahl, hat hier aber keine Route.

**Node `Route by Emailfrom`** → dasselbe.
Grund: leeres oder unbekanntes `Emailfrom`.

Beide neuen Extra-Ausgänge ebenfalls auf `Execute zuruecksetzen` ziehen.

---

## Schritt 6 — Signaturen

### 6a) Code-Node „Build Signature"

Neuer Node, Typ **Code**, Mode **Run Once for All Items**.
Einfügen zwischen `Edit Fields` und `Switch`:

1. Verbindung `Edit Fields` → `Switch` löschen
2. `Edit Fields` → `Build Signature` verbinden
3. `Build Signature` → `Switch` verbinden

Code:

```javascript
// Einheitliche Signatur. Konstant: Name, Telefon, Anschrift.
// Variabel: Domain (prominent) und Absenderadresse.
const NAME  = 'Harald Schwack';
const PHONE = '+43 699 1699 2411';
const ADDR  = 'Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien';
const TEAL  = '#00b09a';

// Logo je Domain. Leerer String = rein textliche Signatur.
// Sobald eine PNG-URL vorliegt: hier eintragen, sonst nichts aendern.
const LOGOS = {
  'haraldschwack.at':   '',
  'schwack.com':        '',
  'photocoach.cc':      '',
  'gettingadddone.com': ''
};

function signature(from) {
  const addr = String(from || '').trim();
  if (!addr) return '';

  let domain = (addr.split('@')[1] || '').toLowerCase();
  if (!domain) return '';
  // Private Gmail-Adresse tritt unter der eigenen Domain auf
  if (domain === 'gmail.com') domain = 'schwack.com';

  const logo = LOGOS[domain] || '';
  const logoCell = logo
    ? '<td style="padding-right:14px;vertical-align:top"><img src="' + logo + '" alt="' + domain + '" width="110" style="display:block;border:0"></td>'
    : '';

  return '<table cellpadding="0" cellspacing="0" border="0" style="margin-top:20px;font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:1.55;color:#3C3C3B">'
    + '<tr>' + logoCell
    + '<td style="border-left:3px solid ' + TEAL + ';padding:2px 0 2px 12px;vertical-align:top">'
    + '<div style="font-size:15px;font-weight:bold;color:' + TEAL + '">' + NAME + '</div>'
    + '<div style="font-size:14px;font-weight:bold;padding-bottom:7px">'
    +   '<a href="https://www.' + domain + '" style="color:' + TEAL + ';text-decoration:none">' + domain + '</a>'
    + '</div>'
    + '<div><strong>T:</strong> ' + PHONE + '<br>'
    +   '<strong>E:</strong> <a href="mailto:' + addr + '" style="color:#3C3C3B;text-decoration:none">' + addr + '</a><br>'
    +   ADDR
    + '</div>'
    + '</td></tr></table>';
}

const src = $('Get many database pages').all();

return $input.all().map((item, i) => {
  const j = item.json;
  const sig = signature(j.emailfrom);
  return { json: Object.assign({}, j, {
    pageId:    src[i] ? src[i].json.id : null,
    signature: sig,
    htmlBody:  (j.body || '') + sig
  })};
});
```

### 6b) Versand-Nodes auf `htmlBody` umstellen

| Node | Feld | neuer Wert |
|---|---|---|
| Send email info | HTML | `{{ $json.htmlBody }}` |
| Send contact | HTML | `{{ $json.htmlBody }}` |
| Send support GAD | HTML | `{{ $json.htmlBody }}` |
| Send shop GAD | HTML | `{{ $json.htmlBody }}` |
| Send office schwack | HTML | `{{ $json.htmlBody }}` |
| Send coach photocoach | HTML | `{{ $json.htmlBody }}` |
| Send a message | Message | `{{ $json.htmlBody }}` |

Die bisherigen Anhängsel `<br><br>` und `{{ $('Edit Fields').item.json.Signatur }}`
dabei entfernen — der Abstand steckt jetzt in der Signatur selbst.

### 6c) Aufräumen

Im Node `Edit Fields` das Feld **Signatur** löschen. Es wird nicht mehr gelesen.

---

## Später, bewusst ausgeklammert

* **Meta-DM** (`HTTP: Send Meta DM`): `bodyParameters` ist leer — weder Empfänger
  noch Nachricht werden übertragen. `Type = Send DM` ist funktionslos.
* **Draft/Active-Divergenz**: aktive Version hat `limit: 10`, der gespeicherte
  Entwurf `limit: 1`. Ein Publish würde die Batchgröße unbemerkt ändern.
* **Zwei falsch als „Executed" markierte Zeilen**: „Testantwort an Harald" und
  „Testantwort an Harald (1)".
