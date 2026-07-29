# Signaturen — Schnellweg (1 Feld ändern, 6 Felder ergänzen)

Nutzt das bereits vorhandene Feld `Signatur` im Node **`Edit Fields`**. Es ist
schon da, wird nur bisher fix mit der Mindful-Images-Signatur befüllt und
ausschließlich von `Send email info` gelesen.

Kein neuer Node, keine Verbindungen ändern. Etwa fünf Minuten.

---

## Schritt A — Das Feld `Signatur` ersetzen

Node **`Edit Fields`** öffnen, beim Feld **`Signatur`** den kompletten Inhalt
löschen und das Folgende einfügen. Achte darauf, dass das Feld im
**Expression**-Modus ist (nicht „Fixed") — erkennbar am `=` links davor.

```
{{ (function(){ const a=String($json.property_emailfrom||'').trim(); if(!a) return ''; let d=(a.split('@')[1]||'').toLowerCase(); if(!d) return ''; if(d==='gmail.com') d='schwack.com'; const T='#00b09a'; return '<table cellpadding="0" cellspacing="0" border="0" style="margin-top:20px;font-family:Arial,Helvetica,sans-serif;font-size:13px;line-height:1.55;color:#3C3C3B"><tr><td style="border-left:3px solid '+T+';padding:2px 0 2px 12px;vertical-align:top"><div style="font-size:15px;font-weight:bold;color:'+T+'">Harald Schwack</div><div style="font-size:14px;font-weight:bold;padding-bottom:7px"><a href="https://www.'+d+'" style="color:'+T+';text-decoration:none">'+d+'</a></div><div><strong>T:</strong> +43 699 1699 2411<br><strong>E:</strong> <a href="mailto:'+a+'" style="color:#3C3C3B;text-decoration:none">'+a+'</a><br>Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien</div></td></tr></table>'; })() }}
```

Damit baut sich die Signatur aus der Absenderadresse selbst: Name, Telefon und
Anschrift konstant, Domain prominent und verlinkt, E-Mail = Absenderadresse.
Neue Absender brauchen nie wieder eine Anpassung.

---

## Schritt B — Die sechs Versand-Nodes anhängen

In jedem Node das Feld **HTML** (beim Gmail-Node: **Message**) auf genau das
setzen:

```
{{ $json.body }}{{ $('Edit Fields').item.json.Signatur }}
```

| Node | Feld |
|---|---|
| Send contact | HTML |
| Send support GAD | HTML |
| Send shop GAD | HTML |
| Send office schwack | HTML |
| Send coach photocoach | HTML |
| Send a message | Message |

**`Send email info`** hat die Referenz schon, aber mit `<br><br>` davor. Dort
denselben Wert wie oben eintragen — der Abstand steckt jetzt in der Signatur
selbst (`margin-top:20px`), sonst klafft eine doppelte Lücke.

---

## Schritt C — Prüfen

Testzeile in Notion anhaken, Lauf abwarten, Mail ansehen. Erwartetes Bild:

```
┃ Harald Schwack
┃ photocoach.cc
┃ T: +43 699 1699 2411
┃ E: coach@photocoach.cc
┃ Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien
```

Der senkrechte Balken links ist teal `#00b09a`.

---

## Logo nachrüsten

Sobald eine PNG-URL je Domain vorliegt, wird aus dem Ausdruck in Schritt A eine
Variante mit Logospalte. Das ist eine Ersetzung desselben Feldes — sag mir die
URLs, dann liefere ich den fertigen Ausdruck.

Kein SVG: Gmail und Outlook verwerfen es.

---

## Was das hier *nicht* behebt

Der Schnellweg bringt nur die Signaturen. Weiterhin offen bleiben:

* **Stille Fehlschläge** — `Send email info` und `Send contact` melden einen
  SMTP-Fehler als Erfolg, Notion hakt die Zeile trotzdem ab. Zwei Schalter,
  siehe `umbau-manuell.md` Schritt 1. **Das ist der wichtigste offene Punkt.**
* Kein Retry bei kurzen Störungen, keine Benachrichtigung bei Fehlschlag.
* Fehlende Fallback-Ausgänge — `Briefing` und leeres `Emailfrom` verschwinden
  lautlos und laufen alle 10 Minuten erneut.
* Gmail-Node schickt an den Anzeigenamen statt an die Adresse.
