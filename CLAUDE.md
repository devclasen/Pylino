# Pylino – Projektkontext für Claude Code

## Worum es geht

Pylino ist ein sehr einfacher Python-Editor für den Schulunterricht (Einsteiger, Sek I/II). Er ersetzt Thonny, weil Installation und Portable-Version an der Schule ständig Probleme machen und der IT-Dienstleister nicht hilft.

**Leitidee:** Nichts installieren. Alles läuft im Browser (Chrome/Edge), verteilt als Link oder als eine HTML-Datei.

**Zwei Ziele, umschaltbar oben in der Oberfläche:**
- **Computer:** Python läuft im Browser (Pyodide). Für Python-Basics: Variablen, Schleifen, Funktionen, Listen, `input()`, `print()`, Standardbibliothek.
- **Pico:** MicroPython auf dem Raspberry Pi Pico über USB (Web Serial API). Für Basisprojekte: LED, Taster, Sensoren, PWM.

Bewusst **nicht** im Umfang: Fenster/Grafik (`tkinter`, `turtle`, `pygame`), Zugriff auf PC-Dateien, Threads, Internetzugriff. Das ist gewollt – Pylino ist „kleines Python“ und sagt das auch ehrlich (siehe „Grenzen“ unten).

## Zielgruppe & Ton

- Schülerinnen und Schüler, teils jung. Lehrkraft nutzt es auch am Beamer (Schriftgröße A−/A+).
- **Oberfläche komplett auf Deutsch**, freundlich, nicht kindisch, nicht „unseriös“. Keine Texte, die technisch-holprig klingen (z. B. wurde „wird aktiv, wenn dein Programm input() aufruft“ bewusst ersetzt durch „Keine Eingabe erwartet“ / „Deine Eingabe …“).
- Fehlermeldungen: Original-Fehler + **deutscher Tipp** + Markierung der Zeile im Editor.
- Name **Pylino**, Icon: kleine grüne Comic-Schlange (inline SVG, auch als Favicon). Schrift: Poppins (OFL), als Teilmenge base64-eingebettet – **keine Google Fonts** (Datenschutz).

## Aktueller Stand

Alles steckt in **einer Datei: `index.html`** (≈ 110 KB, davon ~40 KB eingebettete Schrift; früher `pylino.html`, für GitHub Pages umbenannt).

Repo: https://github.com/devclasen/Pylino (öffentlich, Branch `main`). Seite: https://devclasen.github.io/Pylino/

**Lokale Version = dieselbe Datei.** Per Doppelklick (`file://`) bekommt `<body>` die Klasse `local`, alles mit `.online-only` (Impressum-Links, Download-Link in der Hilfe) wird ausgeblendet. Die Hilfe der Webseite bietet `index.html` als `pylino.html` zum Herunterladen an. Keine zweite Kopie pflegen.

### Architektur

- **Editor:** CodeMirror 5 (vom CDN cdnjs). Fallback auf `<textarea>`, falls CodeMirror nicht lädt. Tab = 4 Leerzeichen, Strg+Enter/F5 = ausführen, Esc = stoppen, Strg+S = speichern.
- **Python im Browser:** Pyodide v0.27.7 (vom CDN jsDelivr) läuft in einem **Web Worker**, erzeugt aus einem Blob (Quelltext steht in `<script type="text/plain" id="worker-src">`).
  - Vorteil Worker: UI friert nicht ein, Ausgabe kommt live, **Stopp = `worker.terminate()` + Neustart** (bricht auch Endlosschleifen ab).
  - `input()` und `time.sleep()` werden in Python überschrieben und nutzen `pyodide.ffi.run_sync` (**JSPI**, braucht aktuelles Chrome/Edge). Kein SharedArrayBuffer → keine COOP/COEP-Header nötig, funktioniert auch von `file://` und einfachem Static Hosting.
  - Nachrichten Worker ↔ Seite: `init`, `run`, `inputReply` / `ready`, `out`, `err`, `info`, `input`, `error`, `done`, `fatal`.
  - Nutzerprogramm wird mit `compile(src, "<dein Programm>", "exec")` in frischem Namespace ausgeführt; Fehler werden als `{name, msg, line, text}` gemeldet.
  - `loadPackagesFromImports` lädt z. B. numpy automatisch nach.
- **Pico:** Web Serial, gefiltert auf USB-Vendor `0x2E8A` (Raspberry Pi), 115200 Baud.
  - Ausführen über den **Raw REPL** (Ctrl-A, Code, Ctrl-D; Antwort `OK` … `\x04` stdout … `\x04` stderr … `>`). Vorher Soft-Reset im Raw REPL (wie Thonny, startet `main.py` nicht).
  - Danach Ctrl-B zurück in den normalen REPL; Begrüßungsbanner wird ausgeblendet.
  - `input()` auf dem Pico funktioniert: Eingabefeld sendet Zeile + `\r`, Pico macht das Echo selbst.
  - Stopp = Ctrl-C. „Auf Pico speichern“ schreibt `main.py` (in Stücken per `f.write(JSON-String)`), „main.py vom Pico laden“ liest sie zurück.
  - Im Leerlauf ist die Konsole ein einfacher REPL-Durchgriff (Befehle direkt an den Pico).
- **Speicherung:** Programmtext, Ziel, Schriftgröße, Dateiname in `localStorage` (Schlüssel `pylino.*`), immer in try/catch. Öffnen/Speichern von `.py`-Dateien lokal (File System Access API mit Download-Fallback).
- **Grenzen-Prüfung:** Tabelle `LIMITS` + `checkLimits()` prüfen vor dem Start die `import`-Zeilen und zeigen „Das kann Pylino (noch) nicht“ mit Erklärung. `machine` & Co. auf „Computer“ → Hinweis, auf „Pico“ umzuschalten. `hintFor()` nutzt dieselbe Tabelle für Laufzeit-Importfehler. Abschnitt „Was Pylino kann – und was nicht“ in der Hilfe.
- **Beispiele:** 5× Python-Grundlagen (Hallo+Eingabe, Zahlenraten, Einmaleins, Notenschnitt, Countdown), 5× Pico (LED blinken, Blinken mit `input()`, Temperatursensor ADC(4), Taster GP14, PWM-Dimmen GP15). Auswahl schaltet das Ziel passend um. Sollen so bleiben.
- **Design:** Farben als CSS-Variablen auf `:root`, Dark Mode über `prefers-color-scheme` und `data-theme`. Konsole immer dunkel. Grün = Marke/Ausführen, Himbeerrot = Pico.

## ⚠️ Noch nicht live getestet

Der Prototyp wurde nur auf Syntax/Logik geprüft (Node `--check`, Python-Setup unter CPython, Screenshots ohne CDN-Zugriff). **Zuerst in echtem Chrome/Edge prüfen:**
1. Lädt Pyodide 0.27.7 im Blob-Worker – auch per Doppelklick (`file://`)?
2. Funktioniert `input()` über `run_sync`? Kritischer Punkt: Ob `run_sync` aus `callPromising` bzw. `runPythonAsync` heraus erlaubt ist (`can_run_sync()`). Falls nicht: Pyodide-Version/Aufrufweg anpassen (ggf. neuere Pyodide-Version, die JSPI sauber unterstützt).
3. `time.sleep()` im Countdown-Beispiel: Ausgabe kommt sekundenweise?
4. Stopp bei `while True: print(1)` – UI bleibt bedienbar, Python startet neu.
5. Pico: Verbinden, Ausführen, Stopp, Fehler mit Zeilenmarkierung, `input()` auf dem Pico, `main.py` speichern/laden, Kabel ziehen während des Laufs.
6. Deutsche Tipps bei typischen Fehlern (NameError, fehlender Doppelpunkt, Text + Zahl, `int("abc")`).

## Nächste Schritte (Priorität)

1. **Live-Test** (siehe oben) und Fehler beheben.
2. **Hosting**: GitHub Pages (öffentliches Repo) oder Cloudflare Pages, Datei als `index.html`. HTTPS ist Pflicht für Web Serial – beide liefern das.
3. **Abhängigkeiten selbst hosten** statt CDN (Datenschutz, Schule): nur den **Pyodide-Kern** (`pyodide.js`, `pyodide.asm.js`, `pyodide.asm.wasm`, `python_stdlib.zip`, `pyodide-lock.json`, ca. 15 MB) plus CodeMirror-Dateien ins Repo, `PYODIDE_URL` auf relativen Pfad umstellen. Lizenzdateien (Pyodide: MPL 2.0, CodeMirror/MicroPython: MIT) mitliefern.
   - Achtung: Selbst gehostete Pyodide-Dateien lassen sich von `file://` aus nicht laden. Entweder Doppelklick-Version weiter mit CDN anbieten oder nur den Link verwenden (Empfehlung: Link).
   - Danach optional als PWA mit Service Worker → funktioniert auch offline.
4. **Eingabe direkt in der Konsole** (Cursor hinter der Frage wie bei Thonny) wäre die „schönste“ Variante – optional, das Eingabefeld ist aktuell okay.
5. ~~Impressum/Datenschutzerklärung~~ erledigt: `datenschutz.html` (eine Seite, Anker `#impressum` / `#datenschutz`, Anrede „du“), verlinkt in Fußzeile und Hilfe. Anbieterdaten wie bei Tinkify (`Documents/Tinkify/website/anbieter.mjs`: c/o COCENTER, E-Mail tinkify@mail.de, ULD Schleswig-Holstein). Impressum nach § 18 Abs. 1 MStV (privat, nicht kommerziell). **Bei jeder Änderung an geladenen Diensten mitpflegen** – z. B. Abschnitt 3 (cdnjs, jsDelivr) entfällt/ändert sich, sobald Pyodide und CodeMirror selbst gehostet werden.

## Später / Ideen

- **Weitere Mikrocontroller**: Raw REPL ist bei allen MicroPython-Boards gleich. Nötig wären: USB-Filter lockern, Umschalter „Computer / Mikrocontroller“, Beispiele pro Board. Kandidaten: Pico W/Pico 2 (laufen größtenteils schon), BBC micro:bit, Arduino-Boards mit MicroPython, ESP32 (Achtung: unter Windows oft CH340/CP210x-Treiber nötig = Installationsproblem). Calliope mini: prüfen. Vorerst bewusst **nur Pico**.
- Dateimanager für den Pico (mehrere Dateien), Paketinstallation (micropython-lib).
- Aufteilung der einen Datei in Module, sobald es wächst (dann mit kleinem Build-Schritt, Ergebnis weiterhin statisch hostbar).

## Rahmenbedingungen

- Kein Server-Code, kein Backend, kein Tracking, keine Accounts. Python läuft ausschließlich im Browser der Nutzer (Sandbox) – sicherer als Thonny, das vollen PC-Zugriff hat.
- Nur Chrome/Edge werden offiziell unterstützt (Web Serial, JSPI). Firefox/Safari: Computer-Modus evtl. eingeschränkt, Pico gar nicht – freundlich darauf hinweisen.
- Keine externen Dienste hinzufügen, die IP-Adressen an Dritte übertragen (keine Google Fonts, keine Analytics).
- Texte im Code (Kommentare) und in der UI auf Deutsch.
