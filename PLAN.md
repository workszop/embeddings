# Improvement plan — Animal embeddings playground

Plan for upgrading the educational lab (`index.html`) used to demonstrate what
embeddings are and how they encode meaning. Based on the review of 2026-06-12.
Each item is independent unless noted; suggested order is bug fixes → UX →
pedagogy, so the lab is solid before new features land on top.

---

## Phase 1 — Bug fixes (quick wins)

### 1.1 Fix clipped numeric tick labels
- **Problem:** `.tick-label-x` (`bottom: -16px`) and `.tick-label-y`
  (`left: -20px`) are appended inside `#canvas`, which has `overflow-hidden`,
  so the 0–100 axis numbers are clipped and invisible.
- **Fix:** position the labels just *inside* the canvas edge (e.g.
  `bottom: 2px` / `left: 2px` with a subtle background), or append them to the
  canvas's parent `<section>` and offset against the canvas rect.
- **Where:** CSS rules near `index.html` `.tick-label-*`; `addAxisTicks()`.
- **Verify:** after "Generate Embeddings", tick numbers visible on both axes,
  not overlapping cards, and they survive a window resize.

### 1.2 Show axis labels during placement
- **Problem:** "Size: Large/Small" and "Environment: Wild/Domestic" labels are
  hidden until "Generate Embeddings", so students place animals on an
  unlabeled canvas.
- **Fix:** show the four word labels from page load (`initializeAnimals()`
  calls `showAxes()`-lite). Keep the *numeric ticks* as a post-generation
  reveal — "now your judgments became a coordinate system" is a good beat.
- **Where:** `hideAxes()` / `showAxes()` / `initializeAnimals()`; split into
  `showAxisLabels()` and `showAxisTicks()`.

### 1.3 Include z in the coordinate tooltip
- **Problem:** hover/drag tooltip shows `(x, y)` while the inspector and table
  show three values, weakening the "an animal IS a 3-number vector" message.
- **Fix:** tooltip shows `(x, y, z)`; both in `showCoordinateTooltip()` and in
  the live-drag branch of `onPointerMove()`.

---

## Phase 2 — UX hardening

### 2.1 Draw neighbor lines in the 3-D view
- **Problem:** neighbors are computed in 3-D but lines are drawn only on the
  2-D canvas, so connected animals can look far apart (different habitats).
- **Fix:** in `render3D()`, when an animal is selected and embeddings exist,
  draw lines from the selected point to its `NEIGHBOR_COUNT` neighbors using
  the same projected coordinates, same color/opacity ramp as the 2-D SVG
  lines. Reuse the neighbor computation from `drawNeighbors()` (extract a
  `computeNeighbors(name)` helper used by both renderers).

### 2.2 Reset confirmation
- **Problem:** one stray tap on Reset wipes all student placements.
- **Fix:** lightweight confirm — either `confirm()` or a two-step button
  ("Reset" → "Tap again to confirm", reverting after ~3 s). Prefer the
  two-step button (no browser dialog on tablets).

### 2.3 Offline support
- **Problem:** Tailwind CDN + Google Fonts require internet; classroom risk.
- **Fix:** replace the Tailwind CDN with a small hand-written stylesheet
  covering only the classes actually used (the page is one screen; this is
  tractable), and self-host or system-stack the fonts
  (`font-family: 'Plus Jakarta Sans', system-ui, sans-serif` with a local
  fallback). Acceptance: page fully styled with network disabled.
- **Note:** largest mechanical task in the plan; do it in its own commit so
  visual diffs are reviewable.

### 2.4 Minor polish (best-effort, bundle into other commits)
- Keyboard alternative for placement (arrow keys nudge the selected card).
- Reduce 3-D label overlap: skip/offset labels when projected points are
  within a few px of each other, or fade non-selected labels when crowded.

---

## Phase 3 — Pedagogy: closing the loop to real AI

### 3.1 "Bridge to real AI" explainer panel
- **What:** a collapsible panel that appears (expanded once) after
  "Generate Embeddings", explaining:
  1. real embeddings have hundreds–thousands of dimensions, not 3;
  2. the dimensions are *learned* by a model from data, not chosen by people;
  3. individual dimensions usually aren't interpretable — but distance still
     means similarity, exactly as in this lab.
- **Where:** new `<section>` in the footer area; toggled in the
  generate-button handler.

### 3.2 Rename / explain "Generate Embeddings"
- **What:** the button reads off positions the students chose — nothing is
  generated. Keep the button name (it's the hook) but add one sentence next
  to it or in the 3.1 panel: "Here *you* are the embedding model; in a real
  AI system a neural network computes these numbers from data."
- **Optional stretch:** hardcode real precomputed embeddings (e.g. from a
  sentence-transformer, PCA-projected to 3-D, values baked into the JS — no
  network needed) for the same 18 animals, with a "compare with a real
  model" toggle in the 3-D view. Seeing a real model also put Cat near Dog
  is the most persuasive moment available. Treat as its own follow-up task;
  requires generating the data offline first.

### 3.3 Query / semantic-search step
- **What:** demonstrate what embeddings are *for* (retrieval / RAG in
  miniature). Add a draggable "❓ Query" marker to the shelf (enabled after
  embeddings are generated). When placed, the inspector shows its vector and
  a ranked top-3 nearest animals ("Your query matched: …"), reusing the
  neighbor machinery. Habitat slider applies to the query too.
- **Where:** extend `newAnimalsShelf` handling with a special card type that
  is excluded from `currentEmbeddingsData` but participates in
  `computeNeighbors()` as a source.
- **Status copy:** "This is how AI search works: your question becomes a
  vector, and the closest vectors are the answers."

### 3.4 Challenge animals that break the dimensions
- **What:** add **Penguin** (swims + walks, can't fly) and **Duck**
  (water *and* air) to the new-animals shelf. There is no correct z for
  them — that's the point. When a challenge animal is placed, show a prompt:
  "Hard to place? Real models fix this with many more dimensions."
- **Also surface the existing edge case:** Chicken (z=68) lands in the "air"
  bucket — add it to the discussion prompts (3.5) rather than changing data.
- **Note:** keep "place all shelf animals" as the gate for Generate; verify
  the count logic and copy still read correctly with 6 shelf animals.

### 3.5 Built-in discussion prompts
- **What:** a small "Think about it" section with 3–4 questions so the lab
  works without a facilitator script:
  - Why are Cat and Dog nearest neighbors? What property makes them close?
  - What dimension would separate Shark from Goldfish better than size?
  - Where should a Bat go on the habitat slider — and what can't these
    three numbers capture about a Bat?
  - Chicken ended up in the "air" group. Is that right? Who decided?
- **Where:** static section after the embeddings table; reveal after
  generation alongside 3.1.

---

## Out of scope (explicitly dropped)

- Cosine similarity display / Euclidean-vs-cosine toggle and normalized 0–1
  similarity scores — dropped per review follow-up; the lab keeps raw
  Euclidean distance only.

## Suggested commit sequence

1. Phase 1 (one commit — small, related fixes)
2. 2.1 + 2.2 (one commit)
3. 2.3 offline support (own commit)
4. 3.1 + 3.2 explainer copy (one commit)
5. 3.4 + 3.5 challenge animals + prompts (one commit)
6. 3.3 query marker (own commit)
7. Stretch: 3.2 real-embedding comparison (separate task)
