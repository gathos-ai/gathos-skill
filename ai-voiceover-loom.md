---
name: ai-voiceover-loom
version: 1.0.0
description: >
  Generate multilingual AI voiceover for Loom and screen recordings using
  zero-shot voice cloning. Activate when the user asks to "add voiceover",
  "narrate a screen recording", "clone my voice for video", or "replace
  audio in a Loom".
---

# AI Voiceover for Loom

## Role
You turn a narration script and a voice sample into a finished MP3 track
ready to overlay onto a Loom export or screen recording — in any of 600+
languages, using the user's own cloned voice.

## Inputs
- `script` (required): the narration text.
- `voice_sample` (required): path to a 30–90 second WAV or MP3 of the user speaking.
- `language` (optional, default "en"): BCP-47 language code (e.g. "hi", "es", "pt-BR").
- `video_file` (optional): path to an MP4/MOV to overlay the track onto.

## Flow
1. **TTS generation.** POST to `https://gathos.com/api/v1/tts` with
   Bearer `$GATHOS_API_KEY`:
   ```json
   { "text": "<script>", "voice_sample_url": "<uploaded-sample>", "language": "<lang>" }
   ```
   Returns `{ audio_url }` or inline base64 audio. Save as `voiceover.mp3`.
2. **Overlay (if `video_file` provided).** Run ffmpeg to strip the original
   audio track and replace it with the new voiceover:
   ```
   ffmpeg -i video_file -i voiceover.mp3 -map 0:v -map 1:a -c:v copy -shortest output.mp4
   ```
3. **Multi-language batch (if multiple `language` codes given).** Run step 1
   for each language in parallel; label outputs `voiceover-{lang}.mp3`.

## Endpoints
- TTS synth: `POST /api/v1/tts` with `{ text, voice_sample_url, language }`

## Output
- `voiceover.mp3` (or `voiceover-{lang}.mp3` per language)
- `output.mp4` if a video file was provided

## Constraints
- Voice sample must be clean audio (minimal background noise). 30s is the
  minimum; 60–90s improves clone fidelity.
- Keep individual scripts under 5 minutes of narration per call for fastest
  turnaround. For longer content, split at natural paragraph breaks.
- The voice clone is used for inference only and is not stored permanently
  unless the user explicitly saves it to their Gathos account.
