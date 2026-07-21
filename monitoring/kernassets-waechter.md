# Kernassets-Wächter (n8n)

Täglicher Verfügbarkeits-Check der Kernassets, gebaut am 20.07.2026.

**n8n-Workflow (aktiv):** `Kernassets-Wächter v2: Webseiten & Etsy (täglich 07:00)`
https://n8n.haraldschwack.at/workflow/G4jfSndI2rjQjd7P

> Update 21.07.2026: v1 (`idKwb6eMTGqmGV1H`) hatte einen Fehlalarm-Bug —
> der Inhalts-Check las `r.body`, aber bei `responseFormat: 'text'` liegt
> der Seiteninhalt in `r.data`. v1 ist deaktiviert (löschbar), v2 mit
> korrigiertem Check ist aktiv. Der Code unten ist der v2-Stand.

## Was wird geprüft (täglich 07:00 Wien)

| Asset | URL | Zusatz-Check |
|---|---|---|
| haraldschwack.at | https://haraldschwack.at/ | Seite enthält "Schwack" |
| photocoach.cc | https://photocoach.cc/ | Seite enthält "hotocoach" |
| gettingadddone.com | https://gettingadddone.com/ | Seite enthält "ADD" |
| Etsy-Shop | https://www.etsy.com/shop/GettingADDDone | HTTP 403 gilt als Bot-Schutz → OK; 404/5xx = Alarm |

Ein Asset gilt als OK bei HTTP 200–399 **und** wenn der erwartete Inhalt in der
Antwort vorkommt (fängt leere/kaputte Seiten mit Status 200 ab). Beim Etsy-Shop
wird HTTP 403 toleriert, weil Etsy Rechenzentrums-IPs oft per Bot-Schutz
blockt — ein gelöschter/geschlossener Shop liefert dagegen 404.

## Ablauf im Workflow

1. **Schedule Trigger** — täglich 07:00 (Instanz-Zeitzone Wien)
2. **Data Table sicherstellen** — legt `core_assets_status` beim ersten Lauf an (`createIfNotExists`)
3. **Kernassets definieren** (Code-Node, dort URLs/Keywords pflegen)
4. **HTTP-Check** — GET mit Browser-User-Agent, 20 s Timeout, Redirects folgen, `neverError`
5. **Bewertung** (Code-Node, Logik siehe oben)
6. **Historie loggen** — jede Prüfung als Zeile in Data Table `core_assets_status`
   (`checked_at`, `site`, `url`, `status`, `ok`, `note`) → beantwortet "seit wann ist X down?"
7. **Alarm** — nur bei mindestens einem Problem: Telegram an Harald (Chat 622619977),
   gleicher Kanal wie der Posting-Wächter. Kein Alarm = alles OK.

## Wartung

- **URL/Asset ändern:** Code-Node "Definiere Kernassets" im Workflow editieren.
- **Historie ansehen:** n8n → Data Tables → `core_assets_status`.
- **Manuell testen:** Workflow in n8n öffnen → "Execute workflow".
- Der Hilfs-Workflow `SETUP: Data Table core_assets_status` (6eEIpJVB8B9JXDYS)
  ist überflüssig und kann gelöscht werden.

## Warum n8n und keine Claude-Routine?

Die Claude-Cloud-Umgebung hat eine restriktive Netzwerk-Policy (alle externen
Domains werden vom Proxy mit 403 geblockt) — HTTP-Checks sind von dort aus
nicht möglich. n8n hat freien Internetzugang und läuft ohnehin täglich.

## Workflow-Quellcode (n8n Workflow SDK)

Source of truth für spätere Änderungen per MCP (`create_workflow_from_code`):

```javascript
import { workflow, node, trigger, expr } from '@n8n/workflow-sdk';

const dailyTrigger = trigger({
  type: 'n8n-nodes-base.scheduleTrigger',
  version: 1.3,
  config: {
    name: 'Täglich 07:00 Wien',
    parameters: {
      rule: {
        interval: [{ field: 'days', triggerAtHour: 7, triggerAtMinute: 0 }]
      }
    }
  }
});

const ensureTable = node({
  type: 'n8n-nodes-base.dataTable',
  version: 1.1,
  config: {
    name: 'Stelle Status-Tabelle sicher',
    parameters: {
      resource: 'table',
      operation: 'create',
      tableName: 'core_assets_status',
      columns: {
        column: [
          { name: 'checked_at', type: 'date' },
          { name: 'site', type: 'string' },
          { name: 'url', type: 'string' },
          { name: 'status', type: 'number' },
          { name: 'ok', type: 'boolean' },
          { name: 'note', type: 'string' }
        ]
      },
      options: { createIfNotExists: true }
    }
  }
});

const defineAssets = node({
  type: 'n8n-nodes-base.code',
  version: 2,
  config: {
    name: 'Definiere Kernassets',
    parameters: {
      mode: 'runOnceForAllItems',
      language: 'javaScript',
      jsCode: `return [
  { json: { name: 'haraldschwack.at', url: 'https://haraldschwack.at/', keyword: 'Schwack', acceptStatus: [] } },
  { json: { name: 'photocoach.cc', url: 'https://photocoach.cc/', keyword: 'hotocoach', acceptStatus: [] } },
  { json: { name: 'gettingadddone.com', url: 'https://gettingadddone.com/', keyword: 'ADD', acceptStatus: [] } },
  { json: { name: 'Etsy-Shop (GettingADDDone)', url: 'https://www.etsy.com/shop/GettingADDDone', keyword: 'GettingADDDone', acceptStatus: [403] } }
];`
    }
  }
});

const checkSite = node({
  type: 'n8n-nodes-base.httpRequest',
  version: 4.4,
  config: {
    name: 'Prüfe Webseite',
    onError: 'continueRegularOutput',
    parameters: {
      method: 'GET',
      url: expr('{{ $json.url }}'),
      sendHeaders: true,
      specifyHeaders: 'keypair',
      headerParameters: {
        parameters: [
          { name: 'User-Agent', value: 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36' },
          { name: 'Accept', value: 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8' },
          { name: 'Accept-Language', value: 'de-AT,de;q=0.9,en;q=0.8' }
        ]
      },
      options: {
        timeout: 20000,
        redirect: { redirect: { followRedirects: true, maxRedirects: 10 } },
        response: { response: { fullResponse: true, neverError: true, responseFormat: 'text' } }
      }
    }
  }
});

const evaluateResults = node({
  type: 'n8n-nodes-base.code',
  version: 2,
  config: {
    name: 'Bewerte Ergebnisse',
    parameters: {
      mode: 'runOnceForAllItems',
      language: 'javaScript',
      jsCode: `const sites = $('Definiere Kernassets').all().map(i => i.json);
const responses = $input.all();
const out = [];
for (let i = 0; i < sites.length; i++) {
  const site = sites[i];
  const r = (responses[i] && responses[i].json) || {};
  const status = (typeof r.statusCode === 'number') ? r.statusCode : null;
  let ok = false;
  let note = '';
  if (r.error) {
    note = 'Fehler: ' + String(r.error.message || r.error).slice(0, 180);
  } else if (status !== null) {
    ok = status >= 200 && status < 400;
    if (ok && site.keyword) {
      const raw = (typeof r.data === 'string' && r.data) ? r.data : ((typeof r.body === 'string' && r.body) ? r.body : JSON.stringify(r.data ?? r.body ?? ''));
      if (!raw.toLowerCase().includes(String(site.keyword).toLowerCase())) {
        ok = false;
        note = 'HTTP ' + status + ', aber erwarteter Inhalt "' + site.keyword + '" fehlt';
      }
    }
    if (!ok && !note) {
      if (Array.isArray(site.acceptStatus) && site.acceptStatus.includes(status)) {
        ok = true;
        note = 'HTTP ' + status + ' (vermutlich Bot-Schutz, gilt als erreichbar)';
      } else {
        note = 'HTTP ' + status;
      }
    }
  } else {
    note = 'Keine Antwort (Timeout oder DNS-Problem)';
  }
  out.push({ json: {
    checked_at: new Date().toISOString(),
    site: site.name,
    url: site.url,
    status: status === null ? 0 : status,
    ok: ok,
    note: note
  }});
}
return out;`
    }
  }
});

const logStatus = node({
  type: 'n8n-nodes-base.dataTable',
  version: 1.1,
  config: {
    name: 'Logge Status-Historie',
    onError: 'continueRegularOutput',
    parameters: {
      resource: 'row',
      operation: 'insert',
      dataTableId: { __rl: true, mode: 'name', value: expr("{{ 'core_assets_status' }}") },
      columns: {
        mappingMode: 'defineBelow',
        value: {
          checked_at: expr('{{ $json.checked_at }}'),
          site: expr('{{ $json.site }}'),
          url: expr('{{ $json.url }}'),
          status: expr('{{ $json.status }}'),
          ok: expr('{{ $json.ok }}'),
          note: expr('{{ $json.note }}')
        },
        schema: [
          { id: 'checked_at', displayName: 'checked_at', required: false, defaultMatch: false, display: true, type: 'date', canBeUsedToMatch: true },
          { id: 'site', displayName: 'site', required: false, defaultMatch: false, display: true, type: 'string', canBeUsedToMatch: true },
          { id: 'url', displayName: 'url', required: false, defaultMatch: false, display: true, type: 'string', canBeUsedToMatch: true },
          { id: 'status', displayName: 'status', required: false, defaultMatch: false, display: true, type: 'number', canBeUsedToMatch: true },
          { id: 'ok', displayName: 'ok', required: false, defaultMatch: false, display: true, type: 'boolean', canBeUsedToMatch: true },
          { id: 'note', displayName: 'note', required: false, defaultMatch: false, display: true, type: 'string', canBeUsedToMatch: true }
        ]
      }
    }
  }
});

const buildAlarm = node({
  type: 'n8n-nodes-base.code',
  version: 2,
  config: {
    name: 'Baue Alarm-Nachricht',
    parameters: {
      mode: 'runOnceForAllItems',
      language: 'javaScript',
      jsCode: `const results = $('Bewerte Ergebnisse').all().map(i => i.json);
const problems = results.filter(r => !r.ok);
if (problems.length === 0) {
  return [];
}
const lines = results.map(r => {
  const icon = r.ok ? '✅' : '❌';
  const statusText = r.status ? 'HTTP ' + r.status : 'KEINE ANTWORT';
  const noteText = r.note && !r.ok ? '\\n   ' + r.note : '';
  return icon + ' ' + r.site + ' — ' + statusText + noteText;
});
const msg = '🚨 <b>Kernassets-Check</b>: ' + problems.length + (problems.length === 1 ? ' Problem' : ' Probleme') + ' gefunden!\\n\\n' + lines.join('\\n') + '\\n\\n📊 Historie: n8n → Data Tables → core_assets_status';
return [{ json: { message: msg, problemCount: problems.length } }];`
    }
  }
});

const sendAlarm = node({
  type: 'n8n-nodes-base.telegram',
  version: 1.2,
  config: {
    name: 'Telegram-Alarm an Harald',
    parameters: {
      resource: 'message',
      operation: 'sendMessage',
      chatId: '622619977',
      text: expr('{{ $json.message }}'),
      additionalFields: {
        parse_mode: 'HTML',
        appendAttribution: false,
        disable_web_page_preview: true
      }
    },
    credentials: { telegramApi: { id: 'V2G16GFCPmCrgr4x', name: 'Telegram account' } }
  }
});

export default workflow('core-assets-check', 'Kernassets-Wächter: Webseiten & Etsy (täglich 07:00)')
  .add(dailyTrigger)
  .to(ensureTable)
  .to(defineAssets)
  .to(checkSite)
  .to(evaluateResults)
  .to(logStatus)
  .to(buildAlarm)
  .to(sendAlarm);
```
