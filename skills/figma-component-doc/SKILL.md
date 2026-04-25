---
name: figma-component-doc
description: "Generate a component documentation block in Figma. Builds a structured doc frame next to the selected component (or component set) containing description, variants with visual+intent descriptions, anatomy of composite parts, properties with default values, colors grouped by role, subcomponents with thumbnails, and a generation footer. Auto-discovers the file's own design tokens (text styles + color variables) — works on any Figma file regardless of design system. Use when user has a component selected in Figma and says 'document this component', 'document this', 'create docs for this', 'document the [name]', 'generate component documentation', 'add doc page for', 'component doc', 'figma component doc', or shares a Figma component URL asking for documentation. Manual-only. Requires the figma-console MCP / Desktop Bridge plugin (writes back into Figma)."
---

# Component Documentation (Generic)

Create (or update) a component documentation block in Figma for the currently selected component. Auto-discovers the file's own design tokens (text styles + color variables) instead of assuming a specific design system. Works on any Figma file with any design system (or none).

**This skill writes nodes back into Figma.** It uses the local `figma-console` MCP, the same one `/figma-autopilot` and `/figma-fix` use.

---

## Available figma-console tools

| Tool | Purpose |
|------|---------|
| `figma_get_status` | Check plugin connection |
| `figma_navigate` | Open a Figma file in Desktop |
| `figma_execute` | Run JS in Figma plugin context — used for discovery and complex node creation |
| `figma_create_child` | Create RECTANGLE / ELLIPSE / FRAME / TEXT / LINE under a parent |
| `figma_set_text` | Change text content / font size on an existing TEXT node |
| `figma_set_fills` | Change fill color on a node |
| `figma_set_strokes` | Change stroke on a node |
| `figma_resize_node` | Resize a node |
| `figma_move_node` | Reposition a node |
| `figma_rename_node` | Rename a layer |
| `figma_capture_screenshot` | Plugin-based screenshot — use after writes (immediate, fresh state) |
| `figma_take_screenshot` | REST-based screenshot — for initial visual context only |

**Use `figma_execute` for the bulk of node creation.** The dedicated tools handle individual property tweaks but cannot set auto-layout, bind variables to fills/text styles, or build nested structures in one shot. The skill needs both — `figma_execute` for the structural work, dedicated tools for any post-creation property fixes.

**Async API:** Always use `figma.getNodeByIdAsync()` and `figma.setCurrentPageAsync()` inside `figma_execute`. The sync versions throw.

---

## Step 0 — Prerequisites and connection check

Before doing anything, verify connections:

**a) Figma Desktop running with debug port:**
```bash
open -a 'Figma' --args --remote-debugging-port=9222
```

**b) Desktop Bridge plugin active in Figma.** Open Figma → Plugins → Desktop Bridge. Plugin must be running.

**c) Optional — VPN off.** figma-console MCP fails connection through some VPN configs.

**d) Run `figma_get_status`** to confirm the bridge is responding. If it errors with `No cloud relay session` or similar, this session is using the cloud-relay version of figma-console rather than the local one. Restart Claude Code to surface the local MCP, or use `figma_pair_plugin` to fall back to the cloud relay (works but less reliable).

**e) FIGMA_ACCESS_TOKEN in `~/.zshrc`** is only needed if you want to read component descriptions via REST API as a fallback. Skill doesn't require it.

If anything fails here, stop and surface the exact error. Don't keep going.

---

## Step 1 — Inspect the selection

Run a `figma_execute` script to read the current selection:

```javascript
const sel = figma.currentPage.selection[0];
if (!sel) return { error: 'No selection. Select a frame containing the component you want to document.' };

const compSet = (sel.type === 'COMPONENT_SET' || sel.type === 'COMPONENT')
  ? sel
  : (sel.findOne(n => n.type === 'COMPONENT_SET') || sel.findOne(n => n.type === 'COMPONENT'));

if (!compSet) {
  return {
    error: 'Selection contains no component or component set.',
    selectionType: sel.type,
    selectionName: sel.name
  };
}

return {
  compSetId: compSet.id,
  type: compSet.type,
  name: compSet.name,
  isSet: compSet.type === 'COMPONENT_SET',
  width: compSet.width,
  height: compSet.height,
  parentId: compSet.parent?.id,
  parentName: compSet.parent?.name,
  parentType: compSet.parent?.type
};
```

If `error` returned, ask the user to select a frame, component, or component set first.

---

## Step 2 — Gather component context

Run a `figma_execute` script to extract everything needed:

```javascript
const compSet = await figma.getNodeByIdAsync('<compSetId from Step 1>');
const isSet = compSet.type === 'COMPONENT_SET';

// 1. Variants
const variants = isSet
  ? compSet.children.map(c => ({ id: c.id, name: c.name, w: c.width, h: c.height, description: c.description || '' }))
  : [{ id: compSet.id, name: compSet.name, w: compSet.width, h: compSet.height, description: compSet.description || '' }];

// 2. Properties — keep options and defaultValue separate
const props = isSet && compSet.componentPropertyDefinitions
  ? Object.entries(compSet.componentPropertyDefinitions).map(([k, v]) => ({
      name: k.replace(/#\w+$/, ''),
      type: v.type,
      options: v.variantOptions || (v.type === 'BOOLEAN' ? [true, false] : null),
      defaultValue: v.defaultValue
    }))
  : [];

// 3. Set-level description (preferred over auto-written)
const setDescription = compSet.description || '';

// 4. Colors — inspect first variant's fills
const firstVariant = isSet ? compSet.children[0] : compSet;
const colorMap = {};
firstVariant.findAll(n => Array.isArray(n.fills) && n.fills.length).forEach(n => {
  n.fills.forEach(f => {
    if (f.type === 'SOLID') {
      const bv = f.boundVariables?.color;
      const hex = '#' + [f.color.r, f.color.g, f.color.b]
        .map(c => Math.round(c * 255).toString(16).padStart(2, '0')).join('').toUpperCase();
      const key = bv ? bv.id : hex;
      if (!colorMap[key]) colorMap[key] = { varId: bv?.id || null, hex, opacity: f.opacity ?? 1 };
    }
  });
});

// 5. Subcomponents — direct INSTANCE children of first variant
const subcomps = [];
firstVariant.findAll(n => n.type === 'INSTANCE').forEach(inst => {
  if (inst.mainComponent) {
    const parentName = inst.mainComponent.parent?.name || inst.mainComponent.name;
    const propStr = Object.entries(inst.componentProperties || {})
      .map(([k, v]) => `${k.replace(/#\w+$/, '')}=${v.value}`).join(', ');
    const entry = parentName + '  →  ' + propStr;
    if (!subcomps.includes(entry)) subcomps.push(entry);
  }
});

// 6. Usages on the current page
const usages = figma.currentPage.findAll(n =>
  n.type === 'INSTANCE' && (
    n.mainComponent?.parent?.id === compSet.id ||
    n.mainComponent?.id === compSet.id
  )
).map(n => ({ id: n.id, parent: n.parent?.name }));

return { name: compSet.name, isSet, setDescription, variants, props, colorMap, subcomps, usages };
```

Use the results to write the description (or use `setDescription` verbatim if present) and populate all section cards.

---

## Step 3 — Auto-discover the file's design tokens

Inspect the file and pick sensible candidates instead of looking up tokens by hardcoded name. Run once per session:

```javascript
// === TEXT STYLES ===
const styles = await figma.getLocalTextStylesAsync();

function rankBoldness(s) {
  const w = (s.fontName?.style || '').toLowerCase();
  if (w.includes('bold') || w.includes('black')) return 3;
  if (w.includes('semibold') || w.includes('medium')) return 2;
  return 1;
}

const sorted = [...styles].sort((a, b) => b.fontSize - a.fontSize);
const asc = [...styles].sort((a, b) => a.fontSize - b.fontSize);

const ST = {
  // TITLE: readable component-doc heading, NOT hero-size. Prefer 24-40px bold/medium.
  // Falls back through wider bands, then largest bold, then largest of any kind.
  TITLE: (
    sorted.find(s => s.fontSize >= 24 && s.fontSize <= 40 && rankBoldness(s) >= 2) ||
    sorted.find(s => s.fontSize >= 20 && s.fontSize <= 48 && rankBoldness(s) >= 2) ||
    sorted.find(s => rankBoldness(s) >= 2) ||
    sorted[0]
  )?.id,
  BODY: (sorted.find(s => s.fontSize >= 12 && s.fontSize <= 16 && rankBoldness(s) === 1)
         || sorted[Math.floor(sorted.length / 2)])?.id,
  BODY_BOLD: (sorted.find(s => s.fontSize >= 12 && s.fontSize <= 16 && rankBoldness(s) >= 3)
              || sorted.find(s => rankBoldness(s) >= 3))?.id,
  BODY_SB: (sorted.find(s => s.fontSize >= 12 && s.fontSize <= 16 && rankBoldness(s) === 2)
            || sorted.find(s => rankBoldness(s) === 2))?.id,
  // HEADING: card section eyebrow. Prefer 11-14px bold; fall back to smallest bold.
  HEADING: (
    asc.find(s => s.fontSize >= 11 && s.fontSize <= 14 && rankBoldness(s) >= 2) ||
    asc.find(s => rankBoldness(s) >= 2) ||
    asc[0]
  )?.id
};

// === COLOR VARIABLES ===
const vars = await figma.variables.getLocalVariablesAsync();
const colorVars = vars.filter(v => v.resolvedType === 'COLOR');

async function resolveColor(v) {
  const collection = await figma.variables.getVariableCollectionByIdAsync(v.variableCollectionId);
  const modeId = collection.defaultModeId;
  let val = v.valuesByMode[modeId];
  let depth = 0;
  while (val?.type === 'VARIABLE_ALIAS' && depth < 5) {
    const ref = await figma.variables.getVariableByIdAsync(val.id);
    if (!ref) break;
    val = ref.valuesByMode[Object.keys(ref.valuesByMode)[0]];
    depth++;
  }
  if (val && typeof val === 'object' && 'r' in val) {
    const lightness = (val.r + val.g + val.b) / 3;
    return { id: v.id, name: v.name, color: val, lightness };
  }
  return null;
}

const resolved = (await Promise.all(colorVars.map(resolveColor))).filter(Boolean);

function findByPattern(...patterns) {
  for (const p of patterns) {
    const match = resolved.find(r => p.test(r.name.toLowerCase()));
    if (match) return match;
  }
  return null;
}

const byLightness = [...resolved].sort((a, b) => a.lightness - b.lightness);

// Match loose forms: `text-primary`, `text/primary`, `Text/text-primary`,
// `colors/foreground/default`, `fg/primary`, `content/secondary`, etc.
// Try semantic patterns FIRST (before lightness fallback) — most files have
// a Text/* or Foreground/* semantic palette that should be preferred over
// raw numeric scales like `gray/700`.
const VAR = {
  PRIMARY: (findByPattern(
    /(^|\/)text[-/]?(primary|default|foreground)\b/,
    /(^|\/)foreground[-/]?(primary|default)?\b/,
    /(^|\/)fg[-/]?(primary|default)?\b/,
    /(^|\/)content[-/]?(primary|default)\b/,
    /(^|\/)(base\/)?black\b/,
    /neutral.*(900|950|1000)\b/, /gray.*(900|950)\b/, /grey.*(900|950)\b/,
    /\bdark\b/
  ) || byLightness[0])?.id,
  SECONDARY: (findByPattern(
    /(^|\/)text[-/]?secondary\b/,
    /(^|\/)text[-/]?(subtle|sub)\b/,
    /(^|\/)foreground[-/]?secondary\b/,
    /(^|\/)fg[-/]?secondary\b/,
    /(^|\/)content[-/]?secondary\b/,
    /neutral.*(500|600)\b/, /gray.*(500|600)\b/, /grey.*(500|600)\b/
  ) || byLightness[Math.floor(byLightness.length * 0.5)])?.id,
  MUTED: (findByPattern(
    /(^|\/)text[-/]?(tertiary|muted|placeholder|disabled)\b/,
    /(^|\/)foreground[-/]?(tertiary|muted)\b/,
    /(^|\/)fg[-/]?(tertiary|muted)\b/,
    /(^|\/)content[-/]?(tertiary|muted)\b/,
    /neutral.*400\b/, /gray.*400\b/, /grey.*400\b/,
    /\bmuted\b/
  ) || byLightness[Math.floor(byLightness.length * 0.65)])?.id
};

return {
  ST, VAR,
  hasStyles: styles.length > 0,
  hasVars: colorVars.length > 0
};
```

If `hasStyles` is `false`, the text helper falls back to raw font properties. If `hasVars` is `false`, it uses raw `#000000` / `#666666` / `#999999` hex values.

**Print the resolved `ST` and `VAR` map to the user before building.** Lets them sanity-check the picks and override before any nodes are created.

---

## Step 4 — Duplicate detection

Before creating anything, search for an existing doc frame:

```javascript
const compSet = await figma.getNodeByIdAsync('<compSetId>');
const docName = `${compSet.name}.docs`;
const existing = figma.currentPage.findOne(n => n.name === docName);
return { existingId: existing?.id || null };
```

If an existing doc frame is found, ask the user: *"Doc frame already exists for this component — rebuild it?"* If yes, clear its description and data children before repopulating. Never duplicate.

---

## Frame structure

```
[ComponentName].docs              ← VERTICAL, HUG×HUG, 58px gap, 80px padding, WHITE fill, 24 radius
├── description                   ← VERTICAL, FIXED 940px × HUG, 16px gap
│   ├── [component name]          ← TITLE style, PRIMARY color, FILL — compact (24-40px), not hero-size
│   ├── [subtitle]                ← BODY style, MUTED color, FILL — "Component Set · N variants"
│   └── [description]             ← BODY style, SECONDARY color, FILL — ≤2 sentences, purpose only
├── data                          ← VERTICAL, FIXED 940px × HUG, 16px gap
│   ├── [VARIANTS card]
│   ├── [ANATOMY card]             ← omit for primitives
│   ├── [PROPERTIES card]
│   ├── [COLORS card]              ← omit if no fills found
│   └── [SUBCOMPONENTS card]       ← omit if no subcomponents
└── footer                         ← VERTICAL, FILL×HUG, 4px gap
    ├── [generated date]           ← BODY, MUTED — "Generated YYYY-MM-DD via /component-documentation"
    └── [component last edited]    ← BODY, MUTED — only if available
```

Position the doc frame **to the right of the original component set** (e.g. `originalX + originalWidth + 200`, same Y). The original component set stays untouched in place.

**No clone of the component set inside the doc.** Earlier versions cloned the set into a parallel column, but it duplicated the VARIANTS card and left a tall empty stripe of whitespace beside it. The VARIANTS card already shows live instances of every variant — that's the source of truth.

**White wrap fill, 80px padding, 24px radius.** Makes the doc readable on any page background (some files have dark canvases that would hide black text), and gives the doc a self-contained surface.

---

## Card specs

All cards share:
- VERTICAL auto-layout
- 24px padding all sides
- Fill `#FFFFFF`, stroke 1px `#DEDEDE` INSIDE, cornerRadius 16
- Width FILL container, Height HUG

| Card          | itemSpacing |
|---------------|-------------|
| VARIANTS      | 20px        |
| PROPERTIES    | 12px        |
| COLORS        | 12px        |
| SUBCOMPONENTS | 12px        |

> Card chrome (white fill, gray stroke, 16px radius) is fixed neutral so docs read consistent across files. Card **content** uses the file's own tokens.

---

## VARIANTS card

**Title:** HEADING style, MUTED color, layer `"title"` — `"VARIANTS · N <propname>s"` where N is the variant count and `<propname>s` is the pluralized variant property name (e.g., `"VARIANTS · 5 types"`, `"VARIANTS · 3 sizes"`). Center-dot, never em-dash. Avoids redundancy with the per-variant labels below ("type = info", "size = sm" etc.).

**One `variant` frame per variant** (layer named `"variant"`):
- HORIZONTAL or VERTICAL auto-layout (see layout rule below)
- Contains: live component instance + `description` sub-frame

**`description` sub-frame** (VERTICAL, 8px gap, FILL×HUG):
- `state` text — BODY_BOLD style, PRIMARY color, FILL — variant label e.g. `"type = single"`
- `description` text — BODY style, SECONDARY color, FILL — **two beats: visual change first, intent second**. Prefer `variant.description` from Figma if non-empty; auto-write only if empty.
  - Visual beat: name the color/glyph difference inferred from the variant's primary fills ("Red glyph, soft-red surface."). The skill should inspect each variant's distinguishing fill (typically the leading icon) and translate hex to a color name.
  - Intent beat: when to use ("Communicates a blocking failure.")
  - No filler adjectives ("calm", "neutral", "approachable", "warm" etc.)
  - Good: `"Red glyph, soft-red surface. Communicates a blocking failure."`
  - Bad: `"Communicates a blocking failure."` (no visual cue)
  - Bad: `"Brings warmth and approachability with a soft-red surface."` (filler adjectives)
- `usage` text — BODY_SB style, PRIMARY color, FILL — **only included if real usages were found in Step 2's `usages` array**. Format: `"Used in: parentA, parentB"`. If no real usages, omit the line entirely. Do NOT invent placeholder usages.

**Between variant rows:** insert a 1px hairline divider (a Rectangle with FILL×1 height, fill `#EAEAEA`). With 4+ variants, the lack of separators makes the card feel like one blob.

**Layout rule:**
- If `card_inner_width - instance_width - 24 ≥ 280` → **HORIZONTAL** row (instance left, description right, 24px gap, instance pinned, description FILL).
- Else if `instance_width ≤ 64` → HORIZONTAL row (14px gap, tight).
- Else → VERTICAL stack (instance above description, 12px gap).

The HORIZONTAL primary case is what most components hit once the data column is wide enough (940px). It cuts doc length roughly in half compared to vertical stacking.

---

## ANATOMY card (composite components only)

Skip this card for **primitive** components (no nested instances) and for components whose decorated variant has fewer than 2 named child elements.

**Heading:** HEADING style, MUTED color, layer `"heading"` — `"ANATOMY"`

**Layout** (HORIZONTAL, 32px gap, FILL×HUG, counterAxisAlignItems CENTER):
- A live instance of the variant with the most parts visible — typically the default variant with all booleans set to `true`. Pinned width.
- `parts` frame (VERTICAL, 12px gap, FILL×HUG):
  - One `part` row per direct named child of the variant:
    - HORIZONTAL, 12px gap, counterAxisAlignItems CENTER
    - `index` text — BODY_BOLD, MUTED — sequential number (`1.`, `2.`, `3.`…)
    - `info` frame (VERTICAL, 2px gap, FILL×HUG):
      - `label` text — BODY_BOLD, PRIMARY — part name from the layer name (humanize camelCase / kebab-case to readable form)
      - `description` text — BODY, SECONDARY — one short clause about what the part does

**Heuristic for part labels:**
- Prefer descriptive layer names (`"Title"`, `"Body"`, `"CTA"`, `"Close"`).
- For unnamed `"Frame N"` layers, infer from contents: TEXT child with largest font → "Title"; remaining TEXT → "Body"; INSTANCE → use main component name.
- Skip pure decorative frames (no text, no instance, no fill).

The anatomy card teaches readers the structure of a composite component faster than any text description.

---

## PROPERTIES card

**Heading:** HEADING style, MUTED color, layer `"heading"` — `"PROPERTIES"`

**One `property` frame per property** (VERTICAL, 4px gap, FILL×HUG, layer name = property name):
- Line 1 — `name` text, BODY_BOLD style, PRIMARY color
- Line 2 — `meta` text, BODY style, MUTED color — see format below (always include the default)
- Line 3 — `description` text, BODY style, SECONDARY color — what the property does, one short clause

**Meta line format (always surface the default — design system users need to know what they get without overrides):**
- VARIANT: `"VARIANT · default: <defaultValue> · option1, option2, option3"`
- BOOLEAN: `"BOOLEAN · default: <true|false>"`
- TEXT: `"TEXT · default: \"<defaultValue>\""`
- INSTANCE_SWAP: `"INSTANCE · default: <componentName>"`

**Between properties:** insert a 1px hairline divider (a Rectangle with FILL×1 height, fill `#EAEAEA`). Stops 4+ properties from reading as one blob.

No em-dashes anywhere.

---

## COLORS card

**Heading:** HEADING style, MUTED color, layer `"heading"` — `"COLORS"`

**Group colors by their variable prefix folder.** Each unique top-level folder (e.g., `Text/*`, `Icons/*`, `CTA/*`, `Surface/*`) becomes a labeled section inside the card. Same-hex values across different roles (e.g., `Text/text-primary` and `Icons/icon-primary` both `#000000`) read as separate, intentional uses instead of looking like duplicates.

**Group structure** (VERTICAL, 8px gap):
- `group-label` text — HEADING style, MUTED color — uppercase folder name without trailing slash (e.g., `"TEXT"`, `"ICONS"`, `"CTA"`)
- One `color` row per fill in the group (HORIZONTAL, 12px gap, FILL×HUG, counterAxisAlignItems CENTER):
  - `Rectangle` 24×24, cornerRadius 6, 1px `#DEDEDE` stroke, fill = actual color
  - `info` frame (VERTICAL, 2px gap, HUG×HUG):
    - `value` text — BODY_SB style, PRIMARY color — `"variableName · #HEX"` if variable-bound, else `"#HEX"`
    - `description` text — BODY style, SECONDARY color — what this color is used for

**Group spacing:** 16px between groups inside the card.

**Unbound hex fills** (no variable): place in a `"SURFACE"` group if the color is light (lightness > 0.9), or `"OTHER"` otherwise.

Omit this card entirely if no fills are found.

---

## SUBCOMPONENTS card

**Heading:** HEADING style, MUTED color, layer `"heading"` — `"SUBCOMPONENTS"`

**One `subcomponent` row per subcomponent** (HORIZONTAL, 16px gap, FILL×HUG, counterAxisAlignItems CENTER):
- Live instance of the subcomponent at its used size (cap at 32×32 if larger to keep rows aligned)
- `info` frame (VERTICAL, 2px gap, FILL×HUG):
  - `name` text — BODY_BOLD style, PRIMARY color — component name (e.g., `"UI Icon"`)
  - `meta` text — BODY style, MUTED color — `"prop = value, prop = value"` for the props in use

The thumbnail makes the reference instant — no mental translation from a prop string to a visual.

**Between rows:** 1px hairline divider (Rectangle, FILL×1, fill `#EAEAEA`).

Omit this card entirely if no subcomponent instances are found.

---

## FOOTER

A small metadata strip at the bottom of the wrap. Tells readers when the doc was generated so they can gauge freshness.

**Layout** (VERTICAL, 4px gap, FILL×HUG, separated from the data column by the wrap's 58px gap):
- `generated` text — BODY style, MUTED color — `"Generated YYYY-MM-DD via /component-documentation"`
- `last-edited` text — BODY style, MUTED color — `"Component last edited YYYY-MM-DD"` — only include if `compSet.lastModified` or equivalent is available, otherwise omit this line.

---

## Step 5 — Build incrementally

Per figma-pilot rules: **at most ~10 logical operations per `figma_execute` call.** Split building across multiple calls:

1. **Skeleton call** — create outer wrapper (white fill, 80 padding, 24 radius) + description frame (940 wide) + data frame (940 wide) + footer frame as empty placeholders. **No documentation horizontal column. No clone.** Set `placeholder = true` on each. Return all created IDs.
2. **Description call** — populate name + subtitle + description text. Description body must be ≤2 sentences focused on purpose — no variant lists, no property lists, no marketing voice.
3. **VARIANTS card call(s)** — build the card heading + variant entries with hairlines between rows. Use visual+intent variant descriptions (color/glyph cue first, semantic intent second). Wire in real usages from Step 2 — omit the "Used in" line if none exist.
4. **ANATOMY card call** — build the card (skip if primitive). Pull direct named children of the most-decorated variant.
5. **PROPERTIES card call** — one `property` block per property (3 stacked text rows each, defaults always shown), with hairline dividers between rows.
6. **COLORS card call** — build grouped by variable folder (skip if empty).
7. **SUBCOMPONENTS card call** — build with thumbnails (skip if empty).
8. **Footer call** — populate generation date + (optionally) component last edited.
9. **Final call** — set sizing modes (`layoutSizingHorizontal = 'FILL'` after parenting), reset `placeholder = false` everywhere, take a screenshot.

**Always return created node IDs from every call** so subsequent calls can reference them.

---

## Reusable helper code

Adaptive helpers — fall back to raw font/color values when the file has no design system. Inline these inside each `figma_execute` call that creates nodes:

```javascript
async function txt(chars, styleId, varId, fallbackHex = '#000000', fallbackSize = 12) {
  const t = figma.createText();

  if (styleId) {
    const style = await figma.getStyleByIdAsync(styleId);
    if (style?.fontName) await figma.loadFontAsync(style.fontName);
    else await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });
  } else {
    await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });
  }

  t.characters = chars;

  if (styleId) await t.setTextStyleIdAsync(styleId);
  else t.fontSize = fallbackSize;

  if (varId) {
    const v = await figma.variables.getVariableByIdAsync(varId);
    if (v) {
      t.fills = [figma.variables.setBoundVariableForPaint(
        { type: 'SOLID', color: { r: 0, g: 0, b: 0 } }, 'color', v
      )];
    }
  } else {
    const r = parseInt(fallbackHex.slice(1, 3), 16) / 255;
    const g = parseInt(fallbackHex.slice(3, 5), 16) / 255;
    const b = parseInt(fallbackHex.slice(5, 7), 16) / 255;
    t.fills = [{ type: 'SOLID', color: { r, g, b } }];
  }

  t.name = chars;
  return t;
}

function wrapFrame(name) {
  const f = figma.createFrame();
  f.name = name;
  f.layoutMode = 'VERTICAL';
  f.primaryAxisSizingMode = 'AUTO';
  f.counterAxisSizingMode = 'AUTO';
  f.itemSpacing = 58;
  f.paddingTop = f.paddingBottom = f.paddingLeft = f.paddingRight = 80;
  f.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];  // white surface, readable on any page
  f.cornerRadius = 24;
  return f;
}

function card(name, spacing) {
  const f = figma.createFrame();
  f.name = name;
  f.layoutMode = 'VERTICAL';
  f.primaryAxisSizingMode = 'AUTO';
  f.counterAxisSizingMode = 'AUTO';
  f.itemSpacing = spacing;
  f.paddingTop = f.paddingBottom = f.paddingLeft = f.paddingRight = 24;
  f.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
  f.strokeWeight = 1;
  f.strokes = [{ type: 'SOLID', color: { r: 0.87, g: 0.87, b: 0.87 } }];
  f.strokeAlign = 'INSIDE';
  f.cornerRadius = 16;
  return f;
}

function dataFrame(width = 940) {
  const f = figma.createFrame();
  f.name = 'data';
  f.fills = [];
  f.layoutMode = 'VERTICAL';
  f.counterAxisSizingMode = 'FIXED';
  f.resize(width, 10);
  f.primaryAxisSizingMode = 'AUTO';  // MUST be set AFTER resize() — resize() overrides it
  f.itemSpacing = 16;
  return f;
}
```

**Critical gotchas (from figma-pilot rules):**
- Always `await` async APIs — `getNodeByIdAsync`, `setCurrentPageAsync`, `loadFontAsync`, `setTextStyleIdAsync`
- `t.textStyleId = id` is deprecated — use `await t.setTextStyleIdAsync(id)`
- Load font BEFORE setting `t.characters`, even when applying a style afterwards
- `resize()` overrides `primaryAxisSizingMode` — set `'AUTO'` AFTER calling `resize()`
- `layoutSizingHorizontal/Vertical = 'FILL'` only works AFTER `parent.appendChild(child)`
- `figma.currentPage = page` throws — use `await figma.setCurrentPageAsync(page)`
- Colors are 0–1 range, not 0–255
- Position new top-level nodes away from (0,0) — page-level placement defaults to origin and overlaps existing content
- Failed `figma_execute` scripts are atomic — no partial state. Read the error, fix the script, retry.
- `console.log()` is not returned — use `return` for output

---

## Writing good descriptions

If `setDescription` from Step 2 is non-empty, use it verbatim and skip auto-writing.

If empty, auto-write following these rules:
- **≤2 sentences. Hard cap.** First sentence = purpose. Second sentence = primitive/composite note (only if it adds info).
- Don't enumerate variants ("informational, error, warning, success" etc.) — already shown in subtitle and VARIANTS card.
- Don't list properties ("Toggleable Title, Left Icon, CTA…") — already shown in PROPERTIES card.
- No marketing voice ("communicate context to the user", "scale from a quiet inline note to a full action-bearing alert"). Plain, direct.
- Present tense, no filler adjectives.

**Subtitle format:** `"Component Set · N variants"` for sets, `"Component · 1 variant"` for single components. Center-dot separator, never em-dash.

**Variant descriptions** (per entry in VARIANTS card):
- One short sentence stating what the variant does or when to use it. That's it.
- No filler adjectives like "calm", "neutral", "warm", "approachable" — they read as AI-padded.
- Good: "Communicates a blocking failure."
- Bad: "Communicates a blocking failure or destructive condition the user must resolve before continuing in a clear, urgent tone."

---

## When to skip a section

- No fills found → skip COLORS card
- No nested instances → skip SUBCOMPONENTS card
- No `componentPropertyDefinitions` (single component) → skip VARIANTS card, document as single-state
- Always include PROPERTIES card if the component has properties

---

## Post-creation checklist

1. `figma_capture_screenshot` on the doc frame (plugin-based, immediate — captures fresh state)
2. Verify all cards visible, no clipped text
3. Verify the `data` frame height is HUG (not stuck at 10px — sign that `primaryAxisSizingMode` wasn't reset after `resize()`)
4. Verify the wrap has a white fill (not transparent) — black text on a transparent wrap renders invisible on dark-canvas pages
5. Verify variant rows used HORIZONTAL layout where the inner card width allowed it (only fall back to VERTICAL stack for components too wide to sit beside their description)
6. Verify auto-written description is ≤2 sentences and doesn't enumerate variants/properties
7. Verify "Used in: …" lines only appear where Step 2 found real usages (no invented placeholders)
8. If text shows wrong fonts/colors, the Step 3 token discovery picked wrong candidates — print the resolved `ST` and `VAR` maps and let the user override specific tokens by name before re-running

---

## Token discovery override

If auto-discovery picks wrong tokens (e.g., file has both a marketing and a product type system, picked the wrong one), let the user override by name:

```javascript
const styles = await figma.getLocalTextStylesAsync();
const ST = {
  TITLE: styles.find(s => s.name === 'text/lg/bold')?.id,
  BODY: styles.find(s => s.name === 'text/sm/regular')?.id,
  // ...
};
```

Always print the resolved token map before building so the user can sanity-check it.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `figma_get_status` errors | Figma desktop not running with debug port, or Desktop Bridge plugin not active | Run `open -a 'Figma' --args --remote-debugging-port=9222`, then open Plugins → Desktop Bridge |
| `No cloud relay session` error | Session is using cloud-relay figma-console instead of local | Restart Claude Code to surface the local MCP, or run `figma_pair_plugin` to use cloud relay |
| `Cannot read properties of null` for `getNodeByIdAsync` | Wrong page, or node was deleted | Verify page context with `figma.setCurrentPageAsync`, re-fetch ID from a stable parent |
| Text shows in wrong font | `loadFontAsync` not called before `t.characters = ...` | Always load font first |
| Frame stays at 10px height | `primaryAxisSizingMode` set before `resize()` | Set `'AUTO'` AFTER resize() |
| Doc panels appear collapsed | `layoutSizingHorizontal = 'FILL'` set before `appendChild` | Append first, then set FILL |
| `figma.currentPage = page` throws | Sync setter not supported | Use `await figma.setCurrentPageAsync(page)` |
