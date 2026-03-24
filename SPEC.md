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
  track_length: number
  segments: Segment[]
}
```

---

# UI Layout

## 1. Header Bar

* Buttons:

  * **New**
  * **Load JSON**
  * **Save / Download JSON**
* Field:

  * `track_length` (number input)

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

## 4. Error Display

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

---

## Export

* Clean JSON only
* No UI-specific fields
* Format:

  * 2-space indentation

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

### B. Segment preview label

For complex corner:

```id="sn2av3"
Les Combes (T5–T7)
```

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
* No persistence (optional localStorage)
* No collaboration

---

# Acceptance Criteria

* User can:

  * Create full JSON from scratch
  * Edit all fields inline
  * Insert anywhere
  * Convert types
  * Reorder segments

* Tool:

  * Prevents invalid schema
  * Detects overlap/order issues
  * Exports valid JSON for Python system
