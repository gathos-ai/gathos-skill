---
name: script-to-reel
version: 1.0.0
description: >
  Turn a short narration script into a vertical 9:16 reel with
  split-screen visuals, dissolve transitions, and a cloned or preset
  voiceover. Instagram-, TikTok-, and YouTube-Shorts-ready. Activate
  when the user asks for a "reel", "short", "TikTok", or "vertical video".
---

# Script-to-Reel

## Role
You turn narration into a vertical 1080×1920 MP4 that looks designed,
not template-y. Output is upload-ready for Instagram Reels, TikTok,
and YouTube Shorts.

## Inputs
- `script` (required): the narration text (15-60 seconds of VO).
- `voice` (optional, default "koko"): preset voice or saved clone.
- `language` (optional, default "en"): TTS language.
- `hook_style` (optional, default "split-tone"):
  `split-tone` | `quote-overlay` | `data-card` | `question-first`.

## Flow
1. **Segment the script** into 4-8 beats (~1 sentence each).
2. **Visual prompts.** For each beat, write a vertical-composition
   image prompt optimized for 9:16 phone viewing. Prefer tight crops,
   human subjects at torso-up, bold color contrast.
3. **Image generation.** POST each prompt to
   `https://gathos.com/api/v1/image-generation` · `width=864, height=1536`
   (9:16, divisible by 16). Poll → decode → save PNG.
4. **Voiceover.** POST the full script to
   `https://gathos.com/api/v1/tts` · `{ voice, language }`. Save MP3.
5. **Edit.** ffmpeg:
   - One shot per VO beat (auto-align via silence detection)
   - 0.3s dissolve between shots
   - Subtle 1.04x Ken-Burns on each still
   - Burn-in captions (pulled from the script) in bold sans-serif
   - Output 1080×1920 H.264, ~8 Mbps, 60fps
6. **Caption file.** Emit an SRT alongside the MP4 for platforms that
   accept uploaded captions.

## Endpoints
- Image: `POST /api/v1/image-generation` · Bearer `$GATHOS_API_KEY`
- TTS: `POST /api/v1/tts`

## Output
- `reel-slug.mp4` (1080×1920, H.264)
- `reel-slug.srt`
- `reel-slug-shots/` (numbered PNGs)

## Constraints
- Keep total duration under 60 seconds for Instagram Reels.
- Divisible-by-16 dimensions. Max 1536 long edge.
- Prefer high-contrast compositions — reels are watched muted 70% of
  the time; visuals must carry meaning on their own.
