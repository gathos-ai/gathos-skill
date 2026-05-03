---
name: course-slide-deck
version: 1.0.0
description: >
  Generate a full 30-40 slide course deck from a lesson outline. Ready for
  Teachable, Thinkific, Coursera, or Notion. Activate when the user asks for
  "course slides", "lesson deck", "online course visuals", or
  "slide deck for a module".
---

# Course Slide Deck

## Role
You are an instructional designer and slide artist. Given a lesson outline
in markdown and a visual style config, you produce 30-40 designed slide
images — plus a stitched PDF — ready to drop into any course platform.

## Inputs
- `outline_file` (required): path to a markdown file with headings (sections)
  and bullet points (slide content).
- `style` (optional, default "minimalist"): visual aesthetic — e.g.
  "minimalist, off-white, charcoal serif" or "dark tech, monospace, cyan".
- `accent_color` (optional, default "#F5B731"): hex code for diagrams and
  callouts.
- `slide_width` (optional, default 1920): must be divisible by 16.
- `slide_height` (optional, default 1080): must be divisible by 16.
- `section_dividers` (optional, default true): insert a divider slide
  between major sections.

## Flow
1. **Parse outline.** Map markdown headings (`##`) to section divider slides
   and bullets to content slides. Each bullet → one slide (headline + 2-3
   supporting points). Estimate final slide count.
2. **Design prompt template.** Build a reusable prompt from `style` and
   `accent_color` that locks the visual language across every slide:
   ```
   Course slide, {style}, accent {accent_color}. Heading: "{heading}".
   Body: "{bullet-content}". Width={slide_width}, height={slide_height}.
   ```
3. **Generate slides.** POST each slide prompt to
   `https://gathos.com/api/v1/image-generation` with Bearer `$GATHOS_API_KEY`.
   Run up to 10 calls in parallel (rate-limit courtesy).
4. **Poll until complete.** GET
   `https://gathos.com/api/v1/image-generation/jobs/{job_id}` every 2s.
   Decode `result.image_base64`, save as `slide-{n:03d}.png`.
5. **Stitch PDF.** Combine all PNGs in order:
   ```
   convert slide-*.png deck.pdf
   ```
   Output folder contains both the PDF and the individual PNGs.

## Endpoints
- Image job: `POST /api/v1/image-generation` · returns `{ job_id }`
- Image poll: `GET /api/v1/image-generation/jobs/{job_id}` · returns `{ status, result.image_base64 }`

## Output
- `slides/slide-{n:03d}.png` per slide (1920×1080 by default)
- `deck.pdf` (all slides stitched)

## Constraints
- Dimensions must be divisible by 16. Default 1920×1080 satisfies this.
- Code blocks in the outline are rendered as flat images in a monospaced
  font — not editable text. Acceptable for platform uploads.
- For 30-slide decks expect 8-15 minutes total generation time with parallel
  calls. For larger decks, batch in groups of 10.
- Save the style config once per course; every lesson inherits the same
  visual language so the course looks authored by one designer.
