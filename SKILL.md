---
name: "figma-color-palette"
description: "Generate a complete, Tailwind-compatible Figma color palette (variables + a reusable \"Color Tag\" badge component + a documented \"Color Section\" frame, assembled into a page-level \"Documentation\" frame with Header/Footer and a wrapping 3-column grid) from a base hex or a full list of colors. Works even in a brand-new file with no existing \"Color Tag\" component — the skill creates it from scratch on first use. Use this skill whenever the user asks to create, generate, or add a color palette in Figma, wants Figma color variables for a design system, mentions \"primitives-{name}\", \"Color Tag\", \"gap-6\", \"Color Section\", \"Documentation\", or \"grid\" in a Figma color-system context, or wants a reusable Figma skill for generating color scales (050-950). Always trigger this before writing any Figma Plugin API code for palette generation — it defines the mandatory intake questions, the HSL scale algorithm, the exact variable/component/frame specs, and a validation script, so nothing has to be invented or guessed."
---

# Figma Color Palette Generator

Generates a complete, reusable color palette inside any Figma file — for
any designer, not just one project. Produces: Figma variables (color +
string placeholders), a reusable `Color Tag` component (created from
scratch if the file doesn't already have one), and a documented
`Color Section` frame — all wired together with bound variables so a
designer can rename/recolor later without touching instances. Each
`Color Section` is assembled into a page-level `Documentation` frame
(Header + wrapping 3-column `grid` + Footer), so multiple palettes
accumulate into one consistent page instead of scattering loose sections.

**Always writes through `use_figma`** (Figma Plugin API), following the
`figma-use` skill's rules: work incrementally, validate after each step,
return all created/mutated node IDs, never guess font style names, always
set explicit variable scopes. Before writing any creation code, skim
`references/figma-api-patterns.md` once per session — it has the exact,
tested API call shapes (page persistence, variable binding, auto-layout
sizing) that are easy to get subtly wrong from memory. Reaching for that
file when unsure is expected, not optional overhead.

---

> ## 🔒 Non-negotiables (read before anything else)
> These are things past runs of this skill have actually skipped or
> gotten wrong. If a generation doesn't tick every one of these, it
> isn't done — no exceptions, no "I'll add it after":
> 1. **Every `Color Section` lives inside a `Documentation` → `grid`,
>    built in that order.** Phase 6 (the container) always completes
>    before Phase 7 (the content) starts — never appended straight to
>    the page/canvas.
> 2. **Every swatch and label is bound to a variable**, never a
>    hardcoded hex or hardcoded string — see Phase 4 and 7.
> 3. **`Description`'s height-fill needs `layoutSizingVertical = "FILL"`
>    set explicitly** — `layoutAlign: STRETCH` alone silently fails to
>    stretch it (see references/figma-api-patterns.md).
> 4. **All text Claude generates and writes into Figma (titles,
>    descriptions, footer copy, any auto-generated string) defaults to
>    English**, regardless of the conversation's language — unless the
>    designer supplies exact text themselves, which is always used
>    verbatim, untranslated, in whatever language they wrote it.
> 5. **Every intake question (Q0–Q5) uses the native tappable-options
>    tool**, asked in the person's own conversation language — never
>    plain prose with markdown bullets, never forced into any one fixed
>    language.
> 6. **Run Phase 8's validation script and read its returned booleans
>    before calling anything done** — "it looks right" is not a pass.
> 7. **Never assume a page, component, or variable persisted between
>    `use_figma` calls** — page creation has silently failed to persist
>    before. Re-verify existence before building on top of anything
>    created in an earlier call (see references/figma-api-patterns.md).
> 8. **If the `Color Tag` component doesn't exist in the file yet,
>    create it (Phase 4.1)** — don't skip it and don't invent an ID for
>    a component that isn't actually there.
> 9. **Every component this skill creates (`Color Tag`, and any future
>    reusable component) is built on a dedicated `❖ Local Components`
>    page — never on the same page as the `Documentation` frame that
>    consumes it.** A component created directly on the working page
>    lands at (0,0) by default, which is exactly where `Documentation`
>    is also built — the component silently overlaps/hides behind the
>    frame. Find-or-create that page first (Phase 4.0/4.1), build the
>    component there, then reference it by ID from wherever
>    `Documentation` lives.
> 10. **`Color Info` has no fill and stretches to its parent's width**
>     (`fills = []`, `layoutSizingHorizontal = "FILL"`), and its
>     `Subtitle`/`Description` text children are `layoutSizingHorizontal
>     = "FILL"` with `textAutoResize = "HEIGHT"` — never a manually
>     `resize()`-ed fixed text width. Fixed-width text is what causes
>     titles/descriptions to overlap the swatch column or get clipped
>     when a `Color Section`'s column width changes.
> 11. **Shade steps are iterated from an ordered array, never a plain
>     JS object keyed by shade.** `"100"`–`"950"` are canonical numeric
>     strings (no leading zero), so a JS engine silently reorders them
>     numerically ahead of `"050"` (which has a leading zero and isn't
>     canonical) when you do `Object.keys(shadesObject)` — `050` ends up
>     last instead of first. Always drive shade iteration from
>     `const orderedShades = ["050","100","200","300","400","500","600","700","800","900","950"]`
>     and look up hex/values by key from that array — never rely on
>     object key insertion order for the shade sequence.
> 12. **A newly created palette page defaults to a generic name,
>     `🎨 Color Palette` — never a name that embeds the palette's
>     technical or brand name** (e.g. not `🎨 Palette –  Core Colors
>     Colors`, not `🎨 Palette – {palette-name}`), unless the designer
>     explicitly asks for a specific page name in their request. See
>     Q5.

---

## Phase 0 — Execution order (the whole pipeline, at a glance)

Always in this order. Never partially skipped, never reordered:

1. **Phase 1** — Ask Q0→Q5 (native tool, in the person's own language).
2. **Phase 2** — Compute the HSL scale (isolated-hex case only).
3. **Phase 3** — Create or reuse the `color` + `placeholder` collections.
4. **Phase 4** — Find-or-create the `❖ Local Components` page, then
   reuse the `Color Tag` component there, or create it from scratch on
   that page if this file doesn't have one yet.
5. **Phase 6** — Build or find `Documentation` → `grid`, fully, before
   Phase 7 (next step) starts.
6. **Phase 7** — Build the `Color Section`(s) directly inside that
   `grid`, using Phase 5's `gap-6` + the component from step 4 + the
   variables from step 3.
7. **Phase 8** — Run the validation script, show the filled-in
   Definition of Done to the designer.

(Phase 5 has no separate pipeline step — it's the `gap-6` sub-structure
assembled as part of step 6/Phase 7.)

---

## Phase 1 — Mandatory intake questions

> All question text below is written in English because that's this
> skill's documentation language — it is **not** a fixed script. Always
> ask Q0–Q5 in the person's own conversation language, translating
> naturally (including the quick-pick option labels). Ask them, in order,
> every time this skill is used, using the native tappable-options tool
> (never plain prose with markdown bullets — see Non-negotiable #5).

Two question formats:
- **Closed questions** (Q1, and the incomplete-list case in Q2b) have a
  fixed, exhaustive set of options — present only those.
- **Open questions** (Q0, hex values, palette name, title/description)
  ask for arbitrary data that can't be predicted. Quick-pick options may
  be offered *in addition to* free text, but free text always stays
  available and takes priority over any quick pick.

Group at most 2–3 questions per tool call when they're naturally asked
together (e.g. Q2a's hex + naming prompts), and keep the accompanying
message short.

### Q0 — Target Figma file
"Which Figma file am I working in? (link or fileKey)"

Required before anything else — no `use_figma` call is possible without a
valid `fileKey`. Extract `fileKey` from a pasted URL
(`figma.com/design/{fileKey}/...`). Skip if already given earlier in the
conversation.

Quick picks + free text:
1. "You'll give it to me later — for now you just want a simple listing
   of palettes"
2. "You don't have these details yet, wait"
3. [free text — link or fileKey]

### Q1 — Input mode (closed choice)
"Are you giving me one or more isolated base hex codes (I'll generate the
050→950 scale for each), or a full list of colors you've already defined
(every shade already provided)?"

- **A. Isolated base hex(es)** → go to Q2a. Each hex is independent and
  generates its own full 050-950 scale (Phase 2 algorithm). Multiple
  hexes = multiple independent palettes.
- **B. Full color list** → go to Q2b.

### Q2a — If isolated hex(es)
"Give me the base hex(es)."

Quick picks + free text:
1. "Generate 2 random color palettes"
2. "Generate a single palette, preferably an accent color"
3. [free text — one or more hex codes]

"For each hex, what name do you want for the palette?" (one name per hex)

Quick picks + free text:
1. "Generate the naming yourself"
2. "Use \"placeholder\" for now, we'll name it later"
3. [free text — palette name]

### Q2b — If full color list
1. "Give me the list of colors (hex), in 050→950 order."
2. "What name do you want for this palette?" (same quick picks as Q2a)
3. "Do the colors have exact names to keep, or should I auto-number them
   050→950?"
   - Precise names given → use them verbatim, never modify.
   - No names given → auto-number in the order provided.
   - **If the count doesn't match the 11 standard steps** (050, 100, 200,
     300, 400, 500, 600, 700, 800, 900, 950) → **stop and ask for
     explicit confirmation** before completing, inferring, or dropping
     anything. Never guess missing shades.

### Q3 — Existing collection check (automatic, not a question)
Before creating anything:
- Check whether a `color` collection already exists.
- Check whether a `placeholder` collection already exists.
- Check whether the requested palette name already exists as a subfolder
  in either collection.

If a conflict is found → ask the designer for explicit confirmation
before overwriting.

### Q4 — Title and description
"What title should be displayed for this palette?" (can differ from the
technical variable name, e.g. technical `primitives-brand`, displayed
title `Brand`)

"What description do you want displayed under the title?"

Quick picks + free text (applies to both):
1. "Generate the copy yourself"
2. "Use \"title\" and \"description\" as placeholders for now"
3. [free text]

If option 1 is picked, the generated title/description is written in
**English** (Non-negotiable #4) — free text (option 3) is always used
exactly as given, untranslated.

### Q5 — Placement
"Should I create this on a new page, or add it to an existing one?"

If new page → the default page name is always the generic
`🎨 Color Palette` (Non-negotiable #12) — never build the name from the
palette's technical or brand name (not `🎨 Palette – {name}`, not
`🎨 Palette – Core Colors`, etc.). Only deviate from that
generic default when the designer explicitly states the page name they
want, either as free text answering this question or stated earlier in
their request — use their wording verbatim in that case. If a page named
`🎨 Color Palette` already exists in the file and this is a distinct,
unrelated set of palettes (not simply adding to the existing one, which
is Q5's "add to an existing one" / Phase 6.0 reuse path), ask the
designer to confirm or provide a distinguishing name rather than silently
creating a second page with the same name.

Once the target page is known, check for an existing `Documentation`
frame per **Phase 6.0** before building anything — reuse it (append to
its `grid`) rather than duplicating the Header/Footer/stroke structure.

---

## Phase 2 — Scale generation (isolated-hex case only)

HSL algorithm, validated against `#424CF9`:

1. Convert base hex → HSL (hue H, lightness L, saturation S).
2. Base color is anchored exactly at shade **500** — no distortion.
3. Lightness curve across the 11 steps: start near-white (~0.97) at 050,
   descend to near-black (~0.20×L base) at 950, with 500 equal to base L.
4. Saturation curve: stay close to base S across the whole scale, slightly
   reduced at both extremes (very light / very dark) to avoid garish or
   muddy colors — never push saturation to 100% at the light end.
5. Convert each HSL point back to hex.

This mode applies **only** to isolated hexes (Q2a). For a full list
(Q2b), no computation happens — provided colors are used as-is.

**Always keep the 11 steps in an ordered array**
(`["050","100","200","300","400","500","600","700","800","900","950"]`),
never a plain object keyed by shade — see Non-negotiable #11. Build the
palette as an array of `{ shade, hex }` pairs (or a `Map`) in that fixed
order, and drive every later loop (variable creation, Color Tag creation,
gap-6 population) from that same ordered structure so `050` always ends
up first, both in the data and in the rendered stack.

---

## Phase 3 — Figma variable structure

> Creating the collections/variables themselves uses the Figma Variables
> API, which has its own non-obvious method names — see
> "Creating variable collections & variables from scratch" in
> `references/figma-api-patterns.md` before writing this code. Always
> reuse-or-create (check `getLocalVariableCollectionsAsync` /
> `getLocalVariablesAsync` first) — never blind-create, or re-running this
> skill on a file that already has palettes will duplicate collections.

### `color` collection (single collection for the whole file)
- Name it exactly `color` (lowercase). Never create a new collection per
  palette — always reuse the existing one (see Q3).
- Organize as subfolders per palette: `{palette-name}/{shade}`
  (e.g. `primitives-brand/500`).
- Each variable's resolved type is `COLOR` (a fixed Figma API value —
  unrelated to the collection's own name, which stays lowercase). Set
  explicit scopes (`FRAME_FILL`, `SHAPE_FILL`) — never leave the default
  `ALL_SCOPES`.

### `placeholder` collection (single collection, STRING type)
- Also a single collection for the whole file, subfoldered per palette,
  with two groups per palette:
  - `{palette-name}/hex/{shade}` → hex code as text
    (e.g. `primitives-brand/hex/500` = `"#424cf9"`)
  - `{palette-name}/name/{shade}` → full shade name
    (e.g. `primitives-brand/name/500` = `"primitives-brand-500"`)
- Lets a designer change displayed names/hex later without touching
  instances.

---

## Phase 4 — `Color Tag` component

A badge showing one color shade: a swatch + its name.

### 4.0 — Dedicated components page (reuse check, always run first)
Search the whole file for a page named exactly `❖ Local Components`
(check `figma.root.children`, not just the current page). If it doesn't
exist yet, create it with `figma.createPage()` before doing anything
else in this phase — every component this skill builds lives there, not
on the page that will hold `Documentation` (see Non-negotiable #9).

Then, within that page, search for a component named exactly `Color Tag`.
If found, keep its ID and reuse it for every instance in Phase 7 — never
rebuild it from scratch. If a `Color Tag` component is found sitting on
any *other* page (e.g. a leftover from before this rule existed), leave
it in place and reuse it rather than duplicating — but prefer moving it
to `❖ Local Components` (`localComponentsPage.appendChild(component)`)
when a designer asks for a fix, so future lookups stay in one place.

### 4.1 — If it doesn't exist yet, create it
Build it once, **on the `❖ Local Components` page**
(`await figma.setCurrentPageAsync(localComponentsPage)` first), as a real
Figma component (`figma.createComponent()`, not `figma.createFrame()` —
same layout API, natively instanceable, see
references/figma-api-patterns.md):
- Container: horizontal auto-layout, white fill, `#cbd5e1` stroke,
  `border-radius: 8px`, padding `8px`, gap `8px`, centered content, HUG
  sizing (no fixed width — it varies with the palette-name string).
- Swatch (`Color Tag Swatch`): `20×20px` square, `border-radius: 5px`,
  `1.25px #cbd5e1` stroke — a plain placeholder fill at creation time
  (never bind a variable at the master-component level; binding happens
  per-instance in Phase 7).
- Label (`Color Tag Label`): `Inter Medium`, `14px`, `line-height: 24px`,
  black, placeholder text at creation time (bound per-instance later).
- Name the finished node exactly `Color Tag`.
- Position it away from (0,0) on the `❖ Local Components` page if other
  components already live there (Rule 13 in figma-use applies to this
  page too, independently of wherever `Documentation` is built).

Once built, treat it exactly like a found component for the rest of the
flow (back to the 4.0 "reuse" path from here on) — **never** leave the
swatch fill or label text hardcoded on the actual instances used in
Phase 7; those must always be bound (Non-negotiable #2). Instances
themselves (`colorTag.createInstance()`) are created directly inside
`gap-6` on the `Documentation` page in Phase 7 — only the master
component lives on `❖ Local Components`.

---

## Phase 5 — `gap-6` frame

Groups the 11 `Color Tag` instances of one palette. Assembled as part of
Phase 7 (see Phase 0's execution order) — not a standalone build step.

- Vertical auto-layout, named exactly `gap-6`.
- **Gap: 24px.**
- **No padding** (0 on all 4 sides).
- **No fill** (empty `fills`).
- Populate it by iterating the Phase 2 ordered shade array/list — never
  a plain object's `Object.keys()` — so `050` lands first and `950` last
  (Non-negotiable #11).

---

## Phase 6 — `Documentation` frame (build or find this BEFORE Phase 7)

Every `Color Section` is born inside a `grid`, inside a `Documentation`
frame — never on the bare canvas. This phase always **completes** before
Phase 7 starts: Phase 7 needs the `grid` node produced here, so there is
no valid path that skips it. `Documentation` lives on whichever page
Phase 1/Q5 designated (its own palette page, or an existing one) — never
on the `❖ Local Components` page from Phase 4.

### 6.0 — Check first
Inspect the target page for a frame named `Documentation` containing a
`grid` child.
- **Found** → skip straight to Phase 7, using this `grid` as the parent.
  Don't rebuild Header/Footer/stroke, don't re-ask the page-level
  title/description.
- **Not found** → build 6.1 → 6.5 below, asking for the page-level
  title/description once (same pattern as Q4; English default when
  generated, Non-negotiable #4).

### 6.1 — Root frame `Documentation`
- Vertical auto-layout, `itemSpacing: 32px`.
- Padding: `64px` top and bottom, **0 left/right**.
- Fixed width `2716px` (reference canvas width — adjust only if the
  designer explicitly asks for a different frame width).
- Fill: white.

### 6.2 — `stroke` (decorative overlay)
A single rectangle marking the horizontal extent of the `grid` content
area.
- **Absolute-positioned** (`layoutPositioning = "ABSOLUTE"`, set only
  *after* appending to `Documentation`), set as the **first child** of
  `Documentation` (sits behind everything else).
- Inset **`64px`** from both the left and right edges of `Documentation`
  (matches the `grid`'s own `padding-x`, see 6.4 — intentional: the
  stroke marks the grid's content width, not the Header/Footer's, which
  are indented further at `96px`).
- Height: full height of `Documentation` (must be re-measured and updated
  every time a Color Section is added — see 6.6).
- No fill. Left/right border only: `0.5px`, `#cbd5e1`. No top/bottom
  border. In practice a Rectangle has no independent per-side stroke
  control, so build this as two thin (`0.5px`-wide) absolute-positioned
  rectangles — one at the left inset, one at the right inset — both
  resized to `Documentation`'s current height and both kept in sync at
  6.6.

### 6.3 — `Header`
Vertical auto-layout, `itemSpacing: 8px`, full width. Exactly 2 children,
both sharing the same block pattern (horizontal auto-layout, centered,
padding `96px` horizontal / `16px` vertical, border-top **and**
border-bottom `0.5px #cbd5e1`, full width):

1. **`gap-4`** (Title block) → contains **`Title`** text: `Inter Semi
   Bold`, 36px, black, `line-height: 40px`, fills available width.
2. **`gap-0`** (Description block) → contains **`Description`** text:
   `Inter Regular`, 16px, `#64748b`, `line-height: 24px`, fills available
   width.

This Title/Description is **page-level** (describes the whole color
system, e.g. brand name + purpose) — distinct from the per-palette
title/description asked in Q4, which lives inside each `Color Section`'s
"Color Info" (Phase 7), and distinct from the page *name* itself (Q5,
Non-negotiable #12) — the page can be named generically `🎨 Color
Palette` while this Header `Title` still says something specific like
"Brand Color System", since one is Figma's page-list label and the other
is in-canvas content.

### 6.4 — `grid` (empty, ready to receive Color Sections)
- Horizontal auto-layout with **`layoutWrap = "WRAP"`**.
- Padding: **`64px` left/right, `0` top/bottom**.
- `itemSpacing: 64px` (horizontal gap) **and** `counterAxisSpacing: 64px`
  (vertical row gap) — both required together for the wrap to space rows
  and columns identically (see references/figma-api-patterns.md).
- `primaryAxisSizingMode: "FIXED"`, width `2716px` (same as
  `Documentation`).
- No fill.
- **Confirmed behavior**: 3 `Color Section`s at `820px` fit one row; a
  4th wraps automatically to a new row (validated with 4 test palettes:
  3 on row 1, 1 on row 2).
- This is the exact node Phase 7 appends into. Keep its ID.

### 6.5 — `Footer`
Vertical auto-layout, centered, padding `96px` horizontal / `16px`
vertical, border-top **and** border-bottom `0.5px #cbd5e1`, full width.
Contains one child, **`gap-2.5`** (horizontal auto-layout, centered),
containing a **`Footer`** text: `Inter Medium`, 12px, black,
`line-height: 16px`.

### 6.6 — After Phase 7 adds a Color Section
Resize `stroke`'s height to the new `Documentation` height — it grows
every time a palette is added, so this happens every time, not just at
creation.

---

## Phase 7 — `Color Section` (created directly inside `grid`)

> ⛔ There is exactly **one** valid parent for a `Color Section`: the
> `grid` node from Phase 6.4. If the code about to run is calling
> `page.appendChild(...)` or `figma.currentPage.appendChild(...)` on a
> `Color Section`, stop — that target is always wrong, and it means
> Phase 6 got skipped. Create the frame and call
> `grid.appendChild(section)` in the same step, not later as a separate
> reparenting pass.

One instance per generated palette. Duplicate this template for each new
palette, always as a direct child of `grid`, resized to a fixed column
width: `colWidth = (2716 - 64×2 padding - 64×2 gaps) / 3 = 820px`, then
`layoutSizingHorizontal = "FIXED"` on the section.

- Root frame **"Color Section"**: horizontal auto-layout, gap **156px**,
  padding **40px** (all sides), fill `#f8fafc`.
- Exactly 2 children:
  1. **"Description" frame**: vertical auto-layout, `justify-between`,
     `flex-1` horizontally (`layoutSizingHorizontal = "FILL"`) **and
     explicit height-fill** (`layoutSizingVertical = "FILL"`), **no fill**
     (empty `fills`).
     > ⚠️ **Known gotcha**: `layoutAlign = "STRETCH"` alone is **not**
     > enough — it silently leaves the frame in `HUG` mode vertically.
     > `layoutSizingVertical` must be set explicitly to `"FILL"` (after
     > the frame has been appended to its parent), or the "Description"
     > block won't stretch to match "gap-6"'s height and
     > `justify-between` will have no effect.
     Contains:
     - **"Color Info"** subgroup (vertical auto-layout, gap 8px):
       - **No background fill** (`fills = []`) and
         **`layoutSizingHorizontal = "FILL"`** so it always follows its
         parent ("Description")'s current width instead of hugging a
         stale one — this is what keeps it from overlapping or clipping
         against the swatch column when a Color Section's width changes
         (Non-negotiable #10).
       - **"Subtitle"** text: `Inter Semi Bold`, 18px, black — the
         palette's displayed title (Q4). `layoutSizingHorizontal =
         "FILL"`, `textAutoResize = "HEIGHT"` — never a manually
         `resize()`-ed fixed width.
       - **"Description"** text: `Inter Regular`, 16px, `#64748b` — the
         displayed description (Q4). Same rule: `layoutSizingHorizontal
         = "FILL"`, `textAutoResize = "HEIGHT"`, no manual fixed-width
         `resize()`.
       > 📏 **No truncation, ever.** "Subtitle" and "Description" must
       > never be cut off. "Color Info" hugs their full height
       > (`layoutSizingVertical = "HUG"`, never a fixed height) with
       > `clipsContent = false`. The outer "Description" frame (above)
       > must also have `clipsContent = false` — even though it's
       > height-fill to match "gap-6", it must let "Color Info" grow
       > past that height rather than clipping it if the title or
       > description text is long.
     - **"Primary Color"** square: `32×32px`, `border-radius: 8px`, fill
       bound to the palette's `500` color variable.
  2. **"gap-6" frame** (Phase 5) listing the palette's 11 `Color Tag`
     instances (from the `❖ Local Components` page's master component,
     Phase 4), each bound to its variables (Non-negotiable #2), created
     in the Phase 2 ordered-array sequence so `050` renders first and
     `950` last (Non-negotiable #11).

Once this Color Section is complete, go back to **Phase 6.6** to resize
`stroke`, then run **Phase 8**.

---

## Phase 8 — Validation (run a script, don't just eyeball it)

A prose checklist is too easy to "mentally tick off" without actually
checking. Instead, **run this exact script through `use_figma`** after
every `Color Section` generation, and read its returned booleans — don't
paraphrase compliance from memory.

### 8.1 — Validation script

```javascript
async function validateColorSection(sectionId) {
  const section = await figma.getNodeByIdAsync(sectionId);
  const report = {};

  // 1. Structural: must be a direct child of a frame named "grid"
  report.parentIsGrid = section.parent?.name === "grid";

  // 2. Structural: that "grid" must live inside a "Documentation" frame
  report.gridInsideDocumentation = section.parent?.parent?.name === "Documentation";

  // 3. Structure: exactly "Description" + "gap-6" children
  const description = section.findOne(n => n.name === "Description" && n.type === "FRAME");
  const gap6 = section.findOne(n => n.name === "gap-6");
  report.hasDescriptionAndGap6 = !!description && !!gap6;

  // 4. Count: exactly 11 Color Tag instances
  const tags = gap6 ? gap6.children.filter(c => c.name === "Color Tag") : [];
  report.tagCount = tags.length;
  report.tagCountOk = tags.length === 11;

  // 5. Variable binding: swatches must be bound, never a hardcoded hex
  report.swatchesBound = tags.length > 0 && tags.every(tag => {
    const swatch = tag.findOne(n => n.name === "Color Tag Swatch");
    return !!swatch?.fills?.[0]?.boundVariables?.color;
  });

  // 6. Variable binding: labels must be bound to the placeholder variable
  report.labelsBound = tags.length > 0 && tags.every(tag => {
    const label = tag.findOne(n => n.name === "Color Tag Label");
    return !!label?.boundVariables?.characters;
  });

  // 7. Known gotcha: Description must be explicitly height-fill
  report.descriptionHeightFill = description?.layoutSizingVertical === "FILL";

  // 8. stroke height must match the current Documentation height
  const doc = section.parent?.parent;
  const stroke = doc?.findOne(n => n.name === "stroke");
  report.strokeHeightMatchesDoc = stroke && doc ? stroke.height === doc.height : null;

  // 9. Color Info: no fill, follows parent width
  const colorInfo = section.findOne(n => n.name === "Color Info");
  report.colorInfoNoFillAndFillsWidth = !!colorInfo &&
    colorInfo.fills.length === 0 &&
    colorInfo.layoutSizingHorizontal === "FILL";

  // 10. Shade order: first tag must be 050, last must be 950
  report.shadeOrderOk = tags.length === 11 && (() => {
    const first = tags[0].findOne(n => n.name === "Color Tag Label");
    const last = tags[tags.length - 1].findOne(n => n.name === "Color Tag Label");
    return first?.characters?.endsWith("050") && last?.characters?.endsWith("950");
  })();

  report.allPass = report.parentIsGrid && report.gridInsideDocumentation &&
    report.hasDescriptionAndGap6 && report.tagCountOk && report.swatchesBound &&
    report.labelsBound && report.descriptionHeightFill &&
    report.strokeHeightMatchesDoc !== false &&
    report.colorInfoNoFillAndFillsWidth && report.shadeOrderOk;

  return JSON.stringify(report, null, 2);
}

return await validateColorSection("PASTE_COLOR_SECTION_ID_HERE");
```

### 8.2 — Definition of Done

Only report a generation as complete once every line below is checked
**from the script's returned booleans**, not from visual impression.
Display this checklist to the designer, filled in, at the end of the
generation:

- [ ] `parentIsGrid` — Color Section is a direct child of `grid`
- [ ] `gridInsideDocumentation` — that `grid` lives inside `Documentation`
- [ ] `hasDescriptionAndGap6` — structure is Description + gap-6
- [ ] `tagCountOk` — exactly 11 Color Tag instances
- [ ] `swatchesBound` — every swatch bound to a color variable
- [ ] `labelsBound` — every label bound to a placeholder variable
- [ ] `descriptionHeightFill` — Description is explicitly height-fill
- [ ] `strokeHeightMatchesDoc` — stroke resized to the final Documentation
      height
- [ ] `colorInfoNoFillAndFillsWidth` — Color Info has no fill and follows
      its parent's width
- [ ] `shadeOrderOk` — the stack renders 050 first and 950 last

Also separately verify (not scriptable via node inspection alone, but
check once per session): the `Color Tag` master component lives on the
`❖ Local Components` page, not on the page holding `Documentation`
(Non-negotiable #9) — `colorTag.parent.name === "❖ Local Components"`;
and the palette page's name is the generic `🎨 Color Palette` unless the
designer explicitly asked for something else (Non-negotiable #12).

If any box would be unchecked, the task is **not done** — fix it and
re-run the script before telling the designer it's finished. Always also
return: page ID, Color Section ID, `Documentation` ID, `❖ Local
Components` page ID, created variable IDs.

---

## Cross-cutting rules
- Always inspect existing pages, collections, and components before
  creating anything — never duplicate what's already there without
  confirmation.
- Work in small `use_figma` steps, validate after each one.
- Always return created/mutated node IDs.
- Never hardcode a color or name in a final instance — everything must be
  a bound variable.
- All text Claude generates for Figma content is in English by default,
  regardless of the conversation's language; designer-supplied free text
  is always used verbatim, untranslated (Non-negotiable #4).
- Components (`Color Tag`, and any future reusable component this skill
  introduces) always live on the dedicated `❖ Local Components` page,
  never on the same page as the `Documentation` frame that instances them
  (Non-negotiable #9).
- Shade steps (050–950) are always driven from an ordered array, never a
  plain object's key order, so `050` never gets silently sorted after
  `950` (Non-negotiable #11).
- A newly created palette page always defaults to the generic name
  `🎨 Color Palette`, never one built from the palette's technical or
  brand name, unless the designer explicitly requested a specific page
  name (Non-negotiable #12).
- Phase 8's validation script must run and return `allPass: true` before
  a generation is reported as finished (Non-negotiable #6).
- This skill is written to work for any designer, in any Figma file —
  never assume a specific file, palette name, or prior context beyond
  what Q0–Q5 and Phase 6.0's existence checks actually establish.