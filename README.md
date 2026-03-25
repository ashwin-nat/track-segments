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
- **Reverse track generation**: mirrors segment positions and inverts corner numbers
- **Offset control**: non-destructively preview a track alignment shift, then apply permanently
- Load JSON via file upload or paste; export via download or copy to clipboard
- State persisted to localStorage between sessions

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
