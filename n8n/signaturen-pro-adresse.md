# Signaturen — eine je Absenderadresse

Sieben eigenständige Signaturen als Felder im vorhandenen Node **`Edit Fields`**.
Jede ist unabhängig bearbeitbar: eigene Farbe, eigener Text, eigenes Logo,
ohne dass die anderen davon berührt werden.

Kein neuer Node, keine Verbindungen ändern.

---

## Schritt A — Sieben Felder in `Edit Fields` anlegen

Node **`Edit Fields`** öffnen. Das bestehende Feld **`Signatur`** löschen
(wird nicht mehr gebraucht) und stattdessen die sieben Felder unten anlegen.

Jeweils **Type: String**, Modus **Fixed** (nicht Expression) — es ist reines HTML.

### `SigInfo` → info@haraldschwack.at

```html
<table cellpadding="0" cellspacing="0" border="0" style="margin-top:20px;font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:1.55;color:#3C3C3B"><tr><td style="border-left:3px solid #AFA362;padding:2px 0 2px 12px;vertical-align:top"><div style="font-size:15px;font-weight:bold;color:#AFA362">Harald Schwack</div><div style="font-size:14px;font-weight:bold;padding-bottom:7px"><a href="https://www.haraldschwack.at" style="color:#AFA362;text-decoration:none">haraldschwack.at</a></div><div><strong>T:</strong> +43 699 1699 2411<br><strong>E:</strong> <a href="mailto:info@haraldschwack.at" style="color:#3C3C3B;text-decoration:none">info@haraldschwack.at</a><br>Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien</div></td></tr></table>
```

### `SigContact` → contact@schwack.com

```html
<table cellpadding="0" cellspacing="0" border="0" style="margin-top:20px;font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:1.55;color:#3C3C3B"><tr><td style="border-left:3px solid #00b09a;padding:2px 0 2px 12px;vertical-align:top"><div style="font-size:15px;font-weight:bold;color:#00b09a">Harald Schwack</div><div style="font-size:14px;font-weight:bold;padding-bottom:7px"><a href="https://www.schwack.com" style="color:#00b09a;text-decoration:none">schwack.com</a></div><div><strong>T:</strong> +43 699 1699 2411<br><strong>E:</strong> <a href="mailto:contact@schwack.com" style="color:#3C3C3B;text-decoration:none">contact@schwack.com</a><br>Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien</div></td></tr></table>
```

### `SigSupport` → support@gettingadddone.com

```html
<table cellpadding="0" cellspacing="0" border="0" style="margin-top:20px;font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:1.55;color:#3C3C3B"><tr><td style="border-left:3px solid #00b09a;padding:2px 0 2px 12px;vertical-align:top"><div style="font-size:15px;font-weight:bold;color:#00b09a">Harald Schwack</div><div style="font-size:14px;font-weight:bold;padding-bottom:7px"><a href="https://www.gettingadddone.com" style="color:#00b09a;text-decoration:none">gettingadddone.com</a></div><div><strong>T:</strong> +43 699 1699 2411<br><strong>E:</strong> <a href="mailto:support@gettingadddone.com" style="color:#3C3C3B;text-decoration:none">support@gettingadddone.com</a><br>Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien</div></td></tr></table>
```

### `SigShop` → shop@gettingadddone.com

```html
<table cellpadding="0" cellspacing="0" border="0" style="margin-top:20px;font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:1.55;color:#3C3C3B"><tr><td style="border-left:3px solid #00b09a;padding:2px 0 2px 12px;vertical-align:top"><div style="font-size:15px;font-weight:bold;color:#00b09a">Harald Schwack</div><div style="font-size:14px;font-weight:bold;padding-bottom:7px"><a href="https://www.gettingadddone.com" style="color:#00b09a;text-decoration:none">gettingadddone.com</a></div><div><strong>T:</strong> +43 699 1699 2411<br><strong>E:</strong> <a href="mailto:shop@gettingadddone.com" style="color:#3C3C3B;text-decoration:none">shop@gettingadddone.com</a><br>Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien</div></td></tr></table>
```

### `SigOffice` → office@schwack.com

```html
<table cellpadding="0" cellspacing="0" border="0" style="margin-top:20px;font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:1.55;color:#3C3C3B"><tr><td style="border-left:3px solid #00b09a;padding:2px 0 2px 12px;vertical-align:top"><div style="font-size:15px;font-weight:bold;color:#00b09a">Harald Schwack</div><div style="font-size:14px;font-weight:bold;padding-bottom:7px"><a href="https://www.schwack.com" style="color:#00b09a;text-decoration:none">schwack.com</a></div><div><strong>T:</strong> +43 699 1699 2411<br><strong>E:</strong> <a href="mailto:office@schwack.com" style="color:#3C3C3B;text-decoration:none">office@schwack.com</a><br>Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien</div></td></tr></table>
```

### `SigCoach` → coach@photocoach.cc

```html
<table cellpadding="0" cellspacing="0" border="0" style="margin-top:20px;font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:1.55;color:#3C3C3B"><tr><td style="border-left:3px solid #00b09a;padding:2px 0 2px 12px;vertical-align:top"><div style="font-size:15px;font-weight:bold;color:#00b09a">Harald Schwack</div><div style="font-size:14px;font-weight:bold;padding-bottom:7px"><a href="https://www.photocoach.cc" style="color:#00b09a;text-decoration:none">photocoach.cc</a></div><div><strong>T:</strong> +43 699 1699 2411<br><strong>E:</strong> <a href="mailto:coach@photocoach.cc" style="color:#3C3C3B;text-decoration:none">coach@photocoach.cc</a><br>Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien</div></td></tr></table>
```

### `SigGmail` → harald.schwack@gmail.com

```html
<table cellpadding="0" cellspacing="0" border="0" style="margin-top:20px;font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:1.55;color:#3C3C3B"><tr><td style="border-left:3px solid #00b09a;padding:2px 0 2px 12px;vertical-align:top"><div style="font-size:15px;font-weight:bold;color:#00b09a">Harald Schwack</div><div style="font-size:14px;font-weight:bold;padding-bottom:7px"><a href="https://www.schwack.com" style="color:#00b09a;text-decoration:none">schwack.com</a></div><div><strong>T:</strong> +43 699 1699 2411<br><strong>E:</strong> <a href="mailto:harald.schwack@gmail.com" style="color:#3C3C3B;text-decoration:none">harald.schwack@gmail.com</a><br>Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien</div></td></tr></table>
```

---

## Schritt B — Jeden Versand-Node auf sein Feld zeigen lassen

Feld **HTML** (beim Gmail-Node: **Message**), Modus **Expression**:

| Node | Absender | Wert |
|---|---|---|
| Send email info | info@haraldschwack.at | `{{ $json.body }}{{ $('Edit Fields').item.json.SigInfo }}` |
| Send contact | contact@schwack.com | `{{ $json.body }}{{ $('Edit Fields').item.json.SigContact }}` |
| Send support GAD | support@gettingadddone.com | `{{ $json.body }}{{ $('Edit Fields').item.json.SigSupport }}` |
| Send shop GAD | shop@gettingadddone.com | `{{ $json.body }}{{ $('Edit Fields').item.json.SigShop }}` |
| Send office schwack | office@schwack.com | `{{ $json.body }}{{ $('Edit Fields').item.json.SigOffice }}` |
| Send coach photocoach | coach@photocoach.cc | `{{ $json.body }}{{ $('Edit Fields').item.json.SigCoach }}` |
| Send a message | harald.schwack@gmail.com | `{{ $json.body }}{{ $('Edit Fields').item.json.SigGmail }}` |

Bei **`Send email info`** steht dort aktuell noch `<br><br>` plus die alte
Referenz auf `Signatur`. Komplett durch den Wert aus der Tabelle ersetzen — der
Abstand steckt jetzt in der Signatur selbst (`margin-top:20px`), sonst entsteht
eine doppelte Lücke.

---

## Farben

| Signatur | Akzent | Warum |
|---|---|---|
| SigInfo | `#AFA362` Gold | Farbe der bisherigen Mindful-Images-Signatur, beibehalten |
| alle übrigen | `#00b09a` Teal | Akzentfarbe aus dem Brand-System |

Jeder Block ist unabhängig. Willst du Photocoach in einer eigenen Farbe, änderst
du in `SigCoach` die drei Vorkommen von `#00b09a` — die anderen sechs bleiben
unberührt. Das war der Punkt an getrennten Signaturen.

---

## Logo einbauen

Sobald du eine PNG-URL hast, in den betreffenden Block direkt nach `<tr>`
einfügen:

```html
<td style="padding-right:14px;vertical-align:top"><img src="HIER_DIE_URL" alt="Logo" width="110" style="display:block;border:0"></td>
```

Kein SVG — Gmail und Outlook verwerfen es. PNG oder JPG, auf einer erreichbaren
Adresse gehostet.

---

## Wartungshinweis

Telefonnummer und Anschrift stehen jetzt siebenmal im System. Wenn sich eines
davon ändert, sind **alle sieben Felder** anzupassen. Am schnellsten geht das,
indem du die sieben Blöcke aus dieser Datei in einem Editor per Suchen-Ersetzen
änderst und wieder einfügst — dann kann keiner übersehen werden.

---

## Weiterhin offen

* **Stille Fehlschläge** — `Send email info` und `Send contact` melden SMTP-Fehler
  als Erfolg, Notion hakt trotzdem ab. Siehe `umbau-manuell.md` Schritt 1.
* Kein Retry, keine Benachrichtigung bei Fehlschlag.
* Fehlende Fallback-Ausgänge: `Briefing` und leeres `Emailfrom` verschwinden lautlos.
* Gmail-Node schickt an den Anzeigenamen statt an die Adresse
  (`sendTo` → `{{ $json.toemail }}`).
