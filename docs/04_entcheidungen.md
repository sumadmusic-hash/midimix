# Entscheidungen & Konsequenzen (Update nach Antwortrunde 2)

Datum: 2026-10-31. Beantwortung der Rückfragen R1–R7 durch den Nutzer;
offene Punkte daraus abgeleitet (O-Punkte). **Kein Code, nur Dokumentation.**

---

## 1. Antworten des Nutzers

| # | Frage | Antwort | Konsequenz für den Plan |
|---|---|---|---|
| R1 | Host? | **Audiotool (www.audiotool.com)**, Online-DAW mit „VST Bridge“ – dort wird das Plugin geladen | Komplette Neubewertung der Machbarkeit → Abschnitt 2 + docs/05 |
| R2 | Anbindung MIDImix | **Gerät wird im Plugin direkt ausgewählt** | Modus B (direkter CoreMIDI-Zugriff aus dem Plugin) ist Pflicht; Details/Risiken in docs/05 §3 |
| R3 | Tech-Basis | **JUCE** | Build via CMake/JUCE-Module `juce_audio_processors`, `juce_audio_devices` (CoreMIDI), `juce_gui_basics`. Lizenzfrage bleibt offen (siehe O-A) |
| R4 | Step-Mapping | **Bestätigt:** MUTE-Reihe = Steps 1–8, REC-ARM-Reihe = Steps 9–16 der selektierten Spur | docs/02 §3.2 Variante „Spalten=Spur“ fixiert |
| R5 | Sampler-Tiefe | **Drum-Sampler, 1 Sample pro Spur, mit Filter + ADSR + Pitch** | Voice-Design fixiert: 1-Shot-Wiedergabe, Pitch (Semitone+Cent), Amp-ADSR, Filter-Env/ADSR, Gain/Pan |
| R6 | MIDI-Out zu externen Geräten | **noch unklar** | Vorschlag in docs/05 §6 (Option A/B/C); Entscheidung hat nur Einfluss auf Phase 3, nichts blockiert |
| R7 | Pattern-Ketten | **unklar / keine Präferenz** | Empfehlung in docs/05 §6: v1 = 1 Pattern, aber Datenmodell erlaubt später A–D ohne Migration |

## 2. Kern-Konsequenz aus R1 (Audiotool) – ehrliche Einordnung

Audiotool ist eine browserbasierte DAW. Ein lokales macOS-VST3 kann dort nicht direkt
laufen; Audiotool nutzt dafür eine lokale **„VST Bridge“-Brücken-App**, die installierte
VST(-2/3)-Plugins in den Browser durchreicht. Das hat drei tiefegreifende Folgen,
die wir **vor jeder Zeile Code verifizieren müssen** (Phase 0, Checkliste in docs/05):

1. **MIDI-Eingang:** Kommt MIDImix-MIDI überhaupt durch die Bridge im Plugin an?
   (Normaler Weg: Audiotool zeigt das Gerät als MIDI-In an und routet auf die Plugin-Spur.)
2. **MIDI-Ausgang / LED-Feedback:** Sendet die Bridge empfangene MIDI-Ausgangs-Events
   des Plugins zurück an ein wählbares Gerät? Wenn nein → LEDs müssen über einen
   **direkten CoreMIDI-Endpunkt** im Plugin angesteuert werden (genau das ist ja
   desiderata R2: „Midimix wird im Plugin direkt ausgewählt“).
3. **Direkter Gerätezugriff aus der Bridge-Umgebung:** Die Bridge-App läuft lokal auf
   der Mac und lädt unser Plugin in sich → unser Plugin kann dort prinzipiell
   CoreMIDI-Geräte öffnen (`juce::MidiInput/MidiOutput`). Risiken: Sandbox der Bridge-App,
   Exclusive-Access auf das MIDImix, Latenz/Thread-Fragen. → Verifikationstest T2/T3.

Zusätzlich offen: Unterstützt die Audiotool-Bridge aktuell **VST3 oder nur VST2**?
Das entscheidet, ob wir zwei Plugin-Targets bauen müssen (JUCE kann beides aus
demselben Code: `set_format(VST3)` + `VST2`-Target; VST2 erfordert die alte
VST2-SDK von Steinberg, Weitergabe eingeschränkt → rechtlicher Hinweis nötig).

## 3. Was sich am Architekturplan ändert (Delta zu docs/02/03)

* **Clock:** In einer Web-DAW mit Bridge ist `ProcessContext.ppqPos` ggf. unzuverlässig
  oder gar nicht gefüllt (Bridge-Transport-Sync unbekannt). Plan: SequenceEngine bekommt
  eine Abstraktion `ClockSource` mit zwei Implementierungen:
  - `HostClock` (ppqPos-basiert, wie bisher geplant) und
  - `InternalClock` (Samplezähler + Tempo-Parameter, host-unabhängig; Fallback, wenn
    Bridge-Transport nicht sauber durchgereicht wird).
  Beide laufen sample-genau im Audio-Callback; kein SystemTimer für Audio.
* **Controller-Schicht:** Statt „Plugin-MIDI-Bus“ (Modus A) wird **Modus B primär**:
  `MidiDevicePicker` im GUI (Liste via `juce::MidiInput::getDevices()`), Input-Port
  = MIDImix, Output-Port = MIDImix (LEDs). Parallel bleibt der normale Plugin-MIDI-
  Eingang aktiv (Noten zum Testen/Triggern), damit funktioniert auch jeder andere Host.
* **GUI:** JUCE statt VSTGUI. 8×16-Grid, Track-Strips, Device-Picker, Transport.
* **Phasenplan:** Phase 0 wächst um die Audiotool-spezifischen Verifikationstests
  (docs/05 §2, Tests T1–T5). Erst danach fällt die endgültige Clock-/Routing-Entscheidung.

## 4. Repo-Hinweis

Die Dokumente docs/02 (Kapitel „Architektur (VST3)“ mit VSTGUI/Steinberg-SDK) und
docs/03 (Phase 1/2 mit VST3-SDK) sind dadurch teilweise überholt. Sie bleiben als
Historie unverändert; maßgeblich sind ab jetzt **docs/04 und docs/05**.
