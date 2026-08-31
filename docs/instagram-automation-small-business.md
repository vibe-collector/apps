# Instagram automatisiert bespielen — ohne Notion

Arbeitspapier, 31.08.2026 — Gesprächsnotiz mit Dorian.
Vier Ausbaustufen, ein Startpfad mit fünf n8n-Workflows und eine Bewertung der
Datenschicht-Alternativen.

Interaktive Fassung: https://claude.ai/code/artifact/d91045ba-9195-45a6-8126-17a11cfa24a1

---

## Vorab: die Datenstruktur ist nicht der Engpass

Für automatisiertes Instagram-Posting braucht es genau fünf Felder:

- **Bild** — als öffentlich erreichbare URL, nicht als Datei-Anhang
- **Caption** — Text inkl. Hashtags
- **Zeitpunkt** — Datum + Uhrzeit, eine Zeitzone, festgeschrieben
- **Kanal** — z. B. Mindful Images / Photocoach / GettingADDdone
- **Status** — geplant → gepostet → fehlgeschlagen

Jedes hier genannte Tool kann das. Die Entscheidung fällt woanders: wie angenehm ist
das Befüllen, wie zuverlässig der Abruf — und **wo liegen die Bilder**.

## Die harte Randbedingung: Instagram nimmt keine Dateien, nur URLs

Die Content Publishing API arbeitet zweistufig (`/media` → `/media_publish`). Meta lädt
das Bild **selbst** von der angegebenen URL. Daraus folgt:

- **Google Drive scheidet als Bildquelle aus.** Freigabelinks liefern HTML statt JPEG;
  der `uc?export=download`-Trick ist ratenbegrenzt und bricht bei größeren Dateien an
  der Virenscan-Zwischenseite. Drive bleibt gut als Archiv, nicht als Auslieferungsort.
  Tauglich: Publisher-Medienbibliothek, Attachment-Feld (Airtable/Notion/Baserow),
  echter Bucket (Cloudinary, Supabase Storage, S3/R2).
  **Eine Ausnahme** — Grundlage von Szenario 0: ein Workflow, der die Datei über den
  Drive-Node authentifiziert *selbst herunterlädt* und Meta anschließend unter einer
  eigenen URL bereitstellt. Meta sieht nie einen Drive-Link, nur das fertige JPEG.
- Professional-Account (Business/Creator) + Meta-App mit `instagram_content_publish`.
  Über Instagram Login inzwischen auch ohne verknüpfte Facebook-Seite — Meta ändert das
  laufend, im Zweifel aktuelle Doku prüfen.
- JPEG, max. 8 MB, Seitenverhältnis 4:5 bis 1.91:1. PNG wird abgelehnt.
- Video braucht Wartezeit vor dem Publish (~45 s ist die richtige Größenordnung).
- 50 Posts / 24 h — für ein Small Business nie das Limit.
- **Token laufen nach 60 Tagen ab.** Klassische Wartungsfalle: alles läuft ein Quartal,
  dann steht es still, ohne sichtbaren Fehler. Wer selbst baut, braucht Token-Refresh
  *und* Alarmierung.

---

## Szenario 0 — Startoption: geteilter Betrieb, null neue Tools

`Google Drive (Bilder)` → `Google Sheet (Plan)` → `Claude (Bedienung)` → `ein n8n-Workflow bei Harald` → `Instagram`

Dorian bekommt keine Automatisierungsplattform, keinen Publisher, kein Abo. Er bekommt
einen Ordner, eine Tabelle und ein Gespräch. Das Veröffentlichen läuft als **ein einziger
Workflow** auf Haralds bestehender Instanz mit. Bevor man für einen Workflow ein Make
einführt, lässt man den Workflow eben mitlaufen.

**Wie der Drive-Widerspruch verschwindet:** Meta bekommt den Drive-Link nie zu sehen. Der
Workflow lädt die Datei über den Google-Drive-Node authentifiziert herunter und stellt sie
für die Sekunden der Container-Erstellung unter einer eigenen Webhook-URL bereit. Kein
Bucket, kein Cloudinary, kein zusätzliches Konto — ein Node mehr.

**Die Tabelle, sieben Spalten:**
`Datum · Uhrzeit · Bild-Dateiname · Caption · Kanal · Status · Fehlertext`
Die letzten beiden schreibt der Workflow zurück. Damit sieht Dorian in derselben Tabelle,
ob gepostet wurde — und wenn nicht, warum. Das ersetzt das Fehler-Dashboard eines Publishers.

**Claude ist die Oberfläche.** Der Teil, der sonst Airtable-Views braucht, wird ein Satz:
*„Ich hab vier Bilder vom Wochenende hochgeladen — plan mir daraus die nächste Woche."*
Claude sieht Ordner und Sheet, wählt die Bilder, schreibt die Captions, füllt die Zeilen,
legt die Termine. Dorian pflegt keine Tabelle, er redet über seinen Content.

### Was Dorian selbst erledigen muss

Alles andere macht Harald. Diese sieben Schritte hängen an Dorians Konto:

1. **Instagram auf Business oder Creator umstellen** — Profil → Konto → Kontotyp. Ein
   privates Konto kann die API grundsätzlich nicht bedienen.
2. **Eine eigene Meta-App anlegen** — auf `developers.facebook.com`, Produkt „Instagram"
   hinzufügen. Bewusst *seine* App, nicht Haralds: dann braucht es weder App-Review noch
   Rollen-Jonglage, weil sein Konto im Entwicklungsmodus bereits berechtigt ist.
3. **Kurzlebiges Token erzeugen** — im Graph API Explorer, mit `instagram_basic` und
   `instagram_content_publish`.
4. **Gegen ein 60-Tage-Token tauschen** — ein Aufruf:
   `oauth/access_token?grant_type=fb_exchange_token`. Dazu die Instagram-User-ID abfragen,
   die der Workflow als Adresse braucht.
5. **Token und User-ID übergeben** — über einen Passwort-Manager oder einen ablaufenden
   Link. Nicht per Chat, nicht per Mail: das ist ein Vollzugriff auf sein Konto.
6. **Drive-Ordner und Sheet freigeben** — für den Google-Zugang, mit dem n8n arbeitet.
   Bearbeitungsrecht auf das Sheet, damit der Status zurückgeschrieben werden kann.
7. **Alle 60 Tage Token erneuern** — oder den Refresh einmalig in den Workflow bauen.
   Empfehlung: einbauen, sonst steht es im November still und keiner weiß warum.

> **Die Kehrseite, ehrlich benannt.** Dorian hängt an Haralds Instanz. Fällt sie aus,
> postet niemand. Ist Harald zwei Wochen weg, ruft Dorian trotzdem an. Und Harald verwahrt
> fremde Zugangsdaten. Als Anschub völlig in Ordnung — nur ist es **eine Zusage, kein
> Werkzeug**. Ein Satz dazu, wer bei einem Fehler was tut, erspart später ein unangenehmes
> Gespräch.

**Dafür:** null neue Abos und Logins für Dorian · Bedienung ist ein Gespräch statt
Tabellenpflege · läuft auf einem Workflow, der seit Monaten trägt · in ~2 Stunden
eingerichtet, Tokens inklusive · jederzeit nach Stufe 1 oder 2 erweiterbar (Tabelle
wandert als CSV).

**Dagegen:** Harald ist der Betreiber und wird angerufen · fremde Zugangsdaten in fremder
Hand, gehört sauber geregelt · Sheet ohne Bildvorschau, geplant wird über Dateinamen · ab
etwa vier Posts pro Woche wird es eng · Dorian lernt das System nicht kennen, er nutzt es nur.

| Neue Tools | Kosten | Aufbau | Wartung |
|---|---|---|---|
| 0 (Drive + Sheet + Claude) | 0 € für Dorian | ~2 Std. inkl. Tokens | bei Harald, ein Workflow mehr |

**Fazit:** Die richtige erste Bewegung. Kostet nichts, beweist in zwei Wochen, ob Dorian
überhaupt regelmäßig Content liefert — und erst diese Antwort rechtfertigt jedes Tool danach.

## Szenario 1 — Minimalstufe: Google Sheets + ein Publisher

`Google Sheets` → `Publer / Metricool` → `Instagram`

Redaktionsplan im Sheet, Bilder in der Medienbibliothek des Publishers, einmal monatlich
CSV-Import. Ab da postet das Tool eigenständig (Feed, Reels, Stories, Karussell).
Kein API-Zugriff, keine Tokens, keine Automatisierungsplattform. Wer auch den Import
loswerden will, hängt Make (Free) dazwischen: neue Zeile → Publer-API.

**Dafür:** an einem Nachmittag fertig · Publisher übernimmt Medien-Hosting, Formatprüfung
und Token-Pflege · Fehler sofort in der UI sichtbar · Analytics inklusive · ohne
IT-Kenntnisse bedienbar.

**Dagegen:** keine Bildvorschau, man plant blind · keine Verknüpfungen (Post kennt weder
Thema noch Kunde noch Kampagne) · ab ~200 Zeilen unübersichtlich · mobil kaum bedienbar ·
KI-Unterstützung nur per Copy-Paste aus dem Chat.

| Tools | Kosten | Aufbau | Wartung |
|---|---|---|---|
| 2 | 0–20 €/Monat | ~3 Std. | gering |

**Fazit:** Die ehrlichste Empfehlung für jemanden, der noch nichts hat. Kostet fast
nichts, geht nie kaputt — und wächst nicht mit.

## Szenario 2 — Mittelweg: Airtable als Content-DB, Make als Motor

`Airtable Interface` → `Airtable (Daten + Bilder)` → `Make / n8n Cloud (+ KI)` → `Instagram Graph API`

Airtables Attachment-Feld ist Datenbank-Spalte **und** Bild-Hosting zugleich — das
Drive-Problem verschwindet, ohne dass ein Bucket dazukommt. Dazu Views: dieselben Daten
als Kalender, Kanban und Galerie, plus Formular/Interface fürs Handy.

Hier wird KI erstmals Teil des Systems statt eines Nebengesprächs: ein Claude- oder
OpenAI-Modul schlägt Captions vor, leitet Hashtags aus dem Bildinhalt ab, liefert
Hook-Varianten. Freigabe ist ein Häkchen, kein Copy-Paste.

**Dafür:** Bild und Metadaten in einer Zeile · Kalender-/Kanban-/Galerie-Ansicht auf
demselben Datensatz · native Connectoren in Make, Zapier, n8n · Relationen Post ↔ Thema ↔
Kunde ↔ Kampagne · KI-Vorschläge landen direkt im Datensatz.

**Dagegen:** Free-Tarif eng (1.000 Records, 1 GB Anhänge pro Base) · Team-Tarif ~20 $
pro Nutzer/Monat · Attachment-URLs kurzlebig (für Automation ok, zum Verlinken nicht) ·
US-Hosting · ohne Publisher pflegst du die Meta-Tokens selbst.

| Tools | Kosten | Aufbau | Wartung |
|---|---|---|---|
| 2–3 | 0–35 €/Monat | 2–3 Tage | mittel (Token alle 60 Tage) |

**Fazit:** Für ein Small Business, das ein Jahr durchhalten soll, der richtige Punkt.
Genug Struktur für echte Redaktionsarbeit, kein eigener Server.

## Szenario 3 — Vollausbau: selbst gehostetes n8n

`Baserow / Notion` → `n8n (eigener Server)` → `MinIO / R2` → `Claude / OpenAI` → `Instagram · Stories · Reels`

Hetzner-Server für 4–6 €, n8n im Docker-Compose, Postgres daneben, Caddy davor. Grenze
ist ab da nicht mehr das Tool, sondern die eigene Zeit: Bildzuschnitt, Wasserzeichen,
DM-Auswertung, Community-Einsendungen, Fehler-Alarmierung nach Telegram.

**Das ist bei Harald keine Theorie, sondern der Ist-Zustand** — 144 Workflows, darunter
`instapost Photocoach Execute` (Container bauen, 45 s warten, publishen),
`instastory Execute (MI + PC + GAD)` (abendliche Story über drei Kanäle) und der
Community-Sammler (Instagram-DMs → Drive → Notion → Telegram).

Datenschicht ist hier frei wählbar:

- **n8n Data Tables** — eingebaut, null Zusatztool. Reicht, wenn kein Mensch reinschauen
  muss (bei Harald z. B. `core_assets_status`).
- **Baserow / NocoDB self-hosted** — Airtable-Ersatz mit UI, ohne Record-Limit, ohne
  US-Hosting.
- **Postgres direkt** mit Grist oder Directus als Oberfläche — maximal robust, braucht
  Entwicklerdenken.
- **Notion bleibt Frontend**, n8n ist der Motor. Aktueller Weg — und die Query-Proxy-
  Workflows zeigen: der Flaschenhals ist die Notion-API, nicht n8n.

**Dafür:** kein Operations-Limit, keine Preisstaffel pro Nutzer · Bildbearbeitung und
Branding im selben Lauf · Daten und Bilder auf eigener Infrastruktur (EU) · beliebige
LLMs, auch lokal via Ollama · Workflows versioniert und testbar.

**Dagegen:** Updates, Backups, Restore-Test dauerhaft in Eigenverantwortung · stiller
Ausfall bleibt ohne Monitoring unbemerkt · bei 144 Workflows ist niemand außer Harald
auskunftsfähig · Meta-Token, Berechtigungen, App-Review alles selbst · für einen Betrieb
ohne IT-Affinität die falsche Stufe.

| Tools | Kosten | Aufbau | Wartung |
|---|---|---|---|
| 4–6 + Server | 10–30 €/Monat | 1–2 Wochen | hoch, dauerhaft |

**Fazit:** Richtig genau dann, wenn jemand im Haus sie betreiben *will*. Für Dorian kein
Startpunkt, sondern ein Ziel — falls überhaupt.

---

## Startpfad — n8n ausprobieren, ohne gleich Stufe 3 zu bauen

Der Sprung von Stufe 1 auf Stufe 3 wirkt größer, als er ist. Man kann n8n laufen lassen,
ohne einen Server zu betreiben — und später umziehen, ohne etwas neu zu bauen: Workflows
exportieren als JSON, Daten als CSV.

### Drei Wege, n8n laufen zu lassen

| Weg | Kosten | Wartung | Wofür |
|---|---|---|---|
| **n8n Cloud Starter** | ~24 €/Monat | keine | Die ersten drei Monate. Sofort startklar, nichts steht still, weil ein Update schiefging. |
| **Hetzner CX22 + Docker** (Community Edition) | 4–6 €/Monat | Updates, Backups, Restore-Test | Ab dem Moment, wo es trägt. Unbegrenzte Executions — der eigentliche Kostenvorteil, nicht der Serverpreis. |
| **Lokal per Docker** | 0 € | Rechner muss laufen | Nur zum Lernen. Für etwas, das um 09:00 posten soll, untauglich. |

Activepieces und Windmill sind quelloffene Alternativen mit ähnlichem Baukasten. Einen
Grund zu wechseln gibt es aber nicht: self-hosted ist n8n bereits die günstigste Variante
seiner selbst.

### Fünf Workflows, in dieser Reihenfolge

1. **Der Publisher** — Zeitplan 09:00, holt die Zeile mit Status `geplant` und heutigem
   Datum, baut den Container, wartet, veröffentlicht, schreibt den Status zurück. Der
   ganze Kern, sechs Nodes.
   `Schedule → DB lesen → POST /media → Wait 45s → POST /media_publish → DB schreiben`
2. **Der Wächter** — Error-Trigger über alle Workflows, Meldung nach Telegram. Der
   wichtigste Workflow überhaupt — und der, den fast alle zuletzt bauen.
   `Error Trigger → Telegram`
3. **Der Ideen-Eingang** — Telegram-Bot: Foto plus Satz hinschicken, landet als Zeile mit
   Status `Idee`. Löst das Handy-Problem, das jede Tabelle hat.
   `Telegram Trigger → Bild in Bucket → DB anlegen`
4. **Der Caption-Vorschlag** — Zeilen ohne Text: Bild an ein Vision-Modell, drei Varianten
   zurück in die DB, Status `zu prüfen`. Nie direkt auf `geplant` — der Mensch gibt frei.
   `Schedule → DB lesen → LLM → DB schreiben`
5. **Der Story-Nachzieher** — abends dasselbe Bild zusätzlich als Story. Zwei Nodes mehr,
   spürbar mehr Reichweite.
   `Schedule 20:00 → heutigen Post lesen → media_type=STORIES → publish`

> **Die Reihenfolge ist der halbe Rat.** 01 und 02 zuerst, dann vier Wochen nichts. Erst
> wenn fehlerfrei gepostet wurde, kommt 03 dazu. Wer mit 04 beginnt, hat eine hübsche
> Demo und keinen Betrieb.

### Der günstige Nachbau, Baustein für Baustein

Dorian braucht dieselbe Architektur, nicht dieselben Rechnungen.

| Baustein | Bei Harald heute | Günstig für Dorian | Kosten |
|---|---|---|---|
| Content-Datenbank | Notion | Baserow Cloud Free oder Airtable Free | 0 € |
| Automatisierung | n8n, eigener Server | n8n Cloud Starter, nach drei Monaten Hetzner | 24 € → 5 € |
| Bild-Archiv | Google Drive | Google Drive, 15 GB reichen lange | 0 € |
| Auslieferung an Meta | eigener Bucket | Cloudinary Free oder Supabase Storage | 0 € |
| Freigabe & Alarm | Telegram | Telegram | 0 € |
| KI-Texte | Claude / OpenAI | Gemini Flash oder Haiku für Entwürfe, großes Modell nur zur Politur | 3–10 € |
| Wissensablage | Notion | Notion Free — persönlich weiterhin gratis | 0 € |
| **Summe** | — | Start in der Cloud, später eigener Server | **~27 € → ~8 €** |

**Empfehlung für Dorian:** Baserow Free, n8n Cloud Starter, Cloudinary Free, Telegram. Ein
Baustein kostet Geld, der Rest nichts. Trägt es nach drei Monaten, wandert alles auf einen
Hetzner — Workflows als JSON, Daten als CSV, kein Neubau.

---

## Notion-Alternativen im Vergleich

Bewertungsfrage: *Wie gut trägt das Tool einen automatisierten Content-Betrieb?* — nicht,
wie schön es ist. (Grist ist das bessere Tabellenwerkzeug und trotzdem schlechter
bewertet, weil ihm die Connectoren fehlen.)

| Tool | Stärke | Schwäche | Eignung |
|---|---|---|---|
| **Airtable** (SaaS/US) | Attachment-Feld löst Bild-Hosting nebenbei; Kalender-/Kanban-/Galerie-Views; native Nodes in Make, Zapier, n8n | Free-Tarif eng; Preissprung pro Nutzer; US-Hosting | ●●●●● Erste Wahl Stufe 2 |
| **Baserow** (OSS/EU) | Airtable-Logik, self-hostbar oder EU-Cloud; n8n-Node; keine Record-Limits | kleineres Ökosystem, UI rauer | ●●●●○ Beste Wahl Stufe 3 |
| **SeaTable** (SaaS/DE) | deutsches Hosting, Airtable-nah, n8n-Node — DSGVO ohne eigenen Server | kleine Community, dünnere Doku | ●●●●○ wenn Hosting-Ort zählt |
| **Google Sheets** | kennt jeder, gratis, stabile API, CSV-Import überall | keine Bilder, keine Relationen, kein Status-Workflow; mobil zäh | ●●●○○ nur Einstieg |
| **NocoDB** (OSS) | Oberfläche über echtes Postgres — Daten bleiben in der DB | denkt in Datenbanken statt Redaktion; mehr Setup als Baserow | ●●●○○ wenn Postgres da ist |
| **Grist** (OSS/EU) | sauberstes Datenmodell, echte Python-Formeln, EU-Hosting | keine nativen Automations-Nodes (nur HTTP); Anhänge schwach | ●●○○○ Auswertung, nicht Betrieb |
| **Supabase** | Storage liefert echte öffentliche URLs — genau was Instagram verlangt | keine Redaktionsoberfläche | ●●●○○ als Bild-Unterbau ideal |
| **n8n Data Tables** | kein Zusatztool, kein API-Overhead, kein Rate-Limit | keine UI, kein Kalender, keine Bilder | ●●●○○ Zustand, nicht Inhalt |
| **Notion** (SaaS/US) | beste Schreibumgebung, echte Verknüpfungen, alles an einem Ort | API langsam und ratenbegrenzt, Datei-URLs laufen ab, Abfragen teils plan-gesperrt, Felder verwildern | ●●●●○ Frontend ja, Motor nein |
| **Trello** | in fünf Minuten verstanden, mobil erstklassig | ohne Power-Ups keine Felder, keine Auswertung | ●●○○○ nur Ideen-Eingang |

**Urteil zu Notion in einem Satz:** als Redaktionsoberfläche weiterhin schwer zu schlagen,
als Automatisierungs-Datenbank die schwächste Wahl auf dieser Liste. Die bestehenden
Query-Proxy-Workflows sind kein Zufall — sie sind bereits Umgehungen der Notion-API. Der
Preis ist bezahlbar, solange Notion Frontend bleibt.

## Publisher, falls die API nicht selbst angefasst werden soll

| Tool | Warum | Haken |
|---|---|---|
| Publer | günstig, CSV-Bulk-Import, brauchbare API | Analytics dünn |
| Metricool | stärkste Auswertung, EU-Anbieter | teurer, Import weniger flexibel |
| Buffer | einfachste Bedienung, Abrechnung pro Kanal | wenig Automatisierungstiefe |
| Meta Business Suite | kostenlos, direkt von Meta | keine Schnittstelle von außen — reine Handarbeit |

## Entscheidungshilfe

- **Dorian will starten, ohne etwas Neues zu abonnieren** → Szenario 0. Drive, Sheet,
  Claude — Haralds Publisher-Workflow läuft mit. Dorian erstellt die Meta-Tokens, sonst nichts.
- **Soll morgen laufen, niemand betreut es** → Stufe 1. Sheet + Publer, CSV monatlich.
- **Mehrere Kanäle, wachsende Bildmengen, mehr als eine Person** → Stufe 2.
- **Jemand im Haus *will* Automatisierung betreiben** → Stufe 3, aber mit Backup,
  Monitoring und Token-Refresh ab Tag eins. Ein n8n ohne Alarmierung ist eine Zeitbombe
  mit langer Zündschnur.
- **Notion behalten?** Ja — als Frontend. Nicht als das, worauf Workflows im Minutentakt
  lesend und schreibend zugreifen.

## Der Satz, der bleiben soll

Nicht das Tool entscheidet, sondern wer das System in zwölf Monaten noch betreiben kann.
Bei Harald ist das Stufe 3, weil er sie betreiben *will*. Bei den meisten kleinen
Betrieben ist es Stufe 1 — kein Rückschritt, sondern die passende Antwort.

Drei ehrliche Startpunkte, in dieser Reihenfolge zu prüfen: **Stufe 0**, wenn Harald bereit
ist, den einen Workflow mitlaufen zu lassen — der billigste Test, ob überhaupt regelmäßig
Content entsteht. **Stufe 1**, wenn Dorian unabhängig sein will, ohne etwas zu betreiben.
**Der Startpfad** mit den Workflows 01 und 02, wenn er selbst bauen will. In allen drei
Fällen gilt dasselbe: den Redaktionsplan von Beginn an mit den fünf Feldern oben führen.
Dann ist jeder spätere Umzug ein Export — und kein Neuanfang.

---

*Preise und Meta-Limits sind Stand August 2026 und ändern sich.*
