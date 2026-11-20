# FAH-Jarvis

Persönlicher Sprachassistent — Weckwort, Bildschirm-Sicht, Sprachbefehle,
der Mac (später auch den Windows-GPU-PC) steuert. Vorbild: Jarvis aus Iron Man.

## Aufbau, Bauen, Installieren (seit 16.11.2026)
- **Eine App: `/Applications/FAH-Friday.app`.** Nur die läuft, auch beim Autostart (Anmeldeobjekt). Nie aus dem Projektordner starten.
- **In der App (nur lesen):** Swift-App, Friday-Code (`Contents/Resources/payload`: jarvis.py, voice/*.py, brain/, hooks/),
  Sprechererkennung (`Frameworks` + `Resources/stimme`) und lokale Stimme (`Resources/lokale-stimme`). Nichts wird kopiert,
  kein Bytecode wird in die App geschrieben (`-B`) → die Signatur bleibt intakt.
- **Persönliche Daten:** `~/Library/Application Support/FAH-Friday/` (0700) mit `voice/` (alle Einstellungen und Zustände:
  `*.txt`, `licht.json`, `sauger.json`, `stimmprofil.json`, Kontakte …), `logs/` und `gehirn-einstellungen.json`.
  **Alle Angaben `voice/…` und `logs/…` in dieser README meinen diesen Datenordner.** Python findet ihn über `FRIDAY_DATEN`
  (setzt die App; ohne Variable gilt derselbe Standardpfad). Tests immer mit `FRIDAY_DATEN=<Testordner>`.
- **Git-Sicherung:** `voice/licht_sicherung.json` und `voice/sauger_raeume.json` legt die App zusätzlich in den Projektordner
  (nur auf dem Entwickler-Mac, `FRIDAY_GIT_SPIEGEL`) → danach committen.
- **Bauen + installieren:** `bash gui/mac/bauen.sh` baut die komplette App, signiert (Apple Development), beendet die laufende
  Friday, ERSETZT `/Applications/FAH-Friday.app` und startet sie neu (`NUR_BAUEN=1` = ohne Installieren). Zwischenstand
  `gui/mac/build/` wird danach gelöscht. `gui/mac/FAH-Friday.app` im Git ist nur der Bau-Stand der Swift-App (ohne Code und
  Stimme; geöffnet beendet er sich mit Hinweis).
- **Verteil-ZIP:** `bash gui/mac/paket.sh` → `~/claude/NEUE ZIPs/FAH-Friday_release.zip` (generischer System-Prompt, keine
  persönlichen Daten, ad hoc signiert, stoppt bei persönlichen Pfaden im Paket).
- **Stimm-Vorlage:** `gui/mac/lokale-stimme/` (nicht im Git): `lib/` (Piper, feste Versionen mit Prüfsummen, `--no-compile`)
  und `friday.onnx(.json)` (gegen `voice/lokal_modell.sha256` geprüft). Fehlt `lib/`, holt `app_bauen.sh` sie neu.

## Entscheidungen (09.11.2026)
- **Geräte:** beide (Mac + Windows-GPU-PC), Start auf dem Mac.
- **Gehirn:** hybrid — einfaches lokal, Schweres über **Login-Claude** (Claude Code
  headless, **keine API**, kein Pro-Nutzung-Preis außer dem Abo).
- **Autonomie:** voll automatisch für Umkehrbares; unumkehrbare Aktionen
  (senden/kaufen/löschen/posten) nur nach Rückfrage: Friday fragt „Soll ich wirklich …?“, NUR „Ja“ führt aus;
  Nein, Stopp oder 30 s keine Antwort = nicht ausgeführt; im Telefonat gar nicht (Brake, seit 16.11.2026).

## Aufbau
```
Weckwort + Sprache ──► Gehirn (Login-Claude) ──► Aktionen ──► Stimme + GUI-Overlay
  (voice/)              (brain/ + Claude Code)    (Mac/PC)      (say / gui/)
                              │
                              └── Sicherung: hooks/brake.py (PreToolUse-Hook)
```

- `brain/jarvis_brain.sh` — treibt Claude Code headless an (Login, bypassPermissions,
  Brake-Hook). Das ist Gehirn **und** Hände in einem: Claude Code führt die Aktionen
  über seine eigenen Werkzeuge (Bash/osascript/Browser) selbst aus.
- `brain/system_prompt.md` — macht aus Claude Code „Jarvis".
- `hooks/brake.py` — fängt unumkehrbare Aktionen ab (Ja/Nein-Rückfrage über `logs/rueckfrage*.json`, Test: `tests/brake_test.py`).
- `actions/screen_read.sh` — Screenshot für „sieh, was läuft".
- `jarvis.py` — MVP-Steuerung im **Textmodus** (Anweisung → Gehirn → Stimme).

## Stand
**Stufe 1 — Kern (gebaut + getestet):** Gehirn über Login, Sicherung, Screenshot, Textmodus.
Testbar: `python3 jarvis.py "öffne die Wetter-Webseite"`.

**Stufe 2 — Sprache (gebaut + getestet, 09.11.):** `voice/listen.py` — Weckwort „Hey Jarvis"
(openWakeWord, lokal) → aufnehmen bis Stille → Whisper (faster-whisper, lokal) → Gehirn.
Einrichtung: `bash voice/setup.sh` (venv + Modelle, einmalig). Selbsttest ohne Mikro:
`voice/.venv/bin/python voice/listen.py --test <16k.wav>` — Weckwort 0.997, Transkription ok.
Freihändiger Start: `voice/.venv/bin/python voice/listen.py` (braucht einmalig Mikro-Freigabe).

**Stufe 3 — native Mac-App (gebaut + signiert, 09.11.):** `gui/mac/` — SwiftUI-Menüleisten-App.
Die App holt das **Mikrofon** (signiert → bekommt den TCC-Dialog), wandelt auf 16 kHz und füttert
die Engine per stdin (`listen.py --stdin`); die Engine selbst braucht dadurch keinen Mikro-Zugriff.
Das war die Ursache der stillen Aufnahme im Terminal: ein CLI-Python bekommt keinen Mikro-Dialog und
nur Stille. Menü-Symbol zeigt den Zustand (💤/🟢/👂/🧠/⚙️), Start/Stopp, STOP-Knopf.
Bauen/Installieren: siehe „Aufbau, Bauen, Installieren (seit 16.11.2026)“.

**Noch nicht / als Nächstes:**
- Schwebendes HUD-Overlay (zusätzlich zum Menü-Fenster).
- „Stop" als immer-aktives Sprachkommando während laufender Aktionen.
- ~~App von fester venv/Pfad-Bindung lösen (portabel), dann Release-ZIP.~~ Erledigt 16.11.2026 (siehe Aufbau).
- Windows-GPU-PC + lokales Modell für den „einfachen" Hybrid-Teil.

## Diktat-Modus (12.11.2026)
„Hey Friday, Diktat“ → Ton, Orb grün (⌨️). Ab jetzt wird alles Gesagte nach jeder kurzen
Sprechpause (1,1 s) ins Textfeld der vorderen App getippt – nichts davon wird als Befehl
ausgeführt oder ans Gehirn geschickt, und ins Log kommt nur die Zeichenzahl.
„Hey Friday, stopp“ (auch „Friday stop“, „Hey Friday, Diktat beenden“) beendet das Diktat;
Text davor wird noch getippt, das Stoppwort nie. Danach hört Friday wieder normal auf Befehle.
- „neue Zeile“ / „neuer Absatz“ → Umbruch; Satzzeichen setzt die Apple-Erkennung selbst.
- Einfügen wie FlowSpeech: erst Bedienungshilfen-API (geprüft), sonst Zwischenablage + Cmd+V,
  alte Zwischenablage nach 0,6 s zurück. Passwortfelder werden übersprungen.
- 3 Minuten Stille beenden das Diktat von selbst; im Menü gibt es „Diktat beenden“.
- „Diktiere <Text>“ (mit Text) bleibt das Einmal-Diktat aus `voice/schnell.py`.

## Nur meine Stimme (Stimmerkennung, 12.11.2026)
Einstellungen → „Nur meine Stimme“ → „Meine Stimme anlernen …“: 8 Sätze vorlesen (~1 Minute).
Danach vergleicht Friday jede Äußerung, die etwas auslösen würde (Weckwort, erwarteter Befehl,
Diktat-Satz), mit dem Stimmprofil; fremde Stimmen lösen nichts aus. „Friday, stopp“ wirkt bei
jeder Stimme. Alles lokal:
- `gui/mac/Stimme.swift` – Ringpuffer (letzte 30 s Mikro, 16 kHz), Stimmabdruck, Profil, Schwellen.
- `gui/mac/stimme/` – sherpa-onnx v2.0.1 + ONNX Runtime (dylibs, unverändert ins App-Bundle unter
  `Contents/Frameworks`), Sprecher-Modell 3D-Speaker ERes2Net (VoxCeleb), Lizenzen in `LIZENZEN.txt`.
  Modellwahl gemessen: CAM++ trennte schlecht (Fremde bis 0,87), ERes2Net sauber (gleich ≥ 0,61, fremd ≤ 0,35).
- Profil `voice/stimmprofil.json` (0600, gitignored, nie im Verteil-Paket). Ins Log nur Zahlen
  („STIMME ok 0.62/0.40“), nie Inhalte.
- Strenge Locker/Normal/Streng = Kosinus-Schwelle 0,38/0,45/0,55; rückt mit, wenn die eigenen
  Proben sich wenig ähneln (Untergrenze 0,38 = knapp über dem höchsten gemessenen Fremdwert). „Mit diesem Mikro ergänzen“ für Headset + MacBook-Mikro.
- Prüfung: unter 0,4 s Sprache kein Urteil (bloßes „Hey Friday“ macht nur bereit und wird mit dem folgenden
  Befehl zusammen geprüft); 0,4–1 s nur bei eindeutigem Wert (dazwischen „unsicher“ = abgelehnt); unter 3 s Schwelle −0,07; ab 4 s in 3-s-Fenstern, JEDES muss bestehen (wer in
  deinen Satz hineinredet, fällt auf). Rechenzeit ~60 ms je Fenster.
- Fail-closed: fehlt Modell/Audio/lesbares Profil, wird nichts ausgeführt und nichts getippt; die Einstellungen
  zeigen die Störung rot, der Schalter bleibt bedienbar. Barge-in bricht mit Schutz nur ab.
- Anlernen: erkannter Text muss zu ≥ 60 % zum Satz passen; beim Ergänzen muss jede Probe zum Profil passen;
  endet nach 2 min ohne Satz und beim Schließen der Einstellungen; angefangene Sätze werden nie ausgeführt.
- Kein Schutz wie Face ID: Eine Aufnahme der Stimme käme durch.

## ElevenLabs-Stimme (14.11.2026)
Einstellungen → „Stimme & HUD“ → Stimme „ElevenLabs (Cloud)“ → API-Schlüssel eintragen →
„Meine Stimme“ und Modell wählen → „Probe hören“.
- Schlüssel nur im macOS-Schlüsselbund (Dienst „FAH-Friday ElevenLabs“). Die App gibt ihn beim
  Agent-Start über stdin weiter (`__elevenkey__`), nie über Datei, Argument oder Log.
  Einmal-Import: `voice/elevenlabs.key` wird beim App-Start in den Schlüsselbund übernommen und gelöscht.
- `voice/elevenlabs.json` = Stimme + Modell (ohne Schlüssel, gitignored); `tts.py` liest sie je Satz → Wechsel wirkt sofort.
- Credits-Anzeige (Rest, Balken, Tarif, Auffüllung, ca. Antworten) über `/v1/user/subscription`.
- Standard-Modell `eleven_v4_turbo` (halber Zeichenverbrauch). Kurze Ansagen als MP3 im Cache (`logs/tts_cache`).
- Ohne Netz, bei leerem Kontingent oder falschem Schlüssel spricht Anna; Grund steht als Code in `logs/tts.log`.
- Datenschutz: Friday schickt dafür ihre Antworttexte an ElevenLabs (nicht deine Sprache, kein Mikro-Ton).

## Friday lokal (Piper auf dem Mac, 14.11.2026)
Einstellungen → „Stimme & HUD“ → Stimme „Friday lokal (Mac)“: eigene Friday-Stimme ohne Internet, ohne GPU-PC,
kostenlos (~1 s Rechenzeit je Satz, Modell ~0,7 s Ladezeit beim Agent-Start).
- Stimmpaket = `lib/` (piper-tts 1.8.0 + onnxruntime 1.19.2 für /usr/bin/python3 3.9, feste Versionen mit Prüfsummen
  in `voice/lokal_requirements.txt`) + `friday.onnx(.json)` (Piper „high“, trainiert auf der lokalen F5-Kopie der
  ElevenLabs-Stimme „Friday“ – nur privat/nicht-kommerziell, F5-Modell cc-by-nc).
- Die Stimme steckt in JEDER App (`Contents/Resources/lokale-stimme`), eigene wie verteilte; Bauvorlage
  `gui/mac/lokale-stimme/` (siehe Aufbau). Erststart der Verteil-App wählt „Friday lokal“.
  Eigenes Paket im Datenordner (`lokale-stimme`) hat Vorrang (Einrichten / Modell wählen /
  Entfernen in den Einstellungen). Bundle ist für Apple Silicon; Intel-Macs: „Einrichten“.
- Tempo (Piper length_scale) in `voice/lokal.json`, Regler 0,8–1,6, Standard 1,2 (= Tempo des ElevenLabs-Originals); Rauschen fest 0,2/0,3 (weniger = kein Metallklang).
- Freigegebenes Modell (15.11.): 500 Epochen auf F5-Daten mit Tempo 0,8, Ähnlichkeit 0,91 zum ElevenLabs-Original; Prüfsumme in `voice/lokal_modell.sha256` – `paket.sh` nimmt nur dieses Modell.
- Ohne Paket oder bei Fehler spricht Anna; Gründe als Code in `logs/tts.log` („LOKAL …“), nie Text.
- Modell + Bibliothek nicht im Git (Größe); Training: Piper1-gpl in einer temporären WSL auf dem GPU-PC.

## Licht (WiZ-Lampen, 15.11.2026)
- `voice/licht.py`: WiZ-Lampen direkt per UDP 38899 (getPilot/setPilot/getSystemConfig), ohne Cloud. Suche = Rundruf + jede Adresse der eingetragenen Netze (Lampen im IoT-VLAN). Lampen ohne Antwort: einmal neu suchen (Adresse gewechselt) und nachfassen.
- Räume: Raum-ID aus der Lampe; Namen nur in der WiZ-Cloud → in Einstellungen → Licht per Blinken zuordnen. Gleicher Name = ein Raum. „Ohne Raumangabe“ = Raum für „Licht an/aus“ ohne Raum.
- Sprache: `schnell.py` fragt `licht.befehl()` vor der „und“-Regel und vor Lautstärke („auf 30 %“). Gehirn bekommt den Aufruf im Systemprompt (warm.py).
- Zustand `voice/licht.json` (gitignored, nicht im Paket). Namen + Zuordnung zusätzlich lesbar in `voice/licht_sicherung.json` (im Git, ohne Adressen/Zeiten, nicht im Paket); fehlt licht.json, liest Friday die Sicherung. Test ohne Senden: `SCHNELL_DRY=1`; Testdatei/-port: `LICHT_DATEI`, `LICHT_PORT`.
- Menü: Zeile „Licht An/Aus“, sobald Lampen bekannt sind. Info.plist: NSLocalNetworkUsageDescription.

## Sauger (Roborock, 15.11.2026)
- `voice/sauger.py`: lokal über TCP 58867, Protokoll L01 (AES-256-GCM) oder 1.0 (AES-ECB). AES über CommonCrypto (ctypes), GCM selbst darauf (gegen CryptoKit geprüft) – keine Zusatzpakete. Leere Nachrichten (Hallo-Antwort) haben Prüfsumme 0.
- Anmeldung einmalig (Roborock-Cloud v4: E-Mail-Code, Signatur `/api/v3/key/sign`, Hawk für Gerätedaten) → Geräteschlüssel + Raumnamen. Schlüssel nur Schlüsselbund (`FAH-Friday Roborock`), über stdin an sauger.py bzw. als `__saugerkey__` an den Agenten; `voice/sauger.json` ohne Geheimnisse (gitignored, nicht im Paket).
- Gerät finden: gemerkte Adresse, sonst TCP-Suche im Lampen-Netz + eigenen /24. Räume: `get_room_mapping` (Segment → Raum-ID) + Raumnamen aus der Anmeldung → `app_segment_clean`.
- Sprache: `schnell.py` fragt `sauger.befehl()` vor Licht. Menüzeile „Sauger Start/Station“, Einstellungen → Sauger.

## HUD
Bleibt fest auf dem Hauptmonitor (Bildschirm mit Menüleiste), wandert nicht mehr mit der Maus.
Themen (Einstellungen → Stimme & HUD → HUD-Thema): Arc-Reaktor, Friday-HUD, HAL – Grafik + Menüleisten-Symbol
(`gui/mac/Themen.swift`); bewegte Teile als Core-Animation-Ebenen (kaum App-CPU).
Position per Ziehen am Orb, gemerkt relativ in `voice/hud_pos.txt`.

## Erster Live-Test (braucht dich)
Autonome Mac-Steuerung braucht einmalige Freigaben (Bedienungshilfen,
Bildschirmaufnahme, Automation) und einen gemeinsamen ersten Lauf — der Agent darf
beim ersten Mal nicht unbeaufsichtigt loslegen. Siehe `ERSTER-TEST.md` (folgt).

## Sicherheit
- Unumkehrbares ist durch `hooks/brake.py` abgesichert; die Muster dort sind das
  Sicherheitsnetz und werden erweitert (Mail-Senden, Kauf, Löschen, Posten, git push,
  shutdown …). Lieber einmal zu viel nachgefragt als ein Fehlversand.
- `bypassPermissions` gilt nur innerhalb dieses Assistenten; nichts davon ändert deine
  normalen Claude-Code-Einstellungen.
