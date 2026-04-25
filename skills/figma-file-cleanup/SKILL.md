---
name: figma-file-cleanup
description: Audit and clean up Figma files — remove orphan library references, rebind colors and text to design system tokens, audit dark mode compatibility, fix naming conventions, and ensure file hygiene. Use when user says 'clean up figma', 'figma audit', 'fix figma styles', 'orphan variables', 'detach variables', 'dark mode check', 'fix dark mode', or shares a Figma URL and wants it cleaned up.
---

# Figma File Cleanup Skill

You are performing a Figma file cleanup using the figma-console MCP tools. The scope can range from a single focused task (e.g., "remap colors") to a full deep audit. Always confirm changes with the user before executing — they are a designer, not a programmer, so explain everything in plain English.

---

## Phase 0: Discovery Interview

Before scanning anything, determine the **scope** and gather context. Skip any questions already answered in conversation or saved in memory.

### 0.1 Determine Scope

First, understand what the user needs. Ask:

**"What would you like to clean up?"** Then classify their answer into one of these modes:

| Mode | Trigger phrases | What runs |
|---|---|---|
| **Focused** | "remap colors", "fix dark mode", "detach orphan variables", "fix text styles" | Skip full audit. Run only the relevant phase (2, 2.5, 3, or 4) with a lightweight scan of just that area. |
| **Targeted** | "clean up this page", "audit the colors and typography" | Run Phase 1 audit for the requested areas only, then execute those phases. |
| **Full** | "full cleanup", "deep audit", "clean up figma", no specific scope mentioned | Run all phases sequentially with checkpoints. |

If the user's intent is unclear, ask: **"Do you want me to focus on a specific area (like colors or typography), or do a full deep audit of the file?"**

For **Focused mode**: skip to the relevant phase directly after answering only the questions needed for that phase (e.g., for colors you only need Q1 and Q2 below).

For **Targeted** or **Full mode**: ask all applicable questions below.

### 0.2 Context Questions

Ask these as needed based on scope. Skip any already answered or saved in memory.

1. **Which page(s) should we audit?** All pages, current page, or specific ones?
2. **Which libraries are your source of truth?** Ask for exact library names (e.g., "Design System", "Enterprise Dashboard"). These are the ONLY libraries whose variable/style bindings we keep.
3. **Are there known orphan libraries to auto-flag?** e.g., "anything shadcn, Tailwind, community kits"
4. **Naming convention for frames/layers?** Numbered, descriptive, or "don't worry about naming" *(Full mode only)*
5. **Audit-only or audit-and-fix?** Generate a report first, or fix as we go with confirmation at each phase?
6. **Known exceptions?** Any frames, components, or patterns to leave alone
7. **Instance-level inherited issues** — Flag only, or attempt to override at instance level?
8. **Priority order** — Colors first? Typography? Or full sequential cleanup? *(Full mode only)*

Store answers as session context. Check Claude's saved memory for any previously stored design system preferences before re-asking.

### 0.3 Connection Check

Run `figma_get_status` to verify MCP connection. If not connected, STOP and guide the user through reconnection:
1. Figma Desktop must be open
2. Desktop Bridge plugin must be running (Plugins > Development > Figma Desktop Bridge)
3. The figma-console-mcp Desktop Bridge plugin must be installed and loaded (run `npx figma-console-mcp` once to fetch the package, then load the plugin manifest from your npx cache)

If 2+ status checks fail, STOP and tell the user what to fix before proceeding.

### 0.4 Pre-Cleanup Snapshot

Before making ANY changes, capture a visual baseline:

1. Get all sections/frames on the target page(s).
2. Take a screenshot of each major section using `figma_take_screenshot`.
3. Store the screenshot references as the "before" state.
4. These will be compared against "after" screenshots in Phase 6 to confirm nothing changed visually.

This step is critical for building confidence that automated fixes didn't break anything. For **Focused mode**, only snapshot the area being worked on.

---

## Phase 1: Audit & Discovery

Run these scans and present findings as a summary table BEFORE proceeding to any fixes.

### 1.1 Library Inventory

Use `figma_execute` to run:

```js
const collections = await figma.teamLibrary.getAvailableLibraryVariableCollectionsAsync();
return collections.map(c => ({ libraryName: c.libraryName, name: c.name, key: c.key }));
```

- Cross-reference each collection against the user's approved source-of-truth list.
- Flag any collection NOT in the approved list as "orphan".
- Present a table: Library Name | Collection Name | Status (Approved / Orphan)

### 1.2 Variable Audit

Use `figma_execute` to scan all nodes on target page(s):

```js
const nodes = page.findAll(n => n.boundVariables != null && Object.keys(n.boundVariables).length > 0);
```

For each bound variable, resolve its collection. Group results by:
- Collection name -> property type -> count
- Separate into: approved bindings vs orphan bindings

Present a report:
- Total nodes scanned
- Nodes with bindings vs without
- Orphan binding count grouped by collection

### 1.3 Color Audit

Find all nodes with solid fills or strokes that are NOT bound to any variable:

```js
const unbound = page.findAll(n => {
  if (!n.fills || n.fills.length === 0) return false;
  const hasSolidFill = n.fills.some(f => f.type === 'SOLID' && f.visible !== false);
  const hasBoundFill = n.boundVariables?.fills?.length > 0;
  return hasSolidFill && !hasBoundFill;
});
```

- Group by hex value, count occurrences, note node types and sample parent names.
- For each unbound color, attempt to match to the nearest semantic token from the approved design system.
- Import and resolve semantic token hex values for comparison.
- Present a table: Hex Value | Count | Node Types | Suggested Token | Confidence (Exact / Close / No Match)

### 1.4 Text Style Audit

Find all TEXT nodes on the page and check their `textStyleId`:

```js
const textNodes = page.findAll(n => n.type === 'TEXT');
const detached = textNodes.filter(n => !n.textStyleId || n.textStyleId === '');
```

- Group detached nodes by font family + weight + size combination.
- Cross-reference with available library text styles to find the closest match.
- Present a table: Font Combo | Count | Closest Library Style | Match Type (Exact / Close / None)

### 1.5 Local Variable Cleanup Audit

Check for local variable collections that duplicate library tokens or were auto-generated:

```js
const localCollections = await figma.variables.getLocalVariableCollectionsAsync();
```

For each local collection:
- List all variables with their types and values
- Compare against library tokens — flag duplicates (e.g., a local `color/white/solid` when `neutral/white` exists in the DS)
- Flag auto-generated variables with suspicious names (e.g., `item spacing/94_87`, `height/414_5`) — these are created by paste-from-code or plugin imports
- Report: Local Collection Name | Variable Count | Duplicates Found | Recommendation (rebind + delete / keep / investigate)

### 1.6 Component Audit (only if user requested or Full mode)

- Find instances detached from their main component.
- Find components used from non-approved libraries — suggest equivalent components from approved libraries.
- For each orphan component, check if an approved-library equivalent exists by comparing:
  - Component name/type (Button, Input, Modal, etc.)
  - Visual similarity
- Present: list of orphan components with suggested approved-library replacements.

### 1.7 Cross-Page Consistency Audit (Full mode only)

Compare how the same element types are styled across different pages:

1. For each approved page in the file, catalog the tokens used for common patterns:
   - Modals: background, border, shadow, text colors
   - Tables: header bg, row bg, divider color, text hierarchy
   - Cards: surface color, border, padding tokens
   - Navigation: active/inactive states, background
   - Buttons: primary, secondary, destructive variants
2. Cross-reference: flag when the same component type uses different tokens on different pages.
3. When inconsistencies are found, identify the "reference" version (the one that looks correct / is on an approved page) and recommend normalizing others to match.

This catches the "someone copied a frame from another file and it brought its own styles" problem.

### 1.8 Naming Audit (only if user requested or Full mode)

Find frames/groups with generic names:

```js
const generic = page.findAll(n =>
  /^(Frame|Group|Rectangle|Vector|Ellipse|Line|Polygon|Star)\s*\d*$/.test(n.name)
);
```

- Count and list by parent section.

### 1.9 Generate Audit Report

Write a comprehensive markdown report to the project directory: `[project-root]/figma-cleanup-audit.md`

Include:
- Executive summary
- Findings by category (colors, typography, variables, components, naming)
- Severity counts (critical / warning / info)
- Recommended action plan with estimated node counts per phase

Present the summary to the user and ask which phases to execute.

---

## Phase 2: Color Cleanup

### 2.1 Remap Unbound Colors to Semantic Tokens

For each unbound color with a high-confidence match:

1. Import the semantic token variable:
   ```js
   const collections = await figma.teamLibrary.getAvailableLibraryVariableCollectionsAsync();
   // Find the approved collection
   const vars = await figma.teamLibrary.getVariablesInLibraryCollectionAsync(collectionKey);
   const imported = await figma.variables.importVariableByKeyAsync(varKey);
   ```

2. Bind the variable to matching nodes:
   ```js
   const fills = node.fills;
   const newFill = figma.variables.setBoundVariableForPaint(fills[0], 'color', imported);
   node.fills = [newFill];
   ```

3. Split by context when choosing tokens:
   - TEXT nodes -> `text/` tokens
   - VECTOR nodes -> `icon/` tokens
   - FRAME, RECTANGLE, COMPONENT nodes -> `background/` or `surface/` tokens

**IMPORTANT:** Present the full mapping table to the user for confirmation BEFORE executing any changes. Explain what each change means in plain English.

Execute in batches of 100-200 nodes per batch to avoid Figma timeouts.

Report: count of nodes rebound per color.

### 2.2 Remap Orphan Color Variables to DS Equivalents

For each orphan color variable:

1. Resolve its current hex value by finding a node that uses it and reading the computed paint.
2. Find the closest semantic token from the approved design system.
3. Present the mapping to the user for confirmation.
4. Rebind fills:
   ```js
   const newFill = figma.variables.setBoundVariableForPaint(fill, 'color', dsVariable);
   node.fills = [newFill];
   ```
5. Rebind strokes similarly:
   ```js
   const newStroke = figma.variables.setBoundVariableForPaint(stroke, 'color', dsVariable);
   node.strokes = [newStroke];
   ```

**IMPORTANT:** Check both fills AND strokes for each variable.

Report: count of nodes remapped per variable.

### 2.3 Verify

Re-scan all fills and strokes on the target pages. Report:
- Remaining unbound color count
- Remaining orphan-bound color count
- Any that could not be resolved (with reason)

---

## Phase 2.5: Dark Mode Audit & Repair

This phase ensures all colors work correctly in both light and dark modes by using semantic tokens that have both mode values defined.

### 2.5.1 Identify Reference Designs

Before fixing dark mode, establish a visual baseline by finding approved/working designs in the file:

1. Ask the user: **"Are there any screens or components in this file (or other files) that already look correct in dark mode?"**
2. If yes, screenshot those reference designs in dark mode and catalog the semantic tokens they use for:
   - Page backgrounds
   - Card/surface backgrounds
   - Text colors (primary, secondary, tertiary)
   - Border/divider colors
   - Input field backgrounds and borders
   - Button styles
   - Status indicators and badges
3. Build a **reference color map** — "for this type of element, use this token" — derived from the working designs.
4. Use this reference map when fixing broken screens, so fixes are consistent with the rest of the app rather than invented from scratch.

### 2.5.2 Audit for Non-Semantic Colors

After Phase 2 color cleanup, check if any remaining bound colors are using **primitive tokens** (e.g., `grey/700`, `neutral/black`) instead of **semantic tokens** (e.g., `text/secondary`, `text/primary`):

```js
// Find nodes bound to primitive color collections instead of semantic ones
const page = figma.root.children.find(p => p.id === targetPageId);
const allNodes = page.findAll(n => n.boundVariables != null);

for (const node of allNodes) {
  const bv = node.boundVariables;
  for (const [prop, bindings] of Object.entries(bv)) {
    if (prop !== 'fills' && prop !== 'strokes') continue;
    for (const binding of bindings) {
      const v = await figma.variables.getVariableByIdAsync(binding.id);
      const col = await figma.variables.getVariableCollectionByIdAsync(v.variableCollectionId);
      // Flag if collection is "Colors" (primitives) instead of "Semantic Colors"
      if (col.name === 'Colors') flagAsPrimitive(node, v);
    }
  }
}
```

- Primitive tokens don't have light/dark mode values — they're single fixed colors.
- Semantic tokens alias to different primitives per mode — these are what enable dark mode.
- Report: list of nodes using primitive tokens, with suggested semantic equivalent.
- Present to user for confirmation before remapping.

### 2.5.3 Switch to Dark Mode and Screenshot

Use `figma_execute` to switch the page to dark mode:

```js
// Find the Semantic Colors collection and switch to dark mode
const collections = await figma.variables.getLocalVariableCollectionsAsync();
// Or for library collections, set the explicit mode on the page
const page = figma.currentPage;
// Set the mode to dark for each collection that has a dark mode
```

Alternatively, guide the user to manually switch the page mode to Dark in Figma's UI (bottom of the layers panel > Page section > Mode dropdown), then take screenshots of each key section using `figma_take_screenshot`.

### 2.5.4 Visual Dark Mode Review

For each section/screen on the page:

1. Take a screenshot in dark mode.
2. Analyze the screenshot for common dark mode issues:
   - **Black text on dark background** — text using a hardcoded dark color instead of a semantic text color
   - **White/light backgrounds that didn't flip** — surfaces still using literal white instead of `background/surface`
   - **Invisible borders** — borders using light colors that disappear on dark backgrounds
   - **Broken status indicators** — badges or pills that lose contrast
   - **Input fields blending into background** — input backgrounds not distinct from page background
   - **Shadows that look wrong** — drop shadows that were designed for light mode
3. For each issue found, identify the offending node and its current color binding.
4. Cross-reference with the **reference color map** from step 2.5.1 to find the correct token.
5. Present findings with before screenshots.

### 2.5.5 Fix Dark Mode Issues

For each identified issue:

1. If a semantic token exists that would fix it → rebind to that token.
2. If no suitable semantic token exists:
   - Flag it: "No semantic token covers this use case."
   - Propose creating a new semantic token in the DS file (name, light value, dark value).
   - Track as a follow-up action item.
3. After fixes, re-screenshot in dark mode and compare to reference designs.
4. Iterate until dark mode is visually consistent with approved designs.

### 2.5.6 Cross-Screen Consistency Check

After fixing individual screens:

1. Screenshot all screens in dark mode side by side.
2. Check for inconsistencies:
   - Same component type (e.g., modals) using different background tokens across screens
   - Headers/navigation using different text colors
   - Cards with mismatched surface colors
3. Normalize by applying the reference color map consistently.
4. Present final dark mode screenshots to user for sign-off.

---

## Phase 3: Typography Cleanup

### 3.1 Re-link Detached Text Styles

For each detached text combo with an EXACT library style match:

1. Present the match to the user for confirmation.
2. Apply the style:
   ```js
   node.textStyleId = styleId;
   ```

For close-but-not-exact matches: present options and ask the user to choose.

Execute in batches.

### 3.2 Detach Orphan Typography Variables

Find all TEXT nodes with bound fontSize, lineHeight, letterSpacing, fontWeight, or fontFamily from orphan libraries.

**CRITICAL: Pre-load all required fonts BEFORE processing text nodes.**

First, run an audit pass to enumerate every font family + style combo used in the target text nodes. Then load each one before processing — Figma will throw on any text operation that touches an unloaded font.

```js
// Load fonts one family/style combo at a time, based on what your audit found.
// Replace these examples with the fonts your library/file actually uses.
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
await figma.loadFontAsync({ family: "Inter", style: "Medium" });
await figma.loadFontAsync({ family: "Inter", style: "Bold" });
// Add every other family/style discovered during the audit.
```

**CRITICAL: Cannot use `setBoundVariable(prop, null)` for text properties.** Instead, overwrite with the current explicit value to break the variable binding while preserving the visual appearance:

```js
const len = node.characters.length;
const currentSize = node.fontSize;
node.setRangeFontSize(0, len, currentSize);

const currentLineHeight = node.lineHeight;
node.setRangeLineHeight(0, len, currentLineHeight);

const currentLetterSpacing = node.letterSpacing;
node.setRangeLetterSpacing(0, len, currentLetterSpacing);

const currentFont = node.fontName;
node.setRangeFontName(0, len, currentFont);
```

Process in batches of 100-200 nodes to avoid timeouts.

Report: count of typography bindings detached.

### 3.3 Verify

Re-scan for remaining orphan typography bindings.
Flag any that could not be detached (usually instance-inherited). Note: "These must be fixed in the source library file."

---

## Phase 4: Structural Cleanup

### 4.1 Detach Orphan Spacing and Radius Variables

Find nodes with bound `itemSpacing`, `paddingTop`, `paddingBottom`, `paddingLeft`, `paddingRight`, `topLeftRadius`, `topRightRadius`, `bottomLeftRadius`, `bottomRightRadius` properties from orphan libraries.

Detach by setting the bound variable to null:

```js
node.setBoundVariable('itemSpacing', null);
node.setBoundVariable('paddingTop', null);
// etc.
```

Process in batches. Report count detached.

### 4.2 Detach Orphan Shadow/Effect Variables

Find nodes with bound effects from orphan libraries. Create clean effect objects with the same visual values but no variable bindings:

```js
const cleanEffects = node.effects.map(e => ({
  type: e.type,
  visible: e.visible,
  blendMode: e.blendMode,
  color: { ...e.color },
  offset: { ...e.offset },
  radius: e.radius,
  spread: e.spread
}));
node.effects = cleanEffects;
```

Report: count of effect bindings detached.

### 4.3 Local Variable Cleanup

If Phase 1.5 found local variables that duplicate library tokens:

1. For each duplicate local variable, find all nodes that reference it.
2. Rebind those nodes to the equivalent library token.
3. After all references are remapped, delete the local variable:
   ```js
   const localVar = await figma.variables.getVariableByIdAsync(localVarId);
   localVar.remove();
   ```
4. After removing duplicate variables, check if the entire local collection is now empty. If so, remove the collection too.

Present the list of local variables to delete and get user confirmation first.

### 4.4 Unused Variable/Style Purge

After all cleanup phases, scan for imported library variables that are no longer referenced by any node on any page:

1. Get all imported variables in the file.
2. For each, search all pages for any node that references it (fills, strokes, effects, or other properties).
3. If a variable is imported but has zero references, it's safe to remove.
4. Removing unused imported variables is what actually makes orphan libraries disappear from the Mode picker.

**Note:** This requires scanning ALL pages in the file, not just the target page. An imported variable referenced on ANY page must be kept.

Present the list of unused variables to the user before removing.

### 4.5 Flag Instance-Level Issues & Maintain Cleanup Tracker

Any remaining orphan bindings are likely inherited from library main components. These CANNOT be fixed at the instance level through the plugin API.

List them with:
- Source component name
- Orphan collection name
- Property type
- Count of affected instances

Note clearly: "These must be fixed in the [Library Name] source file."

**IMPORTANT: Maintain a persistent cleanup tracker file** at `~/Downloads/design-system-cleanup.md` (or the user's preferred location). This file tracks work that needs to happen in OTHER files (library components, design system file, other consumer files). Update it throughout the cleanup process — not just at the end.

The tracker should contain:
- **Library-level fixes needed** — grouped by library file, with checkboxes, instance counts, and what the fix should be
- **Potential new semantic tokens** — tokens that don't exist yet but are needed
- **Files cleaned so far** — running log with date, page, and summary of changes
- **Notes** — recurring patterns, decisions made, conventions established

Update the tracker whenever you:
1. Discover an instance-level issue that needs a library fix
2. Find a missing semantic token
3. Complete a cleanup session on any file
4. Learn about a pattern that applies across files

This ensures nothing gets lost between sessions. Check if the tracker already exists before creating a new one — append to the existing file.

---

## Phase 5: Naming & Organization (Optional)

Only execute if the user requested naming cleanup in the discovery interview.

### 5.1 Layer Naming

- Find all nodes with generic names (Frame 1, Group 2, Rectangle 3, Vector, etc.).
- Suggest meaningful names based on content, node type, and parent context.
- Present suggestions to the user for approval BEFORE renaming.
- Use `figma_rename_node` to apply approved names.

### 5.2 Hidden and Invisible Layers

Find:
- Hidden layers (`visible === false`)
- Zero-opacity nodes (`opacity === 0`)
- Off-canvas elements (positioned far outside the frame bounds)

Present the full list to the user. Ask which to delete vs. keep.

Use `figma_delete_node` only for user-confirmed deletions.

---

## Phase 6: Verification & Final Report

### 6.1 Post-Cleanup Snapshot & Visual Diff

1. Take screenshots of the same sections captured in Phase 0.4.
2. Compare "before" and "after" screenshots side by side.
3. Flag any visual differences — these could be:
   - **Expected:** color shifted slightly due to token remapping (e.g., #10b981 → #4caf50)
   - **Unexpected:** something broke — investigate and fix before proceeding
4. Present the visual diff to the user for sign-off: "Here's what changed visually. Everything else looks identical."

### 6.2 Final Orphan Scan

Re-run the full variable audit from Phase 1.2. Compare before/after counts for:
- Orphan variable bindings
- Unbound colors
- Detached text styles
- Unused imported variables
- Local variable duplicates
- Generic layer names (if naming cleanup was done)

### 6.3 Generate Final Report

Write to `[project-root]/figma-cleanup-report.md`. Include:

- **Summary of all changes made** — in plain English
- **Before/after comparison table** — total orphan bindings, unbound colors, detached text styles
- **Remaining issues** with reason for each:
  - "Inherited from library component — must fix in source file"
  - "No matching token in design system"
  - "Ambiguous match — needs designer decision"
- **Recommended follow-ups:**
  - Library-level fixes needed
  - Token gaps to fill in the design system
  - Pages or components that need manual review
- **Total nodes modified** — broken down by change type

Present the final summary to the user.

---

## Critical Technical Notes

These rules apply throughout ALL phases. Violating them will cause errors or data loss.

### Batching
Always process nodes in batches of 100-200. Figma plugin execution times out at ~30 seconds. Structure batch operations like:

```js
const BATCH_SIZE = 150;
for (let i = 0; i < nodes.length; i += BATCH_SIZE) {
  const batch = nodes.slice(i, i + BATCH_SIZE);
  // process batch
}
```

If a single `figma_execute` call processes one batch, make separate calls for each batch.

### Font Loading
Load ALL required fonts BEFORE processing any text nodes. Load one family/style combo at a time. Always run an audit pass first to enumerate every font family + style combination used in the target text, then `loadFontAsync` each one. Failing to pre-load fonts will cause text operations to throw errors.

### Paint Variable Binding
Use `figma.variables.setBoundVariableForPaint(paint, 'color', variable)` to bind a color variable to a paint. Do NOT use `node.setBoundVariable('fills', ...)` for color properties.

### Text Variable Detaching
Cannot use `setBoundVariable(prop, null)` for text properties (fontSize, lineHeight, letterSpacing, fontName). Instead, overwrite with the current explicit value using `setRangeFontSize()`, `setRangeLineHeight()`, `setRangeLetterSpacing()`, `setRangeFontName()`. This breaks the variable binding while preserving the visual value.

### Instance Limitations — CRITICAL
**NEVER modify styles or variable bindings on component instances.** This applies to ALL phases of cleanup:

1. **Inherited bindings** from main components CANNOT be detached at the instance level through the plugin API — they will fail silently or throw errors.
2. **Even when technically possible**, overriding styles on instances creates hidden overrides that make the design harder to maintain and won't be fixed when the source component is updated.
3. **If a component instance has wrong colors, broken dark mode, or orphan variables** — the fix belongs in the **source component** (in the library file), not on the instance.
4. When you encounter a broken instance, **prompt the user**: "This issue is coming from the [Component Name] in the [Library Name] library. I can't fix it here — you'll need to update the source component. Want me to add this to the follow-up list?"
5. Only modify **non-instance nodes** (FRAME, RECTANGLE, TEXT, VECTOR, GROUP, etc.) that are direct children of the page or section — not nodes nested inside instances.

### Importing Library Variables
The correct sequence for importing library variables:
1. `figma.teamLibrary.getAvailableLibraryVariableCollectionsAsync()` — get all available collections
2. `figma.teamLibrary.getVariablesInLibraryCollectionAsync(collectionKey)` — get variables in a collection
3. `figma.variables.importVariableByKeyAsync(variableKey)` — import a specific variable for use

### Confirmation Gates
ALWAYS present findings and proposed changes to the user before executing. Never auto-fix without showing what will change. The user is a designer, not a programmer — explain every change in plain English:
- Instead of "Rebinding 47 fills from collection X to variable Y", say "47 elements are using a color from an old library. I'll switch them to use the same color from your Design System instead. They'll look exactly the same."
- Instead of "Detaching orphan fontSize bindings", say "Some text sizes are linked to a library you don't use anymore. I'll unlink them but keep the same size so nothing changes visually."
