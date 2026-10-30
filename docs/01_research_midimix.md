# Akai MIDImix – Recherche (quellbasiert)

Alle Angaben in diesem Dokument sind belegt durch:
- **[UG]** `research/midimix_userguide.pdf/.txt` (offizielles User Guide)
- **[PROT]** `research/midimix_unofficial_protocol.pdf` ("(Unofficial) Akai Professional MidiMix Communications Protocol Guide v0.04" von Julian Ceipek)
- **[AL9]** `research/ableton_live_midi_map.py` + `ableton_live_midimix_main.py` (Ableton-Live-9-Remote-Script, Hanz Petrov / st4rchild)
- **[ALD]** `research/midimixd_elements.py`, `midimixd_mappings.py`, `midimixd_init.py` (Ableton-Live-10+-Control-Surface-Definition "midimixd")

Als **OFFEN** markierte Punkte müssen mit der echten Hardware verifiziert werden (Phase 0).

---

## 1. Hardware-Ausstattung [UG S. 3–5]

| Bedienelement | Anzahl | Anmerkung |
|---|---|---|
| Kanal-Fader | 8 | senden CC |
| Master-Fader | 1 | senden CC |
| Encoder/Knobs (270°) | 24 | 3 pro Kanal, senden CC |
| Mute-Buttons (beleuchtet) | 8 | Bank 1 (obere Reihe) |
| Record-Arm-Buttons (beleuchtet) | 8 | Bank 2 (untere Reihe) |
| Solo-Taste (dauerhaft, nur Modifikator) | 1 | **keine LED** [PROT S.5: "SOLO button does not have physical LEDs according to Akai support"] |
| Bank Left / Bank Right | 2 | verschieben die 8 Kanäle |
| Send All | 1 | sendet CC-Werte aller Knobs+Fader auf einmal; **eigene Taste sendet keine Note/CC**, sondern löst nur die CC-Bursts aus [PROT S.4]. Note-ID laut [ALD elements.py, Kommentar Z.7]: **Note #62**. Keine LED [PROT S.5]. |

- Verbindung: **nur USB** (Bus-Powered, kein MIDI-DIN) [UG].
- Vendor-ID 2536 = Akai Professional, Product-ID 49 ("MIDI Mix") [ALD init.py]; Manufacturer-SysEx-ID **0x47 (71)**, Modell-ID **0x31 (49)** [PROT S.7–8].
- Identity-Response des MIDImix: Bytes `(71, 49, 0, 25)` [ALD init.py / PROT S.8].

## 2. Standard-MIDI-Ausgang (Host → bekommt Daten vom MIDImix)

### 2.1 Formate [PROT S.3–4]
- **CC:** `0xB<chan> <ccid> <val>` (chan 0-basiert im Byte, val 0–127)
- **Note On/Off:** `0x9<chan>/<0x8><chan> <ntid> 0x7F` – Velocity beim MIDImix **immer 0x7F** [PROT S.4].

### 2.2 Default-Identifier (Note-Modus, alle Buttons auf MIDI-Kanal 1) [ALD elements.py]

Elemente sind spaltenorientiert: Spalte n (0–7) = ein Kanalstreifen.

| Element | Note-IDs Spalte 0→7 | Typ |
|---|---|---|
| MUTE-Buttons (Bank 1) | 1, 4, 7, 10, 13, 16, 19, 22 | NT |
| SOLO (+Solo gedrückt, MUTE-Buttons ändern IDs!) | 2, 5, 8, 11, 14, 17, 20, 23 | NT |
| REC-ARM-Buttons (Bank 2) | 3, 6, 9, 12, 15, 18, 21, 24 | NT |
| Bank Left | 25 | NT |
| Bank Right | 26 | NT |
| Solo-Taste (Modifikator) | 27 | NT |
| Send All | 62 | NT (per Kommentar ALD; **OFFEN**: verhält sich laut [PROT] anders, s.o.) |

| Element | CC-IDs (Spalte 0→7) |
|---|---|
| Encoder Reihe 1 (oben) | 16, 20, 24, 28, 46, 50, 54, 58 |
| Encoder Reihe 2 | 17, 21, 25, 29, 47, 51, 55, 59 |
| Encoder Reihe 3 | 18, 22, 26, 30, 48, 52, 56, 60 |
| Fader 1–8 | 19, 23, 27, 31, 49, 53, 57, 61 |
| Master-Fader | 62 |

Wichtig: Die Encoder-CCs springen nach Spalte 4 (CC 28/29/30) auf 46… weiter – das ist das Original-Mapping und muss so behandelt werden.

## 3. LED-Ansteuerung (Host → MIDImix) [PROT S.5]

- Format: **Note On auf Kanal 1**: `0x90 <NT-ID> <vel>`; vel ≠ 0 ⇒ LED an, vel = 0 ⇒ LED aus.
- Es wird immer die **ursprüngliche Note-ID** verwendet, auch wenn der Button per SysEx auf eine andere ID/CC umkonfiguriert wurde [PROT S.5].
- **0x80 (Note Off) wird vom MIDImix NICHT verstanden** – LEDs gehen nur mit `0x90 id 0` aus [PROT S.5].
- Beispiel: linke Bank-Taste LED an = `[0x90, 0x19, 0x7F]` (25 = 0x19) [PROT S.5].
- MUTE+SOLO-LEDs: Während Solo gedrückt ist, reagieren die MUTE-LEDs auf die **MUTE+SOLO-IDs** (2,5,8,…23) [PROT S.5–6]. → Für unser Design relevant, wenn wir die MUTE-Reihe als zweite Step-Hälfte nutzen.
- Keine LEDs bei: SOLO-Taste, SEND ALL [PROT S.5].
- Helligkeit: Werte >0 schalten nur an/aus, "light intensity appears to be unaffected" [PROT S.5]. **Kein Dimmen/RGB** ⇒ Playhead-Feedback nur binär (an/aus) oder durch Blinken vom Plugin realisiert.

## 4. SysEx-Konfiguration (MIDImix Editor / "Send To Hardware") [PROT S.10–18]

Generisches Akai-Format: `F0 47 <dev> 31 <msgid> <lenMSB> <lenLSB> <daten...> F7`.

| Msg | ID | Richtung | Zweck |
|---|---|---|---|
| Introduction | 0x60 | Host→Device | teilt Firmware Versions-/Mode-Info mit (Ableton sendet Mode 0x41) |
| Configuration Set | 0x64 | Host→Device | **überschreibt das Mapping** – genau das, was der MIDImix Editor bei "File → Send To Hardware" schickt |
| Request Config | 0x66 | Host→Device | Last-from-Hardware (Antwort = 0x67) |
| Configuration Response | 0x67 | Device→Host | enthält aktuelles Mapping; `.midimix`-Editor-Dateien haben exakt dieses Format |

Datensektion der 0x64/0x67 (138 Bytes):
- `<Dial Config>`: 24 × 2 Byte (Kanal 0–15, CC-ID 0–127), Bytes 8–55
- `<Slider Config>`: 9 × 2 Byte (8 Kanäle + Master), Bytes 56–73
- `<MUTE Button Config>`: 8 × 3 Byte (Kanal, Mode 0=Note/1=CC, ID), Bytes 74–97
- `<REC ARM Button Config>`: 8 × 3 Byte, Bytes 98–121
- `<MUTE+SOLO Button Config>`: 8 × 3 Byte, Bytes 122–145

Konsequenzen für uns:
1. Buttons können wahlweise **Notes oder CCs** senden; Kanal und ID je Button frei wählbar.
2. LED-Rückmeldung existiert **nur für die 16 Buttons + Bank/Solo-Keys**; Encoder/Fader haben keinen Rückkanal. Konfigurationsdateien enthalten keine Einträge für Bank/Solo/Send-All → deren IDs sind **nicht änderbar** ([PROT S.6]: nicht konfigurierbare Identifier).
3. Mit dem offiziellen MIDImix Editor (Download akaipro.com) lassen sich dieselben Änderungen grafisch vornehmen und als `.midimix` speichern [UG S.3, PROT S.13–14]. Unser Plugin könnte 0x64/0x66 sogar selbst sprechen (optionaler Komfort: automatisches Hardware-Setup beim Laden des Presets).

## 5. Risiken / offene Punkte (Phase-0-Checkliste)

| # | Frage | Wie verifizieren |
|---|---|---|
| O1 | Sendediagramm wirklich identisch zu [ALD]? | MIDI-Monitor (z.B. macOS Audio MIDI Setup / Schnellaufnahme in DAW / midimonitor), jedes Element bewegen |
| O2 | Funktioniert LED-Set zuverlässig mit `0x90 id vel` über CoreMIDI aus einem VST3? | Testprogramm |
| O3 | Verhält sich "Send All" (CC-Burst) störend? | Test |
| O4 | Reagieren MUTE-LEDs während Solo-Gedrückt-Halten tatsächlich auf 2,5,8…? | Test |
| O5 | Werden per SysEx geänderte Button-IDs korrekt gesendet, LED aber weiterhin über Original-ID erreichbar? ([PROT] sagt ja) | Test |
| O6 | CoreMIDI Exclusive Access: Was passiert, wenn gleichzeitig eine DAW verbunden ist? | Test |
| O7 | Bleibt ein per SysEx gesetztes Template nach Neustart erhalten? | Stromlos machen, prüfen |

## 6. Bewertung für das Projekt

Eignung als Step-Sequencer-Controller für ein eigenes VST3: **gut**, weil
- 16 beleuchtete Buttons (MUTE + REC-ARM) direkt als Step-Raster nutzbar,
- LED-Rückmeldung vorhanden (binär),
- 24 Encoder + 9 Fader für Parameter,
- SysEx erlaubt sauberes Neu-Mapping.

Einschränkungen, die ins Design eingehen:
- Nur 16 physische Steps sichtbar → Pattern >16 Steps braucht Banks (unser Konzept: 16 Steps = genau eine Vollbank, s. 02_konzept.md).
- Keine LED-Helligkeit → Playhead nur durch schnelles Umschalten (Plugin-Timer) darstellen.
- Encoder sind endlos ohne Positionsrückmeldung → relative Werte/Parameter-Snap nötig.
- Kein Aftertouch/Velocity (immer 0x7F).
