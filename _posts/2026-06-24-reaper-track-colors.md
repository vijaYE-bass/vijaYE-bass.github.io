---
layout: post
title: "Setting REAPER Track Colors with Python/Claude/ReaperMCP"
date: 2026-06-24
---

One of the first things I set up with the REAPER MCP integration was a coherent
color scheme across all 9 tracks. Here's the full procedure — what works, what
doesn't, and how to keep the TCP and mixer in sync.

## Prerequisites

- REAPER is open
- reapy server is running inside REAPER (port 2306 — starts automatically if
  you have the reapy REAPER extension installed)
- Python environment: `~/claude/reaper-mcp` with `uv`

Run all snippets from that directory:

```bash
cd ~/claude/reaper-mcp
uv run python3 -c "..."
```

## Check track order first

Colors are applied by index (0-based). Always verify the order before running
the color script — tracks may have been reordered since the session was created:

```python
import reapy
reapy.connect()
p = reapy.Project()
for i, t in enumerate(p.tracks):
    print(i, t.name.strip())
```

Confirm the output matches the order you expect before proceeding.

## Before and after

Without colors, all mixer channels look identical. At a glance you can't tell
which strip is which, especially once you're deep in a session.

**Before:**
![Mixer with no track colors](/assets/images/reaper-mixer-before.png)

**After:**
![Mixer with color scheme applied](/assets/images/reaper-mixer-after.png)

Each track now has a color band at the top of its mixer strip and a colored
number tab at the bottom. The TCP (track list on the left) gets the same color
as a full background fill.

## The sync problem

REAPER has two views that both display track colors:
- **TCP** — the track control panel (left side of the arrange view)
- **MCP** — the mixer control panel

They're driven by the same underlying data (`I_CUSTOMCOLOR`) but don't always
redraw together when colors are set via the API. Calling `update_arrange()` only
refreshes the arrange view. To get both in sync you need two extra calls:

```python
rpr.TrackList_AdjustWindows(True)   # redraws TCP
rpr.ThemeLayout_RefreshAll()         # redraws MCP / mixer
```

Without `ThemeLayout_RefreshAll()`, the mixer strips stay grey even though the
color is stored correctly.

## The wrong encoding

The first attempt used manual bit-packing:

```python
# WRONG on macOS — swaps R and B channels
t.set_info_value('I_CUSTOMCOLOR', r | (g << 8) | (b << 16) | 0x1000000)
```

On macOS, `I_CUSTOMCOLOR` expects `(r << 16) | (g << 8) | b | 0x1000000`.
Getting this wrong produces wrong hues — teal becomes yellow, pink becomes purple.

## The correct approach

Use `SetTrackColor` with `ColorToNative` to handle the platform encoding, then
force both panels to redraw:

```python
import reapy

reapy.connect()
p  = reapy.Project()
rpr = reapy.reascript_api

colors = [
    ( 76, 175,  80),  # Guitar      — green
    (255,  87,  34),  # Force       — deep orange
    (255, 152,   0),  # MPC         — amber
    ( 33, 150, 243),  # FL Studio   — blue
    (  0, 188, 212),  # DDJ 400     — teal
    (  0,   0,   0),  # Elektron    — black
    (233,  30,  99),  # SP404       — pink
    ( 80, 100, 230),  # Hydrasynth  — indigo
    (150, 150, 150),  # Sync        — grey
]

for idx, (r, g, b) in enumerate(colors):
    rpr.SetTrackColor(p.tracks[idx].id, rpr.ColorToNative(r, g, b))

rpr.TrackList_AdjustWindows(True)
rpr.ThemeLayout_RefreshAll()
```

`ColorToNative(r, g, b)` does the platform-specific conversion. `SetTrackColor`
sets both the custom color and enables the custom color flag in one call — no
need to manually OR in `0x1000000`.

## Color grouping logic

Colors are grouped by instrument family so related tracks cluster visually:

| Track | Instrument | Family | Color |
|-------|-----------|--------|-------|
| Guitar | Live → Axe-FX | Live | Green |
| Force | Akai Force | Groove box | Deep orange |
| MPC | Akai MPC | Sampler | Amber |
| FL Studio | DAW via ADAT | Software | Blue |
| DDJ 400 | DJ controller | DJ | Teal |
| Elektron | Digitakt / Syntakt | Drum machine | Black |
| SP404 | Roland SP-404 | Sampler | Pink |
| Hydrasynth | ASM Hydrasynth | Synth | Indigo |
| Sync | Click / MIDI clock | Utility | Grey |

Samplers stay warm (orange → amber → pink), synths stay cool (indigo, black),
and the utility track is neutral so it never draws the eye accidentally.
