# Image-Tagger-Prozess (Drive → Notion Files DB)

Stand: 31.08.2026

## Grundidee

Jeder der drei Kanäle hat auf Google Drive unter `Meine Ablage/n8n` einen
Quellordner mit drei Unterordnern. Der Ordner, in dem eine Datei liegt, **ist**
ihr Status — es gibt keinen zusätzlichen Merker.

```
Quellordner  ──Tagger sagt OK──►  ready  ──Picker nimmt es──►  used
     │                                                          (im Web freigegeben,
     └────Tagger sagt nein──►  ausgeschieden                      Container/Posting)
```

Das Auswahlkriterium des Taggers ist ausschließlich: **die Datei liegt im
Quellordner.** Kein Namensfilter, keine Dateiendung, kein Datenbank-Lookup.
Wer nicht getaggt werden kann, wird aus dem Pool entfernt — nicht ignoriert.
Damit kann keine Datei den Tagger mehr blockieren.

## Die drei Kanäle

| Kanal | Quellordner | ready | ausgeschieden | Workflow |
|---|---|---|---|---|
| Mindful Images | `insta Workflow` | `ready` | `ausgeschieden mindful images` | `yWgIPSRkLjV8oEaI` |
| Photocoach | `Insta Workflow Photocoach` | `ready photocoach` | `ausgeschieden photocoach` | `wDhrqL46WBCOHWGw` |
| GettingADDDone | `insta gettingadddone` | `ready Gettingadddone` | `ausgeschieden gettingadddone` | `9dKbe180TOLs9o0Z` |

Drive-Ordner-IDs:

| Ordner | ID |
|---|---|
| insta Workflow | `1VwyYpJZuaTPC4jYIbIDUwy3VCFvL55Vg` |
| ready | `1nR9fVUGeDhO9tZlqSPwzjHQI5uHInsDZ` |
| used | `1bsXqZCAotF7kv5nct1l3HaJB7RgXA06j` |
| ausgeschieden mindful images | `1p4c4L2i1cwt97e7wAkQJz7iWbX3VUXV_` |
| Insta Workflow Photocoach | `1fIE6uwsomYzfvGG24-xjn264zEyNm710` |
| ready photocoach | `1lZVDvWETL2oLhZtbTLMv-Jd6oG6JRy6y` |
| used photocoach | `1Z3oqQ9c7FLihVYwdf_kfum0wuESBuB2G` |
| ausgeschieden photocoach | `1Oln6Am9WYtRO7auKh1aFfeWLJ4bmvWCj` |
| insta gettingadddone | `1Pw4Kc9iRuFuT9bBo1AqjjQ1xELQRt5JP` |
| ready Gettingadddone | `1NlX7EJYN0u-F4Qjvz_C337cg6qd7g16o` |
| used gettingadddone | `1mhLY9P69y-7f05kuzltl0CXmH6MAddKA` |
| ausgeschieden gettingadddone | `1NSJl0B5YvRVXzpkeqWBCaR7eUWDba3zs` |

## Ablauf im Workflow

```
Cron Tagger ─┐
Manual Test ─┴─► Pick 1 PNG ──► Datei-Metadaten ──► Format pruefen ──► Geeignet?
                                                                        │
                                    ┌───────────────────────────────────┴──── nein
                                    │ ja                                       │
                                    ▼                                          ▼
                            Download Image                        Move to Ausgeschieden
                                    ▼                                          ▲
                              Vision Tag ───────────────────────────────────┐  │
                                    ▼                                       │  │
                              Pack Result ──────────────────────────────────┤  │
                                    ▼                                       │  │
                             Move to Ready ──────────────────────────────── ┤  │
                                    ▼                                       │  │
                         Create a database page ────────────────────────────┴──┘
                                                        (jeweils Fehlerausgang)
```

**Pick 1 PNG** (Google Drive, fileFolder/search, Query-Modus, limit 1):

```
'<QUELLORDNER-ID>' in parents and trashed = false
and mimeType != 'application/vnd.google-apps.folder'
```

Der Ausschluss von `application/vnd.google-apps.folder` ist nötig, weil sonst
die Unterordner `ready`, `used` und `ausgeschieden` selbst als Treffer
zurückkämen.

**Datei-Metadaten** (HTTP Request, Drive-Credential als predefinedCredentialType):

```
GET https://www.googleapis.com/drive/v3/files/{{ $json.id }}
    ?fields=id,name,mimeType,size,imageMediaMetadata(width,height,rotation)
    &supportsAllDrives=true
```

Die Bildabmessungen liefert nur dieser Aufruf — der n8n-Drive-Node gibt
`imageMediaMetadata` nicht heraus.

**Format pruefen** (Code-Node) — Ausschlusskriterien in dieser Reihenfolge:

| Prüfung | Grenze | Ausschluss-Grund |
|---|---|---|
| MIME-Typ | nur `image/jpeg`, `image/jpg`, `image/png`, `image/webp` | Videos, PDFs, Dokumente, Markdown |
| Abmessungen lesbar | width und height > 0 | defekte oder untypisierte Datei |
| max. Seitenverhältnis | 1.91 : 1 | Panorama / zu breit |
| min. Seitenverhältnis | 0.80 : 1 (= 4:5) | zu hoch für Instagram |
| Mindestauflösung | kurze Seite ≥ 600 px | Logos, Icons, Miniaturen |

Die Ratio-Grenzen sind Instagrams zulässiger Bereich. `rotation` 90/270 wird
berücksichtigt, sonst würde ein hochkant aufgenommenes Foto falsch bewertet.

Ausgabe: `{ fileId, filename, mimeType, width, height, ratio, ok, grund }`.
Der Grund steht im Execution-Log der n8n-Ausführung.

**Fehlerpfade.** Jeder Knoten ab `Datei-Metadaten` hat
`onError: continueErrorOutput` und seinen Fehlerausgang auf
`Move to Ausgeschieden` verdrahtet. Egal wo es klemmt — Download, Vision-API,
kaputtes JSON in `Pack Result`, Notion nicht erreichbar — die Datei landet im
Ausschuss und blockiert den Quellordner nicht.

Die Reihenfolge `Move to Ready` **vor** `Create a database page` ist bewusst:
Scheitert der Notion-Eintrag, wandert die Datei aus `ready` heraus in den
Ausschuss. Es entsteht nie eine Datei in `ready` ohne DB-Zeile und nie eine
DB-Zeile ohne Datei.

## Warum das gebaut wurde

Vorher filterte `Pick 1 PNG` über den Dateinamen (`insta` bzw. `Firefly`).
Alles, was anders hieß, war für den Tagger unsichtbar und blieb liegen; alles,
was passend hieß, aber nicht verarbeitbar war, wurde bei jedem Lauf erneut
gezogen und ließ den Workflow scheitern — eine Sackgasse, die den Kanal
tagelang stilllegen konnte.

## Verifiziert am 31.08.2026

| Fall | Datei | Ergebnis |
|---|---|---|
| Dokument | `posts-batch-01.md` (text/markdown) | abgewiesen → `ausgeschieden gettingadddone` |
| Panorama / Logo | `photocoach_logo_auf_petrol_300.png`, 300×51, 5.88:1 | abgewiesen → `ausgeschieden photocoach` |
| Unterordner | `ready` / `used` / `ausgeschieden` | korrekt nicht als Datei gezogen |
| leerer Quellordner | Mindful Images | 0 Items, sauberer Leerlauf |

Der Gutfall (gültiges Bild wird getaggt und nach `ready` verschoben) war zum
Umbauzeitpunkt nicht prüfbar, weil in keinem Quellordner ein gültiges Bild lag.
Er wird beim nächsten echten Lauf verifiziert.

## Offen

- **Kontrollagent**: Abgleich Drive-Bestand gegen Notion Files DB je Kanal —
  Datei ohne DB-Zeile, DB-Zeile ohne Datei, DB-Zeile ohne Drive-ID.
- **323 ID-lose Zeilen** vom 19.08.2026 (Photocoach-Portfolio-Kuratierung mit
  Lehrpunkt-Annotationen) — nicht löschen, Herkunft geklärt, Zielzustand offen.
- **Temporärer Filter** `ID is_not_empty` im Format-Korrektor `86jeHSmpFo0fhk2t`
  entfernen, sobald der Datenbestand sauber ist.
- **Ausschuss-Grund sichtbar machen**: aktuell nur im Execution-Log. Optional
  könnte der Kontrollagent den Inhalt der `ausgeschieden`-Ordner melden.
