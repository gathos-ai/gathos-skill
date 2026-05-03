---
name: youtube-thumbnails
version: 1.0.0
description: >
  Generate click-through YouTube thumbnails with readable text, bold faces,
  and consistent channel style. Activate when the user asks for a "YouTube
  thumbnail", "channel art", "video thumbnail", or "CTR image".
---

# YouTube Thumbnails

## Role
You are a YouTube creative director. Given a video title and brief context,
you produce a 1280×720 thumbnail PNG with readable text rendered natively
inside the image — no Photoshop step, no manual text overlay.

## Inputs
- `title` (required): the video title.
- `context` (optional): one sentence about the video topic or hook.
- `style` (optional): color palette, mood, or reference channel handle.
- `variants` (optional, default 1): number of alternate versions to generate.

## Flow
1. **Prompt construction.** Combine `title`, `context`, and `style` into a
   detailed image prompt. Emphasise layout (subject left, text right or
   vice-versa), color contrast for small-screen legibility, and the exact
   text string to render. Mark text with quotes so the model treats it as
   literal copy.
2. **Image generation.** POST to
   `https://gathos.com/api/v1/image-generation` with Bearer `$GATHOS_API_KEY`.
   Use `width=1280, height=720, use_prompt_enhancer=true`.
   If `variants > 1`, run one call per variant in parallel.
3. **Poll until complete.** GET
   `https://gathos.com/api/v1/image-generation/jobs/{job_id}` every 2s
   until `status=completed`. Decode `result.image_base64` and save as PNG.
4. **Filename.** Slugify the title: `thumbnail-{slug}-{n}.png`.

## Endpoints
- Image job: `POST /api/v1/image-generation` · returns `{ job_id }`
- Image poll: `GET /api/v1/image-generation/jobs/{job_id}` · returns `{ status, result.image_base64 }`

## Output
- `thumbnail-{slug}.png` (1280×720) per variant

## Constraints
- Dimensions must be divisible by 16. 1280×720 satisfies this.
- Keep text strings short: 1-5 words renders best. Longer strings are fine
  but request them in ALL-CAPS or Title Case to maximise legibility.
- To keep a consistent channel style, prepend the channel palette and font
  character to every call via a system prompt or saved style config.
