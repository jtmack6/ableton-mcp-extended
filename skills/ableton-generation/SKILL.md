---
name: ableton-generation
description: Snapshot and restore Ableton "generations" — full creative state captures (tempo, tracks, plugins, clip notes, arrangement, mix) saved as plain JSON for later iteration, A/B comparison, or post-crash recovery. Distinct from code/dev saves (use git for those).
---

# Ableton Generation (Save / Load)

## Objective
Persist the full creative state of an Ableton session as a portable, plain-text snapshot — independent of Ableton's `.als` save — so that:
- musical iterations are versioned and named explicitly,
- a crash or "I forgot to save" no longer loses work,
- past generations can be restored or A/B-compared.

This is **session saving for generative work**. It is not a substitute for `.als` saves and not a substitute for git.

## When to Use
- User invokes `/save-generation <label>` to snapshot the current state.
- User invokes `/load-generation <label>` to rebuild a saved generation.
- After any substantial musical revision (instrument swap, structural change, transposition, new section) the assistant may *offer* a save — but never saves silently.

## Do Not Use For
- Code changes — use git.
- Single trivial parameter tweaks — wait for a meaningful checkpoint.
- Replacing the user's Cmd+S habit — always remind them to also save the `.als`.

## Core Rules
1. Every save is **full state**. No diff-only saves. Each generation folder is self-contained.
2. Folders are **date-prefixed** for chronological listing: `YYYY-MM-DD_HHMM_<kebab-label>`.
3. Saves are **explicit only**. Never auto-save. Never overwrite an existing folder — append a numeric suffix if a collision occurs.
4. Generative saves live in `generations/` at the project root. Do not commit them in dev-focused PRs unless explicitly requested.
5. After every save, **remind the user to Cmd+S the `.als`** — Claude cannot trigger Ableton's save.

## Folder Layout
```
generations/
  2026-04-27_1430_bass-octave-up/
    MANIFEST.md          (human-readable summary)
    state.json           (machine-readable full state)
    notes/
      drums_intro.json
      drums_a.json
      bass_a.json
      ...                (one JSON file per session clip with notes)
```

- `MANIFEST.md` is the source of truth for humans (form, tracks, what changed).
- `state.json` is the source of truth for machines (used by `/load-generation`).
- `notes/*.json` files are individual clip note arrays — same schema as `add_notes_to_clip` accepts: `[{pitch, start_time, duration, velocity}, ...]`.

## Save Workflow (`/save-generation <label>`)

1. **Validate label.** Lowercase kebab-case, no spaces. If missing or invalid, ask the user for one. Suggested: 1-4 words describing what's new ("bass-octave-up", "added-bridge", "synth-to-zebra3").

2. **Compute folder name** = `YYYY-MM-DD_HHMM_<label>` from current date/time. Check for collision; if exists, append `-2`, `-3`, etc.

3. **Pull current state** via MCP tools (in parallel where possible):
   - `get_session_info` → tempo, time signature, track count
   - `get_track_info` for each track → name, devices, mixer (vol/pan/mute/solo), clip slot summary
   - `get_arrangement_info` → arrangement clip placements
   - `get_cue_points` → cue list
   - For each session clip with notes: capture the notes (assistant typically already has them in conversation context or in `/tmp` from generation; if not, attempt to read; if unrecoverable, note this in MANIFEST as "notes not captured")

4. **Ask the user (1-2 sentences) what changed from the last generation.** Keep it brief — this becomes the "Changed from parent" section. If there's no obvious parent, ask which previous generation this is descended from (or "none" if first).

5. **Write the folder:**
   - `MANIFEST.md` — populated from [manifest-template.md](references/manifest-template.md)
   - `state.json` — see [state-schema.md](references/state-schema.md)
   - `notes/*.json` — one file per session clip, named after the clip (kebab-cased): `bass-a.json`, `drums-intro.json`, etc.

6. **Final reminder:** "Save the .als now (Cmd+S) so Ableton's project file matches this snapshot."

## Load Workflow (`/load-generation <label-or-folder-name>`)

1. **Resolve target.** Match against folder names in `generations/` (substring or exact). If multiple match, list them and ask the user to pick.

2. **Read MANIFEST.md and state.json** to understand what to rebuild.

3. **Confirm before mutating.** Show a brief summary of what will be created (X tracks, Y clips, arrangement length, plugins) and ask the user to confirm before running. **Do not** automatically wipe their current session — ask whether to (a) load into the current empty session, (b) load into a fresh new session (user creates manually), or (c) abort.

4. **Rebuild in this order:**
   - Set tempo + time signature
   - Create/rename tracks
   - Load instruments and effects (external plugins by name; stock devices by URI from state.json)
   - Set track mix (volume, pan, mute, solo)
   - Create session clips and add notes from `notes/*.json`
   - Place arrangement clips (delete existing first if any, in highest-index-first order)
   - Add cue points (skip duplicates if Live rejects them — known quirk)

5. **Final reminder:** "Verify playback then Cmd+S to write the rebuilt state to the .als."

## Naming Conventions

- **Folder names:** `YYYY-MM-DD_HHMM_<kebab-label>`. Date is local time. Do not use 24h colons (`:`) — use `HHMM` packed.
- **Note files:** kebab-case clip name + `.json`. Strip trailing version markers from clip names (`Bass A +12` → `bass-a-plus-12.json`). Track index is recorded in `state.json`, not the filename.
- **Labels:** short, factual. Prefer "bass-octave-up" over "fixed-bass". Prefer "added-bridge" over "v2".

## Distinction from Dev / Code Saves

| Concern              | Tool        | Trigger                          |
|----------------------|-------------|----------------------------------|
| Music iteration      | this skill  | user runs `/save-generation`     |
| Code change          | git         | `git commit` after edits         |
| Ableton project file | `.als`      | user hits Cmd+S in Live          |

These are orthogonal. A single Claude session may produce all three; they don't interfere.

## Guardrails
- Never run `/save-generation` automatically. Always require user invocation.
- Never overwrite an existing generation folder. Always append a suffix.
- Never `git add` or `git commit` files inside `generations/` unless the user explicitly asks.
- Never delete a generation folder without explicit user confirmation including the folder name.
- If state capture is incomplete (e.g., notes not recoverable for some clips), record this honestly in MANIFEST under "Limitations" — do not pretend the snapshot is complete.

## Offering Saves Proactively

After completing a meaningful musical change (new section, instrument swap, transposition), end the response with a short offer:

> Save this as a generation? Run `/save-generation <suggested-label>`.

Skip the offer for trivial changes, mid-iteration states, or when the user is mid-task. Do not nag.

## References
- MANIFEST template: [manifest-template.md](references/manifest-template.md)
- state.json schema: [state-schema.md](references/state-schema.md)
