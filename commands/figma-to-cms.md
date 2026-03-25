Analyze a Figma design frame and extract a CMS content schema — what's editable content vs decorative structure. Then annotate the design directly in Figma.

**Core principle:** A CMS schema is a contract between the editor and the frontend — not a mirror of the design. The design shows one state. Your schema must account for all reasonable states.

## 0. Environment

This command requires the Figma MCP server connected to Claude Code:

- **Read** — `get_design_context` + `get_screenshot` to fetch the frame's node tree and visual reference
- **Write** — `use_figma` to create annotation frames via the Plugin API (load `figma-use` skill first)

If write access isn't available, skip step 7 (annotations) and output the schema as text only.

## 1. Read the design

The user provides a Figma URL. Extract `fileKey` and `nodeId` from it:
- `figma.com/design/:fileKey/:name?node-id=X-Y` → fileKey, nodeId = `X:Y`
- Branch URLs: `figma.com/design/:fileKey/branch/:branchKey/...` → use branchKey

Fetch in parallel:
1. **Node tree** — the frame's structure, properties, text content, fills, layout
2. **Screenshot** — visual reference of the frame

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
  title: string (required)
  slug: slug (from title)
  seo: object { metaTitle: string, metaDescription: text, ogImage: image }

  hero: object
    heading: string (required)
    subheading: text
    cta: object { label: string, url: url }

  featuredPosts: array of ref → Post
```

**Types:** string, text, richText, image, url, slug, number, boolean, date, datetime, reference, array, object.

For each field: type, required/optional, constraints, relationships.

## 6. Stress-test

Run before presenting:

| Test | Question | Action |
|------|----------|--------|
| Editor | Field name clear without the design? | Rename |
| Empty state | Breaks if blank? | Mark required |
| Overflow | 200 chars in a string? | Note max |
| Duplication | Same content in multiple blocks? | Model once |
| Type precision | Really a `string`? | Could be text, richText, url, slug, number, date |
| Alt text | Every image has alt? | Add — no exceptions |

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
- Keep field names short, camelCase, readable
- Don't model layout as content
- Don't over-nest — flatten 1-2 field objects
- Don't under-type — use precise types
- Placeholder text is not structure
