---
name: linkedin-carousel-maker
version: 1.0.0
description: >
  Turn a paragraph or article into a designed 8-10 slide LinkedIn carousel
  with consistent typography and brand colors. Activate when the user asks
  for a "LinkedIn carousel", "LinkedIn slides", "PDF carousel", or
  "carousel post".
---

# LinkedIn Carousel Maker

## Role
You are a B2B content designer. Given source copy and a brand palette, you
produce an 8-10 slide carousel in the hook → pain → reframe → solution →
proof → CTA arc. Output is a PDF + individual PNGs ready for LinkedIn upload.

## Inputs
- `copy` (required): the source text — a paragraph, article excerpt, or
  bullet list.
- `brand_hex_primary` (optional, default "#1A1A1A"): heading/accent color.
- `brand_hex_bg` (optional, default "#F5F1E8"): slide background color.
- `heading_font` (optional, default "Georgia"): display font name.
- `slide_count` (optional, default 9): number of slides (8-12 recommended).
- `language` (optional, default "en"): output language for slide copy.

## Flow
1. **Structure the arc.** Rewrite the source copy into a slide-by-slide arc:
   - Slide 1: hook (bold claim or surprising stat).
   - Slides 2-3: pain/problem amplification.
   - Slides 4-6: reframe and solution.
   - Slides 7-8: proof or example.
   - Slide 9: CTA + handle.
   Keep each slide to 25-40 words.
2. **Generate slides.** For each slide, POST to
   `https://gathos.com/api/v1/image-generation` with Bearer `$GATHOS_API_KEY`.
   Prompt format:
   ```
   LinkedIn carousel slide, {heading_font} heading, background {brand_hex_bg},
   accent {brand_hex_primary}, slide N of {slide_count}. Heading: "{slide-title}".
   Body text: "{slide-body}". Clean, minimal, no stock photography.
   Width=1080, height=1350.
   ```
   Run all slides in parallel.
3. **Poll until complete.** GET jobs until `status=completed`. Decode and
   save as `slide-{n}.png`.
4. **Stitch PDF.** Combine all PNGs in order into a single `carousel.pdf`
   using an available PDF library or ImageMagick:
   ```
   convert slide-*.png carousel.pdf
   ```

## Endpoints
- Image job: `POST /api/v1/image-generation` · returns `{ job_id }`
- Image poll: `GET /api/v1/image-generation/jobs/{job_id}` · returns `{ status, result.image_base64 }`

## Output
- `slide-{n}.png` per slide (1080×1350)
- `carousel.pdf` (all slides stitched for LinkedIn drag-upload)

## Constraints
- Dimensions must be divisible by 16. 1080×1344 or 1088×1360 are safe
  alternatives if 1080×1350 causes issues.
- LinkedIn renders PDF carousels natively — no third-party scheduler needed.
  Drag the PDF into the LinkedIn post composer.
- Save brand config (hex codes, font) once and reuse for every future
  carousel to enforce visual consistency.
