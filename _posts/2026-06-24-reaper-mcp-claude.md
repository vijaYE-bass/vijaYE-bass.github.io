---
layout: post
title: "Controlling REAPER with Claude Code via MCP"
date: 2026-06-24
---

REAPER has a Python API called [reapy](https://python-reapy.readthedocs.io/) that
exposes the full REAPER extension API over a local socket. This post documents how
I wired it up to Claude Code so I can control REAPER through natural language in my
terminal.

## Architecture

Claude Code's MCP server subprocesses run inside a sandbox that blocks direct TCP
connections to localhost. So a direct reapy connection from the MCP server isn't possible.
The solution is a bridge process that runs outside the sandbox:

```
Claude Code (MCP client)
    │
    ▼ file IPC via /tmp/reaper_rpc/
bridge.py  (runs in terminal, outside sandbox)
    │
    ▼ TCP localhost:2306
REAPER + reapy server
```

The MCP server writes tool call payloads to files in `/tmp/reaper_rpc/`. The bridge
watches that directory, executes the calls via reapy, and writes responses back.

## Setup

**1. Start REAPER.** The reapy server starts automatically on port 2306 once you've
installed the reapy REAPER extension.

**2. Start the bridge** (keep this terminal open):

```bash
cd ~/claude/reaper-mcp
uv run python3 scripts/start_bridge.py
```

**3. Reconnect MCP** in Claude Code: `/mcp`

## What it can do

With 58 tools exposed, you can do things like:

- Create, rename, delete, color tracks
- Set volume, pan, mute, solo
- Add FX and set parameters
- Create MIDI items and drum patterns
- Analyze loudness, dynamics, frequency spectrum
- Render stems or the full project
- Set tempo and time signature

## Falling back to reapy directly

Some things aren't exposed by the MCP tools (e.g. recording input assignment).
For those, I drop down to reapy directly:

```python
import reapy
reapy.connect()
p = reapy.Project()
t = p.tracks[0]
t.set_info_value('I_RECINPUT', 1034)  # ADAT 1/2
```

## Track color encoding

One gotcha: `set_info_value('I_CUSTOMCOLOR', ...)` uses the wrong byte order on macOS
if you encode manually. Use reapy's native `track.color = (r, g, b)` property instead,
which handles the platform-specific encoding. Then call `reapy.update_arrange()` to
force a redraw.

## Current session setup

9 tracks, each colored by instrument family:

| Track | Instrument | Color |
|-------|-----------|-------|
| Guitar | Live guitar → Axe-FX | Green |
| Force | Akai Force | Deep orange |
| MPC | Akai MPC | Amber |
| FL Studio | DAW via ADAT | Blue |
| DDJ 400 | DJ controller | Teal |
| Elektron | Digitakt / Syntakt | Black |
| SP404 | Sampler | Pink |
| Hydrasynth | Synth | Indigo |
| Sync | Click / sync track | Grey |
