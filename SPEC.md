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
  sectors?: {        // optional — omitted entirely when not configured
    s1: number       // integer, meters; end of sector 1
    s2: number       // integer, meters; end of sector 2
  }
}
```

---

# UI Layout

## 1. Header Bar

* Buttons:

  * **New**
  * **Load JSON**
  * **Copy JSON**
  * **Generate Reverse Track**
  * **Download JSON**
  * **Auto-adjust** toggle (end_m → next start_m)

* Collapsible fields section:

  * `circuit_name` (text input, required)
  * `circuit_number` (number input, required)
  * `track_length` (number input, required)
  * **Sectors (Optional)** — see below
  * Offset control — non-destructive preview with modulo wraparound; "Apply Permanently" commits

---

## 1a. Sectors (Optional)

Shown in the collapsible header section.

* **Enable sectors** checkbox/toggle
* When enabled, two integer inputs appear:
  * `S1 boundary (m)` — end of sector 1
  * `S2 boundary (m)` — end of sector 2
* When disabled — inputs hidden, `sectors` omitted from export
* When loading JSON that contains `sectors` — toggle auto-enables and fields pre-fill

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

---

### If `complex_corner`

* `corner_numbers`

  * Input style:

    * comma-separated string OR chip list
  * Example:

    ```id="n2zpl7"
    5,6,7
    ```

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

## 4. Sector Validation

When sectors are enabled:

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

## 5. Error Display

* Inline field errors (red border)
* Segment-level error badge
* Global error summary at top:

  * “X errors found”

---

# JSON Import / Export

## Load

* Paste JSON OR upload file
* Validate before accepting
* Show errors if invalid
* If `sectors` present → auto-enable toggle and pre-fill `s1` / `s2`
* If `sectors` absent → toggle remains disabled

---

## Export

* Clean JSON only — no internal fields (`_id`, `corner_numbers_raw`)
* `sectors` included only when toggle is enabled and values are valid
* `sectors` omitted entirely (not `null`, not `{}`) when toggle is off
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

### C. Sectors toggle

* Optional track-level metadata
* Does not affect segment layout or validation
* When disabled: no `sectors` key in output

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
  * Optionally configure sector boundaries (S1, S2)
  * Load existing JSON with or without sectors

* Tool:

  * Prevents invalid schema
  * Detects overlap/order issues
  * Validates sector boundaries (integer, ordered, within track length) when enabled
  * Blocks export if sector errors exist
  * Omits `sectors` entirely from output when toggle is off
  * Exports valid JSON for Python system
