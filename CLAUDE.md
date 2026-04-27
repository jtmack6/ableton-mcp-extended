# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`ableton-mcp-extended` exposes Ableton Live as an MCP server. Natural-language commands from an AI assistant are routed through `MCP_Server/server.py` (FastMCP, runs on the host) which talks over a TCP socket to a Live Remote Script (`AbletonMCP_Remote_Script/__init__.py`) loaded inside Ableton.

## Architecture

Two processes, one socket. Knowing this split is essential before changing anything:

- **`MCP_Server/server.py` (host process, Python 3.10+)**: FastMCP server registering ~45 `@mcp.tool()` endpoints. Builds a JSON command, sends it over TCP to `localhost:9877`, parses the JSON response, returns a string to the AI client. Stateless except for the singleton `_ableton_connection` and an external-plugin discovery cache (`_EXTERNAL_PLUGIN_CACHE_TTL_SECONDS = 120s`, invalidated on disconnect).
- **`AbletonMCP_Remote_Script/__init__.py` (runs inside Ableton, Python 2/3 compatible)**: A `_Framework.ControlSurface` subclass that listens on TCP 9877. Receives JSON commands and dispatches to `_<command>` methods. **Live API state mutations must run on the main thread**: the script uses a `queue.Queue` + `self.schedule_message(0, main_thread_task)` pattern to marshal state-modifying calls (any command in the long whitelist near `_process_command`). Read-only commands run inline on the socket thread. When adding a state-modifying command, you must update **both** whitelists — the one in `MCP_Server/server.py:107` (`is_modifying_command` adds a 100ms post-send delay and longer timeout) and the one in `AbletonMCP_Remote_Script/__init__.py:229` (which decides whether to schedule on the main thread).
- **`Ableton-MCP_hybrid-server/AbletonMCP_UDP/__init__.py`**: Optional second Remote Script that adds a UDP listener on 9878 alongside the TCP one (port 9877 is shared with the standard script — only install one of them, or use this one to get both protocols). Used by `experimental_tools/xy_mouse_controller/` for low-latency continuous parameter control.

### Adding a new MCP tool

1. Add `@mcp.tool() def my_tool(ctx: Context, ...)` in `MCP_Server/server.py`. Convert 1-based caller indices to 0-based with `_to_zero_based()` / `_optional_to_zero_based()` before sending to the Remote Script.
2. If the tool mutates Live state, append the command type to the `is_modifying_command` list in `send_command()` (`server.py:107`).
3. In the Remote Script, add a `_my_command()` method on `AbletonMCP` and route it from `_process_command()`. If it touches the Live API/song model, add the command type to the main-thread whitelist (`__init__.py:229`) so it's scheduled via the `response_queue` pattern; otherwise dispatch inline.
4. Use `normalize_param`/`denormalize_param` for device parameter values (caller-facing API is always 0.0–1.0). Use `bar_to_beat`/`beat_to_bar` for transport/arrangement positions (caller-facing API is 1-based bars).

### Index/parameter conventions

- All MCP tools accept **1-based** track/clip/device indices; the Remote Script always works in **0-based** indices. Don't bypass `_to_zero_based()`.
- Bar numbering is **1-based** (bar 1 = beat 0). The conversion respects time signature numerator/denominator.
- Device parameter values exposed externally are normalized to [0.0, 1.0]; the Remote Script converts to/from each parameter's raw `min`/`max` range.

### Plugin handling

- `MCP_Server/plugin_aliases.py` is a registry mapping friendly parameter names (e.g. `"filter cutoff"`) to the real Live API parameter name (`"Fil Cutoff"`) per known plugin (currently Serum). Use `resolve_alias()` when a user asks for a parameter by friendly name.
- External plugin discovery (`list_external_plugins`, `load_external_plugin`) walks the Live browser tree under root candidates `plugins`, `vst3`, `vst2`, `au`, `plug-ins`. Results are cached for 120s. Force a rescan with `refresh_cache=True`.

## Commands

```bash
pip install -e .                      # editable install
pytest                                # unit tests only (integration tests skipped by default)
pytest -m integration                 # integration tests (requires running Ableton with Remote Script)
pytest tests/unit/test_bar_beat_conversion.py::TestBarToBeat::test_4_4_bar_2  # single test
python MCP_Server/server.py           # run the MCP server in stdio mode (what the AI client launches)
ableton-mcp-extended                  # same, via console script entry point
```

`pyproject.toml` sets `addopts = "-m 'not integration'"`, so plain `pytest` skips anything marked `@pytest.mark.integration`. Tests that import `MCP_Server.server` mock `mcp`/`mcp.server`/`mcp.server.fastmcp` at module load (see `tests/unit/test_bar_beat_conversion.py`) because importing FastMCP at test time pulls in unwanted side effects.

## Installing the Remote Script (required for any end-to-end work)

The Remote Script source in `AbletonMCP_Remote_Script/__init__.py` only takes effect once it's copied into Ableton's user scripts directory and selected as a Control Surface:

- macOS (Live 11+): `~/Music/Ableton/User Library/Remote Scripts/AbletonMCP/__init__.py` — **the README's `~/Library/Preferences/Ableton/Live <Version>/User Remote Scripts/` path is stale** and silently ignored by Live 11+. Folder name must be exactly `AbletonMCP` (it's the dropdown label and is verified end-to-end).
- Windows: `%USERPROFILE%\Documents\Ableton\User Library\Remote Scripts\AbletonMCP\__init__.py`

Then in Ableton: Preferences → Link, Tempo & MIDI → Control Surface = `AbletonMCP`, Input/Output = None. Status bar should read `AbletonMCP: Listening for commands on port 9877`. The TCP and UDP scripts can co-exist if installed in separate folders (`AbletonMCP/` and `AbletonMCP_UDP/`).

After editing `AbletonMCP_Remote_Script/__init__.py`, you must re-copy it to that user folder and **restart Ableton** (the script is loaded once at Live startup) for changes to take effect.

## Skills

Project-local skills live under `skills/` and are exposed as slash commands via `.claude/commands/`.

### `ableton-songwriter` — `/ableton-songwriter <request>`

`skills/ableton-songwriter/SKILL.md` defines a structured songwriting workflow (intake MCQ → production brief → build → quick mix → handoff). Key rules: never delete user material without confirmation, ask at most 3 + 2 follow-up questions, prefer external plugins when available and fall back to stock devices clearly, and stop retry loops on platform-blocked operations (e.g. trying to delete the last session track — see `get_track_deletion_status` and the safety guard in `delete_track`).

### `ableton-generation` — `/save-generation <label>` and `/load-generation <label>`

`skills/ableton-generation/SKILL.md` defines a session-saving workflow for **generative/musical work** distinct from code/dev saves. Snapshots write to `generations/YYYY-MM-DD_HHMM_<kebab-label>/` containing `MANIFEST.md`, `state.json`, and `notes/*.json` (one per session clip). `/load-generation` rebuilds a saved generation. Key rules baked in: never auto-save (explicit invocation only), never overwrite an existing folder (append `-2`, `-3`), never `git add` or `git commit` files under `generations/` unless explicitly requested, and always remind the user to Cmd+S the `.als` after a snapshot (Claude can't trigger Ableton's save). Use this for musical iterations; use git for code changes.

## Optional ElevenLabs MCP

`elevenlabs_mcp/` is a separate FastMCP server bundled in this repo for voice/SFX generation that imports directly into Ableton. Independent of the Ableton bridge — configure it as a second `mcpServers` entry. Requires `ELEVENLABS_API_KEY` (and optionally `ELEVENLABS_OUTPUT_DIR`) — see `.env.example`. Generate a Claude Desktop config block with `python -m elevenlabs_mcp --print`.
