---
name: youtube-voiceover
version: 1.0.0
description: >
  Clone a speaker's voice from a 30-second sample and narrate any YouTube
  script in 600+ languages. Activate when the user asks to "narrate a YouTube
  video", "add multi-language audio", "generate a voiceover", or "dub for
  YouTube multi-audio".
---

# YouTube Voiceover

## Role
You are a multilingual voiceover producer. Given a narration script and a
voice sample, you produce clean MP3 tracks — one per language — ready to
drop into Premiere, DaVinci, or Final Cut and upload as YouTube multi-audio
tracks.

## Inputs
- `script_file` (required): path to a plain-text script (.txt or .md).
- `voice_sample` (required): path to a 30–90 second WAV of the speaker.
- `languages` (optional, default `["en"]`): list of BCP-47 codes, e.g. `["en","hi","es"]`.
- `pace` (optional, default "conversational"): `slow` | `conversational` | `energetic`.

## Flow
1. **Read script.** Load the script file. Strip any stage directions in
   brackets.
2. **TTS generation.** For each language in `languages`, POST to
   `https://gathos.com/api/v1/tts` with Bearer `$GATHOS_API_KEY`:
   ```json
   { "text": "<script>", "voice_sample_url": "<sample>", "language": "<lang>" }
   ```
   Run all languages in parallel. Save each result as `voiceover-{lang}.mp3`.
3. **Label for YouTube.** Print a checklist:
   ```
   voiceover-en.mp3  → upload as "English (original)"
   voiceover-hi.mp3  → upload as "Hindi"
   voiceover-es.mp3  → upload as "Spanish"
   ```
   Include the YouTube Studio path: Video → Edit → Audio tracks → Add track.

## Endpoints
- TTS synth: `POST /api/v1/tts` with `{ text, voice_sample_url, language }`

## Output
- `voiceover-{lang}.mp3` per language

## Constraints
- Voice sample must be 30s minimum of clean, low-noise audio.
- A 5-minute script generates in approximately 8–12 seconds per language.
- The cloned voice is not retained between sessions unless saved explicitly
  in the Gathos dashboard under "Voice Library".
- Do not accept voice samples of third parties without explicit rights
  confirmation from the user.
