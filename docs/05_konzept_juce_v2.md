# Konzept v2 (maßgeblich): „MidiMixSeq“ – JUCE-Drum-Sampler + Step-Sequencer mit MIDImix

Ersetzt die Technik-Kapitel von docs/02 (§4 Architektur, §5 GUI) und passt docs/03 an.
Grundlage der Entscheidungen: docs/04. MIDImix-Protokelldaten: docs/01 (unverändert gültig).

---

## 1. Produktdefinition (finalisiert)

* **Typ:** Instrument-Plugin aus einem JUCE-C++-Codebase. Ziel-Host: **Audiotool VST Bridge**
  auf macOS. Zusätzlich baubar: Standalone-App (zum Testen ohne jeden Host – wichtig für uns!).
* **Klangengine:** Drum-Sampler, **8 Spuren, je 1 Sample** (WAV/AIFF, Drag&Drop oder Datei-Dialog),
  pro Spur: Gain, Pan, **Pitch** (Semitones ±24 + Cent), **Filter** (Cutoff, Resonanz, Env-Amount),
  **Amp-ADSR**, optionale **Filter-ADSR**. Ein-Shot-Wiedergabe (kein Loop nochnachhall-Abschneide-Zwang;
  Release = Sample endet natürlich).
* **Sequencer:** 8 × 16 Steps, Step = on/off (v1), Swing 0–70 %, Auflösung 16tel fix,
  Play/Stop/Reset, Clock wahlweise Host oder Intern (siehe §4).
* **Hardware:** Akai MIDImix wird im Plugin als Ein-/Ausgabegerät ausgewählt (CoreMIDI direkt,
  unabhängig vom Host-Routing), inkl. LED-Feedback.

## 2. Verifikations-Checkliste Audiotool/Bridge (Phase 0, NEU – blockiert Design-Detalles)

| # | Test | Frage | Werkzeug |
|---|---|---|---|
| T1 | Audiotool-Konto öffnen, VST Bridge installieren, Plugin-Ordner einlesen | Unterstützt die Bridge **VST3, VST2 oder beide**? Welche Scan-Pfade? | Bridge-UI + unsere Test-Plugins (leeres JUCE-Plugin in beiden Formaten) |
| T2 | Transport/Clock | Bekommt das Plugin in der Bridge korrekte Tempo/Play-Position (`ppqPos`)? | Debug-JUCE-Plugin, das ProcessContext-Felder loggt (FileLogger) |
| T3 | MIDI-Eingang durch die Bridge | Kommen MIDImix-Noten/CCs im Plugin an, wenn Audiotool sie routet? | Testplugin listet alle empfangenen Bytes |
| T4 | MIDI-Ausgang durch die Bridge | Kann Plugin-MIDI zurück zu einem Gerät geroutet werden? (wahrscheinlich nein → dann LEDs nur über §3) | Testplugin sendet Note-Ons auf Ausgangsbus |
| T5 | Direkter CoreMIDI-Griff | Darf/dann kann unser Plugin in der Bridge `MidiInput::start()` / `MidiOutput` auf MIDImix öffnen (Sandbox? Exclusive?)? | Testplugin mit Device-Picker |
| T6 | Audio-Gerät der Bridge | Welches CoreAudio-Device nutzt die Bridge (Built-in/Audio-MIDI-Setup-Agent)? Latenz/Puffergröße? | bridge-einstellungen + Messung Roundtrip |

**Regel:** Erst wenn T1–T5 dokumentierte Ergebnisse haben, fallen die finalen Entscheidungen
Clock-Modus (Standard), LED-Weg und Plugin-Formate. Bis dahin bauen wir alles so, dass beide
Wege möglich bleiben (Abstraktionen unten).

## 3. Geräte-Anbindung (R2: Auswahl im Plugin)

```
GUI-Tab "Controller":
  [MIDI Input ▼]  Liste aller CoreMIDI-Inputs (juce::MidiInput::getDevices())
                  → Standardmatch: Name enthält "MIDImix"
  [MIDI Output▼]  dito Outputs für LED-Sendungen
  [ ] Enable direct device access   (abschaltbar, falls Bridge/Host stört)
  Statuszeile: "MIDImix verbunden, letzte Activity: Note 10 ch1 vel 127"
```

* Implementierung mit `juce::MidiInput`/`MidiOutput` (`juce_audio_devices`) – läuft lokal in der
  Bridge-Prozess-Umgebung, nicht im Browser.
* **Konfliktfall:** Wenn Audiotool/Bridge das MIDImix bereits als MIDI-Source belegt, darf unser
  Direct-Access trotzdem mitlesen (CoreMIDI erlaubt mehrere Sources; Senden an den Output ist unkritisch).
  Getestet in T5. Fallback: Modus A (über Host-MIDI-Bus) bleibt im Code erhalten –
  Parser und LedDriver sind routing-agnostisch (sie arbeiten auf `RawMidipacket`-Level).
* Thread-Regeln: Callback des MidiInput → Ring-Buffer → UI-Thread (Controller-Logik);
  LED-Sende nie aus dem Audio-Thread, sondern aus dem Timer/UI-Thread (LEDs müssen nicht sample-genau).

## 4. Clock-Design (wegen Web-DAW-Unklarheit doppelt ausgelegt)

```
class ClockSource (abstract)
├── HostClock      : liest ppqPos/tempo/transportInfo aus process()     (präzise, DAW-synced)
└── InternalClock  : eigener Sample-Zähler + BPM-Parameter im Plugin    (host-unabhängig)
```

* Beide erzeugen intern gleichartige `StepTick{sampleOffset, stepIndex}`-Events in einer
  Lock-free Queue, die `process()` abarbeitet.
* User-Umschalter „Sync: Host / Internal“. Für Audiotool wählen wir nach T2 den zuverlässigen Modus
  als Default.
* Beat-Auflösung: 16 Steps = 1 Takt (4/4), Substep-Raster fester Sample-Offset
  (`samplesPerBeat/4`). Swing: gerade 16tel um Anteil nach hinten.

## 5. Datenmodell & Persistenz (JUCE)

* Pattern/Samplepfade/Parameter → binärer Block via `getInitialPageState`/`getCurrentProgramStateInformation`
  (ValueTree-basiert, versioniert; Migration wie üblich). Samples **nicht** einbetten in v1
  (nur Pfade + relative Suche), optional „Sample ins State einbetten“ als späteres Feature.
* ValueTree-Root `PluginState`: tracks[8]{name,samplePath,pitch,gain,pan,cutoff,res,envAmt,
  a,d,s,r, steps[16]{on}}, selectedTrack, swing, clockMode, bpm, midiInName, midiOutName.
* Undo/Redo über `UndoManager` (UI-Thread).

## 6. Offene Punkte R6/R7 – meine Empfehlung (bitte bestätigen oder überstimmen)

* **R6 (MIDI-Out an externe Geräte):** Vorschlag: In v1 **keine** separate MIDI-Out-Funktion;
  aber die Engine feuert interne TriggerEvents – ein später zuschaltbarer Ausgang
  („Spur n → Note C3+n auf Plugin-MIDI-Out“) ist damit trivial nachrüstbar. Kostet jetzt nichts.
* **R7 (Pattern-Ketten):** v1 = **ein Pattern**, aber das Datenmodell speichert
  `patterns[]`-Array mit Länge 1 – spätere 4 Pattern ohne Migrationsbruch. GUI zeigt ab v1
  einen A/B/C/D-Reiter nur deaktiviert (Planungs-Transparenz).

## 7. GUI (JUCE)

* Links: 8 Track-Strips (Name, Drop-Zone fürs Sample, Mute/Solo/Select, Mini-Parameter Pitch/Gain).
* Mitte: 8×16 Step-Grid (Klick toggles; aktuelle Spur hervorgehoben; Playhead-Spalte blinkt).
* Oben: Play/Stop, Sync-Mode, BPM (nur bei Internal), Swing, Clear/Randomize.
* Rechts/Tab: Controller (Device-Picker §3), Configure-MIDImix-Button (SysEx, Phase 6).
* Alle Werte spiegeln 1:1 die MIDImix-Belegung (docs/02 §3.2/3.3, unverändert gültig).

## 8. Build & Packaging

* CMake + JUCE 7/8 (`juce_add_plugin`, Formate: VST3 + AU + Standalone; VST2 nur falls T1 es verlangt
  – dann separates SDK, Lizenzhinweis beachten).
* Targets arm64+x86_64 universal; ad-hoc Signatur, optional Developer-ID+Notarization (Kostenpunkt:
  Apple-Dev-Account 99 $/Jahr → Rückfrage O-B).
* Testmatrix: Standalone (alle Funktionen), Reaper (Referenz-Host, VST3), Audiotool-Bridge (Ziel).

## 9. Lizenz-Hinweis JUCE (Rückfrage O-A)

JUCE ist GPLv3 **oder** kommerziell. Für privates, nicht verteiltes Selbst-Nutzen ist GPL kein Problem.
Sobald das Plugin weitergegeben/verkauft wird und der Quellcode nicht offen gelegt werden soll,
wird eine JUCE-Lizenz nötig (Tier nach Umsatz). Bitte bestätigen: nur Eigennutzung?
