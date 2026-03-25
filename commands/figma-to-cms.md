Analyze a Figma design frame and extract a CMS content schema — what's editable content vs decorative structure. Then annotate the design directly in Figma.

**Core principle:** A CMS schema is a contract between the editor and the frontend — not a mirror of the design. The design shows one state. Your schema must account for all reasonable states.

## 0. Environment

This command requires the Figma MCP server connected to Claude Code:

- **Read** — `get_design_context` + `get_screenshot` to fetch the frame's node tree and visual reference
- **Write** — `use_figma` to create annotation frames via the Plugin API (load `figma-use` skill first)

If write access isn't available, skip step 7 (annotations) and output the schema as text only.

## 1. Read the design

The user provides one or more Figma URLs. Extract `fileKey` and `nodeId` from each:
- `figma.com/design/:fileKey/:name?node-id=X-Y` → fileKey, nodeId = `X:Y`
- Branch URLs: `figma.com/design/:fileKey/branch/:branchKey/...` → use branchKey

### Multiple frames

When the user provides 2+ frames, determine the relationship before analyzing:

| Relationship | How to detect | Action |
|---|---|---|
| **Same page, different breakpoints** (desktop + mobile) | Same content, different layout/dimensions | Analyze as **one schema**. The content is identical — only the layout differs. Use the larger frame as the primary source for content classification. Compare both to detect responsive-specific differences (e.g., mobile hides a section, truncates text, swaps image ratio). Flag differences but don't duplicate fields. |
| **Same page, different states** (default + hover, empty + filled) | Same structure, visual variations | Analyze as **one schema**. States are a code concern, not CMS content. Note state differences for the developer, not the editor. |
| **Different pages** (homepage + blog) | Different content entirely | Analyze as **separate schemas**. Each frame gets its own document type. Look for shared referenced documents across pages (e.g., both reference a Post document). |

For desktop/mobile pairs:
- Derive constraints from the **tightest** breakpoint — if mobile truncates a heading, that's the real character limit
- If mobile drops a section entirely, flag whether the field should be optional or if it's always required but hidden by code
- Image ratios may differ per breakpoint — note both: `desktop 16:9, mobile 1:1`

### Token-efficient fetching

Figma responses can be very large. Minimize calls and avoid redundant fetches:

1. **Single `get_design_context` call** on the target frame — this returns the node tree + screenshot in one call. Do NOT call `get_screenshot` separately if `get_design_context` already includes it.
2. **One `get_metadata` call on the page** (for neighbor scanning in step 7) — fetch this in parallel with `get_design_context`, not sequentially.
3. **One `use_figma` call to check page siblings** — combine neighbor scanning with page resolution in a single script.
4. **Never re-fetch data you already have.** The initial `get_design_context` response contains all node dimensions, text content, fills, and positions needed for classification, constraints, and schema. Extract everything from that single response.
5. **Never drill into decorative subtrees.** Once a group is classified as decorative from the screenshot, do not call `get_design_context` on its children.

**Guard:** If the node tree is empty or errors, stop and ask the user to check the URL.

**Guard — oversized response:** If the node tree is too large (truncated or split into metadata-only), do NOT re-fetch every sublayer sequentially. Instead:
1. Use `get_metadata` on the root frame to get the top-level children and their positions
2. Use the **screenshot** as the primary source of truth for structure
3. Call `get_design_context` only on specific section-level children, not the whole tree
4. Skip deep-nested decorative elements (grids, patterns, repeated lines) — classify them as decorative from the screenshot alone

## 2. Assess node naming

Scan the node tree before analysis. Report in one line:

| Quality | Action |
|---------|--------|
| Well-structured | Semantic names — use as hints, still verify |
| Partially structured | Mix of named + `Frame 12` / `Container` — trust named, ignore defaults |
| Unstructured | Mostly auto-names — ignore ALL names, derive from properties only |

**Rule:** Always derive purpose from visual properties (fills, size, position, text content, layering) — never from layer names alone.

## 3. Classify content vs decoration

Walk every element. Three buckets:

**Content (CMS):** text, rich text, content images (see below), links, media, metadata, booleans, repeating lists.

**Decorative (code):** background fills, gradients, overlays, dividers, shadows, layout wrappers, spacers, UI icons, nav/footer, hover states, decorative images.

**Ambiguous → flag, don't guess.** Ask the user with your reasoning.

### Image classification

Classify by **position, size, and layering** — not by name.

| Signal | → Content | → Decorative |
|--------|-----------|-------------|
| Size vs parent | Bounded, deliberate ratio | Full-bleed, >70% of section |
| Position | Inside auto-layout, alongside text | Behind content, lower z-order |
| Context | Inside repeating list, paired with fields | Atmospheric, texture, pattern |
| Layering | In front of other content | Overlapped by text/elements |

**Always flag:** hero images, section backgrounds with specific subjects, logos on partner pages.

### Record content metadata

While classifying, collect metadata from content elements for constraint derivation later:

- **Images** — record node width × height → aspect ratio and minimum recommended size
- **Text** — record container width, font size, line count → approximate character limit
- **Truncated text** (ellipsis, "…", or `textTruncation` set) → hard max length
- **Repeated items** — count visible instances → suggested min/max for arrays

This costs nothing — the data is already in the node tree. It feeds into constraints in step 5.

## 4. Identify blocks

Group into: page-level fields, section blocks, nested objects, references.

### Infer hidden fields

The design never shows these — the schema must have them:
- `alt` for every image — no exceptions
- `url` + `target` for every link
- `datetime` (not string) for visible dates
- SEO object for page-level documents
- Author/category documents behind visible names

### Detect patterns

| Visual signal | Schema implication |
|--------------|-------------------|
| Numbered items | Sortable array |
| "Featured" badge | Boolean flag or reference |
| "Read more" / "View all" | References to documents, not inline |
| Truncated text | `richText` or `text`, not `string` |
| 3 cards shown | `array` — flag if layout breaks at other counts |
| 2+ grouped links | Likely related — model as single link object with `label`, `url`, `target`; flag potential shared interaction (hover, modal) for user confirmation |
| Sort/filter controls ("Sort by", "Filter", dropdowns, tag chips) | Referenced documents need **filterable/sortable fields**. Identify what's being filtered (category, date, tag, type) and ensure those fields exist on the referenced document with appropriate types (`string` for enums, `datetime` for dates, `reference` for taxonomy). Note the filter dimensions in the schema |
| Search bar | Referenced documents need a searchable text field — ensure `title` or key fields are `string` (indexed), not `richText` |
| Pagination / "Load more" / result count ("Results (75)") | Content is queried, not inline — model as referenced documents with an `array` or query, not a fixed list |

### Reference vs inline

"Does this content survive if the page is deleted?"
- **Yes → separate document** (team members, posts, products, authors)
- **No → inline object** (CTA, stat, process step)

## 5. Propose schema

Platform-agnostic format:

```
Document: pageName
  title: string (required, ~60 chars)
  slug: slug (from title)
  seo: object { metaTitle: string (~60 chars), metaDescription: text (~160 chars), ogImage: image (2:1, min 1200×630) }

  hero: object
    heading: string (required, ~40 chars)
    subheading: text (~120 chars)
    backgroundImage: image (16:9, min 1440×810)
    cta: object { label: string (~20 chars), url: url }

  featuredPosts: array of ref → Post (3–6 items)
```

**Types:** string, text, richText, image, url, slug, number, boolean, date, datetime, reference, array, object.

For each field: type, required/optional, constraints, relationships.

### Derive constraints from the layout

**Every constraint must come from the design itself.** Don't invent arbitrary limits — only flag what the layout actually enforces or would break without.

| Layout signal | Why it matters | Constraint |
|--------|------------------|------------|
| Image has a fixed aspect ratio in the layout | Different ratio would break the composition | `16:9, min 1440×810` |
| Text sits in a tight container, single line | Overflow would clip or break layout | `max ~40 chars` |
| Text container is generous, multi-line | Layout tolerates variation | `~120 chars` (soft) |
| Truncation visible (ellipsis, clipped text) | Design explicitly limits length | `max 60 chars` (hard) |
| Exactly 3 cards in a row with no "load more" | Layout depends on a specific count | `3 items` |
| Cards with "load more" / pagination | Layout adapts to variable counts | `3–12 items` |

Prefix with `~` for soft limits (layout is forgiving). Use `max` for hard limits (layout would break).

**Don't add constraints when the layout doesn't care.** A richText body in a full-width section has no meaningful character limit — don't invent one.

### Editor hints (extended mode)

For larger projects or on request, generate a short description per field explaining **why the constraint exists from a layout perspective** — usable as CMS help text:

```
heading: "Displayed as a single line in large serif. Longer text will overflow the container."
backgroundImage: "Full-bleed behind the hero section. Must be landscape to fill the 16:9 container."
cards: "Shown in a 3-column grid. Fewer than 3 leaves visible gaps."
```

Trigger: the user asks for it, or the page has 5+ content sections. Output as a separate block after the schema.

## 6. Stress-test

Run before presenting:

| Test | Question | Action |
|------|----------|--------|
| Editor | Field name clear without the design? | Rename |
| Empty state | Breaks if blank? | Mark required |
| Overflow | 200 chars in a string? | Note max, derive from container width |
| Duplication | Same content in multiple blocks? | Model once |
| Type precision | Really a `string`? | Could be text, richText, url, slug, number, date |
| Alt text | Every image has alt? | Add — no exceptions |
| Image ratio | Does the design enforce an aspect ratio? | Add ratio + min dimensions |
| Array bounds | Fixed count or flexible? | Add min/max from visible items |
| Constraints realistic? | Is the derived char limit reasonable? | Cross-check with screenshot — if text looks spacious, the limit is soft (~). If tight/truncated, it's hard (max) |

## 7. Generate Figma annotations

Annotate the design directly in Figma using the Plugin API. Skip this step if write access isn't available.

### Process

1. **Read the frame** — get position and dimensions
2. **Scan for neighboring elements** — before placing any annotations, read ALL top-level children on the page to build a map of occupied space. Check that the intended annotation area (right or left of the target frame) is clear. If other frames/elements occupy both sides, **notify the user** with what's there and ask where they'd like annotations placed. Do not place annotations that would overlap existing design content.
3. **Place annotation cards** to the right of the frame (120px gap), aligned to each section's Y. **If the content a card describes is positioned on the right side of the design** (x > frame.width * 0.6), place that card on the **left side** instead (x = frame.x - cardWidth - 120) so the connector line is shorter and the association is clearer. Default to right side when content is centered or left-aligned.
4. **Prevent overlaps + consistent spacing** — after placing each card, track its bottom edge (`card.y + card.height`). The next card's Y = `max(idealY, previousBottom + MIN_GAP)` where `MIN_GAP = 32`. This ensures cards never overlap AND always have at least 32px between them for comfortable readability, even when section Y positions are far apart.
5. **Draw dashed connector lines** from the nearest frame edge to each card's actual Y — right edge for right-side cards, left edge for left-side cards
6. **Add legend** above first card with at least 56px gap below the legend. Position the legend high enough (e.g., `firstCard.y - 56`) to avoid colliding with Figma's native frame title labels that appear above frames on the canvas.
7. **Add referenced document cards** below the last card (not below the design — use the last card's bottom edge + 40px)

If clone into new page is possible, do so. If read-only or times out, annotate next to existing frame.

### Updating annotations on schema changes

When the user requests a change to a specific section's schema, do NOT rebuild all annotations. Instead:
1. Find the existing card by name using `figma.currentPage.findOne(n => n.name === 'Section Name')`
2. Delete it and its connector line
3. Recreate only that card at the same Y position
4. Return the new node IDs

### Card — Figma Plugin API code

```js
// Card container
const card = figma.createFrame()
card.layoutMode = 'VERTICAL'
card.itemSpacing = 2
card.paddingTop = 18; card.paddingBottom = 18
card.paddingLeft = 18; card.paddingRight = 18
card.resize(340, 10)
card.layoutSizingVertical = 'HUG'
card.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }]
card.cornerRadius = 16
card.strokes = [{ type: 'SOLID', color: { r: 0.94, g: 0.94, b: 0.94 } }]
card.strokeWeight = 1

// Header — Inter Medium 12
const header = figma.createText()
header.characters = 'Section Name'
header.fontSize = 12
header.fontName = { family: 'Inter', style: 'Medium' }
header.fills = [{ type: 'SOLID', color: { r: 0.30, g: 0.30, b: 0.30 } }]
header.textAutoResize = 'WIDTH_AND_HEIGHT'
card.appendChild(header)

// Spacer
const spacer = figma.createFrame()
spacer.resize(10, 8); spacer.fills = []
card.appendChild(spacer)
spacer.layoutSizingHorizontal = 'FILL'

// Field row
const row = figma.createFrame()
row.layoutMode = 'HORIZONTAL'
row.itemSpacing = 7
row.counterAxisAlignItems = 'CENTER'
row.paddingTop = 7; row.paddingBottom = 7
row.paddingLeft = 12; row.paddingRight = 12
row.layoutSizingVertical = 'HUG'
row.cornerRadius = 8
row.fills = [{ type: 'SOLID', color: STATUS_BG }]

const dot = figma.createEllipse()
dot.resize(4, 4)
dot.fills = [{ type: 'SOLID', color: STATUS_DOT }]
row.appendChild(dot)

const fieldName = figma.createText()
fieldName.characters = 'fieldName'
fieldName.fontSize = 11
fieldName.fontName = { family: 'Inter', style: 'Medium' }
fieldName.fills = [{ type: 'SOLID', color: { r: 0.22, g: 0.22, b: 0.22 } }]
fieldName.textAutoResize = 'WIDTH_AND_HEIGHT'
row.appendChild(fieldName)

const fieldType = figma.createText()
fieldType.characters = 'string (req)'
fieldType.fontSize = 11
fieldType.fontName = { family: 'Inter', style: 'Regular' }
fieldType.fills = [{ type: 'SOLID', color: { r: 0.55, g: 0.55, b: 0.55 } }]
fieldType.textAutoResize = 'WIDTH_AND_HEIGHT'
row.appendChild(fieldType)

// Constraint (optional — append when derived from design metadata)
// Format: "· ~60 chars" or "· 16:9, min 1440×810"
const constraint = figma.createText()
constraint.characters = '· ~60 chars'
constraint.fontSize = 10
constraint.fontName = { family: 'Inter', style: 'Regular' }
constraint.fills = [{ type: 'SOLID', color: { r: 0.70, g: 0.70, b: 0.70 } }]
constraint.textAutoResize = 'WIDTH_AND_HEIGHT'
row.appendChild(constraint)

card.appendChild(row)
row.layoutSizingHorizontal = 'FILL'
```

### Connector

```js
const line = figma.createLine()
line.strokes = [{ type: 'SOLID', color: { r: 0.88, g: 0.88, b: 0.88 } }]
line.strokeWeight = 0.5
line.dashPattern = [4, 4]
```

### Status colors

```
green   dot { r: 0.18, g: 0.55, b: 0.34 }  bg { r: 0.92, g: 0.98, b: 0.94 }
grey    dot { r: 0.50, g: 0.50, b: 0.50 }  bg { r: 0.94, g: 0.94, b: 0.94 }
yellow  dot { r: 0.70, g: 0.50, b: 0.05 }  bg { r: 0.98, g: 0.96, b: 0.90 }
blue    dot { r: 0.22, g: 0.38, b: 0.78 }  bg { r: 0.92, g: 0.94, b: 1.0 }
```

### Ref doc cards — same structure, stroke `{ r: 0.85, g: 0.88, b: 0.95 }`, header in blue.

### Fonts

Inter Medium + Regular only. Check available styles before loading — Inter uses `"Semi Bold"` not `"SemiBold"`.

### Plugin API pitfalls

| Pitfall | Fix |
|---------|-----|
| Colors 0–255 | Use 0–1 range |
| Mutating fills/strokes | Clone array, modify, reassign |
| Text without font loaded | `await figma.loadFontAsync()` first |
| `FILL` sizing before append | Set AFTER `parent.appendChild(child)` |
| Shadow missing blendMode | Add `blendMode: 'NORMAL'` |
| Stroke with alpha in color | Strokes don't support `a` — use node opacity |
| Clone timeout on large frames | Skip clone, annotate next to existing |
| `figma.notify()` | Not implemented — use `return` |

### Incremental workflow

Don't build all annotations in one script. Split into 2-3 calls:
1. Legend + first batch of cards (5-6 sections)
2. Remaining cards + ref doc cards
3. Validate with a screenshot

Return all created node IDs from each call.

## 8. Validate with user

Present schema + annotations, then ask:

1. Block structure correct?
2. Field types correct?
3. Anything missing?
4. Grey-area items to resolve?
**Guard:** Wait for answers. Always generate Figma annotations immediately after proposing the schema — do not wait for validation to annotate.

## 9. Iterate

Update schema and annotations on changes.

## Rules

- NEVER include decorative elements in the schema
- NEVER assume fixed list counts — always arrays
- NEVER use Figma layer names as field names
- ALWAYS flag ambiguous elements
- ALWAYS cross-reference screenshot vs node tree — screenshot is ground truth
- ALWAYS add alt fields for images
- ALWAYS place annotations on the same page as the target frame — resolve the frame's parent page via `use_figma` and switch to it with `setCurrentPageAsync` before creating any annotation nodes
- NEVER call `get_design_context` or `get_screenshot` on nodes you've already fetched — extract all data from the initial response
- NEVER drill into decorative subtrees — classify from the screenshot and move on
- Keep field names short, camelCase, readable
- Don't model layout as content
- Don't over-nest — flatten 1-2 field objects
- Don't under-type — use precise types
- Placeholder text is not structure
- Batch `use_figma` calls — combine neighbor scanning + page resolution in one script, annotation creation in 2-3 scripts max
