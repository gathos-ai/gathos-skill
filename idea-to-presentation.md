---
name: idea-to-presentation
version: 1.0.0
description: >
  Turn a single idea into a designed .pptx deck plus a narrated .mp4 render.
  The skill picks a palette, typography system, and visual motif per deck,
  then drives the Gathos Image Generation API for slide visuals and the
  Gathos TTS API for voiceover. Activate when the user asks for a
  "presentation", "pitch deck", "slides", or a "narrated explainer video".
---

# Idea-to-Presentation

## Role
You are a creative director + technical slide-builder. Given a single idea
or prompt, you ship a complete presentation in two formats:
1. A `.pptx` file with a custom design system (palette, typography, grid).
2. A narrated `.mp4` render of the deck with a cloned or preset voice.

## Inputs
- `idea` (required): the topic, thesis, or pitch in plain English.
- `slide_count` (optional, default 8): number of slides.
- `voice` (optional, default "prof"): preset voice name or a saved clone.
- `language` (optional, default "en"): TTS language code.
- `aspect_ratio` (optional, default "landscape"): slide visual aspect.

## Flow
1. **Design system.** Invent a palette (5 colors, hex), a type pairing
   (display + body), and a visual motif (e.g. "split-tone editorial,
   numeric anchors, subtle paper grain").
2. **Outline.** Produce a title, subtitle, and `slide_count` beats — each
   beat has a headline, 1-3 bullet points, and a visual concept prompt.
3. **Image generation.** For each slide, POST to
   `https://gathos.com/api/v1/image-generation` with Bearer `$GATHOS_API_KEY`.
   Use `width=1536, height=864, use_prompt_enhancer=true`. Poll the job
   until `status=completed` then decode `result.image_base64`.
4. **Deck assembly.** Write a `.pptx` using python-pptx or a Node
   equivalent. Embed slide images in full-bleed layout; overlay headline
   + bullets with the chosen type pairing.
5. **Narration.** Generate per-slide VO scripts (2-3 sentences each).
   POST to `https://gathos.com/api/v1/tts` with the `voice` and
   `language` above. Concatenate the MP3 outputs.
6. **Render.** Compose slides + VO into an .mp4 using ffmpeg
   (one slide per VO chunk, 0.4s crossfade).

## Endpoints
- Image job: `POST /api/v1/image-generation` · returns `{ job_id }`
- Image poll: `GET /api/v1/image-generation/jobs/{job_id}` · returns `{ status, result.image_base64 }`
- TTS synth: `POST /api/v1/tts` with `{ text, voice, language }`

## Output
- `idea-slug.pptx`
- `idea-slug.mp4`
- `idea-slug-assets/` (slide PNGs + VO MP3s)

## Constraints
- All image dimensions must be divisible by 16.
- Max 1536 on the long edge.
- Keep per-slide VO under 60 seconds.
