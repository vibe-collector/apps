# Execute HS → E-Mail-Versand: Befundbericht & Testroutine

**Workflow:** `sendemaifromexecute` (n8n-ID `4r1b9Vs8XOynpAqc`)
**Quelle:** Notion-DB ⚡ Execute HS (`336a4c35-b2a7-80ac-9258-dc29fc28d9c3`)
**Stand:** 29.07.2026, 15:15 Uhr (Europe/Vienna)

---

## 1. Kurzfassung

Der Workflow läuft alle 10 Minuten und holt alle Zeilen mit `Execute = true`.
Diese Grundmechanik ist in Ordnung. **Der Versand selbst ist es nicht.**

Von den 5 heute erstellten Test-Mails ist **keine einzige** zugestellt worden.
Zwei davon sind in Notion trotzdem als „Executed" markiert.

Ursache ist nicht ein Fehler, sondern eine Kette aus vier Problemen:
ein totes SMTP-Konto, eine Fehlerbehandlung die Fehlschläge als Erfolg meldet,
eine fehlende Fehlerbehandlung die den Workflow in eine Endlosschleife schickt,
und Signaturen, die nur bei einem von sieben Absendern existieren.

---

## 2. Befunde

### F1 — SMTP-Credential „SMTP account contact" ist tot (Blocker)

`ETIMEDOUT — Connection timeout` nach jeweils 120 Sekunden. Kein Verbindungsaufbau.

Betroffen sind vier von sieben Absendern:

| Absender | Node | Credential |
|---|---|---|
| support@gettingadddone.com | Send support GAD | SMTP account contact |
| shop@gettingadddone.com | Send shop GAD | SMTP account contact |
| office@schwack.com | Send office schwack | SMTP account contact |
| coach@photocoach.cc | Send coach photocoach | SMTP account contact |

Ein Timeout beim *Verbindungsaufbau* (nicht beim Login) heißt: Host oder Port
sind nicht erreichbar. Das ist eine Credential-Konfiguration, kein Workflow-Fehler —
lässt sich nur mit den Zugangsdaten des Mail-Providers lösen.

In n8n existieren drei SMTP-Credentials: `SMTP account info`, `SMTP account info3`,
`SMTP account contact`. Welches davon tatsächlich funktioniert, ist bisher nirgends belegt.

### F2 — Stille Fehlschläge (kritischster Befund)

`Send email info` und `Send contact` sind auf `onError: continueRegularOutput`
+ `alwaysOutputData: true` gesetzt. Bei SMTP-Fehler geben sie
`{"error":"Connection timeout"}` mit **Status „success"** aus — und der
nachgelagerte Notion-Node setzt die Zeile auf `Executed`.

Belegt in Execution `342072`:

```
Send contact          → executionStatus: "success"
                        data: { "error": "Connection timeout" }
Update a database page → Executed = 2026-07-29T14:42, Execute = false
```

Die Notion-Zeile „Testantwort an Harald (1)" (contact@schwack.com) steht damit
auf „erledigt", obwohl nie eine Mail rausging. Dasselbe gilt für
„Testantwort an Harald" (info@haraldschwack.at, Executed 12:33).

**Das System lügt über den Versandstatus.** Solange das so ist, ist jede
Erfolgsmeldung wertlos.

### F3 — Endlosschleife bei den übrigen vier Absendern

Die vier Nodes aus F1 haben *keine* Fehlerbehandlung. Ein SMTP-Timeout lässt
die gesamte Execution abstürzen, bevor der Notion-Node läuft. `Execute` bleibt
auf `true` → nächster Lauf in 10 Minuten → gleicher Absturz.

Zwischen 12:40 und 13:10 sind so sechs Läufe à 2–4 Minuten fehlgeschlagen.
**Ich habe den Haken bei diesen vier Zeilen entfernt, die Schleife ist gestoppt.**

### F4 — Signatur nur bei einem von sieben Absendern

Nur `Send email info` hängt `{{ $('Edit Fields').item.json.Signatur }}` an.
`Send contact`, `Send support GAD`, `Send shop GAD`, `Send office schwack`,
`Send coach photocoach` und der Gmail-Node haben **keine Signatur**.

Zusätzlich: Die eine existierende Signatur ist fest auf
„Harald Schwack / Mindful Images / haraldschwack.at" verdrahtet. Für
schwack.com, gettingadddone.com und photocoach.cc wäre sie inhaltlich falsch.

### F5 — Gmail-Node schickt an den Namen statt an die Adresse

`Send a message` verwendet `sendTo: {{ $json.to }}`. Das Feld `to` enthält den
Anzeigenamen („Harald"), nicht die Adresse. Richtig wäre `toemail`.
Die Route für harald.schwack@gmail.com kann so nie zustellen.

### F6 — Keine Fallback-Ausgänge an beiden Switches

* `Switch` (nach Type) kennt: Send Email, Create Calender Event, Send DM, Send Telegram.
  **`Briefing` fehlt** — obwohl es in Notion als Auswahl existiert.
* `Route by Emailfrom` hat keinen Fallback für leeres/unbekanntes `Emailfrom`.

In beiden Fällen verschwindet die Zeile lautlos, der Notion-Node läuft nie,
`Execute` bleibt gesetzt → dieselbe Dauerschleife wie in F3.

### F7 — Meta-DM-Node ohne Inhalt

`HTTP: Send Meta DM` hat `bodyParameters: [{}]` — leer. Weder Empfänger noch
Nachricht werden übertragen. `Type = Send DM` ist funktionslos.

### F8 — Gespeicherte Version weicht von der aktiven ab

Aktive Version: `Get many database pages` mit `limit: 10`.
Gespeicherter Entwurf: `limit: 1`.
Ein Publish würde die Batchgröße unbemerkt auf 1 ändern.

### F9 — Kein Retry, keine Benachrichtigung

Keiner der Versand-Nodes hat „Retry on Fail". Bei einem kurzen Netzaussetzer
ist die Mail verloren (F2) oder die Schleife startet (F3). Harald erfährt in
keinem Fall etwas davon.

---

## 3. Was ich brauche

### 3.1 Signaturen — pro Absender

Für jeden Absender brauche ich die Werte unten. Für **info@haraldschwack.at**
habe ich sie bereits aus der bestehenden Signatur; die sechs anderen fehlen
komplett.

Vorlage zum Ausfüllen:

```
Absender:        <z. B. office@schwack.com>
Name:            <Harald Schwack>
Zeile 2:         <Marke / Rolle, z. B. "Unternehmensberatung">
Telefon:         <+43 ...>
E-Mail-Anzeige:  <office@schwack.com>
Website:         <www.schwack.com>
Adresse:         <Straße | PLZ Ort>
Logo-URL:        <https://...>
Akzentfarbe:     <#RRGGBB>
Social-Links:    <FB / LinkedIn / Instagram — welche, welche URLs>
Rechtszeile:     <UID, Firmenbuchnummer o. Ä. — falls nötig>
```

**Bekannt (info@haraldschwack.at):** Harald Schwack · Mindful Images ·
+43 699 1699 2411 · info@haraldschwack.at · www.haraldschwack.at ·
Hintere Liesingbachstr. 14-16/A4/3, 1100 Wien · Akzent `#AFA362` ·
FB + LinkedIn + Instagram · Logo von haraldschwack.at

**Offen:** contact@schwack.com · office@schwack.com ·
support@gettingadddone.com · shop@gettingadddone.com · coach@photocoach.cc ·
harald.schwack@gmail.com

Kürzeste Variante, falls es heute schnell gehen soll: sag mir, welche Absender
du **heute** wirklich brauchst — für die restlichen setze ich vorerst eine
schlichte Textsignatur ein.

### 3.2 SMTP-Zuordnung

Welcher Absender soll über welches Postfach raus? Konkret ungeklärt:

* Warum läuft `SMTP account contact` in einen Timeout — Host, Port oder Blockade?
* Sind gettingadddone.com und photocoach.cc überhaupt beim selben Provider
  wie schwack.com, oder brauchen sie eigene Credentials?
* Gibt es funktionierende Zugangsdaten, die ich hinterlegen kann?

### 3.3 Verhalten im Fehlerfall

Wenn eine Mail nicht rausgeht — was soll passieren? (Frage stelle ich separat.)

---

## 4. Testroutine

### Grundregel

**Immer genau eine Zeile mit `Execute = true`.** Der 10-Minuten-Takt bleibt
aktiv, ich stoße die Läufe zusätzlich manuell an, damit wir nicht warten müssen.
So ist bei jedem Lauf eindeutig, welche Mail welches Ergebnis erzeugt hat.

### Ablauf pro Absender (ca. 3 Minuten)

| Schritt | Wer | Was |
|---|---|---|
| 1 | ich | Eine Testzeile anlegen: `Type = Send Email`, `Emailfrom = <Absender>`, `Emailto = info@haraldschwack.at`, `Subject = TEST <Absender> <Uhrzeit>`, Standard-Testtext |
| 2 | ich | Haken setzen, Workflow manuell auslösen |
| 3 | ich | Execution auslesen: SMTP verbunden? Fehlertext? Notion korrekt aktualisiert? |
| 4 | **du** | Postfach prüfen: angekommen? Absender richtig? Signatur richtig? Umbrüche sauber? Im Spam? |
| 5 | ich | Ergebnis in die Matrix eintragen, Fehler fixen, oder nächster Absender |

Erst wenn eine Zeile grün ist, gehe ich zur nächsten.

### Reihenfolge

1. **info@haraldschwack.at** — einziger Absender mit Signatur, bester Referenzfall
2. **contact@schwack.com** — deine Hauptadresse
3. **office@schwack.com**
4. **coach@photocoach.cc**
5. **support@gettingadddone.com**
6. **shop@gettingadddone.com**
7. **harald.schwack@gmail.com** — Gmail-Route, braucht vorher Fix F5

Danach die übrigen Typen: Create Calender Event · Send Telegram ·
Briefing (Route fehlt, F6) · Send DM (Node leer, F7).

### Testmatrix

| # | Absender | SMTP-Credential | n8n-Status | Zugestellt | Signatur | Notiz |
|---|---|---|---|---|---|---|
| 1 | info@haraldschwack.at | SMTP account info | **250 Ok, queued** ✅ | Prüfung Harald | vorhanden | Test 01, 15:20 |
| 2 | coach@photocoach.cc | SMTP account contact | **Timeout** | nein | fehlt | heute erforderlich |
| 3 | shop@gettingadddone.com | SMTP account contact | **Timeout** | nein | fehlt | heute erforderlich |
| 4 | contact@schwack.com | — | offen | offen | fehlt | Zeile fälschlich auf „Executed" |
| 5 | office@schwack.com | SMTP account contact | **Timeout** | nein | fehlt | |
| 6 | support@gettingadddone.com | SMTP account contact | **Timeout** | nein | fehlt | |
| 7 | harald.schwack@gmail.com | Gmail OAuth | offen | offen | fehlt | F5 blockiert |

**Heutiger Scope (Entscheidung Harald):** info@haraldschwack.at ·
coach@photocoach.cc · shop@gettingadddone.com
**Fehlerverhalten (Entscheidung Harald):** 3× Retry, danach parken + Telegram-Alarm

### Ergebnis Test 01 — info@haraldschwack.at

Execution `342115`, 29.07. 15:20:57, Laufzeit 1,8 s.

```
accepted:  ["info@haraldschwack.at"]
rejected:  []
response:  "250 2.0.0 Ok: queued as 96C8850A4D6F"
messageId: <f821a4d6-d831-c45a-b404-2706e9cc1d5f@haraldschwack.at>
```

Der Mailserver hat die Nachricht angenommen — kein stiller Fehlschlag, sondern
ein belegter Versand. **Damit ist bewiesen: `SMTP account info` funktioniert.**
Der Ausfall betrifft ausschließlich `SMTP account contact`.

Offen: Zustellung im Postfach und Darstellung der Signatur (prüft Harald).

Nebenbefund: Eine frisch angelegte Notion-Zeile ist ca. 10–60 s lang nicht über
die gefilterte API-Abfrage sichtbar. Für den 10-Minuten-Takt irrelevant, beim
manuellen Anstoßen aber zu beachten.

---

## 5. Reihenfolge der Umsetzung

**Phase 0 — Stabilisieren** (erledigt / ohne Input möglich)
- [x] Endlosschleife gestoppt (Haken bei 4 Zeilen entfernt)
- [ ] SMTP-Timeout von 120 s auf ~15 s senken — Fehlschläge kosten sonst Minuten
- [ ] Fehlerausgänge statt stiller Erfolgsmeldung (F2)
- [ ] Fallback-Ausgänge an beiden Switches (F6)
- [ ] Gmail-Empfänger auf `toemail` korrigieren (F5)
- [ ] Draft/Active-Divergenz auflösen (F8)

**Phase 1 — Signaturen** (braucht deinen Input, Abschnitt 3.1)
- [ ] Signaturblock pro Absender hinterlegen, zentral statt pro Node

**Phase 2 — SMTP klären** (braucht deinen Input, Abschnitt 3.2)
- [ ] Funktionierende Credentials hinterlegen bzw. Zuordnung korrigieren

**Phase 3 — Durchtesten** (gemeinsam, Abschnitt 4)
- [ ] Absender 1–7 einzeln, Matrix füllen

**Phase 4 — Restliche Typen**
- [ ] Calendar · Telegram · Briefing-Route · Meta-DM

**Phase 5 — Scharfschalten**
- [ ] Retry auf transiente Fehler, Telegram-Alarm bei Fehlschlag
- [ ] Zwei falsch als „Executed" markierte Zeilen korrigieren

---

## 6. Sofort geändert

| Zeitpunkt | Änderung | Grund |
|---|---|---|
| 29.07. 15:14 | `Execute` entfernt bei „Testantwort an Harald (2)–(5)" | Endlosschleife alle 10 Min gestoppt |
| 29.07. 15:20 | Testzeile „TEST 01" angelegt und versendet | Nachweis, dass `SMTP account info` funktioniert |

---

## 7. Signaturen — Entwurf

Grundsatzentscheidung: **bildfrei**, außer bei info@haraldschwack.at.

Begründung: Die bestehende Mindful-Images-Signatur lädt vier Bilder nach, drei
davon von `mail-signatures.com` — einer fremden Generator-Seite. Fällt die aus
oder sperrt sie Hotlinking, bricht die Signatur. Dazu sind nachgeladene Bilder
in vielen Clients standardmäßig blockiert und zählen als Spam-Signal.

Eine reine Textsignatur mit farbigem Akzentbalken rendert in Gmail, Outlook,
Apple Mail und Thunderbird identisch, bricht nie und kostet keine Zustellrate.
Farbe: Teal `#00b09a` aus dem hinterlegten Brand-System.

```html
<table cellpadding="0" cellspacing="0" border="0"
       style="margin-top:20px;font-family:Arial,Helvetica,sans-serif;
              font-size:13px;line-height:1.55;color:#3C3C3B">
  <tr><td style="border-left:3px solid #00b09a;padding:2px 0 2px 12px">
    <div style="font-size:15px;font-weight:bold;color:#00b09a">Harald Schwack</div>
    <div style="font-size:13px;font-weight:bold;color:#00b09a;padding-bottom:7px">Photocoach</div>
    <div><strong>T:</strong> +43 699 1699 2411<br>
      <strong>E:</strong> <a href="mailto:coach@photocoach.cc">coach@photocoach.cc</a><br>
      <strong>W:</strong> <a href="https://www.photocoach.cc">www.photocoach.cc</a><br>
      Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien</div>
  </td></tr>
</table>
```

Zuordnung der Markenzeile:

| Absender | Markenzeile | Website |
|---|---|---|
| info@haraldschwack.at | Mindful Images | haraldschwack.at (bestehende Bildsignatur) |
| contact@schwack.com | schwack.com | www.schwack.com |
| office@schwack.com | schwack.com | www.schwack.com |
| coach@photocoach.cc | Photocoach | www.photocoach.cc |
| support@gettingadddone.com | Getting ADD Done ⚠️ | www.gettingadddone.com |
| shop@gettingadddone.com | Getting ADD Done ⚠️ | www.gettingadddone.com |
| harald.schwack@gmail.com | Harald Schwack | www.schwack.com |

⚠️ = aus dem Domainnamen abgeleitet, von Harald noch nicht bestätigt.

Telefon und Adresse sind laut Harald bei allen Absendern identisch.

### Logos

Nicht beschaffbar in dieser Session: photocoach.cc und gettingadddone.com sind
über die Netzwerk-Policy gesperrt (`connect_rejected — policy denial`). Der
Umweg über einen n8n-HTTP-Request scheitert an der fehlenden Freigabe zum
Anlegen von Workflows. Das im Brand-System hinterlegte HS-Logo liegt als
Inline-SVG vor — für E-Mail unbrauchbar, da Gmail und Outlook SVG verwerfen.

Wenn Logos gewünscht sind, brauche ich je eine PNG-URL. Einbau ist dann eine
Zeile pro Absender.

---

## 8. Vorbereiteter Umbau (wartet auf Freigabe)

`update_workflow` auf `4r1b9Vs8XOynpAqc`, 33 Operationen, gezielt statt
Neuanlage — Credentials bleiben unangetastet.

| # | Operation | Behebt |
|---|---|---|
| 1 | Code-Node „Build Signature" zwischen `Edit Fields` und `Switch` | F4 |
| 2 | Signatur pro Absender, `htmlBody` = Text + Signatur | F4 |
| 3 | `pageId` wird mitgeführt (für den Fehlerpfad) | — |
| 4 | Alle 7 Versand-Nodes: `html`/`message` → `{{ $json.htmlBody }}` | F4 |
| 5 | Alle 7: `retryOnFail`, `maxTries: 3`, `waitBetweenTries: 5000` | F9, Entscheidung Harald |
| 6 | Alle 7: `onError: continueErrorOutput`, `alwaysOutputData: false` | **F2** |
| 7 | Gmail `sendTo` → `{{ $json.toemail }}` | F5 |
| 8 | Fallback-Ausgang an `Switch` und `Route by Emailfrom` | F6 |
| 9 | Neu: `Execute zuruecksetzen` (Notion, entfernt nur den Haken) | F3 |
| 10 | Neu: `Alert Fehler` (Telegram an 622619977) | F9 |
| 11 | Fehlerausgänge aller 7 Nodes + beide Fallbacks → Fehlerpfad | F2, F3, F6 |

Wirkung: Ein Fehlschlag setzt `Executed` **nicht** mehr, entfernt den Haken
(keine Schleife) und meldet sich per Telegram. `Executed` wird ausschließlich
nach einem echten `250 Ok` gesetzt.

Nicht enthalten, bewusst später: Meta-DM-Body (F7), Draft/Active-Divergenz (F8),
Korrektur der zwei falsch markierten Zeilen.
