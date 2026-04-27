---
description: "Snapshot the current Ableton creative state into generations/<date>_<label>/ — for musical iterations, NOT code changes"
---

Use the Ableton generation save workflow defined in:
- `skills/ableton-generation/SKILL.md`
- `skills/ableton-generation/references/manifest-template.md`
- `skills/ableton-generation/references/state-schema.md`

Execution requirements:
- This is GENERATIVE saving for musical work. Do NOT touch git. Do NOT run `git add` or `git commit`.
- Pull the current state via AbletonMCP tools (get_session_info, get_track_info per track, get_arrangement_info, get_cue_points).
- Folder name format: `YYYY-MM-DD_HHMM_<kebab-label>` under `generations/` at project root.
- Never overwrite an existing folder — append `-2`, `-3`, etc. on collision.
- Write `MANIFEST.md`, `state.json`, and `notes/*.json` (one per session clip with notes the assistant has access to).
- Ask the user (briefly) what changed from the previous generation. Record it in MANIFEST under "Changed from parent".
- After saving, remind the user: "Save the .als now (Cmd+S) so Ableton's project file matches this snapshot."

Label (kebab-case, 1-4 words describing what's new):
`$ARGUMENTS`

If the label is missing or invalid (spaces, uppercase, special chars), ask for one before proceeding.
