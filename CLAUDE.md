# Track Segment Editor — Claude Context

## Project Overview

A single-file browser-based editor for creating and editing F1 track segment JSON files used by the "Pits n' Giggles" telemetry system. No build step, no backend — everything runs in-browser.

## File Structure

| File | Role |
|------|------|
| [index.html](index.html) | Entire application — React components, reducer, validation, styling |
| [README.md](README.md) | User-facing usage docs and deployment instructions |
| [SPEC.md](SPEC.md) | Requirements, data structures, validation rules, UX patterns |

## Tech Stack

- **React 18** + **Babel standalone** — JSX transpiled in-browser, no build step
- **Tailwind CSS 3** — all styling via CDN
- **Bootstrap Icons 1.11.3** — icon set
- **No npm, no bundler, no build process**

## Data Model

### Track JSON (exported format)
```json
{
  "circuit_name": "string",
  "circuit_number": 1,
  "track_length": 5300,
  "segments": [ ... ],
  "sectors": { "s1": 1800, "s2": 3600 }
}
```
`sectors` is **required**: Pits n' Giggles refuses to load a track file without it.

Optional track-level fields, all lap distances in `[0, track_length)`; a zone may cross the start line
(`end_m < start_m`) but may not be empty:
```json
"speed_trap_m": 716.3,
"drs_zones": [{ "id": 1, "detection_m": 2757.6, "activation": [{ "start_m": 3055.8, "end_m": 3889.1 }] }],
"overtake_dist": { "detection_m": 5154.4, "activation_m": 5253.6 },
"active_aero_zones": [{ "mode": "full", "start_m": 1158.3, "end_m": 2113.1 }]
```
A `corner` may carry `brake_boards: [{ "distance_m": 100, "at_m": 2403.5, "side": "left" }]`
(`distance_m` 50 to 300 in 50s, `side` left/right/both). A `complex_corner` may carry `corners`: sub-corners
numbered like `corner_numbers`, tiling the segment exactly, each with an optional name and its own boards.
A complex corner itself carries no boards.

The schema is `lib/track_segments_classifier` in pits-n-giggles, which refuses unknown fields
(`extra="forbid"`). Any other field is still carried through in its original key order, but flagged.

### Segment Types

**straight**
```json
{ "type": "straight", "name": "...", "start_m": 0, "end_m": 200 }
```

**corner**
```json
{ "type": "corner", "name": "...", "start_m": 200, "end_m": 350, "corner_number": 1 }
```

**complex_corner**
```json
{ "type": "complex_corner", "name": "...", "start_m": 350, "end_m": 600, "corner_numbers": [2, 3] }
```

### Internal Editor Fields (not exported)
- `_id` — unique ID (`seg_<counter>_<random>`), used as React key and error map key
- `corner_numbers_raw` — string representation for the `complex_corner` input (e.g. `"2,3"`)
- `brake_boards` — corner boards as `{ _id, distance_m, at_m, side }` with string values
- `sub_corners` — complex corner parts as `{ _id, corner_number, name, start_m, end_m, brake_boards }`, strings.
  `subCornersFollow()` keeps them tiling: the first start and last end follow the segment, and a same-length
  `corner_numbers` change renumbers them. `splitSubCorners()` builds or rebuilds them evenly spaced
- `_extra` — fields the editor does not know, merged back on export and flagged
- `_keyOrder` — the segment's key order in the source file, reused on export
- Top-level equivalents in reducer state: `extraTop` and `topKeyOrder`
- Track-level points and zones in reducer state: `speedTrap`, `overtake`, `drsZones`, `aeroZones` (strings, `_id` per row),
  built by `editorMarkers()` / `markersFromFile()`

## Architecture

- **Single `useReducer`** manages all state — no useState for app logic
- **`validate(segments, trackLength)`** runs on every state update, returns error map keyed by `_id`. Includes the file-wide corner numbering rule from Pits n' Giggles: 1, 2, 3... in lap order, no gaps or repeats, and at least 2 numbers per `complex_corner`. A `complex_corner` with sub-corners must keep them numbered like `corner_numbers` and tiling its range exactly (`subCornerErrors`, per sub-corner under `sub_corners`, count mismatch under `corners`)
  It also checks corner brake boards, sub-corners and their boards, and flags unknown segment fields
- **`validateSectors(s1, s2, trackLength)`** and **`validateMarkers(state, trackLength)`** — computed inline in App (not stored in reducer state). `validateMarkers` covers the track-level points and zones and unknown top-level fields, mirroring `lib/`
- Any validation error (segments, sectors, zones, points, boards, unknown fields) blocks Copy JSON and Download JSON, so every file the editor writes loads in Pits n' Giggles
- **`exportSegment(seg, includeExtra = true)`** strips internal fields for JSON output. `name` is written for straights and whenever non-empty, never as `""`, matching `lib/track_segments_classifier`. `CHANGE_TYPE` drops `corners` (not a complex corner) and `brake_boards` (not a corner). Never pass it straight to `.map()`, which would supply the index as `includeExtra`
- **`importSegment(seg)`** converts raw JSON to editor state (adds `_id`, `corner_numbers_raw`, `brake_boards`, `_extra`, `_keyOrder`)
- **`buildPayload(state)`** builds the exported file; the only place Copy JSON and Download JSON get it from. Top-level keys follow `LIB_TOP_ORDER`, so every shipped file round-trips byte for byte
- **`validateLoadedData(data)`** refuses a file whose zones or boards have fields the editor would drop
- **localStorage key:** `track-segment-editor.state.v1`

### Action Types (reducer)
`SET_CIRCUIT_NAME`, `SET_CIRCUIT_NUMBER`, `SET_TRACK_LENGTH`, `ADD_SEGMENT`, `INSERT_ABOVE`, `INSERT_BELOW`, `UPDATE_SEGMENT`, `CHANGE_TYPE`, `MOVE_UP`, `MOVE_DOWN`, `DELETE_SEGMENT`, `LOAD_JSON`, `NEW`, `APPLY_OFFSET`, `SET_S1`, `SET_S2`, `SET_MARKERS` (any of `speedTrap`, `overtake`, `drsZones`, `aeroZones`), `SET_BRAKE_BOARDS`, `SET_SUB_CORNERS`, `OPEN_GITHUB_MODAL`, `CLOSE_GITHUB_MODAL`

### GitHub Track Source
- **`GitHubTrackSource`** — self-contained IIFE, no React/editor knowledge; fetches branches + `assets/track-segments/*.json` from the public `ashwin-nat/pits-n-giggles` repo via the GitHub REST API + `raw.githubusercontent.com`. Unauthenticated (60 req/hr/IP); throws `GitHubError` with user-facing messages.
- **`GitHubModal`** — thin UI (branch selector → track selector → Fetch). On success dispatches the existing `LOAD_JSON` with `fileName` = the repo filename, so the fetched track enters the normal load/validation flow and the Save dialog defaults to that name. GitHub failures never leave the editor unusable.

### Sectors state (in reducer)
- `s1: string` — raw input value for S1 boundary (meters)
- `s2: string` — raw input value for S2 boundary (meters)
- Persisted to localStorage; restored on load
- `LOAD_JSON` pre-fills `s1`/`s2` when loaded JSON contains a valid `sectors` object; otherwise both show as required

### Positions that do not follow the segments
- Zones, points and boards are lap distances of their own; moving or resizing a segment does not move them
- Offset is blocked when the file has any of them or other extra fields, since it would shift segments only
- Reverse generation drops them with a notice (reverse layouts get their own) and mirrors sectors as a starting point
- localStorage snapshots from before the editor knew these fields (kept in `extraTop` / `_extra`) are migrated on restore

## Key Features

- **Auto-adjust mode** — editing a segment's `end_m` automatically updates next segment's `start_m`
- **Reverse track generation** — mirrors positions (`start_m = trackLen - oldEnd_m`) and inverts corner numbers
- **Offset preview** — non-destructive visualization with modulo wraparound; "Apply Permanently" commits changes
- **Sectors (required)** — S1/S2 boundary inputs, validated separately from segments; export is blocked until valid
- **Unknown fields preserved** — anything the editor does not understand round-trips unchanged
- **Custom modal/toast system** — app-level modals for alert/confirm/prompt; toasts auto-dismiss at 2.5s
- **Move animations** — CSS bounce+scale on segment move, flash overlay

## Running Locally

```bash
python -m http.server 8000
# open http://localhost:8000
```

Or just open `index.html` directly in a browser.

## Deployment

```bash
npx vercel
```

## Conventions

- Dark theme: bg `#1a1a1a` / `#0d0d0d`, zinc-gray accents, red for errors, amber for offset controls
- All edits are inline — no separate edit modal
- Validation errors surface as field-level badges + global header banner
- "Go to Next Error" button scrolls to the next invalid segment
- Segment IDs use a counter synced via `syncIdCounter()` to survive localStorage restores
