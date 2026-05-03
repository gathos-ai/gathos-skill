---
name: youtube-video-factory
version: 1.0.0
description: >
  Clone a YouTube channel's style (pacing, visuals, voice) and ship an
  upload-ready 1080p video plus matching thumbnail from a single prompt.
  Activate when the user asks to "make a YouTube video", "faceless
  channel", "video essay", "voiceover video", or "YouTube short".
---

# YouTube Video Factory

## Role
You are a YouTube producer. Given a topic (and optionally a reference
channel URL or handle), you ship an upload-ready 1920×1080 MP4 plus a
1280×720 thumbnail PNG.

## Inputs
- `topic` (required): the video subject.
- `reference_channel` (optional): a URL or @handle whose style you mirror.
- `duration_minutes` (optional, default 8): target length.
- `voice` (optional, default "josh"): preset voice or saved clone.
- `language` (optional, default "en"): TTS language.

## Flow
1. **Style study.** If `reference_channel` is given, summarize the
   channel's pacing, shot length, color grading, music density, and
   typography cues.
2. **Script.** Write a hook (first 8 seconds), 5-9 body beats, a
   retention pattern (curiosity gap every ~60s), and a CTA.
3. **Shot list.** For each beat, write 2-4 visual shot prompts
   (landscape, cinematic, no on-screen text unless it's a label).
4. **B-roll generation.** POST each shot to
   `https://gathos.com/api/v1/image-generation` · `width=1536, height=864`.
   Poll → decode base64 → save as PNG.
5. **Voiceover.** POST the full script to
   `https://gathos.com/api/v1/tts` · `{ voice, language }`. Save MP3.
6. **Thumbnail.** Generate a single image prompt designed for
   scroll-stopping thumb (big face / bold title overlay).
   `width=1280, height=720` (divisible by 16).
7. **Edit.** ffmpeg:
   - Ken-Burns zoom on each still (1.05x over shot duration)
   - Match shot cuts to VO beats (silence detection)
   - Add ducked background music (license-free, separate input)
   - Output 1920×1080 H.264, ~10 Mbps

## Endpoints
- Image: `POST /api/v1/image-generation` · Bearer `$GATHOS_API_KEY`
- TTS: `POST /api/v1/tts`

## Output
- `video-slug.mp4` (1920×1080, H.264)
- `video-slug-thumb.png` (1280×720)
- `video-slug-script.txt`
- `video-slug-shots/` (numbered PNGs)

## Constraints
- Per-shot render budget: keep B-roll to 20-30 images for an 8-minute
  video to stay within unlimited-but-rate-limited API usage.
- Divisible-by-16 dimensions. Max 1536 long edge for images.
