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
