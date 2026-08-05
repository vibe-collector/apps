# Scheduled Task `orchestrator` — Einrichtung

Stand: 05.08.2026. **Der Task ist noch NICHT angelegt** (Grund siehe „Blocker").

`PROMPT.md` enthält den vollständigen Seiteninhalt der Notion-Seite
[🧠 Orchestrator (COO)](https://app.notion.com/p/390a4c35b2a7818dbffde3156f190a73)
(ID `390a4c35-b2a7-818d-bffd-e3156f190a73`) — wortwörtlich, ungekürzt, inklusive
Lauf-Weiche. Er gehört als Task-Prompt eingefügt, nicht als Link: der Task hat zur
Laufzeit keinen garantierten Notion-Zugriff auf diese Seite.

## Konfiguration

| Feld | Wert |
|---|---|
| Name | `orchestrator` |
| Cron | `20 */4 * * *` |
| Modell | Opus |
| Session | eigene Session pro Lauf |
| Connector | Notion |
| Zeitplan initial | deaktiviert |

### Achtung Zeitzone

Cron wird in **UTC** ausgewertet. Wien liegt im Sommer auf CEST (UTC+2), damit
trifft `20 */4 * * *` genau die gewünschten Zeiten 02:20 / 06:20 / 10:20 / 14:20 /
18:20 / 22:20 Wien.

Ab Ende Oktober (CET, UTC+1) verschiebt derselbe Ausdruck die Läufe auf
01:20 / 05:20 / … Wien. Sollen die Uhrzeiten ganzjährig stehen, muss der Cron zur
Zeitumstellung auf `20 1,5,9,13,17,21 * * *` geändert werden — und im Frühjahr zurück.

## Blocker

Alle Aufrufe des MCP-Servers `claude-code-remote` scheitern mit:

```
MCP error -32003: MCP tool call requires approval
```

Das betrifft auch reine Lese-Aufrufe (`list_triggers`, `list_environments`), also den
gesamten Server — nicht einzelne Werkzeuge. Ein Freigabe-Dialog erreicht die Oberfläche
nicht. Eine Allow-Regel in `~/.claude/settings.json` (siehe unten) hat den Fehler in der
laufenden Session nicht behoben; ob sie nach einem Neustart greift, ist offen.

```json
{ "permissions": { "allow": ["mcp__claude-code-remote", "mcp__bf7c680d-5fdc-5ef4-b4a0-abadb619bf0a"] } }
```

## Offener Punkt: `state.json`

Die Lauf-Weiche unterscheidet Voll-Lauf und Rücklauf daran, ob `state.json` bereits einen
Eintrag mit dem heutigen Datum hat. Diese Datei existiert nirgends, und jeder Lauf startet
mit frischem Clone.

Konsequenz ohne Fix: **jeder** Lauf gilt als erster Lauf des Tages, also 6× täglich
Voll-Lauf mit bis zu 3 neuen Vorschlägen statt 1× — genau die Vervielfachung, die die
Weiche verhindern soll. Vor dem Scharfschalten zu klären.

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
