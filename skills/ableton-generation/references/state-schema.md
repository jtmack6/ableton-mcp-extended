# state.json Schema

Machine-readable full-state capture. Used by `/load-generation` to rebuild the session.

```json
{
  "generation": {
    "label": "bass-octave-up",
    "saved_at": "2026-04-27T14:30:00-07:00",
    "parent": "2026-04-27_1200_extended-84bar"
  },
  "transport": {
    "tempo": 180.0,
    "signature_numerator": 4,
    "signature_denominator": 4
  },
  "tracks": [
    {
      "index": 1,
      "name": "Drums",
      "is_midi_track": true,
      "volume": 0.82,
      "panning": 0.0,
      "mute": false,
      "solo": false,
      "devices": [
        {
          "kind": "external_plugin",
          "name": "Superior Drummer 3"
        }
      ],
      "session_clips": [
        { "slot": 1, "name": "Drums Intro",  "length_beats": 16, "notes_file": "notes/drums-intro.json" },
        { "slot": 2, "name": "Drums A",      "length_beats": 48, "notes_file": "notes/drums-a.json" },
        { "slot": 3, "name": "Drums B",      "length_beats": 64, "notes_file": "notes/drums-b.json" },
        { "slot": 4, "name": "Drums Bridge", "length_beats": 32, "notes_file": "notes/drums-bridge.json" }
      ]
    },
    {
      "index": 2,
      "name": "Bass",
      "is_midi_track": true,
      "volume": 0.78,
      "panning": 0.0,
      "mute": false,
      "solo": false,
      "devices": [
        { "kind": "external_plugin", "name": "Mariana" }
      ],
      "session_clips": [...]
    },
    {
      "index": 3,
      "name": "Rhythm Gtr L",
      "is_midi_track": true,
      "volume": 0.78,
      "panning": -0.55,
      "devices": [
        { "kind": "stock_instrument", "uri": "query:Synths#Tension:Guitar%20&%20Plucked:FileId_33772", "display_name": "Tension - Drive Guitar" }
      ],
      "session_clips": [...]
    }
  ],
  "arrangement": {
    "clips": [
      { "track": 1, "session_slot": 1, "start_bar": 1,  "name": "Drums Intro" },
      { "track": 1, "session_slot": 2, "start_bar": 5,  "name": "Drums A" },
      { "track": 1, "session_slot": 3, "start_bar": 17, "name": "Drums B" },
      { "track": 2, "session_slot": 5, "start_bar": 5,  "name": "Bass A +12" }
    ]
  },
  "cue_points": [
    { "bar": 1,  "name": "Intro" },
    { "bar": 5,  "name": "Verse 1" },
    { "bar": 17, "name": "Chorus 1" }
  ],
  "limitations": [
    "Plugin presets inside Mariana / Zebra 3 / Superior Drummer 3 are not captured — they default to whatever the plugin loads with.",
    "MIDI / audio effect chains beyond the primary instrument device are not captured.",
    "Automation lanes are not captured."
  ]
}
```

## Field Notes

### `tracks[].devices[]`
Each device entry has a `kind`:
- `"external_plugin"` — VST/AU loaded by name. Required: `name`. Reload via `load_external_plugin`.
- `"stock_instrument"` / `"stock_effect"` — Ableton built-in. Required: `uri`. Reload via `load_instrument_or_effect`. Optional: `display_name` for human readability.

### `tracks[].session_clips[]`
- `slot` is 1-based.
- `notes_file` is a path relative to the generation folder. Empty session slots can be omitted from the array.
- For arrangement-only clips that are NOT mirrored to a session slot, they still need a session slot for restore — the load workflow creates a temporary session clip, places it into the arrangement, and leaves the session slot populated.

### `arrangement.clips[]`
- `start_bar` is 1-based.
- `session_slot` references which session slot the arrangement clip was duplicated from. The load workflow uses `duplicate_clip_to_arrangement(track, session_slot, start_bar)`.

### `cue_points[]`
- `bar` is 1-based.
- Live's API sometimes rejects cue points that conflict with internal positions; the load workflow logs and continues if a cue point can't be created.

## What's NOT Captured

- Plugin internal patches/presets (VST state) — only the plugin identity.
- Automation lanes.
- Audio clips and audio effect parameters.
- Return tracks and sends.
- Master track settings beyond volume/tempo.
- Warp markers, clip envelopes.

If any of these become important, extend the schema and capture them — but always note in MANIFEST when something is incomplete.
