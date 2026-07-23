# System-Watchdog — Architektur (Stand 20.07.2026)

Mehrschichtige Überwachung von Haralds Automatisierungs-Infrastruktur.
Grundprinzip: **Ein Watchdog, der in n8n läuft, kann nicht melden, wenn n8n
selbst steht.** Deshalb zwei Schichten mit unabhängigen Meldewegen.

## Meldeweg (der "Briefing-Kanal")

Wie Morgen-/Abend-Briefing: Eine Zeile in der Notion **Execute-DB**
(data_source_id `336a4c35-b2a7-8079-95dc-000bc83ddbc6`) mit
`Type = Briefing`, `Subject`, `Text` (Telegram-HTML), `Executed` leer.
Der n8n-Workflow **Daily Briefing Sender** (`vG0YBpLid2viAXk4`) pollt alle
30 Min (05:00–21:30) und stellt an Haralds Messenger-Chat (622619977) zu.

**Fallback bei n8n-Ausfall:** Claude-Push-Notification direkt aufs Handy —
unabhängig von n8n, Notion und Telegram-Zustellung.

## Schicht 1 — in n8n (fachliche Checks)

| Wächter | Prüft | Rhythmus | Alarmweg |
|---|---|---|---|
| Kernassets-Wächter v2 (`G4jfSndI2rjQjd7P`) | haraldschwack.at, photocoach.cc, gettingadddone.com, Etsy-Shop per HTTP + Inhalts-Check; Historie in Data Table `core_assets_status` | täglich 07:00 | Telegram direkt |
| Posting-Wächter (`8iYJgvNv6S5nHiWv`) | Social-Post heute Published? | täglich 19:30 | Telegram direkt |
| Fehler-Log-Check (Cowork-Skill `daily-fehler-log-check`) | n8n-Fehlermeldungen, fehlgeschlagene Workflows, fixt Kleinigkeiten | täglich | Briefing |

## Schicht 2 — außerhalb n8n (Claude-Routine "n8n System-Watchdog")

Claude-Routine `trig_018mUnYFK3ZMQTzqTNfUHb1w`, täglich 05:45 UTC
(07:45 Wien im Sommer, 06:45 im Winter), gebunden an die Session
`session_01KRNEPJSVFtw4SkLu4tRLsJ` (dort sind n8n- und Notion-Connector
verfügbar). **Report-only** — repariert nichts.

Prüfungen (Fokus: *stille* Ausfälle, die kein Fehler-Log erzeugen):

1. **n8n erreichbar?** Schlagen alle n8n-MCP-Calls fehl → n8n vermutlich
   down → kritischer Alarm per Push.
2. **Scheduler lebt?** Executions der letzten 26 h; null Executions =
   Scheduler tot.
3. **Kern-Workflows gelaufen?** Daily Briefing Sender, Kernassets-Wächter v2,
   instapost MI, Posting-Wächter, sendemaifromexecute — jeder braucht ≥1
   Lauf in 26 h, sonst Alarm "läuft still nicht mehr".
4. **Fehl-Executions** (error/crashed) → Warnung (Details übernimmt der
   Fehler-Log-Check).
5. **Briefing-Zustellung intakt?** Briefing-Zeilen > 2 h alt ohne
   `Executed`-Stempel = Zustellkette hängt → Alarm per Push (der
   Briefing-Kanal selbst ist dann ja betroffen).

Meldeverhalten: Alarm → Push **und** Briefing-Zeile. Nur Warnungen →
Briefing-Zeile. Alles OK → still; montags ein kurzes Lebenszeichen
("✅ System-Watchdog: alles lief normal"), damit erkennbar bleibt, dass der
Watchdog selbst lebt.

## Abgedeckte Ausfallszenarien

| Szenario | Erkannt durch | Meldeweg |
|---|---|---|
| Webseite/Etsy down (z. B. 403 wie im Juli) | Kernassets-Wächter | Telegram, täglich bis behoben; Dauer aus Data Table ablesbar |
| Workflow wirft Fehler | Fehler-Log-Check + Watchdog-Warnung | Briefing |
| Workflow läuft still gar nicht mehr | System-Watchdog (Schicht 2) | Briefing + ggf. Push |
| n8n komplett down / Scheduler tot | System-Watchdog (Schicht 2) | **Push** (n8n-unabhängig) |
| Briefing-Zustellung kaputt | System-Watchdog (Schicht 2) | **Push** (n8n-unabhängig) |

## Incident 23.07.2026 — "Taskagent"-Meldung

Der E-Mail-Verarbeiter `info@ XI` (`mucRYz36ldoE4Zq6`) meldete per Telegram,
er könne einen Task (Überweisung Domain-Rechnung mindful-images.com,
21,90 EUR, fällig 06.08.2026) nicht anlegen. Ursache: Der Tool-Workflow
**Taskagent** (`uSNXeAwkTdUmW7Qy`) des Agenten "ultimate intern" ist
**archiviert** → jeder Aufruf endet mit "Workflow is not active and cannot
be executed". Prüfung aller 16 Tool-Workflows des ultimate intern ergab
**7 archivierte**: Taskagent, Noteagent, Peopleagent, Areaagent,
Rezeptagent, Reiseagent, Contentcommander — diese Fähigkeiten fehlen dem
Agenten derzeit still.

Behoben/To-do:
- Verpasster Task wurde manuell in der Tasks-DB angelegt (Tickler 05.08.).
- Watchdog-Routine auf 04:30 Wien vorverlegt, damit die Meldung vor dem
  Morgenbriefing ankommt (Zustellung ~05:00 mit dem ersten Sender-Lauf).

### Trigger-Analyse (24.07.2026): Wer ruft den ultimate intern noch?

Alle nachvollziehbaren ultimate-intern-Läufe (16.07., 21.07., 23.07.; die
Läufe 14./15.07. passen ins selbe 5-Minuten-Raster) kamen über **einen**
Pfad: Der aktive Poller **"gmail@ V"** (`IOzKdyNtEABSYoi9`, Schedule alle
5 Min) übergibt Mails von Kontakten mit **AutoAction-Flag in der
People-DB** (`1e7252a0-0a25-4a14-9171-8abe34e3150c`) an den ultimate
intern. Der einzige Absender, der das zuletzt auslöste:
**HOTdomains (servicecenter@hotdomains.at)** — Domain-Rechnungen/Quittungen
(16.07. Rechnung photocoach.cc 23,88 EUR, 21.07. Zahlungsquittung dazu,
23.07. Rechnung mindful-images.com 21,90 EUR).

Zweiter, strukturell noch verdrahteter Pfad: `info@ XI` → Switch
"VIP? attachments" (Flags AutoAction/AutoEvent/AutoResponse/AutoCRM aus
der People-DB) → "Call 'ultimate intern'". Im Ausführungszeitraum nie
gefeuert.

**Dekommissionierungs-Empfehlung** (statt Ent-Archivieren): In der
People-DB alle Kontakte mit gesetzten Auto-Flags filtern (AutoAction /
AutoEvent / AutoResponse / AutoCRM = true) und die Flags abschalten —
mindestens beim Kontakt HOTdomains. Damit ist der letzte lebende Weg in
das Agenten-Konstrukt gekappt; die Mails landen weiterhin in der
Notion-Emails-DB und die Claude-E-Mail-Triage (4×/Tag) erzeugt daraus
Tasks (Regel: Rechnung mit Frist → Task, Tickler = Frist − 1 Tag).
Taskagent & Co. können archiviert bleiben; ultimate intern + Tool-Workflows
danach schrittweise stilllegen.

Randnotiz: Die Fehl-Execution von `info@ XI` am 23.07. 07:05 betraf eine
Phishing-Mail ("Nespresso"/Surface-Köder an paypal@haraldschwack.at) —
nicht klicken, kann gelöscht werden. Auffällig: Für paypal@haraldschwack.at
existiert ein People-Eintrag mit VIP-Flag; Spam an diese Adresse wird
dadurch bevorzugt behandelt — Eintrag prüfen.

## Wartung / offene Punkte

- Test-Workflows `TEST: Trigger-Registrierung via MCP` (`wNRYQ7ImxxTyuBwV`)
  und `TEST: Tages-Trigger-Format via MCP` (`iZojMixasWMuLlgz`) sowie
  `SETUP: Data Table core_assets_status` (`6eEIpJVB8B9JXDYS`) können in n8n
  gelöscht werden (Wegwerf-Artefakte vom 20.07.2026).
- Kernassets-Wächter **v1** (`idKwb6eMTGqmGV1H`) ist deaktiviert (Fehlalarm-Bug
  im Inhalts-Check, am 21.07.2026 durch v2 `G4jfSndI2rjQjd7P` ersetzt) und
  kann ebenfalls gelöscht werden.
- Kern-Workflow-Liste der Routine bei neuen täglichen Workflows ergänzen
  (Routine in claude.ai → Routinen bearbeiten).
- Cron der Routine ist UTC-fix: 05:45 UTC = 07:45 Wien im Sommer,
  06:45 im Winter.
