# Track Segment Editor

A local-first web tool for creating and editing F1 track segment JSON files for the Pits n' Giggles telemetry system.

## Usage

Open `index.html` in any browser — or serve it locally:

```bash
python -m http.server 8000
```

No build step, no install, no backend.

## Features

- Create and edit **straight**, **corner**, and **complex_corner** segments inline
- Reorder segments with move up/down buttons
- Insert segments above or below any existing card
- Convert segment types via dropdown (preserves name and distance fields)
- **Auto-adjust** toggle: editing `end_m` automatically updates the next segment's `start_m`
- Full validation on every edit: required fields, range checks, overlap detection, out-of-bounds
- `complex_corner` validation enforces increasing consecutive turn numbers (e.g. `5,6,7`)
- Corner numbers are checked across the whole file (1, 2, 3... in lap order), and sector boundaries are required, matching what Pits n' Giggles accepts
- **Zones and points**: edit the speed trap, overtake detection and activation, DRS zones and active aero zones
- **Brake boards** on each corner: distance, position on the lap and side of the track
- **Sub-corners**: split a complex corner into one part per corner number, move the boundaries between them, name them, and give each its own brake boards
- Every value Pits n' Giggles would refuse is flagged, so a saved file loads there; fields the editor does not know are kept but flagged
- **Reverse track generation**: mirrors segment positions and inverts corner numbers
- **Offset control**: non-destructively preview a track alignment shift, then apply permanently
- Load JSON via file upload or paste; export via download or copy to clipboard
- **Fetch from GitHub**: pull track definitions straight from the public `ashwin-nat/pits-n-giggles` repo — pick a branch and a JSON file, no clone or Git install needed
- State persisted to localStorage between sessions
- Icon toolbar with hover/focus tooltips

## Output Format

```json
{
  "circuit_name": "Spa-Francorchamps",
  "circuit_number": 7,
  "track_length": 7004,
  "segments": [
    { "type": "straight", "name": "Kemmel Straight", "start_m": 0, "end_m": 550 },
    { "type": "corner", "name": "Turn 1", "start_m": 550, "end_m": 720, "corner_number": 1 },
    { "type": "complex_corner", "name": "Les Combes", "start_m": 720, "end_m": 950, "corner_numbers": [5, 6, 7] }
  ]
}
```

## Deployment

Deploy to Vercel with no configuration:

```bash
npx vercel
```
