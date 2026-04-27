# MANIFEST Template

Each generation folder contains a `MANIFEST.md` with this structure. Sections marked optional may be omitted if not applicable.

```md
# Generation: <kebab-label>

- Saved: <YYYY-MM-DD HH:MM local>
- Parent: <previous-generation-folder-name | none>
- Folder: <this folder name>

## Changed from parent
<1-3 lines describing what is new vs the parent generation. Be specific — mention the track or section.>

## Production Brief
- Genre/Reference:
- Mood/Intent:
- BPM/Groove:
- Key/Mode:
- Song Form:
- Section Lengths:
- Instrument Priorities:
- Vocal Plan:
- Mix Target:
- Constraints/Do-Not-Do:

## Form
| Bars  | Section       | Source clips |
|-------|---------------|--------------|
|  1-4  | Intro         | drums-intro, synth-intro |
|  5-16 | Verse 1       | drums-a, bass-a, gtr-a, synth-a |
| ...   | ...           | ... |

## Tracks / Plugins
| # | Name | Instrument / Plugin | Pan | Volume | Notes |
|---|------|---------------------|-----|--------|-------|
| 1 | Drums       | Superior Drummer 3        | 0    | 0.82 | |
| 2 | Bass        | Moog Mariana              | 0    | 0.78 | |
| 3 | Rhythm Gtr L | Tension "Drive Guitar"   | -0.55| 0.78 | |
| 4 | Lead Gtr R   | Tension "Hard Picked"    | +0.55| 0.74 | |
| 5 | Synth Pad    | u-he Zebra 3             | 0    | 0.62 | |

## Cue Points
- Bar 1   — Intro
- Bar 5   — Verse 1
- Bar 17  — Chorus 1
- Bar 33  — Verse 2
- Bar 45  — Chorus 2
- Bar 61  — Bridge / Solo
- Bar 69  — Final Chorus

## MIDI / Note Conventions
- start_time, duration: beats (quarter-note based)
- pitch: absolute MIDI number (E1=28, E2=40, E3=52, E4=64, E5=76)
- velocity: 0-127
- See `notes/` for per-clip note arrays. Each file matches the schema accepted by `add_notes_to_clip`.

## Session Clip Slot Map
| Track | Slot 1 | Slot 2 | Slot 3 | Slot 4 | ... |
|-------|--------|--------|--------|--------|-----|
| 1 Drums       | drums-intro (16b) | drums-a (48b) | drums-b (64b) | drums-bridge (32b) |
| ...

(Lengths in beats. 16 beats = 4 bars at 4/4.)

## Limitations / Caveats (optional)
<List anything not captured: e.g. plugin presets, automation lanes, audio effects after instruments, MIDI effects, send levels, etc.>

## Revision Options (optional)
<Carry forward from the songwriter handoff if available — what to try next.>
```

## Authoring Tips

- Keep "Changed from parent" tight: 1-3 lines, focused on what's audibly different.
- Put new ideas / open questions in "Revision Options" so the next generation has a starting point.
- Don't paste full note arrays into MANIFEST — they live in `notes/*.json`.
- If a track plugin isn't a true MIDI instrument (e.g., an amp/processor on a MIDI track), call it out explicitly under the Tracks table — that's the kind of subtle issue that causes silent tracks.
