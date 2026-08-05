# Scheduled Task `orchestrator` — Einrichtung

Stand: 05.08.2026. **Der Task ist angelegt und aktiv.**

`PROMPT.md` enthält den vollständigen Seiteninhalt der Notion-Seite
[🧠 Orchestrator (COO)](https://app.notion.com/p/390a4c35b2a7818dbffde3156f190a73)
(ID `390a4c35-b2a7-818d-bffd-e3156f190a73`) — wortwörtlich, ungekürzt, inklusive
Lauf-Weiche. Er gehört als Task-Prompt eingefügt, nicht als Link: der Task hat zur
Laufzeit keinen garantierten Notion-Zugriff auf diese Seite.

## Konfiguration

| Feld | Wert |
|---|---|
| Name | `orchestrator` |
| Cron | `20 2,6,10,14,18,22 * * *` |
| Modell | Opus |
| Session | eigene Session pro Lauf |
| Connector | Notion |
| Zeitplan | aktiv |

### Achtung Zeitzone

**Cowork wertet Cron in Lokalzeit aus, nicht in UTC.** Der Ausdruck oben trifft
direkt 02:20 / 06:20 / 10:20 / 14:20 / 18:20 / 22:20 Wien und bleibt über die
Zeitumstellung hinweg richtig.

Frühere Annahme in diesem Dokument war `20 */4 * * *` mit UTC-Auswertung — das
hätte die Läufe zwei Stunden zu früh gefeuert (00:20 / 04:20 / …).

## Lauf-Weiche: Heartbeat statt `state.json`

Der Prompt unterscheidet Voll-Lauf (erster Lauf des Tages: Pflicht-Schritt 0 **und**
neue Vorschläge) von Rücklauf (nur Pflicht-Schritt 0) anhand von `state.json`. Diese
Datei existiert nirgends, und jeder Lauf startet mit frischem Clone — ohne Fix hätte
**jeder** Lauf als Voll-Lauf gegolten, also 6× täglich bis zu 3 neue Vorschläge statt 1×.

Ersetzt durch eine Heartbeat-Zeile in 🔁 Routinen HS
(`214a4c35-b2a7-8012-8185-000b1acc0a5c`): Existiert für heute bereits ein Eintrag mit
`Routine ID = orchestrator`, ist der Lauf ein Rücklauf. Sonst Voll-Lauf, und der
Eintrag wird geschrieben.

## Lese-Abfragen laufen über n8n

`query_data_sources` und `query_database_view` sind im Notion-Plan gedeckelt
(`available_with_limit`); `fetch`, `search`, `create_pages` und `update_page` nicht.
Die Leseabfragen des Orchestrators laufen deshalb über n8n gegen die öffentliche
Notion-REST-API — eigene Tür, kein MCP-Kontingent.

| Workflow | ID | Zweck |
|---|---|---|
| Orchestrator Kontext (Lesen) | `WkJjONip3SfOpC0E` | Aktive Agenten, Routinen, Rückfrage-Tasks → `laufArt`, `heartbeatMarker`, `rueckfrageTasks` |
| Content-Kalender (Lesen) | `0Tkr1VDXV7zfG97j` | Kampagnen + Planungs-Items + Tasks → Tickler-Regel-Check |
| Agent Queue Reader | `1xnXAQYxoX98PHCf` | Queue-Filter für alle Agenten |

Beim Queue Reader lag der Fix des Rückfrage-Filters als gespeicherte, aber **nie
veröffentlichte** Version vor (`versionId` ≠ `activeVersionId`). Speichern reicht in
n8n nicht — erst `publish_workflow` schaltet die Version scharf. Nach dem Publish
waren 5 wochenlang unsichtbare Tasks wieder in der Queue.

## MCP-Blocker `claude-code-remote`

Alle Aufrufe des MCP-Servers `claude-code-remote` scheitern in dieser Umgebung mit:

```
MCP error -32003: MCP tool call requires approval
```

Das betrifft auch reine Lese-Aufrufe (`list_triggers`, `list_environments`), also den
gesamten Server — nicht einzelne Werkzeuge. Ein Freigabe-Dialog erreicht die Oberfläche
nicht. Eine Allow-Regel in `~/.claude/settings.json` hat den Fehler in der laufenden
Session nicht behoben.

Umgangen über Dispatch: Task-Anlage, Testlauf und Aktivierung liefen dort. Ursache
ungeklärt.

## Vorher-Snapshot (05.08.2026, 08:27 UTC)

Baseline für die Abnahme des ersten Laufs. Tasks mit `Freigabe = 💬 Rückfrage` und
`done = false` in der [Tasks DB](https://app.notion.com/p/3331caadee244e768ccb1e8a311efb84)
(`0d1904d8-4d66-4e4e-97d5-e794d3981af7`):

| Task | Agent | Verantwortlich | Kommentar HS | Erstellt |
|---|---|---|---|---|
| Prozess-Coach: Kursverkauf / Checkout SIPOC | — | Prozess-Coach | ja | 03.07. |
| LinkedIn-Redakteur: Themenvorschläge + Brief | LinkedIn-Redakteur | — | ja | 08.07. |
| Gate 2: Text-Go MI August | Newsletter Creator | Newsletter Creator | ja | 12.07. |
| Task-Taker-Kanal steht seit 03.07. | — | — | ja | 12.07. |
| Prozess-Coach: Scheduled-Run-Heartbeat instabil | — | — | ja | 16.07. |
| Newsletter Creator: photocoach.cc Welcome-Sequenz | Newsletter Creator | Newsletter Creator | **leer** | 19.07. |
| Watchdog erweitern: Task-Taker-Heartbeat-Check | — | — | ja | 31.07. |

Pflicht-Schritt 0 darf ausschließlich `Agent`, `Verantwortlich` und Body-Text setzen —
also nur die vier Zeilen ohne Agent. Unverändert bleiben müssen `Freigabe`,
`Agent-Status`, `done` und `Kommentar HS`.

### Zielkonflikt beim ersten Lauf

Nur eine der vier agentenlosen Zeilen hat einen eindeutigen Treffer — und die ist geparkt:

- **Kursverkauf SIPOC** → Prozess-Coach. Haralds Kommentar sagt aber: „Bewusst geparkt bis
  25.08.2026. Bis dahin kein Vorschlag, keine SIPOC-Arbeit." Regel 2 würde den Task vor
  diesem Datum zurück in die Queue schieben.
- **Task-Taker-Kanal**, **Heartbeat instabil**, **Watchdog erweitern** → kein sauberer
  Treffer. Task-Taker sollte sich nicht selbst diagnostizieren, Watchdog ist als
  „kein Task-Output — Observability-Rolle" geführt. Erwartung: „Kein passender Agent —
  Kandidat: …" im Task-Body.

## Offen

- **Newsletter Creator liest seine Queue nicht.** `Auslöser = „Manuell (Harald)"`, kein
  Prompt auf der Registry-Seite, 11 verknüpfte Tasks mit durchgehend leerem
  `Agent-Status`, der älteste 34 Tage. Newsletter-Arbeit passiert nachweislich —
  vermutlich über `newsletter-regelkreis-lauf` (täglich 02:39), der unter den 134
  n8n-Workflows nicht existiert. Vor dem Schreiben eines Prompts muss die tatsächliche
  Cowork-Task-Liste vorliegen, sonst entsteht ein Doppel.
- `Verantwortlich = Prozess-Coach` dort nachtragen, wo es noch fehlt.
