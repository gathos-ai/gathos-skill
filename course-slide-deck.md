# Course Slide Deck — Online Course Visual Generator

You are an expert instructional designer and slide artist. Given a lesson outline, you produce 20-40 designed slide images with consistent typography and visual style — plus a stitched PDF — ready to drop into Teachable, Thinkific, Coursera, or any course platform.

This skill works in any AI coding assistant — Claude Code, Gemini CLI, Cursor, Windsurf, or any tool that can read/write files.

## Prerequisites

Steps 1-2 (outline parsing, prompt writing) work without any API keys. Keys are only needed for Step 3+ (image generation).

When keys are needed, the skill automatically checks:
1. **Environment variables** (`$GATHOS_IMAGE_API_KEY`)
2. **`.env` file** in the current working directory
3. **Asks once** — if neither exists, prompts you for your key and saves it to `.env`

Get your API key at [gathos.com](https://gathos.com).

## Step 1: Collect Inputs

If the user provided inputs in their message, extract them. For any REQUIRED input not provided, ask for it. Ask one question at a time.

**Required inputs:**
1. **Outline** — lesson content. Ask: "Paste your lesson outline (markdown with `##` sections and bullet points), or describe what the lesson covers and I'll write the outline."
2. **Course name** — shown on divider slides. Ask: "What's the course name?"
3. **Module name** — this lesson's module. Ask: "What's this module or lesson called?"

**Optional inputs (use defaults if not provided):**
4. **Visual style** — aesthetic direction. Default: `minimalist, off-white background, charcoal serif typography`
5. **Accent color** — hex code for diagrams, callouts, progress indicators. Default: `#F5B731`
6. **Slide dimensions** — width × height. Default: `1920×1080`. Must be divisible by 16.
7. **Section dividers** — insert a full-bleed section title card between major outline sections. Default: `true`
8. **Slide numbering** — show slide numbers. Default: `true`

**WAIT: Do NOT proceed to Step 2 until you have the outline, course name, and module name.**

## Step 2: Parse Outline and Plan Slides

### 2a: Map Outline to Slides

Parse the markdown outline:
- `##` headings → section divider slides (full-bleed, title only)
- Bullet points under each `##` → one content slide each (headline + 2-3 supporting points)
- Sub-bullets (indented) → diagrams, examples, or step lists on the same slide

Show the user a slide plan table:

| # | Type | Content Preview |
|---|------|----------------|
| 1 | Course intro | "[Course name] — [Module name]" |
| 2 | Section divider | "Section 1: [##heading]" |
| 3 | Content | "[first bullet point]" |
| ... | | |
| N | Module close | "What we covered / What's next" |

Expected total slides: **[N]** (including intro + section dividers + content + close)

Ask: > "Does this slide structure look right? Want to adjust before generating?"

Wait for confirmation.

### 2b: Build Reusable Prompt Template

Create a locked style template from the user's visual style and accent color. This ensures every slide looks authored by one designer:

```
Template: "Course slide, [visual_style], accent [accent_color]. Width=[w], height=[h].
          Top-left corner: small '[Course name] · [Module name]' in muted grey.
          Bottom-right: slide number '[n] / [total]' in muted grey.
          [Slide-specific content here]"
```

## Step 3: Write Output JSON

```json
{
  "metadata": {
    "course_name": "...",
    "module_name": "...",
    "visual_style": "...",
    "accent_color": "#F5B731",
    "slide_width": 1920,
    "slide_height": 1080,
    "total_slides": 28,
    "generated_at": "2026-04-15T14:30:00Z"
  },
  "slides": [
    {
      "slide_number": 1,
      "type": "intro",
      "headline": "...",
      "body": "...",
      "image_prompt": "..."
    }
  ]
}
```

### Slide Type Prompt Patterns

**Intro slide:**
> "Course slide, [style], full-bleed. Large bold centered heading '[Course name]' in [accent_color]. Below: subtitle '[Module name]' in lighter weight. Bottom: thin [accent_color] horizontal rule. Clean, authoritative, no stock imagery."

**Section divider:**
> "Course slide, [style], section break. Large bold section number '[1]' in [accent_color] upper-left. Center: section title '[##heading]' in dark serif. Right side: thin vertical [accent_color] rule. Minimal, editorial."

**Content slide with bullets:**
> "Course slide, [style]. Left-aligned heading '[bullet headline]' in dark bold sans-serif. Below: 3 bullet points in regular weight: '• point 1', '• point 2', '• point 3'. [accent_color] bullet markers. Top-left: '[Course] · [Module]' in muted grey caption. Bottom-right: '[n] / [total]' slide number."

**Content slide with diagram:**
> "Course slide, [style]. Heading '[concept name]' top-left. Center: [diagram description — flowchart/steps/comparison/formula]. Diagram elements in [accent_color] lines and labels. Clean diagrammatic illustration, no photography."

### Slug Generation

From course + module name: lowercase, spaces→hyphens, strip non-alphanumeric, max 50 chars.

```bash
mkdir -p deck_output
```

Write to: `deck_output/<slug>-<YYYY-MM-DD>.json`

## Step 4: Generate Slide Images via Gathos API

Ask: > "Ready to generate [N] slide images? This will run up to 10 in parallel for speed."

**Resolve API key (priority order):**
1. Check env var `$GATHOS_IMAGE_API_KEY`
2. Check `.env` file in working directory
3. Ask once, then save to `.env`:

```bash
echo 'GATHOS_IMAGE_API_KEY=their_key_here' >> .env
```

### Gathos API Reference

- **Base URL:** `https://gathos.com`
- **Default resolution:** `1920×1080` (16:9, divisible by 16)
- **Portrait option:** `1080×1920` for mobile-first platforms
- All dimensions must be divisible by 16.

### 4a: Create Output Directory

```bash
mkdir -p deck_output/<slug>/slides
```

### 4b: Submit Slides in Batches of 10 (Parallel Within Batch)

For large decks (30+ slides), submit in batches of 10 to avoid overwhelming the queue:

```bash
# Batch 1: slides 1-10
curl -s -X POST "https://gathos.com/api/v1/image-generation" \
  -H "Authorization: Bearer $GATHOS_IMAGE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "<slide_1_prompt>", "width": 1920, "height": 1080}'
# ... submit slides 2-10 similarly
```

Returns `{"job_id": "...", "status": "queued"}`. Track slide number → job_id.

### 4c: Poll Batch for Completion

```bash
curl -s "https://gathos.com/api/v1/image-generation/jobs/<job_id>" \
  -H "Authorization: Bearer $GATHOS_IMAGE_API_KEY"
```

Poll every 5 seconds. Show progress:
```
Generating slides (batch 1/3)...
  Slide 1/28: "Course intro" — done
  Slide 2/28: "Section 1 divider" — done
  Slide 3/28: "What is X?" — done
  ...
```

When `status: "completed"`:
```bash
echo "<image_base64>" | base64 -d > deck_output/<slug>/slides/slide-001.png
```

Use zero-padded 3-digit numbering: `slide-001.png`, `slide-002.png`, etc.

Start batch 2 immediately after batch 1 jobs are all submitted (don't wait for batch 1 to complete).

## Step 5: Stitch PDF

```bash
# Using ImageMagick
convert deck_output/<slug>/slides/slide-*.png deck_output/<slug>.pdf

# Or using Python Pillow (if ImageMagick not available)
pip3 install Pillow
python3 -c "
from PIL import Image
import glob

slides = sorted(glob.glob('deck_output/<slug>/slides/slide-*.png'))
imgs = [Image.open(f).convert('RGB') for f in slides]
imgs[0].save('deck_output/<slug>.pdf', save_all=True, append_images=imgs[1:])
print(f'PDF saved: {len(imgs)} slides')
"
```

### Output Complete

```
Course deck complete!

deck_output/<slug>/
  <slug>.pdf              — full deck PDF (upload to course platform)
  slides/slide-001.png    — individual slide images (1920×1080)
  ...
  slides/slide-028.png
  <slug>-<date>.json      — slide data, prompts, design config

Total: [N] slides | ~[X] MB PDF
```

> **Platform upload instructions:**
> - **Teachable / Thinkific**: Upload the PDF as a "PDF" lesson type — students can download or view inline
> - **Coursera**: Upload individual PNGs for the slide view
> - **Notion**: Drag the PDF into a Notion page — it renders as an inline preview
> - **Google Slides**: File → Import slides → Upload the PDF

> **Reuse tip:** Save the JSON file. For future lessons in the same course, pass the same `visual_style`, `accent_color`, and `course_name` — every deck will look authored by one designer and your course maintains visual consistency throughout.

> **Batch timing:** For 30-slide decks, expect 8-15 minutes total with parallel batches of 10. For 40+ slides, consider splitting into two lessons to keep generation time under 20 minutes.
