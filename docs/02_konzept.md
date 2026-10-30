# Konzept: "MidiMixSeq" – VST3-Sampler mit 8 Spuren + MIDImix-Integration

Zielplattform: **macOS** (VST3; AU als optionaler späterer Schritt). Keine Abhängigkeit von Ableton.

---

## 1. Produktdefinition

Ein VST3-Instrument-Plugin:
- **8 Sampler-Spuren** (je Spur: 1 Sample/Zone, Level, Pan, Filter, Tuning, Hüllkurve …)
- **Step-Sequencer**: 8 Spuren × 16 Steps, je Step: On/Off (+ Velocity/Accent in Ausbaustufe)
- Eigene Audio-Wiedergabe der Samples (Sample-Playhead pro Trigger)
- Eigener Clock-/Scheduler-Layer mit Swing & Tempo (host-synced über `IAudioProcessor::process` Position)
- MIDI-Controller-Ebene nur für den Akai MIDImix (Eingang + SysEx-Setup + LED-Ausgang)
- Extra-MIDI-Ausgang: Der Sequencer kann die 8 Spuren parallel als Note-Ons auf einen externen Synth/Kanal senden (Doppelbetrieb intern+extern = Option im Plugin)

Abgegrenzt (v1 NICHT): Multisample-Keymapping pro Spur, Effekte außer Grundfilter, Pattern-Ketten >1, Automationsspur-Editing via Hardware.

## 2. Datenmodell

```
PluginState
├── transport: tempo(Host), running, swing 0..1, resolution=16th
├── pattern: PatternData          → serialisiert in State (VST3 State / .json)
│   ├── tracks[8]
│   │   ├── name, samplePath, rootPitch, gain, pan, filterCutoff, filterRes,
│   │   │   ampEnv(a,d,s,r), outChannel, mute, solo
│   │   └── steps[16]: { on:bool, vel:0..127, accent:bool }   // v1: nur on
├── ui: selectedTrack 0..7, page 0..1 (Steps 1-8 / 9-16 bei Mini-Modus)
└── controller: ControllerConfig  → nicht in State, sondern separate Preset-Datei
    ├── inputPortName/outputPortName, midiChannelFilter, learnMode
    └── templateApplied: bool     (wurde SysEx 0x64 schon gesendet?)
```

## 3. MIDImix-Mapping (Design-Entscheidung)

### 3.1 Grundsatz
Wir nutzen das **Original-CC/Note-Raster** des MIDImix und legen es im Plugin fest
(vorhersehbar, ohne Editor zwingend). Der Editor/SysEx wird nur für optionale
Ergänzung benutzt (siehe 3.4). Alle Buttons bleiben auf **Noten-Modus**, Kanal 1.

### 3.2 Step-Matrix: 16 beleuchtete Tasten = 16 Steps einer Spur

Anordnung auf dem Gerät: Spalten 0–7, je 2 Reihen (MUTE oben, REC ARM unten).

| Physische Taste | Note-ID | Sequencer-Funktion |
|---|---|---|
| MUTE Spalte s | 1,4,7,10,13,16,19,22 | Step **(s, Reihe A)** togglen |
| REC ARM Spalte s | 3,6,9,12,15,18,21,24 | Step **(s, Reihe B)** togglen |

Damit ergeben die 16 Tasten **ein 8×2-Raster**. Zwei mögliche Lesarten – Entscheidung offen (Rückfrage R4):

- **Variante „8×2-Sicht“ (empfohlen für v1):** Die 16 Tasten zeigen immer einen 16-Step-Ausschnitt, aber spaltenweise: MUTE-Reihe = Steps 1,3,5…, REC-Reihe = Steps 2,4,6… einer einzigen (selektierten) Spur? → unübersichtlich.
- **Variante „Spalten=Spur“:** MUTE-Taste Spalte s toggled Step s der aktuellen Spur, REC-Taste Spalte s toggled Step s+8 der aktuellen Spur. ⇒ **16 Steps genau eine Vollbank, kein Blättern nötig.** ✔ empfohlen
  - LED-Zustand: MUTE-LED[s] = step[s].on, REC-LED[s] = step[s+8].on (gespeicherte Schritte leuchten).
  - Playhead: beim Laufen wird zusätzlich die Current-Step-LED „überbelichtet“ (an/aus im Clock-Takt; da LEDs nicht dimmbar [PROT S.5], blinkt aktueller Step, gesetzte Steps bleiben an, wenn möglich per Toggle-Sequence).

### 3.3 Restliche Bedienelemente

| Element | Nutzung |
|---|---|
| 8 Fader (CC 19,23,27,31,49,53,57,61) | Lautstärke der 8 Spuren (absolut, 0–127→Gain) |
| Master-Fader (CC 62) | Master-Output-Gain |
| Encoder Reihe 1 (CC 16,20,…,58) | Cut-off Filter der selektierten Spur? Oder je Spalte: Track-Parameter 1 aller 8 Spuren – **Vorschlag: pro Spalte = Spur ⇒ Encoder R1=Cutoff, R2=Resonanz, R3=Amp-Decay der jeweiligen Spur** (8×3 Parameter gleichzeitig) |
| Bank Left / Right (Notes 25/26) | **Spurauswahl** (selectedTrack −/+) – erfüllt Anforderung „Spuren schaltet man mit den anderen verfügbaren Buttons um“ |
| Solo-Taste (Note 27) | Modifikator: Hold + Step-Taste = Velocity/Accent ändern (später); Hold + Fader = Pan statt Volume (Pickup-Modus) |
| Send All (Note 62) | ignorieren bzw. „Parameter-Abhören“ (alle Werte ins Plugin übernehmen) |

Hinweis: Bank Left/Right und Solo sind per SysEx **nicht umkonfigurierbar** [PROT S.6] – passt, wir nutzen sie ohnehin.

### 3.4 Optionales SysEx-Hardware-Setup (Komfort, Phase 5)
Das Plugin kann beim Start (Button „Configure MIDImix“) eine 0x64-Configuration-Set-Nachricht senden, damit:
- alle 16 Buttons sauber auf Noten-Kanal 1 stehen (Default), und
- (optional) MUTE+SOLO-Virtual-Buttons auf IDs gesetzt werden, die wir als zweite Step-Bank zweckentfremden können.
Realisierung als eigener Modul `midimix_sysex.{h,cpp}` mit Encoder/Decoder für 0x60/0x64/0x66/0x67 (Format exakt in docs/01, Abschnitt 4). Vorher mit MIDImix Editor gegenprüfen (`.midimix`-Dateien = 0x67-Format [PROT S.14]).

## 4. Architektur (VST3)

```
┌───────────────────────────── VST3 Plug-in (Steinberg VST3 SDK 3.7.x) ─────────────────────────────┐
│                                                                                                    │
│  MidiMixSeq : public audioEffect::IAudioProcessor,                                                  │
│               public IComponentOne (EditController)                                                 │
│                                                                                                    │
│  ├─ Processor (Audio/MIDI-Thread)                                                                   │
│  │   ├─ SequenceEngine   – zähll samples-genau Steps aus HostPosition (PPQ-Umrechnung),             │
│  │   │                     erzeugt interne TriggerEvents (Queue → process())                        │
│  │   ├─ VoicePool        – 8 Stimmen (1/Spur, polyOption aus), Sample-Playback,                     │
│  │   │                     ADSR, OnePole/Ladder-Filter, Gain/Pan                                    │
│  │   ├─ Mixer            – Summierung, Master-Gain                                                   │
│  │   └─ EventBuses       – [in] MIDI (Noten zum Triggern/Learn), [out] Audio Stereo,                │
│  │                          [out] optionaler MIDI-Bus „Sequencer Out“                               │
│  ├─ Controller (UI-Thread)                                                                          │
│  │   ├─ MidimixInput     – lauscht Plugin-MIDI-Eingang auf CC/Notes nach Tabelle 3.2/3.3           │
│  │   ├─ LedDriver        – sendet 0x90 id vel über dieselbe Busverbindung zurück ans Gerät         │
│  │   ├─ UiModel          – Pattern-Model + Undo, spiegelt Zustand in GUI                            │
│  │   └─ GUI              – VSTGUI 6: 8 Zeilen × 16 Spalten Grid, Track-Liste, Transport            │
│  └─ Common: PatternData, ParameterDefs (kIsAutomatable etc.), State-Migration                       │
└────────────────────────────────────────────────────────────────────────────────────────────────┘
```

Wichtige technische Punkte:
1. **Clock:** Kein eigener Timer für Audio! Steps werden aus `ProcessContext.ppqPos/tempo/samplesPerSecond` abgeleitet (host-synced). Freier Lauf („Free Run“) nur als Notbehelf via SystemTimer (genau dann dokumentierte Latimitationen).
2. **MIDI-Rückweg zum MIDImix:** Ein VST3-Plugin hat keinen direkten CoreMIDI-Zugriff auf Geräte außerhalb der DAW. Zwei Betriebsmodi:
   - **Modus A (in der DAW):** MIDImix hängt als MIDI-In/Out an der DAW; Plugin bekommt Noten über seinen MIDI-Eingang und sendet LED-Noten über seinen MIDI-Ausgang (Bus „to MIDImix“), den der User in der DAW aufs Gerät routet. ← Standardfall, funktioniert überall.
   - **Modus B (direct/CoreMIDI):** Plugin öffnet per CoreMIDI (VST3 host-independent) ein Source/Endpoint-Handle direkt. Geht, ist aber heikel (Exclusive Access, Sandbox der DAW, App-Transport-Security). Nur als Fallback/Eigenbau-Standalone denken. → Rückfrage R2.
3. **Threading:** UI-Änderungen an Steps müssen atomisch in den Audio-Thread (Lock-free Ring-Buffer oder doppelte Puffer mit Copy-on-Cycle-Boundary).
4. **Persistence:** Pattern + Sample-Pfade in VST3-State (Base64-Chunks wie üblich (Chunk-basiert)); Sample-Daten selbst NICHT ins State (nur Pfade + Load via `IHostApplication`/Dateidialog; optional Resampling-Einbettung später).
5. **Parameter:** Jede Spur 6 Parameter ×8 + Master = 49 Parameter → VST3-ParameterIDs nach Schema `(track*16 + param)`; alle automationsfähig.

## 5. GUI (minimal, weil Hardware primär)

- 8×16 Step-Grid (klickbar, Rechtsklick=löschen, Shift=Velocity-Stufen in späteren Versionen)
- Track-Strip links: Name, Stummschalt-, Solo-, Select-Button, Sample-Loader
- Top: Play/Stop, Tempo-Anzeige (read-only, kommt vom Host), Swing-Regler, Resolution (16 fixed in v1)
- Controller-Tab: Port-Auswahl (nur Modus B), „Send Template to MIDImix“, Learn-Modus

## 6. Meilensteine (Plan siehe 03_plan.md)

M0 Recherche/Protokoll-Verifikation → M1 SDK-Gerüst & stummer Sampler → M2 Sequencer-Core → M3 MIDImix-Input → M4 LED-Feedback → M5 SysEx-Template → M6 Stability/Docs → M7 Packaging.

## 7. Offene Rückfragen (Stand dieses Dokuments)

- **R1** DAW auf macOS, in der das VST3 laufen soll (Logic unterstützt offiziell kein VST3!, GarageBand auch nicht) → welche Host-App? (Waveform, Reaper, Studio One, Bitwig, Cubase …?)
- **R2** Betriebsmodus A (über DAW-MIDI-Routing) ok, oder willst du Modus B (Plugin redet direkt mit CoreMIDI am Host vorbei)?
- **R3** C++ mit Steinberg VST3 SDK + VSTGUI ok? (Alternative JUCE – kommerzielle Lizenzkosten ab v5; favorisiere SDK pur oder JUCE, wenn Lizenz vorhanden)
- **R4** Step-Mapping-Variante 3.2 bestätigen: MUTE-Reihe = Steps 1–8, REC-Reihe = Steps 9–16 der gewählten Spur?
- **R5** Samples: mono/stereo, vorab geladen per Drag&Drop in die Spur – reicht „1 Sample pro Spur“ für v1?
- **R6** Soll der Sequencer zusätzlich MIDI-Noten ausgeben (externe Synths steuern) oder rein interner Sampler?
- **R7** Brauchst du Pattern-Ketten/Varianten (A/B/C/D) bereits in v1?
