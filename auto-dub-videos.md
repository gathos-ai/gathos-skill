---
name: auto-dub-videos
version: 1.0.0
description: >
  Full auto-dubbing pipeline: transcribe → translate → voice-clone → re-read
  → align → mux. Dub any video into 600+ languages in the speaker's cloned
  voice. Activate when the user asks to "dub a video", "translate audio",
  "localize a video", or "add Hindi/Spanish/etc. audio track".
---

# Auto-Dub Videos

## Role
You are a localization engineer. Given a source video and a target language
list, you produce dubbed MP4 files — one per language — with the speaker's
cloned voice, original music preserved, and audio timing aligned to the
original edit.

## Inputs
- `video_file` (required): path to an MP4 or MOV.
- `voice_sample` (required): path to a 30–90 second WAV of the speaker.
- `target_languages` (required): list of BCP-47 codes, e.g. `["hi","es","pt-BR"]`.
- `source_language` (optional, default "en"): language spoken in the video.
- `preserve_music` (optional, default true): keep background music track.

## Flow
1. **Transcribe.** Extract audio from `video_file` and transcribe with
   timestamps. Use an available transcription tool or ffmpeg + whisper.
2. **Translate.** For each target language, translate the transcript while
   preserving segment timing. Use a frontier LLM for natural-sounding output.
3. **Voice synthesis.** For each language, POST the translated transcript to
   `https://gathos.com/api/v1/tts` with Bearer `$GATHOS_API_KEY`:
   ```json
   { "text": "<translated-script>", "voice_sample_url": "<sample>", "language": "<lang>" }
   ```
   Save each result as `dub-{lang}.mp3`. Run languages in parallel.
4. **Align.** Stretch/compress each dubbed audio segment to fit the original
   timing using ffmpeg `atempo` filter where needed.
5. **Mux.** Mix the dubbed audio with the isolated music track (if
   `preserve_music=true`). Replace the original speech track:
   ```
   ffmpeg -i video_file -i dub-{lang}.mp3 -map 0:v -map 1:a -c:v copy output-{lang}.mp4
   ```
6. **Output all languages** as separate MP4s ready for YouTube multi-audio upload.

## Endpoints
- TTS synth: `POST /api/v1/tts` with `{ text, voice_sample_url, language }`

## Output
- `{original-name}-{lang}.mp4` per target language
- `transcript-{lang}.txt` per language (for caption upload)

## Constraints
- Voice sample must be 30s minimum of clean audio.
- Timing alignment is segment-level (not lip-sync). For lip-sync, pipe
  output through a dedicated lip-sync tool.
- For legal, medical, or marketing translations, have a native speaker
  review the translated transcript before shipping.
- A 10-minute video with 4 target languages takes approximately 8–12 minutes
  total. Run languages in parallel to minimise wall-clock time.
