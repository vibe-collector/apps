# gettingadddone.com – Relaunch-Vorschlag

Stand: 4. September 2026. Basis: `Webseiten/gettingadddone.com/` im Repo `vibe-collector/n8n` (index.html EN, index_de.html DE, style.css).

## Was die Live-Seite heute gut macht

- Klare Botschaft und ehrlicher Ton („Built by someone who gets it").
- Saubere Struktur: Hero → Problem → Templates → How it works → Signal/Noise → Feedback → FAQ → About → Kontakt.
- Bereits kleine Animationen (Fade-in, Count-up, Signal-Balken).
- Zweisprachig, ohne Framework, schnell.

## Was bremst

| Problem | Konkret |
|---|---|
| Zu viel Fließtext | Problem-Karten, Signal-Sektion und About haben je 3 bis 4 Absätze. Auf dem Handy ist das eine Textwand. |
| Hero zeigt nichts Eigenes | Das Laptop-SVG ist ein generisches Mockup. Der eigentliche Aha-Moment (aus 47 Tasks wird eine) wird nur beschrieben, nicht gezeigt. |
| Animationen ohne Tiefe | Alles fadet gleichförmig ein. Kein Parallax, keine Scroll-Verknüpfung, kein interaktives Element. |
| Typografie | Inter für alles. Headlines haben keinen eigenen Charakter, h2 ist nur 1,5rem. |
| Mobile | Der Shop-Button (wichtigster CTA) wird auf dem Handy per CSS ausgeblendet. |
| Performance | Logo-PNG 269 KB, zwei ungenutzte Alt-Logos mit je 1,1 MB im Ordner. Fonts per `@import` im CSS blockieren das Rendering. |
| SEO | Kein `hreflang` zwischen EN und DE, kein `canonical`, kein `og:image`, keine Produkt-Strukturdaten. |
| Etsy-Links | Alle drei Produkte verlinken auf die Shop-Übersicht statt auf das jeweilige Listing. |
| Kein Dark Mode | Seite ignoriert die Systemeinstellung. |

## Vorschlag in 6 Punkten

1. **Hero als These.** Statt Laptop-Mockup eine Bühne: 18 Aufgaben-Chips schweben um eine einzige „One Thing"-Karte. Beim Scrollen driften die Chips in drei Tiefenebenen weg (Parallax), die Karte bleibt. Am Desktop reagieren die Chips zusätzlich leicht auf die Maus. Das ist das Produktversprechen in einer Bewegung.
2. **Text um 60 % kürzen.** Jede Sektion bekommt eine Headline, einen Satz und höchstens drei Stichpunkte. Die drei Problem-Zahlen (47, 20 min, 14 Tage) tragen die Emotion, nicht die Absätze.
3. **Interaktiver Signal/Noise-Regler.** Statt drei statischer Balken ein Slider: Signal-Prozent verschieben, ICES-Score und Status („Signal", „Watch noise", „Noise penalty") ändern sich live. Das Alleinstellungsmerkmal wird erlebbar.
4. **Scroll-gesteuerte Bewegung ohne JavaScript.** Einblendungen, Parallax der Hintergrundzahlen und des Portraits laufen über CSS `animation-timeline: scroll()` / `view()`. Ohne Browser-Support oder bei „Bewegung reduzieren" ist einfach alles sichtbar. Kein Framework, kein Bundle.
5. **Typografie mit Charakter.** Bricolage Grotesque (variabel, schmal gesetzt) für Headlines, Inter bleibt für Fließtext, JetBrains Mono für Labels und Zahlen. Größerer Kontrast in der Hierarchie.
6. **Technik-Hausputz.** Logo als 35 KB WebP (statt 269 KB PNG), Alt-Logos löschen, Fonts per `<link rel=preconnect>`, `hreflang`/`canonical`, JSON-LD für die drei Produkte, Dark Mode über CSS-Tokens, Shop-Button auf Mobile sichtbar.

## Der Prototyp

`gettingadddone.com/index.html` in diesem Ordner ist die englische Seite komplett neu gebaut, als eine Datei ohne Abhängigkeiten (Bilder eingebettet). Er ist noch **kein Drop-in-Ersatz**:

- Texte sind gekürzte Versionen der bestehenden EN-Texte, nicht final abgestimmt.
- Deutsche Version fehlt noch (gleiche Struktur, nur Texte tauschen).
- Etsy-Deep-Links zu den einzelnen Listings fehlen (Harald liefert die URLs).
- Datenschutz-Seiten bleiben unverändert.

## Nächste Schritte

1. Harald schaut den Prototyp an und sagt, was bleibt und was raus soll.
2. Claude baut die DE-Version und die finalen Texte ein.
3. Claude ersetzt die Dateien im n8n-Repo, Harald lädt sie auf den Server.
