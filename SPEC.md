# Track Segment Editor for Pits n' Giggles – Final Spec

## Goal

Build a **simple, local-first web tool** to create and edit track segment JSON files.

The tool:

* Does **not model inheritance**
* Works purely with **explicit segment types**
* Enforces **schema via UI + validation**
* Optimized for **fast manual editing**

---

# Core Design Principles

1. **Type-driven, not model-driven**

   * The tool only knows:

     * segment types
     * fields per type
   * It does NOT know about base classes or inheritance

2. **Strict but simple validation**

   * Prevent obvious mistakes
   * Don’t over-engineer

3. **Fast editing**

   * Minimal clicks
   * Inline editing
   * Reordering is first-class

---

# Supported Segment Types

## 1. straight

Fields:

* `type: "straight"`
* `name: string` (required)
* `start_m: number` (required)
* `end_m: number` (required)

---

## 2. corner

Fields:

* `type: "corner"`
* `name: string` (optional but allowed)
* `start_m: number` (required)
* `end_m: number` (required)
* `corner_number: number` (required)

---

## 3. complex_corner

Fields:

* `type: "complex_corner"`
* `name: string` (required)
* `start_m: number` (required)
* `end_m: number` (required)
* `corner_numbers: number[]` (required, non-empty)

---

# Top-Level Schema

```ts id="j6b4b0"
type TrackData = {
  circuit_name: string
  circuit_number: number
  track_length: number
  segments: Segment[]
  sectors: {         // required: Pits n' Giggles refuses to load a file without it
    s1: number       // integer, meters; end of sector 1
    s2: number       // integer, meters; end of sector 2
  }
  speed_trap_m?: number
  drs_zones?: { id: number, detection_m: number, activation: { start_m: number, end_m: number }[] }[]
  overtake_dist?: { detection_m: number, activation_m: number }   // one per track
  active_aero_zones?: { mode: "full" | "partial", start_m: number, end_m: number }[]
  // Any other field is carried through untouched, but flagged: Pits n' Giggles refuses it.
}

// On a corner segment:
brake_boards?: { distance_m: 50 | 100 | 150 | 200 | 250 | 300, at_m: number, side: "left" | "right" | "both" }[]
// On a complex_corner segment; numbered like corner_numbers and tiling the segment exactly:
corners?: { corner_number: number, name?: string, start_m: number, end_m: number, brake_boards?: BrakeBoard[] }[]
```

The schema is `lib/track_segments_classifier` in Pits n' Giggles; the editor mirrors its rules. Every value
above that is a lap distance is in `[0, track_length)`. Zones may cross the start line (`end_m < start_m`),
segments may not. Top-level keys are written in the order that library writes them.

---

# UI Layout

## 1. Header Bar

* Buttons (icon-only, each with a hover/focus tooltip):

  * **New**
  * **Load JSON**
  * **Fetch from GitHub** — see 1b
  * **Copy JSON**
  * **Generate Reverse Track**
  * **Download JSON**
  * **Auto-adjust** toggle (end_m → next start_m)
  * Header collapse/expand chevron (right-aligned)

* Collapsible fields section:

  * `circuit_name` (text input, required)
  * `circuit_number` (number input, required)
  * `track_length` (number input, required)
  * **Sectors (required)** — see below
  * Offset control — non-destructive preview with modulo wraparound; "Apply Permanently" commits

---

## 1a. Sectors (Required)

Shown in the collapsible header section.

* Two required integer inputs:
  * `S1 boundary (m)` — end of sector 1
  * `S2 boundary (m)` — end of sector 2
* When loading JSON that contains `sectors` — fields pre-fill
* When loading JSON without `sectors` — fields show as required, and export is blocked until filled

---

## 1b. Fetch from GitHub (Remote Source)

Lets non-technical users load the latest track definitions without cloning
the repo or installing Git.

* **Source**: public `ashwin-nat/pits-n-giggles` repo, folder `assets/track-segments`
* **Modal flow**: select branch → select JSON file → **Fetch**
  * Branches are listed from the GitHub REST API; the repo's default branch
    is selected first. Changing the branch refreshes the file list.
  * Only `*.json` files in `assets/track-segments` are shown; the filename
    is displayed as-is (no circuit-name inference).
* **On fetch**: the file is retrieved from the selected branch and handed to
  the normal load path (same validation as paste/upload). The repo filename
  becomes the Save/Download default so the origin is retained.
* **Isolation**: all GitHub logic lives in a self-contained `GitHubTrackSource`
  helper; the editor has no GitHub-specific concepts. Leaves room for other
  sources later.
* **Auth**: none. Unauthenticated GitHub API (60 requests/hour per IP).
* **Error handling**: network/GitHub unavailable, branch list failure, file
  list failure, file retrieval failure, file changed/removed between listing
  and fetch, and invalid/unparseable JSON are all surfaced in the modal. A
  GitHub failure never blocks local editing.
* **Non-goals**: no branch creation, commits, pushes, pull requests, or
  writes to the remote repo.

---

## 1c. Zones and Points panel

Collapsible card above the segment list, with a summary and an error badge.

* **Speed trap** at the top: one input; empty means none. Every game uses it
* **F1 26** (collapsible):
  * **Overtake**: detection and activation inputs, or "+ Overtake" when the track has none
  * **Active aero zones**: mode (full / partial), start, end
* **F1 25 and F2** (collapsible):
  * **DRS zones**: id, detection, and one or more activation ranges each; a new zone takes the next free id
* Each collapsible group shows its count and error badge, and starts open only when it has errors
* None of these move when segments do

---

## 2. Segment List

* Vertical list of segment cards
* Order = execution order (important)
* No grouping, no nesting

---

# Segment Card

Each segment is a **self-contained editable block**

---

## A. Top Row

* **Type dropdown** (required)

  * Options:

    * straight
    * corner
    * complex_corner

* **Name input**

* **Delete button**

---

## B. Range Fields

* `start_m` (number input)
* `end_m` (number input)

---

## C. Type-Specific Fields

### If `corner`

* `corner_number` (number input)
* Brake boards: one row per board (distance, `at_m`, side), "+ Board" to add. `at_m` is where the board stands

---

### If `complex_corner`

* `corner_numbers`

  * Input style:

    * comma-separated string OR chip list
  * Example:

    ```id="n2zpl7"
    5,6,7
    ```
* Sub-corners: a collapsible box. "Split into sub-corners" makes one per corner number, evenly spaced
  * Each row: corner number, optional name, start (the previous boundary) and end; the last end is the segment end
  * Changing an end moves the next start with it; the first start and last end follow the segment's own range
  * A same-length change to `corner_numbers` renumbers them; any other change offers "Rebuild from corner_numbers", keeping names and boards by corner number
  * Each sub-corner has its own brake boards box; a complex corner has no boards of its own
  * "Remove sub-corners" asks first when boards would go with them

---

### If `straight`

* No additional fields

---

## D. Actions

* ⬆ Move up
* ⬇ Move down
* ➕ Insert above
* ➕ Insert below
* 🔁 Convert type (handled via dropdown)

---

# Key Features

## 1. Add Segment

* User clicks:

  * “Insert above” or “Insert below”
* Must choose type immediately (or default to `straight`)

Default values:

```ts id="8mxxhl"
{
  type: "straight",
  name: "",
  start_m: previous.end_m,
  end_m: start_m + 100
}
```

---

## 2. Convert Segment Type

User changes `type` via dropdown.

### Behavior:

* Preserve:

  * `name`
  * `start_m`
  * `end_m`

* Reset type-specific fields:

| From → To            | Action                           |
| -------------------- | -------------------------------- |
| any → straight       | remove extra fields              |
| any → corner         | require `corner_number`          |
| any → complex_corner | initialize `corner_numbers = []` |

---

## 3. Reordering

* Move up/down buttons
* Order directly affects validation

---

## 4. Delete Segment

* Immediate removal
* Optional confirmation

---

# Validation Rules

Validation runs:

* On every edit
* On load
* Before export

---

## 1. Field-Level Validation

### Required Fields

* `type` → required
* `start_m`, `end_m` → required
* `name`:

  * required for:

    * `straight`
    * `complex_corner`
* `corner_number`:

  * required for:

    * `corner`
* `corner_numbers`:

  * required for:

    * `complex_corner`
  * must be:

    * non-empty
    * all integers

---

## 2. Range Validation

```ts id="v7iykn"
start_m < end_m
```

Error if:

* equal
* greater

---

## 3. Ordering + Overlap Validation

For all segments:

```ts id="skh3wl"
segments[i].start_m >= segments[i-1].end_m
```

Error if violated:

* overlap
* out-of-order

---

## 3a. Corner Numbering Validation

Matches Pits n' Giggles, which rejects the whole file otherwise:

* `corner` and `complex_corner` numbers together run 1, 2, 3... in segment order
* No gaps, repeats or out-of-order numbers
* `corner_number` must be an integer
* A `complex_corner` needs at least 2 numbers
* A `complex_corner` that carries `corners` (sub-corners) must have them numbered like `corner_numbers`, starting at its `start_m`, ending at its `end_m` and each starting where the previous ends

---

## 4. Sector Validation

Always applies:

```
s1 > 0
s1 < s2
s2 < track_length
s1 and s2 must be integers (no floats)
```

* Inline errors shown below each input
* Export (Copy JSON / Download JSON) blocked until sector errors are resolved
* Segment validation is unaffected

---

## 4a. Zones, Points and Boards

Matches Pits n' Giggles, which rejects the whole file otherwise:

* Every lap distance is required, numeric and in `[0, track_length)`
* A zone may cross the start line but may not be empty (`start_m` equal to `end_m`)
* DRS zone ids are unique whole numbers; each zone has at least one activation range
* Aero mode is `full` or `partial`; board distance is 50 to 300 in 50s; board side is left, right or both
* Sub-corners match `corner_numbers`, start at the segment start, end at its end, and each starts where the previous ends; their boards are checked like corner boards
* Unknown fields, at top level or on a segment, are kept and flagged
* Loading refuses a file whose zones or boards carry unknown fields, since the editor would drop them

---

## 5. Error Display

* Inline field errors (red border)
* Segment-level error badge
* Zones and Points panel badge
* Global error summary at top:

  * “X segments with errors · Y zone or point errors”

---

# JSON Import / Export

## Load

* Paste JSON OR upload file
* Validate before accepting
* Show errors if invalid
* If `sectors` present → pre-fill `s1` / `s2`
* If `sectors` absent → `s1` / `s2` show as required
* Fields the editor does not understand are kept, at top level and per segment

---

## Export

* Clean JSON only — no internal fields (`_id`, `corner_numbers_raw`, `_extra`, `_keyOrder`)
* `sectors` always included; export blocked until valid
* Unknown fields written back unchanged, keys in the source file's order, so an unedited file round-trips unchanged
* `name` is always written for straights and omitted when empty on corners and complex corners, as Pits n' Giggles writes it
* Export is blocked while any segment, zone, point or board error remains, since Pits n' Giggles would refuse the file
* `drs_zones` and `active_aero_zones` are written when non-empty or when the loaded file had them
* Format: 2-space indentation

---

# UX Details (Important)

## 1. Inline Editing

* No modals
* Everything editable directly

---

## 2. Fast Input

* Number inputs for distances
* Comma input for `corner_numbers`

---

## 3. Visual Feedback

* Highlight invalid segments
* Highlight overlapping segments

---

## 4. Optional Helpers

### A. Auto-adjust toggle

When enabled:

* Editing `end_m` updates next segment’s `start_m`

---

### B. Offset preview

* Non-destructive: shows shifted positions on each segment card without modifying state
* Modulo wraparound using `track_length`
* "Apply Permanently" commits offset to all segment `start_m` / `end_m` values

---

### C. Sectors

* Required track-level metadata
* Does not affect segment layout or validation

### D. Reverse track and offset with zones, points, boards or unknown fields

* Reverse generation mirrors sectors (`s1' = L - s2`, `s2' = L - s1`) as a starting point, and drops zones, points, boards and unknown fields with a notice: reverse layouts get their own
* Offset is blocked when the file has any of them, since it would not shift their positions

---

# Tech Stack

## Recommended

* React (simple functional components)
* Local state only
* No backend

---

## No Dependencies Needed For:

* File load → FileReader
* File save → Blob download

---

# Non-Goals

* No inheritance modeling
* No backend
* No persistence (optional localStorage — implemented)
* No collaboration
* Sectors do not define segment boundaries — segments remain independent

---

# Acceptance Criteria

* User can:

  * Create full JSON from scratch
  * Edit all fields inline
  * Insert anywhere
  * Convert types
  * Reorder segments
  * Configure sector boundaries (S1, S2), which are required
  * Load existing JSON with or without sectors
  * Load and save files with fields the editor does not understand, without losing them
  * Edit the speed trap, overtake points, DRS zones, active aero zones, sub-corners and brake boards
  * Fetch an existing track definition from the public GitHub repo (branch +
    file selection) with no Git install, then edit and save it locally

* Tool:

  * Prevents invalid schema
  * Detects overlap/order issues
  * Validates sector boundaries (required, integer, ordered, within track length)
  * Validates corner numbering across the whole file, as Pits n' Giggles does
  * Flags every zone, point, board or field Pits n' Giggles would refuse
  * Blocks export while any validation error exists
  * Exports valid JSON for Python system
