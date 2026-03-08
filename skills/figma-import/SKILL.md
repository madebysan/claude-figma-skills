---
name: figma-import
description: "Import design tokens from any codebase into Figma as variables. Scans a project for colors, typography, spacing, corner radius, shadows, and opacity tokens — then creates Figma variable collections via figma-console MCP. Works with CSS, Tailwind, shadcn/ui, SwiftUI, Kotlin/Compose, Sass, Less, and design token JSON files. Use when user says 'import tokens to Figma', 'sync tokens to Figma', 'create Figma variables from code', 'push tokens to Figma', or 'figma import'."
---

# Figma Import — Code Tokens to Figma Variables

You import design tokens from a codebase into Figma as organized variable collections. You support any project type — web, iOS, Android, cross-platform.

## Prerequisites

Before starting, verify:
1. **figma-console MCP is connected** — run `figma_get_status` to check. If it fails, tell the user: "The Figma desktop plugin bridge needs to be running (localhost:9222). Open Figma desktop and start the Console MCP plugin."
2. **A Figma file is open** — the variables will be created in the currently active file.
3. **VPN is off** — if connections fail, ask the user if their VPN is on (common cause).

## Workflow

### Phase 1: Detect & Scan

1. **Detect the project stack** by checking files in the working directory:

| Signal Files | Stack | Token Sources |
|---|---|---|
| `Package.swift`, `*.xcodeproj` | SwiftUI/iOS | Color extensions, Asset.xcassets, Font definitions, hardcoded padding/radius values |
| `build.gradle.kts`, `*.kt` | Kotlin/Compose | `Color()` definitions, `MaterialTheme` values, `dp` values, `sp` values |
| `tailwind.config.*` | Tailwind v3 | `theme.extend` in config file |
| `@theme` in CSS files | Tailwind v4 | `@theme` block in CSS |
| `globals.css` with `--` vars | shadcn/ui | CSS custom properties in globals.css |
| `*.css` with `:root` | Plain CSS | CSS custom properties |
| `*.scss` with `$` vars | Sass | Sass variables |
| `*.less` with `@` vars | Less | Less variables |
| `tokens.json`, `*.tokens.json` | Design Token JSON | W3C DTCG or Style Dictionary format |
| `style-dictionary.config.*` | Style Dictionary | Token files referenced in config |

2. **Scan for tokens** across the detected stack. Search for ALL of these categories:

#### Colors
- **CSS**: `--color-*`, `--*-color`, any `--*` with hex/rgb/hsl values
- **Tailwind v3**: `theme.extend.colors` in config
- **Tailwind v4**: `--color-*` in `@theme` block
- **SwiftUI**: `Color(red:green:blue:)`, `Color("name")`, `Color.*` extensions, `.xcassets` color sets
- **Kotlin**: `Color(0xFF...)`, `Color(red, green, blue)`, MaterialTheme color definitions
- **Sass**: `$color-*`, `$*-color`, any `$` variable with color values
- **Less**: `@color-*`, `@*-color`
- **JSON**: `"type": "color"` or `"$type": "color"`

#### Typography
- **CSS**: `--font-size-*`, `--font-weight-*`, `--font-family-*`, `--line-height-*`
- **Tailwind**: `theme.extend.fontSize`, `theme.extend.fontFamily`
- **SwiftUI**: `.system(size:)`, `.font(.*Font)` definitions, custom Font extensions
- **Kotlin**: `*.sp` values, `FontWeight.*`, `FontFamily.*`
- **Sass/Less**: `$font-*`, `@font-*`
- **JSON**: `"type": "fontFamily"`, `"type": "fontSize"`, `"type": "fontWeight"`

#### Spacing
- **CSS**: `--spacing-*`, `--gap-*`, `--padding-*`, `--margin-*`
- **Tailwind**: `theme.extend.spacing`
- **SwiftUI**: `padding(*)`, `spacing:` values in VStack/HStack, `.frame(width:height:)` patterns — extract the unique set of values used
- **Kotlin**: `*.dp` values for padding, arrangement spacing
- **Sass/Less**: `$spacing-*`, `@spacing-*`
- **JSON**: `"type": "spacing"` or `"type": "dimension"`

#### Corner Radius
- **CSS**: `--radius-*`, `--border-radius-*`
- **Tailwind**: `theme.extend.borderRadius`
- **SwiftUI**: `cornerRadius:` values, `RoundedRectangle(cornerRadius:)`
- **Kotlin**: `RoundedCornerShape(*.dp)`, `shape.*` in MaterialTheme
- **Sass/Less**: `$radius-*`, `@radius-*`
- **JSON**: `"type": "borderRadius"`

#### Shadows
- **CSS**: `--shadow-*`, `box-shadow` values
- **Tailwind**: `theme.extend.boxShadow`
- **SwiftUI**: `.shadow(color:radius:x:y:)` definitions
- **Kotlin**: `Elevation.*`, shadow modifiers
- **JSON**: `"type": "shadow"`

#### Opacity
- **CSS**: `--opacity-*`
- **SwiftUI**: `.opacity(*)` values used consistently
- **Kotlin**: `alpha` values
- **JSON**: `"type": "opacity"`

3. **Normalize all found tokens** into a standard format:

```
{
  name: "colors/primary",          // category/name format
  resolvedType: "COLOR",           // COLOR | FLOAT | STRING
  value: "#4085F2",                // normalized value
  lightValue: "#4085F2",           // optional: light mode value
  darkValue: "#1A1A2E",            // optional: dark mode value
  description: "Primary accent",   // source context
  source: "Assets.xcassets"        // where it was found
}
```

**Normalization rules:**
- Convert all colors to hex format (`#RRGGBB` or `#RRGGBBAA`)
- Convert SwiftUI `Color(red: 0.25, green: 0.52, blue: 0.95)` to hex
- Convert Kotlin `Color(0xFF4085F2)` to hex
- Convert `rgb()`, `hsl()`, `oklch()` to hex
- Convert rem/em to px (assume 16px base)
- Convert SwiftUI points and Kotlin dp to px (1:1 mapping)
- Round float values to 2 decimal places
- Use forward slashes for grouping: `colors/primary`, `spacing/sm`, `radius/md`

### Phase 2: Present & Confirm

4. **Present the discovered tokens** to the user in a clear table, grouped by category:

```
Found 32 design tokens in [project name]:

COLORS (14)
| Token Name | Value | Source |
|---|---|---|
| colors/primary | #4085F2 | AccentColor.colorset |
| colors/today | #F2C440 | Enums.swift |
| ... | ... | ... |

SPACING (10)
| Token Name | Value | Source |
|---|---|---|
| spacing/xs | 4 | TaskRowView.swift |
| spacing/sm | 8 | KanbanColumnView.swift |
| ... | ... | ... |

[etc. for each category]
```

5. **Ask the user to confirm** before pushing to Figma:
   - "These 32 tokens will be created as Figma variables. Want me to proceed?"
   - "Should I organize them as one collection ('Design Tokens') or separate collections per category ('Colors', 'Spacing', etc.)?"
   - "Does this project have light/dark modes? If so, I'll create mode variants."

### Phase 3: Push to Figma

6. **Check for existing variables** — run `figma_get_variables` first. If collections with the same names exist, warn the user: "A 'Colors' collection already exists. Should I skip it, merge into it, or create a new one?"

7. **Create variables in Figma** using the batch tools:

**For a fresh setup (no existing collections):**
Use `figma_setup_design_tokens` for each collection (up to 100 tokens per call):

```
figma_setup_design_tokens({
  collectionName: "Colors",
  modes: ["Default"],           // or ["Light", "Dark"] if modes detected
  tokens: [
    { name: "colors/primary", resolvedType: "COLOR", values: { "Default": "#4085F2" } },
    { name: "colors/today", resolvedType: "COLOR", values: { "Default": "#F2C440" } },
    ...
  ]
})
```

**For adding to existing collections:**
Use `figma_batch_create_variables`:

```
figma_batch_create_variables({
  collectionId: "existing-collection-id",
  variables: [
    { name: "colors/primary", resolvedType: "COLOR", valuesByMode: { "mode-id": "#4085F2" } },
    ...
  ]
})
```

**Organize by collection:**

| Collection | Token Types | Variable Type |
|---|---|---|
| Colors | Color values | COLOR |
| Spacing | Padding, margin, gap values | FLOAT |
| Typography | Font sizes, line heights | FLOAT |
| Typography (families) | Font family names | STRING |
| Corner Radius | Border radius values | FLOAT |
| Shadows | Shadow values | STRING (Figma can't natively store shadow composites as variables) |
| Opacity | Opacity values | FLOAT |

8. **If more than 100 tokens in a category**, split into multiple batch calls of 100 each.

9. **Take a screenshot** after creating variables to verify they appear correctly:
   - Run `figma_get_variables` to confirm creation
   - Report the results: "Created 32 variables across 4 collections: Colors (14), Spacing (10), Typography (5), Corner Radius (3)"

### Phase 4: Summary

10. **Report what was created:**
```
Import complete:
- Colors: 14 variables in "Colors" collection
- Spacing: 10 variables in "Spacing" collection
- Typography: 5 variables in "Typography" collection
- Corner Radius: 3 variables in "Corner Radius" collection
Total: 32 variables created

Source: Tasks iOS (/Users/san/Projects/Tasks/ios/)
Target: [Figma file name]
```

11. **Suggest next steps:**
    - "Want me to generate a visual documentation page in Figma showing all the tokens?" (Phase 5)
    - "Run `/figma-extract` to verify the variables by exporting them back"
    - "You can now bind these variables to your Figma components"

### Phase 5: Generate Documentation (Optional)

If the user wants a visual reference page in Figma:

12. **Create a "Design Tokens" page** in the current file using `figma_execute`:
    - Check if the page already exists to avoid duplicates
    - Use `await figma.setCurrentPageAsync(page)` (not `figma.currentPage = page`)
    - Use `await figma.getNodeByIdAsync(id)` (not `figma.getNodeById(id)`)

13. **Look up variable IDs** before building visuals. Run `figma_get_variables` to get the variable IDs for each collection. You need these to bind fills and values to the actual variables (not hardcoded hex).

14. **Build sections** inside a main frame with auto-layout (vertical, 48px gap, 60px padding):

    - **Header** — Project name + "Design Tokens", token count, collection count
    - **Colors** — Circle swatches (48×48) in a wrap grid with name, hex value, and usage context below each
    - **Spacing** — Horizontal bars at proportional widths with label (name), value (Xpt), and visual bar
    - **Corner Radius** — Rounded rectangles at increasing sizes showing each radius value
    - **Opacity** — Squares with black fill at each opacity level, labeled with name and percentage
    - **Typography** — Text samples at each font size with weight name and usage context

15. **Bind variables to every visual element (critical).** The doc page is a living reference — every visual must use the actual Figma variable, not a hardcoded value. When a variable changes, the doc updates automatically.

    **Principle:** If a variable exists for a value, bind it. No hardcoded hex, px, or pt values on elements that represent tokens.

    **COLOR variables → swatch fills:**
    ```js
    const swatch = figma.createEllipse();
    swatch.resize(48, 48);
    swatch.fills = [figma.util.solidPaint('#000000')]; // placeholder

    const variable = await figma.variables.getVariableByIdAsync(variableId);
    const fills = [...swatch.fills];
    fills[0] = figma.variables.setBoundVariableForPaint(fills[0], 'color', variable);
    swatch.fills = fills;
    ```
    `setBoundVariableForPaint` returns a new paint — you must reassign to `node.fills`.

    **FLOAT variables → spacing bars (width):**
    ```js
    const variable = await figma.variables.getVariableByIdAsync(variableId);
    bar.setBoundVariable('width', variable);
    ```

    **FLOAT variables → corner radius:**
    ```js
    const variable = await figma.variables.getVariableByIdAsync(variableId);
    // Bind all four corners for uniform radius
    rect.setBoundVariable('topLeftRadius', variable);
    rect.setBoundVariable('topRightRadius', variable);
    rect.setBoundVariable('bottomLeftRadius', variable);
    rect.setBoundVariable('bottomRightRadius', variable);
    ```

    **FLOAT variables → opacity:**
    ```js
    const variable = await figma.variables.getVariableByIdAsync(variableId);
    rect.setBoundVariable('opacity', variable);
    ```

    **FLOAT variables → typography (font size):**
    ```js
    const variable = await figma.variables.getVariableByIdAsync(variableId);
    textNode.setBoundVariable('fontSize', variable);
    ```

    **FLOAT variables → auto-layout gaps and padding:**
    ```js
    // If spacing tokens are used for section padding or item gaps
    const variable = await figma.variables.getVariableByIdAsync(variableId);
    frame.setBoundVariable('itemSpacing', variable);      // gap between items
    frame.setBoundVariable('paddingTop', variable);       // padding
    frame.setBoundVariable('paddingBottom', variable);
    frame.setBoundVariable('paddingLeft', variable);
    frame.setBoundVariable('paddingRight', variable);
    ```

    **STRING variables → font family (if stored as STRING):**
    ```js
    // STRING variables can't be bound to font family in Figma's API —
    // font family is set via figma.loadFontAsync() and textNode.fontName.
    // Display the variable value as label text instead.
    ```

    **Bindable property reference:**

    | Token type | Figma property | Method |
    |---|---|---|
    | Color | Fill color | `setBoundVariableForPaint(fill, 'color', var)` |
    | Color | Stroke color | `setBoundVariableForPaint(stroke, 'color', var)` |
    | Spacing | Width, height | `setBoundVariable('width', var)` |
    | Spacing | Padding | `setBoundVariable('paddingTop', var)` etc. |
    | Spacing | Item spacing (gap) | `setBoundVariable('itemSpacing', var)` |
    | Corner radius | All corners | `setBoundVariable('topLeftRadius', var)` etc. |
    | Opacity | Node opacity | `setBoundVariable('opacity', var)` |
    | Font size | Text font size | `setBoundVariable('fontSize', var)` |

    `setBoundVariable` modifies the node in place. `setBoundVariableForPaint` returns a new paint object that must be reassigned.

16. **Design rules for the doc page:**
    - All sections use auto-layout for clean alignment
    - Color swatches: use `primaryAxisSizingMode: 'FIXED'`, `layoutWrap: 'WRAP'`, `counterAxisSizingMode: 'AUTO'` to prevent height clipping
    - Section headers: 24px bold, dark gray (#1A1A1A)
    - Labels: 11-12px, medium gray (#666666)
    - Load fonts async before setting text: `await figma.loadFontAsync({ family: "Inter", style: "Regular" })`
    - Frame background: white (#FFFFFF), width ~900px

17. **Take a screenshot** of the completed page to verify it renders correctly and variables are bound (hovering a swatch in Figma should show the variable name, not a raw hex).

## Token Naming Conventions

When naming tokens, follow these rules:

1. **Use the original name when available** — if the code defines `$color-primary`, name it `colors/primary`
2. **For unnamed values (hardcoded)**, infer a semantic name from context:
   - A padding of `8` used everywhere → `spacing/sm`
   - A corner radius of `12` on cards → `radius/card`
   - A color used on checkboxes → `colors/checkbox`
3. **Group with forward slashes** — Figma displays these as folders: `colors/primary`, `colors/secondary`
4. **Use lowercase kebab-case** — `spacing/button-padding`, not `spacing/ButtonPadding`
5. **For SwiftUI system colors** (`.blue`, `.red`, `.green`), use the resolved hex value and name them `colors/system-blue`, `colors/system-red`, etc. Look up the standard iOS system color hex values.

## Handling Edge Cases

- **SwiftUI semantic colors** (`.primary`, `.secondary`, `.tertiary`): These adapt to light/dark automatically. Create two modes in Figma with the resolved light and dark values.
- **SwiftUI system colors** (`.blue`, `.red`): Look up the standard iOS 17+ hex values for both appearances.
- **Tailwind arbitrary values** (`text-[#1a1a2e]`): Skip these — they're one-offs, not tokens.
- **Duplicate values**: If the same hex color appears with different names, keep both — they represent different semantic roles.
- **No tokens found**: If a scan returns very few tokens, tell the user: "This project doesn't have many centralized tokens. Want me to scan all view files for hardcoded values instead? This will find more tokens but they may be less organized."

## Important Notes

- **Never modify source code** — this skill only reads from code and writes to Figma.
- **Always confirm before creating** — show the user what will be created and get approval.
- **Batch for performance** — always use `figma_setup_design_tokens` or `figma_batch_create_variables`, never create variables one at a time.
- **Colors must be hex** — the Figma MCP only accepts hex format for COLOR variables.
- **Figma Enterprise not required** — the figma-console MCP (plugin bridge) works on any Figma plan, unlike the REST API which requires Enterprise for variables.
