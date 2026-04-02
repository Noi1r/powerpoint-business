# PowerPoint Business — SVG-Native Slide Generation

> A Claude Code skill that generates visually polished business presentations through multi-agent collaboration.
> LLM directly generates SVG slides (1280x720), exported as editable SVG-embedded PPTX (Office 365+).

## How It Works

```
User Input (topic / PDF / URL)
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│                   Lead Agent (Orchestrator)              │
│                                                         │
│  Phase 0-1          Phase 2         Phase 3-4           │
│  ┌───────────┐    ┌────────────┐   ┌──────────────┐    │
│  │ Consultant │    │ Researcher │   │   Planner    │    │
│  │ (需求分析) │    │ (网络调研)  │   │ (大纲+规划)  │    │
│  └─────┬─────┘    └─────┬──────┘   └──────┬───────┘    │
│        │ requirements   │ research/       │ outline.json│
│        │ .md            │ chapter-*.md    │ planning-   │
│        ▼                ▼                 ▼ draft.md    │
│                                                         │
│  Phase 5-6                    Phase 6 QA                │
│  ┌───────────────────┐       ┌────────────────┐        │
│  │     Designer      │──────▶│    Reviewer     │        │
│  │ (SVG Draft+Design)│◀──fix─│ (自动+人工 QA) │        │
│  └────────┬──────────┘       └────────────────┘        │
│           │ slides/*.svg                                │
│           ▼                                             │
│  Phase 7: Export                                        │
│  ┌──────────────────────────────────────────────┐      │
│  │ sync_manifest → export_pptx → preview.html   │      │
│  └──────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
        │
        ▼
  output/presentation.pptx  +  output/index.html
```

**Core idea**: LLM is the designer. No templates, no PowerPoint API to manipulate shapes — the LLM generates raw SVG code for every slide, then Python scripts handle PPTX packaging. This gives the LLM full creative control over layout, typography, charts, and decoration.

---

## Architecture

### SVG-Native Approach

Traditional PPT generation tools call a library's API to create shapes, textboxes, and charts. This skill takes a fundamentally different approach:

1. **LLM generates SVG directly** — each slide is a standalone SVG file (1280x720, 16:9)
2. **SVG is embedded into PPTX** via OOXML SVG extension — editable text/shapes in Office 365+
3. **PNG fallback** is auto-generated for older Office versions

Why SVG? Because LLMs are excellent at generating structured markup. An SVG slide is just XML — the LLM can reason about coordinates, colors, typography, and layout in a single coherent document. No API abstraction layer, no "closest shape" compromises.

### Multi-Agent Collaboration

5 specialized agents, each with a dedicated system prompt (`prompts/`), handle different phases:

| Agent | Role | Phases | Key Output |
|-------|------|--------|------------|
| **Consultant** | Needs analysis & requirements interview | 0-1 | `requirements.md` |
| **Researcher** | Web research with confidence tagging | 2 | `research/chapter-*.md` |
| **Planner** | Information architecture & narrative design | 3-4 | `outline.json` + `planning-draft.md` |
| **Designer** | SVG generation (draft + themed) | 5-6 | `drafts/*.svg` + `slides/*.svg` |
| **Reviewer** | Two-layer QA (automated + LLM aesthetic) | 6 | `reviews/*.md` + `scores.json` |

The Lead Agent (the skill itself) orchestrates the pipeline, dispatching each agent as a subagent with the appropriate prompt and upstream artifacts.

**Role boundaries are strict**: each agent reads only upstream outputs and never modifies other agents' artifacts. Reviewer feedback routes back to the correct agent — content issues go to Planner, visual issues go to Designer.

---

## The 8-Phase Pipeline

### Phase 0: Material Analysis (Consultant)

Read user-provided materials (PDF, Word, text, URL). Extract core thesis, logical structure, key data points, audience cues, visual assets, and content gaps. This analysis feeds Phase 1 but is not shown to the user.

### Phase 1: Requirements Interview (Consultant) — HARD STOP

Interactive interview with the user via structured questions:

- **Audience**: C-suite / clients / internal team / general
- **Purpose**: Persuade / inform / teach / pitch
- **Page count**: Calculated from material volume (concise / standard / comprehensive)
- **Visual theme**: 14 themes available (see below)
- **Research**: Whether to supplement with web research

The user must confirm requirements before the pipeline continues.

### Phase 2: Web Research (Researcher) — Optional

Only runs if the user opted in. For each chapter in the outline:

1. Determine search intent (market data, pain points, case studies, etc.)
2. Execute max 3 searches per chapter via grok-search
3. Tag every claim with confidence level: `[confirmed]` / `[probable]` / `[unverified]`

### Phase 3: Pyramid Outline (Planner) — HARD STOP

Apply Barbara Minto's Pyramid Principle as the structural backbone. Choose the best narrative framework:

| Framework | Best For |
|-----------|----------|
| **Pyramid** | Standard business reporting |
| **SCQA** | Problem-solving, proposals |
| **PAS** | Persuasion, sales, fundraising |
| **Hero's Journey** | Brand narrative, transformation stories |

Output: `outline.json` with per-slide structure. The user must approve the outline before proceeding.

### Phase 4: Planning Draft (Planner)

For each slide, produce an 8-field planning card:

| Field | Content |
|-------|---------|
| Objective | What this slide achieves |
| Core Message | Single takeaway |
| Evidence | Data source with confidence tags |
| Layout | Bento Grid type + card arrangement |
| Info Hierarchy | Primary → Secondary → Tertiary |
| Keywords | Key terms for icon/image selection |
| Visual Elements | Charts, diagrams, icons planned |
| Design Notes | Specific guidance for Designer |

### Phase 5: Draft SVG (Designer)

Generate one SVG per slide with neutral gray palette. Focus on:
- Correct content matching the planning draft
- Proper Bento Grid layout
- Semantic IDs on every element (`s{nn}-{role}-{index}`)
- Readable typography with proper hierarchy

### Phase 6: Design SVG + QA (Designer + Reviewer)

**Designer** injects theme style onto Draft SVGs — colors, gradients, shadows, decorative elements. Does NOT change content or layout.

**Reviewer** runs two-layer QA:
- **Layer 1 (Automated)**: `check_svg.py` validates well-formedness, safe area, contrast (WCAG AA ≥ 4.5:1), card gaps, font sizes, semantic IDs
- **Layer 2 (LLM Review)**: Scoring on 5 dimensions — Layout (25%), Readability (25%), Typography (20%), Info Density (20%), Color (10%)

Quality gate: ≥ 7.0 pass, 5.0-6.9 fix loop (max 2 rounds), < 5.0 redesign.

### Phase 7: Export

```bash
# Sync manifest from final SVGs
python scripts/sync_manifest.py slides/ slide-manifest.json

# Export PPTX (SVG-embed, editable in Office 365+)
python scripts/export_pptx.py slides/ output/presentation.pptx

# Generate interactive HTML preview
python scripts/preview.py slides/ output/index.html
```

---

## Design System

### Bento Grid Layouts

A flexible card-based layout system inspired by the Japanese bento box. 20+ layout types driven by content semantics:

| Layout | Use Case |
|--------|----------|
| `hero_stat` | Single big number + context |
| `dashboard` | 3-4 KPIs with optional charts |
| `two_col` | Side-by-side comparison |
| `timeline` | Process, roadmap |
| `chart_focus` | Large chart + annotation sidebar |
| `icon_grid` | Feature list with icons |
| `funnel` | Conversion visualization |
| `flywheel` | Closed-loop / cycle diagram |
| `quote` | Testimonial or key quote |
| `table` | Structured data matrix |
| ... | See `references/bento-grid.md` for all layouts |

Canvas: 1280x720 SVG viewBox, 60px safe area padding, usable region 1160x600.

### 14 Themes

Defined in `references/themes.yaml`, each theme specifies: color scheme (primary, secondary, accent, text, chart colors), typography scale, card style (border-radius, shadow, border), gradients, and decorative elements.

**Business themes (10)**:
Corporate Blue, Warm Earth, Minimal White, Tech Dark, Emerald Gold, Royal Purple, Sunset Coral, Nordic Sage, Ocean Teal, Rose Dusk

**Academic/Humanities themes (4)**:
Zen Ink (禅意水墨), Academic Classic (学术经典), Parchment (自然书卷), Pure Academic (极简学术)

### 206 Lucide Icons

9 categories of SVG icons from the Lucide icon set, pre-indexed in `references/icons.json`:

general, communication, data, business, education, humanities, nature, status, arrows

### 28 Chart Templates

Indexed by analysis goal in `references/charts-index.json`. Simple charts (bar, donut, line) are rendered as inline SVG. Complex statistical charts use matplotlib.

### Cognitive Design Principles

All design decisions are grounded in evidence-based principles documented in `references/cognitive-design.md`:

- **Miller's Law**: Max 4±1 information chunks per slide
- **Mayer's Multimedia Principles**: Spatial contiguity, coherence, signaling, segmenting
- **Gestalt Principles**: Proximity, similarity, continuity, figure-ground, common region
- **Cognitive Load Theory**: Minimize extraneous load, manage intrinsic load, optimize germane load
- **CRAP Design**: Contrast, Repetition, Alignment, Proximity (optional review via `--crap` flag)

---

## Project Structure

```
powerpoint-business/
├── SKILL.md                    # Skill definition (entry point)
├── prompts/
│   ├── consultant.md           # Phase 0-1: needs analysis
│   ├── researcher.md           # Phase 2: web research
│   ├── planner.md              # Phase 3-4: outline + planning
│   ├── designer.md             # Phase 5-6: SVG generation
│   ├── reviewer.md             # Phase 6 QA: two-layer review
│   └── optimizer-crap.md       # Optional CRAP design review
├── references/
│   ├── themes.yaml             # 14 theme definitions
│   ├── bento-grid.md           # Layout system specification
│   ├── svg-conventions.md      # SVG generation rules
│   ├── cognitive-design.md     # Evidence-based design principles
│   ├── icons.json              # 206 Lucide icon index
│   └── charts-index.json       # 28 chart templates
├── scripts/
│   ├── export_pptx.py          # SVG → PPTX export (SVG-embed OOXML)
│   ├── preview.py              # SVG → interactive HTML preview
│   ├── check_svg.py            # Automated QA checks
│   ├── sync_manifest.py        # Build semantic manifest from SVGs
│   ├── embed_icons.py          # Replace icon placeholders with paths
│   └── build_icons.py          # Build icons.json from Lucide source
└── demo/
    ├── slides/                 # Example SVG slides
    └── output/                 # Example exported PPTX
```

---

## Scripts

### `export_pptx.py`

Exports SVG slides to PPTX with editable SVG embedded in OOXML.

Strategy: build a standard PPTX with PNG fallback images (rendered via cairosvg at 2x retina), then inject SVG files into the ZIP package with proper OOXML relationships and content types. Office 365+ renders the SVG; older versions show the PNG.

```bash
python scripts/export_pptx.py slides/ output/presentation.pptx [--speaker-notes notes.md]
```

### `check_svg.py`

Automated quality checks on all SVG slides:

| Check | Condition |
|-------|-----------|
| XML well-formedness | Valid XML |
| viewBox | Exactly `0 0 1280 720` |
| Safe area | All elements within 60px padding |
| Font size | Every `<text>` ≥ 12px |
| Color contrast | WCAG AA ≥ 4.5:1 |
| Card gap | ≥ 20px between adjacent cards |
| Semantic IDs | `s{nn}-{role}-{index}` format |
| Manifest consistency | SVG text matches manifest (optional) |

```bash
python scripts/check_svg.py slides/ [--manifest slide-manifest.json]
```

### `sync_manifest.py`

Parses all SVG files and builds a semantic manifest (`slide-manifest.json`) containing every element's ID, role, bounding box, and text content. This manifest powers natural language editing — when a user says "change the title on slide 3", the system looks up the element in the manifest.

```bash
python scripts/sync_manifest.py slides/ slide-manifest.json [--theme corporate_blue]
```

### `preview.py`

Generates a self-contained HTML file with three viewing modes:
- **Gallery**: thumbnail grid with lightbox zoom
- **Scroll**: vertical scroll through full-width slides
- **Present**: fullscreen presentation with keyboard navigation (arrow keys, space, escape)

```bash
python scripts/preview.py slides/ output/index.html
```

### `embed_icons.py`

Post-processor that replaces `<use data-icon="name"/>` placeholders in SVG files with actual Lucide icon SVG paths from `icons.json`.

```bash
python scripts/embed_icons.py slides/ --icons references/icons.json [--in-place]
```

### `build_icons.py`

Builds `icons.json` from the Lucide icon set source SVGs. Curates 206 icons across 9 categories.

```bash
python scripts/build_icons.py --lucide-path /path/to/lucide/icons --output references/icons.json
```

---

## Usage

### As a Claude Code Skill

This project is designed as a [Claude Code](https://claude.ai/claude-code) skill. Install it to `~/.claude/skills/powerpoint-business/` and it will be triggered by commands like:

```
/powerpoint-business create AI Industry Report
/powerpoint-business create --from paper.pdf
/powerpoint-business edit my-project
/powerpoint-business retheme my-project "Tech Dark"
/powerpoint-business review my-project --crap
```

### Supported Actions

| Action | Description |
|--------|-------------|
| `create [topic/file]` | Full 8-phase pipeline from topic or source material |
| `edit [project]` | Natural language editing via semantic manifest |
| `undo [project] [slide]` | Restore previous slide version from history |
| `retheme [project] [theme]` | Regenerate Design SVGs with a different theme |
| `export [project]` | Re-export PPTX + HTML from current SVGs |
| `preview [project]` | Generate and open HTML preview |
| `review [project]` | Run full Reviewer QA on slides |

### Editing Workflow

After creating a presentation, use natural language to edit:

```
"把第3页的标题改成'市场增长趋势'"
"第5页的配色太暗了，换成暖色调"
"删掉第7页的第二张卡片"
"所有页面的副标题字号改成20px"
```

The system reads the semantic manifest to locate target elements, backs up the current SVG, dispatches the Designer for targeted regeneration, then re-syncs and re-exports.

---

## Dependencies

**Required**:

```bash
pip install python-pptx lxml cairosvg
```

- `python-pptx` — PPTX file generation
- `lxml` — SVG/XML parsing and manipulation
- `cairosvg` — SVG → PNG rendering (for fallback images)

**System dependency** (macOS):

```bash
brew install cairo
```

**Optional**:

- `matplotlib` — complex statistical charts (simple charts are inline SVG)
- `soffice` (LibreOffice) — PPTX → PDF verification

---

## Compatibility

| Office Version | SVG Rendering | Editing |
|----------------|---------------|---------|
| Office 365 (Windows/Mac) | Native SVG | Full: text, shapes, colors editable |
| Office 2021+ | Native SVG | Full |
| Office 2019 | PNG fallback | Not editable (raster image) |
| Office 2016 | PNG fallback | Not editable |
| LibreOffice | SVG partial | Limited |
| Google Slides | PNG fallback | Not editable |

---

## Design Constraints (Hard Rules)

1. **Semantic IDs on every SVG element** — `s{nn}-{role}-{index}` format
2. **60px safe area padding** on all sides
3. **Card gap ≥ 20px**
4. **No external image references** — all assets inline or base64
5. **Draft ≠ Design** — Design phase must not alter text content or card count
6. **Layout diversity** — no same layout for 3+ consecutive slides
7. **Max 7 bullets, 5 cards, 80 CJK chars** per card body
8. **Font size floor: 12px**
9. **WCAG AA contrast ≥ 4.5:1** for all text
10. **Two HARD STOPs** — user confirms requirements (Phase 1) and outline (Phase 3)
11. **Research confidence tags** — `[confirmed]` / `[probable]` / `[unverified]`
12. **`<foreignObject>` for multi-line text** — the only reliable SVG text wrapping method
13. **Version before edit** — backup to `slides/history/` before any modification

---

## License

MIT
