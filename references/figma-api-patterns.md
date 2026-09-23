# Figma Plugin API — validated patterns

These are exact, tested call shapes for this skill's `use_figma` code.
They exist because the Figma Plugin API has several non-obvious defaults
that are easy to get subtly wrong from memory — read this once per
session before writing creation code, not just when something breaks.

## Creating variable collections & variables from scratch
Needed for Phase 3 (`color` and `placeholder` collections). Reuse first,
create only if missing:
```javascript
// Reuse-or-create a collection
async function getOrCreateCollection(name) {
  const collections = await figma.variables.getLocalVariableCollectionsAsync();
  let collection = collections.find(c => c.name === name);
  if (!collection) {
    collection = figma.variables.createVariableCollection(name);
  }
  return collection;
}

const colorCollection = await getOrCreateCollection("color");
const placeholderCollection = await getOrCreateCollection("placeholder");
const modeId = colorCollection.modes[0].modeId; // single-mode collections: modes[0]

// COLOR variable (subfoldered name = the variable's own name, Figma renders the "/" as folders)
async function getOrCreateColorVar(collection, fullName, rgbHex) {
  const existing = (await figma.variables.getLocalVariablesAsync("COLOR"))
    .find(v => v.name === fullName && v.variableCollectionId === collection.id);
  const variable = existing ?? figma.variables.createVariable(fullName, collection, "COLOR");
  variable.scopes = ["FRAME_FILL", "SHAPE_FILL"];
  variable.setValueForMode(collection.modes[0].modeId, hexToRgb(rgbHex)); // {r,g,b} 0-1 floats
  return variable;
}
// e.g. getOrCreateColorVar(colorCollection, "primitives-brand/500", "#424cf9")

// STRING placeholder variable (same pattern, resolvedType "STRING", plain string value)
async function getOrCreateStringVar(collection, fullName, value) {
  const existing = (await figma.variables.getLocalVariablesAsync("STRING"))
    .find(v => v.name === fullName && v.variableCollectionId === collection.id);
  const variable = existing ?? figma.variables.createVariable(fullName, collection, "STRING");
  variable.setValueForMode(collection.modes[0].modeId, value);
  return variable;
}
// e.g. getOrCreateStringVar(placeholderCollection, "primitives-brand/name/500", "primitives-brand-500")
```
Always check-then-create (never blind-create) — re-running this skill on a
file that already has the collection must extend it, not duplicate it.

## Page & node persistence (real failure observed — treat as a hazard)
`figma.createPage()` has, in practice, occasionally returned a successful
ID for a page that **did not persist** by the next `use_figma` call — the
page vanished even though the previous call reported success.
- After creating a page (or anything you'll reference in a *later*
  separate `use_figma` call), don't just trust the returned ID blindly on
  the next call. If a lookup fails, immediately run
  `figma.root.children.map(p => ({id: p.id, name: p.name}))` to see the
  real current state before assuming your write was lost or before
  silently recreating (recreating without telling the designer can
  duplicate work or orphan content).
- Prefer doing related creation steps (page + its first content) inside
  **one** `use_figma` call when practical, to reduce the number of
  cross-call round trips that can hit this.

## Page switching & node retrieval
- `await figma.setCurrentPageAsync(page)` before creating any node on
  that page.
- `await figma.getNodeByIdAsync(id)` for all node lookups, especially
  across separate `use_figma` calls or across pages — synchronous
  `getNodeById` is unreliable for this.
- Reference pages/nodes **by the ID returned from creation**, not by
  name — names aren't a reliable lookup key across calls.

## Auto-layout sizing (FILL vs HUG)
`figma.createFrame()` / `figma.createAutoLayout()` children default to
**HUG**. To make a child stretch to fill its parent:
1. Append the child to its parent first.
2. Set `child.layoutSizingHorizontal = "FILL"` and/or
   `child.layoutSizingVertical = "FILL"` — **after** appending, never
   before (the property doesn't exist meaningfully until the node has an
   auto-layout parent).
3. The parent's own relevant axis must itself be resolved (usually
   `primaryAxisSizingMode = "FIXED"` with an explicit `resize(w, h)`
   call) — a child can't FILL an axis the parent itself is still hugging.

> ⚠️ Known trap: `layoutAlign = "STRETCH"` alone does **not** guarantee
> fill sizing on the cross axis — it can silently leave a frame in HUG.
> Always set `layoutSizingVertical`/`layoutSizingHorizontal` to `"FILL"`
> explicitly when that's the intent; don't rely on `layoutAlign` alone.

## Binding a fill to a color variable
```javascript
const variable = await figma.variables.getVariableByIdAsync(variableId);
const baseFill = { type: "SOLID", color: { r: 0, g: 0, b: 0 } }; // plain object first
const boundFill = figma.variables.setBoundVariableForPaint(baseFill, "color", variable);
node.fills = [boundFill]; // reassign the array — mutating fills[0] in place doesn't take effect
```

## Binding text characters to a placeholder (STRING) variable
```javascript
await figma.loadFontAsync(textNode.fontName); // font must be loaded first
const variable = await figma.variables.getVariableByIdAsync(placeholderVarId);
textNode.setBoundVariable("characters", variable);
```

## Variable scopes
Set explicitly right after creating a variable — the default `ALL_SCOPES`
causes it to show up in unrelated pickers in Figma's UI:
```javascript
variable.scopes = ["FRAME_FILL", "SHAPE_FILL"]; // for COLOR variables
```

## Creating a component vs a frame
`figma.createComponent()` behaves exactly like `figma.createFrame()` for
layout purposes (auto-layout, padding, fills — same API) but produces a
node that's natively instanceable. Use it directly for anything meant to
be reused as a component (e.g. `Color Tag`) — there's no
frame-to-component "conversion" step needed.

## get_metadata / get_design_context vs use_figma
- `get_metadata` and `get_design_context` require the target file to be
  the **active tab** in the Figma desktop app. If they error with "No
  node could be found... make sure the app is open and active tab", that
  means focus, not that a prior write failed.
- `use_figma` (Plugin API) works independently of which tab has focus in
  the desktop app.
- Don't conclude a `use_figma` write failed just because a follow-up
  `get_metadata`/`get_design_context` call errors — ask the designer to
  switch to the right tab and retry the read, or just trust `use_figma`'s
  own returned confirmation.

## Wrapping grid (multi-column, auto-wrap to new rows)
Both of these are required together — one alone won't wrap correctly:
```javascript
grid.layoutMode = "HORIZONTAL";
grid.layoutWrap = "WRAP";
grid.itemSpacing = 64;        // horizontal gap between columns
grid.counterAxisSpacing = 64; // vertical gap between wrapped rows
```

## Absolute-positioned overlay (e.g. a decorative `stroke` rectangle)
```javascript
parent.appendChild(rect);       // append first
rect.layoutPositioning = "ABSOLUTE"; // then detach from auto-layout flow
rect.x = 64; rect.y = 0;        // then position manually
```
