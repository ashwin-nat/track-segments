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
`sectors` is **optional** — omitted entirely (not `null`, not `{}`) when the toggle is off.

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

## Architecture

- **Single `useReducer`** manages all state — no useState for app logic
- **`validate(segments, trackLength)`** runs on every state update, returns error map keyed by `_id`
- **`validateSectors(sectorsEnabled, s1, s2, trackLength)`** — separate function, computed inline in App (not stored in reducer state); returns `{}` when sectors disabled
- **`exportSegment(seg)`** strips internal fields for JSON output
- **`importSegment(seg)`** converts raw JSON to editor state (adds `_id`, `corner_numbers_raw`)
- **localStorage key:** `track-segment-editor.state.v1`

### Action Types (reducer)
`SET_CIRCUIT_NAME`, `SET_CIRCUIT_NUMBER`, `SET_TRACK_LENGTH`, `ADD_SEGMENT`, `INSERT_ABOVE`, `INSERT_BELOW`, `UPDATE_SEGMENT`, `CHANGE_TYPE`, `MOVE_UP`, `MOVE_DOWN`, `DELETE_SEGMENT`, `LOAD_JSON`, `NEW`, `APPLY_OFFSET`, `SET_SECTORS_ENABLED`, `SET_S1`, `SET_S2`

### Sectors state (in reducer)
- `sectorsEnabled: boolean` — controls toggle and export
- `s1: string` — raw input value for S1 boundary (meters)
- `s2: string` — raw input value for S2 boundary (meters)
- Persisted to localStorage; restored on load
- `LOAD_JSON` auto-enables toggle and pre-fills `s1`/`s2` when loaded JSON contains a valid `sectors` object

## Key Features

- **Auto-adjust mode** — editing a segment's `end_m` automatically updates next segment's `start_m`
- **Reverse track generation** — mirrors positions (`start_m = trackLen - oldEnd_m`) and inverts corner numbers
- **Offset preview** — non-destructive visualization with modulo wraparound; "Apply Permanently" commits changes
- **Sectors (optional)** — toggle-gated S1/S2 boundary inputs; validated separately from segments; omitted from export when off
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
