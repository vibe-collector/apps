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
| 2 | coach@photocoach.cc | SMTP coach@photocoach.cc | **250 Ok, 447 ms** | Prüfung Harald | fehlt | Test 02, nach Portwechsel |
| 3 | shop@gettingadddone.com | SMTP shop@gettingadddone.com | **250 Ok, 472 ms** | Prüfung Harald | fehlt | Test 03, nach Portwechsel |
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

### Ergebnis Test 02 — coach@photocoach.cc

Execution `342183`, 29.07. 16:35:12, Laufzeit **120.011 ms**.

```
Send coach photocoach → ETIMEDOUT
messages:   ["Connection timeout", "Connection timeout"]
credential: SMTP coach@photocoach.cc  (aLcKjxqRJrdoTzUc)
```

Wichtig: Das Credential ist **neu und korrekt zugeordnet** — Harald hat je
Postfach ein eigenes angelegt (`aLcKjxqRJrdoTzUc` für coach@,
`vHmoAivaE4lBIYLN` für shop@) und der Node zeigt darauf. Das
Platzhalter-Problem aus F1 ist damit erledigt. Auch die Passwörter wurden in
Plesk und in n8n neu vergeben.

**Der Timeout bleibt trotzdem — und das grenzt die Ursache eindeutig ein.**

### F10 — Es ist nicht das Passwort, es ist Host oder Port

| Beobachtung | Schlussfolgerung |
|---|---|
| `SMTP account info` antwortet in **369 ms** mit `250 Ok` | Der Weg von n8n zu diesem Mailserver ist frei |
| `SMTP coach@photocoach.cc` schweigt **120.000 ms** | Die TCP-Verbindung kommt nie zustande |

Ein falsches Passwort ergäbe einen Authentifizierungsfehler nach
Millisekunden. 120 Sekunden Stille bedeutet: der Server hat auf den
Verbindungsversuch nie geantwortet. Die Zugangsdaten wurden gar nicht erst
gesendet — es gab nichts, wohin man sie hätte senden können.

Mögliche Ursachen, nach Wahrscheinlichkeit:

1. **Port 25.** Wird von den meisten Hostern ausgehend blockiert. Das Symptom
   ist exakt dieses stille Timeout. Richtig sind 587 (STARTTLS) oder 465 (SSL/TLS).
2. **Falscher Host.** Tippfehler, oder ein Hostname der nicht auf den
   Mailserver zeigt.
3. Firewall zwischen n8n und dem Mailserver.

**Lösung:** Host und Port aus `SMTP account info` übernehmen — von dieser
Kombination ist erwiesen, dass sie von n8n aus durchgeht. Da alle Domains auf
demselben Plesk liegen, bedient ein einziger Mailserver sie alle; der Host ist
für coach@photocoach.cc derselbe wie für info@haraldschwack.at. Nur Benutzer
(die vollständige Adresse) und Passwort sind je Postfach verschieden.

### F10 gelöst — es war der Port

Harald hat auf **Port 587 mit ausgeschaltetem SSL** umgestellt, analog zum
funktionierenden `SMTP account info`. Beide Absender gehen seitdem durch:

| | shop@gettingadddone.com | coach@photocoach.cc |
|---|---|---|
| Execution | `342194` | `342196` |
| Antwort | `250 2.0.0 Ok: queued as ADF52515184B` | `250 2.0.0 Ok: queued as 589725151205` |
| Message-ID | `…@gettingadddone.com` | `…@photocoach.cc` |
| Dauer | 472 ms | 447 ms |
| vorher | 120.011 ms Timeout | 120.011 ms Timeout |

Die Message-IDs tragen jeweils die richtige Absenderdomain — für die
DKIM-Zuordnung das gewünschte Bild.

Port 587 ohne SSL ist korrekt: dort wird unverschlüsselt verbunden und per
STARTTLS hochgestuft. Der SSL-Schalter gehört zu Port 465, wo die
Verschlüsselung von der ersten Sekunde an steht. Beides gleichzeitig ergäbe
denselben Timeout aus anderer Ursache.

Damit gilt für die verbleibenden Absender dieselbe Kombination — es unterscheiden
sich nur Benutzername und Passwort. Offen: contact@schwack.com,
office@schwack.com, support@gettingadddone.com.

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

## 7. Signaturen — festgelegt

**Vorgabe Harald:** Es ist ein durchgehendes Thema. Er ist immer Harald Schwack,
mit vollständiger Anschrift und Telefonnummer. Prominent variiert nur die
Domain, um die es geht, sowie die E-Mail-Adresse — und die ist immer die
Absenderadresse.

Daraus folgt: **eine** Vorlage statt sieben, vollständig aus `Emailfrom`
abgeleitet. Kein Pflegeaufwand bei neuen Absendern, keine Divergenz.

| Bestandteil | Wert |
|---|---|
| Name | Harald Schwack (konstant) |
| Prominente Zeile | Domain aus `Emailfrom`, verlinkt |
| Telefon | +43 699 1699 2411 (konstant) |
| E-Mail | `Emailfrom` selbst |
| Anschrift | Hintere Liesingbachstraße 14-16/A4/3 · 1100 Wien (konstant) |
| Akzent | Teal `#00b09a` |

Sonderfall: `harald.schwack@gmail.com` — gmail.com taugt nicht als prominente
Domain, dort steht schwack.com.

Logos werden über eine `LOGOS`-Tabelle im Code-Node nachgerüstet, eine Zeile je
Domain. Ist der Eintrag leer, entfällt die Logospalte ersatzlos — die Signatur
bleibt in jedem Fall gültig.

**Damit entfällt die bisherige Mindful-Images-Signatur für info@haraldschwack.at.**
Falls sie dort erhalten bleiben soll, ist das ein Eintrag in der Zuordnung.

Bildfrei, bis Logo-URLs vorliegen.

Ergebnis je Absender:

| Absender | Prominente Zeile | E-Mail in der Signatur |
|---|---|---|
| info@haraldschwack.at | haraldschwack.at | info@haraldschwack.at |
| contact@schwack.com | schwack.com | contact@schwack.com |
| office@schwack.com | schwack.com | office@schwack.com |
| coach@photocoach.cc | photocoach.cc | coach@photocoach.cc |
| support@gettingadddone.com | gettingadddone.com | support@gettingadddone.com |
| shop@gettingadddone.com | gettingadddone.com | shop@gettingadddone.com |
| harald.schwack@gmail.com | schwack.com (Sonderfall) | harald.schwack@gmail.com |

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

---

## 9. Live-Checkliste (Stand 29.07. 17:05)

### Bestätigt

| Absender | Execution | Serverantwort |
|---|---|---|
| info@haraldschwack.at | 342115 | `250 Ok: queued as 96C8850A4D6F` |
| shop@gettingadddone.com | 342203 | `250 Ok: queued as 5E6F05151205` |
| coach@photocoach.cc | 342201 | `250 Ok: queued as 942305151205` |
| office@schwack.com | 342216 | `250 Ok: queued as 3A87D50D28A9` |
| support@gettingadddone.com | 342217 | `250 Ok: queued as B09CB50D28A9` |
| contact@schwack.com | 342256 | `250 Ok: queued as A7B995151870` |
| harald.schwack@gmail.com | 342247 | Gmail-API, Message-ID `19fae6dc7d7b8366` ⚠️ |

⚠️ Gmail lief nur, weil im Testdatensatz im Feld `to` eine echte Adresse stand.
Mit dem üblichen Inhalt („Harald") schlägt die Route fehl — siehe F5. Der Node
selbst und das OAuth-Token sind in Ordnung, defekt ist ausschließlich das Mapping.

**Damit sind alle sieben Absender belegt.**

Alle Message-IDs tragen die jeweils richtige Absenderdomain. Kalenderzweig
ebenfalls bestätigt (Execution 342208, Kino-Termin angelegt).

### Offen vor dem Livegang

| # | Punkt | Wirkung wenn nicht behoben |
|---|---|---|
| 1 | `alwaysOutputData` + `onError: continueRegularOutput` bei `Send email info` und `Send contact` | Fehlgeschlagene Mails werden in Notion als „Executed" abgehakt. Verlust ohne Spur. |
| 2 | Sieben `Sig*`-Felder in `Edit Fields` fehlen | Keine Mail hat eine Signatur |
| 3 | Tote Referenz auf `Signatur` in `Send email info` | Verweist auf ein gelöschtes Feld |
| 4 | Gmail-Node `sendTo: {{ $json.to }}` | Route kann nicht zustellen, Feld enthält „Harald" |
| 5 | `Get many database pages` steht auf `limit: 1` | Maximal 6 Mails pro Stunde |
| ~~6~~ | ~~contact@ und Gmail ungetestet~~ | **erledigt 17:21** |

Punkt 1 ist der einzige, der stillen Datenverlust verursacht — alle anderen
sind sichtbar, sobald sie auftreten.

### Zustand des Workflows

Unverändert gegenüber dem Ausgangszustand, mit zwei Ausnahmen:
`Send coach photocoach` hat `retryOnFail: true` erhalten, und das Feld
`Signatur` in `Edit Fields` wurde gelöscht.

Sämtliche schreibenden MCP-Aufrufe (`update_workflow`,
`create_workflow_from_code`, `execute_workflow`, `search_nodes`) werden vom
n8n-Connector mit `requires approval` abgelehnt. Die Änderungen sind daher
manuell einzutragen: `umbau-manuell.md` und `signaturen-pro-adresse.md`.

---

## 10. Umsetzung 30.07.2026 — Schreibzugriff zurück, Signaturen live

Nach einem Reconnect des n8n-Connectors funktionierten die schreibenden
MCP-Aufrufe wieder. Der Umbau wurde daraufhin direkt abgesetzt.

### Angewendet

| Änderung | Umfang |
|---|---|
| Signatur je Absender **direkt im Versand-Node** (Haralds Variante 1) | 7 Nodes |
| `retryOnFail` mit 3 Versuchen, 5 s Abstand | alle 7 Versand-Nodes |
| `alwaysOutputData: false` + `onError: stopWorkflow` | `Send email info`, `Send contact` |
| Publiziert | `activeVersionId f3d76208-bd0f-45b4-8a9b-0f76d0813f03` |

`Edit Fields` wurde bewusst **nicht** angefasst — damit sind Haralds manuelle
Änderungen (Gmail-Empfänger) unberührt und die tote Referenz auf das gelöschte
Feld `Signatur` ist mit ersetzt.

### Warum `stopWorkflow` statt `continueErrorOutput`

Ursprünglich war „Continue (using error output)" empfohlen — das galt für die
Version **mit** angeschlossenem Fehlerpfad. Der existiert nicht. Ein
unverbundener Fehlerausgang bedeutet: Das Item wird lautlos verworfen **und die
Execution meldet trotzdem Erfolg** — schlechter für die Diagnose als vorher.

Mit `stopWorkflow` schlägt der Lauf sichtbar fehl, `Executed` bleibt leer, und
der Fehler ist im n8n-Dashboard rot.

### Verifikation

Execution `343097`, 30.07. 07:20:57, Absender info@haraldschwack.at:

```
messageSize: 1326        (Testmails ohne Signatur lagen bei 332–347)
response:    250 2.0.0 Ok: queued as D0B3C514FBBA
messageId:   <49549469-30eb-31a3-c3f3-68a36303f622@haraldschwack.at>
```

Keine Fehl-Execution zwischen 29.07. 15:21 und 30.07. 07:22.

### Einstellungen nach Haralds Entscheidung

* **Intervall 5 Minuten** (statt 10) → 12 Durchläufe pro Stunde
* **`limit: 1` bleibt bewusst** — eine Execution entspricht genau einem Fall,
  was die Fehlersuche erheblich vereinfacht. Bei ~6 Einträgen in 14 Tagen ist
  der Durchsatz reichlich bemessen.

### Bekannte Nebenwirkung von `limit: 1`

Schlägt eine Zeile dauerhaft fehl, bleibt ihr Haken gesetzt. Da pro Lauf nur
**eine** Zeile geholt wird, kann diese Zeile bei jedem Durchlauf erneut gezogen
werden und nachfolgende Einträge blockieren. Bei zehn Zeilen pro Lauf fiele das
nicht auf, bei einer schon.

Zwei Wege damit umzugehen:
1. Rot markierte Executions im Dashboard beobachten und den Haken manuell
   entfernen (aktueller Zustand)
2. Fehlerpfad nachrüsten, der den Haken automatisch entfernt und per Telegram
   meldet — oder den bestehenden Workflow `Error Logger Notion lastruns` als
   Error Workflow in den Settings eintragen. Dann greift auch der tägliche
   Watchdog.

### Weiterhin offen

* Fallback-Ausgänge an beiden Switches (F6) — `Briefing` und leeres `Emailfrom`
* Meta-DM-Node ohne Body (F7)
* Zwei Notion-Zeilen fälschlich auf „Executed" (`Testantwort an Harald`, `(1)`)
* Logos in den Signaturen, sobald PNG-URLs vorliegen

---

## 11. Abschlusscheck 31.07.2026, 13:15

### Betriebsbilanz

| Prüfung | Ergebnis |
|---|---|
| Fehl-Executions seit 30.07. 07:22 | **0** in über 28 Stunden |
| Zeilen mit hängendem `Execute`-Haken | **0** — jede Zeile hat einen `Executed`-Stempel |
| Blockade durch `limit: 1` | nicht eingetreten |
| Erste echte Korrespondenz | „Antwort Sammelgeschenk Florian (50.) — Zusage", 30.07. 18:39 |

Harald hat am 30.07. abends alle sieben Absender zweimal durchgetestet
(Signatur, dann Signatur mit Logo) plus einen finalen Testrun. Logos sind in
der aktiven Version enthalten. Takt steht inzwischen auf **4 Minuten**.

### Offen: Entwurf ist nicht publiziert

`versionId 2b57e9ab` ≠ `activeVersionId 9da93e0c`. Im Entwurf liegen
Änderungen, die nicht live sind:

| Node | Entwurf | Aktiv |
|---|---|---|
| Send email info | `Harald Schwack I Mindful Images <info@…>` | `info@haraldschwack.at` |
| Send contact | `Harald Schwack <contact@…>`, Domainzeile entfernt | `contact@schwack.com`, mit Domainzeile |
| Send support GAD | `GettingADDDone Support <support@…>` | `support@gettingadddone.com` |
| Send shop GAD | `GettingADDDone Shop <shop@…>` | `shop@gettingadddone.com` |
| Send office schwack | `Harald Schwack I schwack.com <office@…>`, Logo auf `www.` | `office@schwack.com` |
| Send coach photocoach | `Harald Schwack I photocoach.cc <coach@…>` | `coach@photocoach.cc` |
| Send a message | Domainzeile entfernt | mit Domainzeile |

Anzeigenamen im From-Header sind eine gute Idee — sie erhöhen die
Wiedererkennung im Posteingang. Sie sind aber noch nie durch einen Testlauf
gegangen. Vor dem Publish einen Testversand je Absender, danach Absenderzeile
im Postfach prüfen.

### Nachgezogen am 31.07.

`Send a message` (Gmail) stand als **einziger** Versand-Node noch auf
`onError: continueRegularOutput` — ein Gmail-Fehler wäre weiterhin als Erfolg
nach Notion durchgereicht worden. Jetzt auf `stopWorkflow` plus Retry, wie die
anderen sechs. Liegt im Entwurf, geht mit dem nächsten Publish live.

### Weiterhin latent

* **Fallback-Ausgänge fehlen** an beiden Switches. Eine Zeile mit unbekanntem
  `Type` oder leerem `Emailfrom` wird lautlos verworfen, `Execute` bleibt
  gesetzt — und blockiert bei `limit: 1` alle vier Minuten die Warteschlange.
  Bisher nicht aufgetreten, weil `Briefing`-Zeilen von einem separaten
  Workflow abgearbeitet werden.
* **Meta-DM-Node** hat einen leeren Body, `Type = Send DM` ist funktionslos.
* Zwei Notion-Zeilen vom 29.07. stehen fälschlich auf „Executed".
