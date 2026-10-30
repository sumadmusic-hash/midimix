# Umsetzungsplan (nur Plan – noch kein Code)

Reihenfolge so gewählt, dass jede Phase mit der echten Hardware/Mac prüfbar ist.
Kein Schritt erfindet Protokoll-Details; alles greift auf `docs/01_research_midimix.md` zurück.

---

## Phase 0 – Verifikation & Werkzeug-Beschaffung (VOR jedem Code)

**Zweck:** Alle als OFFEN markierten Punkte aus docs/01 klären; Risiken aus dem Weg räumen.

| Task | Beschreibung | Ergebnis / Artefakt |
|---|---|---|
| 0.1 | Mac-Setup: Xcode, CMake, Steinberg VST3 SDK v3.7.x + public sdk + VSTGUI herunterladen (offizielle Quellen: github.com/steinbergmedia/vst3sdk, vstgui). macOS Deployment-Target festlegen (Intel/ARM universal?). | `tools/setup.md` |
| 0.2 | Testhost installieren (abhängig von Rückfrage R1 – Reaper empfohlen: kostenlos testbar, VST3-fähig, gutes MIDI-Routing). | – |
| 0.3 | **MIDI-Monitor-Session:** MIDImix per USB anschließen, mit MiniMIDI/Audio MIDI Setup jeden Regler/Fader/Button durchprobieren, gesendete Bytes protokollieren → Vergleich mit docs/01 Tabelle. | `research/midimix_capture.md` |
| 0.4 | **LED-Smoke-Test:** kleines Swift-/C++-Kommandozeilenprogramm über CoreMIDI: `0x90 id vel` an MIDImix senden → leuchten die richtigen Tasten? Auch Solo-Gedrückt-Fall testen (O4). | `tools/ledtest/` |
| 0.5 | **SysEx-Roundtrip:** 0x66 Request senden → 0x67-Antwort parsen; mit MIDImix Editor erstelltes `.midimix`-File byte-genau vergleichen (Dokumentation des exakten Byte-Layouts, inkl. nicht dokumentierter Bytes). Danach gefahrlos 0x64 mit unverändertem Inhalt zurückschicken (Roundtrip-Test O5/O7). | `research/sysex_roundtrip.md` |
| 0.6 | Entscheidung zu R1–R7 einholen und hier ergänzen. | Abschnitt „Entscheidungen“ unten |

**Exit-Kriterium:** Mapping-Tabelle bestätigt, LED- und SysEx-Pfad nachweislich funktionsfähig.

## Phase 1 – Projektgerüst (kein Sound, kein MIDI)

| Task | Beschreibung |
|---|---|
| 1.1 | CMake-Projekt `midimixseq/` mit VST3-SDK-Submodule; Ziel: `MidiMixSeq.vst3` Bundle (macOS xpcbundle), Codesign-Optionen vorbereitet. |
| 1.2 | Processor-Klasse + EditController-Gerüst, Parameter-IDs definiert (8 Spuren × {Gain, Pan, Cutoff, Res, Decay, RootPitch} + Master). |
| 1.3 | CI-lokal: Build-Skript `build.sh`, Smoke-Test „Plugin lädt in Host“. |

## Phase 2 – Sampler-Core (Audio)

| Task | Beschreibung |
|---|---|
| 2.1 | Sample-Loader (WAV via std oder dr_wav – Lizenz prüfen), 1 Zone/Spur, RAM-Buffer. |
| 2.2 | Voice: Abspiel-Position, ADSR (A/D/S/R-Parameter), Lineares/Cos-Interpolation, Stereo/Mono. |
| 2.3 | Filter pro Spur (erst One-Pole, später Ladder), Gain/Pan, Softclip-Mixer, Master. |
| 2.4 | Manuelle Trigger von außen: Plugin empfängt Note-On auf MIDI-Kanal n → spielt Spur n (damit ist alles Weitere testbar, bevor der Sequencer existiert). |
| 2.5 | State save/load (Pattern + Pfade) via `get/setState`. |

## Phase 3 – Step-Sequencer-Engine

| Task | Beschreibung |
|---|---|
| 3.1 | `SequenceEngine`: aus `ProcessContext` (ppqPos, tempo, timeSigInfo, sampleRate) 16tel-Steps ableiten; Substep-Queue; Swing; Start/Stop/Reset. |
| 3.2 | Pattern-Datenstruktur + atomare UI→Audio-Übergabe (double-buffered snapshot pro Block). |
| 3.3 | Trace-Ausgabe (Debug): getriggerte Steps auf stdout, damit Timing ohne GUI prüfbar ist. |
| 3.4 | Optional externer MIDI-Out-Bus „Sequencer Out“ (R6). |

## Phase 4 – MIDImix-Anbindung (Input)

| Task | Beschreibung |
|---|---|
| 4.1 | `MidimixParser` (reine Logik, unit-testbar): Maps aus docs/01 (Note-ID → Step, CC → Parameter). Konstanten zentral in `midimix_map.h`. |
| 4.2 | Anbindung an Plugin-MIDI-Eingang (Modus A): DAW routet MIDImix-Ausgang auf Plugin-In. |
| 4.3 | Interaktion: Step-Toggle, Track-Select (Bank L/R), Fader→Volume, Encoder→Filter, Solo-Hold-Modifikator. |
| 4.4 | Learn-Fallback: Wenn eingehende ID unbekannt → Config-Datei-Eintrag (später GUI). |

## Phase 5 – LED-Feedback (Output)

| Task | Beschreibung |
|---|---|
| 5.1 | `LedDriver`: erzeugt `0x90 <origNTID> <on/off>` für alle 16 Step-LEDs bei Musteränderung; Bank-LEDs für selectedTrack-Anzeige. |
| 5.2 | Playhead-Animation: im Step-Takt aktuelle Step-LED togglen (binär, da keine Helligkeit [PROT S.5]); Taktart-Optionen (alle 4 Steps heller = blinkt doppelt). |
| 5.3 | Routing-Check: LEDs müssen den Plugin-MIDI-Output-Bus Richtung Gerät verlassen (Host-Routing-Doku schreiben!). |

## Phase 6 – SysEx-Template (optional, nur wenn Phase 0.5 sauber lief)

| Task | Beschreibung |
|---|---|
| 6.1 | `midimix_sysex.{h,cpp}`: Builder/Parser 0x60/0x64/0x66/0x67 exakt nach docs/01 §4. |
| 6.2 | Button „Configure MIDImix“ im GUI sendet 0x64 mit unserem Wunsch-Template (Standard belassen! Nur MUTE+SOLO-Virtual-Buttons evtl. umbiegen). |
| 6.3 | Restore-Funktion: aktuelles Device-Config vor Überschreiben per 0x66 sichern (`.midimix`-Export). |

## Phase 7 – Feinschliff, Stabilität, Distribution

| Task | Beschreibung |
|---|---|
| 7.1 | Undo/Redo, Pattern Copy/Paste, Randomize, Clear. |
| 7.2 | Thread-Safety-Review, ASan/TSan-Läufe, CPU-Profil (Ziel: <2 % auf M1). |
| 7.3 | Handbuch-Nutzerseite: DAW-Routing-Screenshot (In/Out, Monitor off!), MIDImix-Belegung als PDF-Cheat-Sheet. |
| 7.4 | Packaging: .vst3 in `/Library/Audio/Plug-Ins/VST3/`, notarization-ready Skript (eigener Dev-Account nötig → Kostenrückfrage). |

---

## Entscheidungen (werden nach Beantwortung der Rückfragen eingetragen)

| # | Frage | Antwort | Konsequenz |
|---|---|---|---|
| R1 | Host-DAW? | offen | Logic geht nicht mit VST3 → ggf. AU-Build nötig |
| R2 | Modus A vs. B | offen | bestimmt Phase 4/5 Implementierung |
| R3 | SDK pur vs. JUCE | offen | Build-System, GUI-Toolkit |
| R4 | Step-Mapping-Variante | offen | docs/02 §3.2 |
| R5 | 1 Sample/Spur ok? | offen | Sampler-Komplexität |
| R6 | MIDI-Out für externe Geräte? | offen | Bus-Design |
| R7 | Pattern-Ketten v1? | offen | Datenmodell |

## Grober Zeitrahmen (Tageseinheiten à ~4 h, nur Schätzung)

Phase 0: 1–2 · Phase 1: 2 · Phase 2: 4–6 · Phase 3: 3–4 · Phase 4: 2–3 · Phase 5: 2 · Phase 6: 1–2 · Phase 7: 3–5

## Repo-Struktur (geplant)

```
/workspace
├── docs/            # dieses Dokument + Recherche
├── research/        # PDFs, Referenzskripte (bestehend)
├── tools/           # ledtest, sysex_probe (Phase 0)
└── midimixseq/      # Plugin-Projekt (ab Phase 1)
    ├── CMakeLists.txt
    ├── source/{processor,controller,midimix_map,midimix_sysex,engine,...}
    └── tests/       # reine Logik unit-testbar (Parser!)
```
