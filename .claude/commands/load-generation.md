---
description: "Rebuild a saved Ableton generation from generations/ into the current session — full state restore"
---

Use the Ableton generation load workflow defined in:
- `skills/ableton-generation/SKILL.md`
- `skills/ableton-generation/references/state-schema.md`

Execution requirements:
- Resolve the target generation folder under `generations/`. Match the argument as substring against folder names. If multiple match, list them and ask the user to pick exactly one.
- Read `MANIFEST.md` (for humans) and `state.json` (for the rebuild).
- Show a brief plan summary (tracks, plugins, # clips, arrangement length, cue points) and CONFIRM with the user before mutating the session.
- Ask whether the current session is empty or should be wiped — DO NOT auto-wipe. If the session has content, ask explicitly how to proceed (load into empty session / abort).
- Rebuild order: tempo+sig → tracks (create + rename) → instruments/effects (external by name, stock by URI) → mixer settings → session clips + notes → arrangement clips (delete-then-place if conflicts) → cue points.
- For unrecoverable items (e.g., plugin presets, automation), report what couldn't be restored in the final summary.
- After rebuild, remind the user: "Verify playback then Cmd+S to write the rebuilt state to the .als."

Generation label or folder name (substring OK):
`$ARGUMENTS`

If the argument is missing, list all folders under `generations/` and ask the user to pick one.
