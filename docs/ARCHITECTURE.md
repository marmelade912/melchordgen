# melchordgen architecture

melchordgen is a VST3 audio-effect plugin (JUCE, C++17) with MIDI output, plus a Python training and evaluation pipeline that lives outside the plugin. The plugin runs inside the DAW on an audio track; it listens to that track, works out the musical context, and emits MIDI that the host can record or that the user can drag out as a `.mid` file.

## Signal and data flow inside the plugin

```
audio in (host track)
   |
   v
LoopCapture ----------> captured buffer (one loop, transport-aligned)
   |
   v
HpcpChroma -----------> 12-bin chroma per frame (harmonic pitch class profile)
   |               \
   v                v
KeyDetect        ChordDetect -----> chord sequence with bar positions
   |                |
   v                v
MusicalContext  (key, chords, energy / density / register / groove descriptors)
   |
   v
Generator -------> candidate notes (rule-based first; model-backed path planned)
   |
   v
ConstraintLayer -> in-key, chord-aligned, register and density limits, voice-leading rules
   |
   v
GenSchedule / TriadSchedule -> notes placed on the host grid, sync-locked to transport
   |                                 |
   v                                 v
MIDI out to the host track      MidiExport -> drag-out .mid file
```

Module map (one header per responsibility):

| Module | Responsibility |
|---|---|
| `LoopCapture` | Captures one loop of the track's audio aligned to the host transport (bars and beats from `getPlayHead()`), so analysis always sees whole bars. |
| `Fft`, `HpcpChroma` | Short-time FFT and harmonic pitch class profile: folds spectral energy into 12 pitch classes per frame, the input for key and chord detection. |
| `KeyDetect` | Correlates the aggregated chroma against key profiles and returns the key with a confidence; the UI lets the user override it. |
| `ChordDetect`, `ChordInitiation` | Segments the chroma timeline into chords, snaps boundaries to beats, and decides when a chord actually starts versus when a passing tone is sounding. |
| `CharacterDescriptors`, `MovementBands` | Reads energy, note density, register, and groove from the loop so the generated part matches the feel of what is playing. |
| `MusicalContext` | The single struct the generator consumes: key, chord timeline, descriptors, tempo, and time signature. |
| `Generator`, `ConstraintLayer`, `SequenceAvoid` | Produces candidate chords and melody, then filters and repairs them: stay in key, agree with the current chord, respect register and density targets, avoid repeating the same figure, resolve chromatic notes. |
| `GenSchedule`, `TriadSchedule` | Turns note candidates into timed MIDI events on the host grid, so playback is sample-accurate against the transport. |
| `MidiExport` | Writes the generated part to a standard MIDI file for drag-and-drop into any DAW, the host-agnostic fallback. |
| `SidecarClient` | Optional link to an out-of-process helper for heavier analysis or model inference, kept off the audio thread. |
| `PluginProcessor`, `PluginEditor`, `ScreenLine` | JUCE plumbing: parameters, state, the UI, and the status line that shows detected key and chords with manual override controls. |

Design rules that shaped this:

- Nothing that can block runs on the audio thread. Capture and analysis are staged; heavier work goes through the sidecar.
- The host transport is the only clock. Tempo, beat, and downbeat come from the DAW, not from onset detection, which keeps generated parts locked to the session.
- Two ways out for the notes: live MIDI to an armed track, and a `.mid` file. Either one works in every major DAW.
- Manual override is first-class. If key or chord detection is wrong, the user corrects it in the UI and generation follows the correction.

## Outside the plugin: the Python pipeline

The `training/` tools build the data the generator learns from and the harness that scores it:

- Harvesting: parse Ableton Live project files (`.als`) and MIDI to extract chord and melody material, sections, and pitch context.
- Dataset building: assemble, transpose-augment, and label the harvested material; build movement profiles and connection statistics between chords.
- Back-testing: replay the generator against held-out loops and score the output with the same constraints the plugin applies.
- Evaluation: the blind rating rounds described in `EVALUATION-METHOD.md`.

## Build

CMake 3.22+, JUCE 8, a C++17 compiler (Visual Studio 2019 or later on Windows). The plugin targets VST3 on Windows first; macOS follows the same CMake project. Source is private; this repository documents the design and the evaluation method.
