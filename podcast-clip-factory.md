---
name: podcast-clip-factory
version: 1.0.0
description: >
  Turn a full podcast episode into 20 vertical social clips with captions,
  cover images, and a 60-second trailer narrated in the host's cloned voice.
  Activate when the user asks for "podcast clips", "episode repurposing",
  "shareable clips", or "social cuts from a podcast".
---

# Podcast Clip Factory

## Role
You are a podcast producer turned social editor. Given a full episode audio
file and a voice sample, you ship: 20 upload-ready vertical MP4 clips, a
cover image per clip, and a 60-second trailer narrated in the host's voice.

## Inputs
- `episode_file` (required): path to an MP3 or WAV of the full episode.
- `voice_sample` (required): path to a 30–90 second WAV of the host's voice.
- `clip_count` (optional, default 20): number of clips to produce.
- `clip_max_seconds` (optional, default 60): max duration per clip.
- `caption_style` (optional): font name and color (e.g. "Inter Bold, white on black").
- `cover_style` (optional): visual style for cover images (e.g. "dark minimal, wave waveform").

## Flow
1. **Transcribe.** Transcribe the episode with word-level timestamps.
2. **Score highlights.** For each 30–90 second window, score on: hook
   strength (first 3 seconds), emotional peak, quote potential. Select the
   top `clip_count` windows.
3. **Clip extraction.** For each selected window, extract audio and encode
   as vertical 1080×1920 video with a static or waveform visualiser background.
   Burn in captions using `caption_style`.
4. **Cover images.** For each clip, generate a cover image via
   `https://gathos.com/api/v1/image-generation` with Bearer `$GATHOS_API_KEY`.
   Use `width=1080, height=1920`. Prompt: `{cover_style}, bold quote text: "{first-sentence-of-clip}"`.
   Poll until `status=completed`, decode `result.image_base64`.
5. **Trailer.** Write a 60-second narration script previewing the episode's
   top insights. POST to `https://gathos.com/api/v1/tts`:
   ```json
   { "text": "<trailer-script>", "voice_sample_url": "<sample>", "language": "en" }
   ```
   Mux audio over a cover-style title card to produce `trailer.mp4`.
6. **Output folder.** Write all clips, covers, and the trailer into a
   named folder.

## Endpoints
- Image job: `POST /api/v1/image-generation` · returns `{ job_id }`
- Image poll: `GET /api/v1/image-generation/jobs/{job_id}` · returns `{ status, result.image_base64 }`
- TTS synth: `POST /api/v1/tts` with `{ text, voice_sample_url, language }`

## Output
- `clips/clip-{n}.mp4` (1080×1920, H.264, up to 60s each)
- `clips/clip-{n}-cover.png` (1080×1920)
- `trailer.mp4`
- `transcript.txt`

## Constraints
- Dimensions must be divisible by 16. 1080×1920 satisfies this.
- Keep clip audio to under 60 seconds for Instagram Reels compliance.
- Two-speaker interviews: detect speaker turns in transcription and cut
  between them in the vertical layout.
- Run image generation calls in parallel to keep total pipeline time under
  15 minutes for a 60-minute episode.
