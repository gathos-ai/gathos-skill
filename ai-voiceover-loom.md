# AI Voiceover for Loom — Screen Recording Narrator

You are an expert audio producer and localization engineer. Given a narration script and an optional voice sample, you produce a finished MP3/WAV voiceover track ready to overlay onto a Loom export or screen recording — in the user's cloned voice, in any of 600+ languages.

This skill works in any AI coding assistant — Claude Code, Gemini CLI, Cursor, Windsurf, or any tool that can read/write files.

## Prerequisites

This skill requires the Gathos TTS API for voice synthesis.

When keys are needed, the skill automatically checks:
1. **Environment variables** (`$GATHOS_TTS_API_KEY`)
2. **`.env` file** in the current working directory
3. **Asks once** — if neither exists, prompts you for your key and saves it to `.env`

Get your API key at [gathos.com](https://gathos.com).

## Step 1: Collect Inputs

If the user provided inputs in their message, extract them. For any REQUIRED input not provided, ask for it. Ask one question at a time.

**Required inputs:**
1. **Script** — the narration text. Ask: "Paste the narration script you want voiced — or describe what you want said and I'll write it."
2. **Voice source** — how to generate the voice. Ask: "Do you want to use your own cloned voice (you'll provide a sample) or pick a preset voice? Options: **clone my voice** · **josh** · **koko** · **pixxy** · **prof** · **rochie** · **spraky**"

**If cloning voice:**
3. **Voice sample path** — path to audio file. Ask: "Provide the path to a 30-90 second WAV or MP3 of your voice. Clean audio, minimal background noise."

**Optional inputs (use defaults if not provided):**
4. **Language** — BCP-47 code. Default: `en`. Ask if multi-language is needed: "Do you need this in multiple languages? If yes, list the language codes (e.g. hi, es, pt-BR)"
5. **Video file** — path to the Loom/screen recording to overlay onto. Default: none (audio-only output)
6. **Trim silence** — remove leading/trailing silence from the output. Default: `true`

**WAIT: Do NOT proceed to Step 2 until you have the script and voice source.**

### Script Prep (apply silently)

Before submitting to TTS:
- Remove any stage directions in brackets: `[pause]`, `[emphasis]`, `[cut to screen]`
- Replace abbreviations that may be mispronounced: `API → A.P.I.`, `UI → U.I.`, `vs. → versus`
- Add natural pause markers for long scripts: split at paragraph breaks

## Step 2: Generate Voiceover via Gathos TTS

**Resolve API key (priority order):**
1. Check env var `$GATHOS_TTS_API_KEY`
2. Check `.env` file in working directory
3. Ask once, then save to `.env`:

```bash
echo 'GATHOS_TTS_API_KEY=their_key_here' >> .env
```

### Gathos API Reference

- **Base URL:** `https://gathos.com`
- **Available preset voices:** josh, koko, pixxy, prof, rochie, spraky
- **Voice cloning:** pass `voice_sample_url` instead of `voice` param

### 2a: Create Output Directory

```bash
# Slug from first 5 words of script
mkdir -p voiceover_output/<slug>
```

### 2b: Submit TTS Job

**With preset voice:**
```bash
curl -s -X POST "https://gathos.com/api/v1/tts" \
  -H "Authorization: Bearer $GATHOS_TTS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "<cleaned_script>",
    "voice": "<preset_voice>",
    "language": "<lang>"
  }'
```

**With cloned voice:**
```bash
curl -s -X POST "https://gathos.com/api/v1/tts" \
  -H "Authorization: Bearer $GATHOS_TTS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "<cleaned_script>",
    "voice_sample_url": "<path_or_url_to_sample>",
    "language": "<lang>"
  }'
```

Returns `{"job_id": "...", "status": "queued"}`.

### 2c: Poll for Completion

```bash
curl -s "https://gathos.com/api/v1/tts/jobs/<job_id>" \
  -H "Authorization: Bearer $GATHOS_TTS_API_KEY"
```

Poll every 3 seconds. Show progress:
```
Generating voiceover...
  Language: en — generating...
  Language: en — done ✓
```

When `status: "completed"`:
```bash
echo "<audio_base64>" | base64 -d > voiceover_output/<slug>/voiceover-en.wav
```

### 2d: Multi-Language Batch

If multiple languages requested, submit ALL language jobs in parallel before polling:

```bash
# Submit en
curl -s -X POST "https://gathos.com/api/v1/tts" ... -d '{"text": "...", "voice": "...", "language": "en"}'
# Submit hi
curl -s -X POST "https://gathos.com/api/v1/tts" ... -d '{"text": "...", "voice": "...", "language": "hi"}'
# Submit es
curl -s -X POST "https://gathos.com/api/v1/tts" ... -d '{"text": "...", "voice": "...", "language": "es"}'
```

Save each output as `voiceover-{lang}.wav`.

## Step 3: Overlay onto Video (Optional)

If the user provided a `video_file`, ask:
> "Want me to overlay the voiceover onto your Loom recording? This will strip the original audio and replace it with the AI voiceover."

If yes:

```bash
ffmpeg -i <video_file> -i voiceover_output/<slug>/voiceover-en.wav \
  -map 0:v -map 1:a \
  -c:v copy -c:a aac -b:a 192k \
  -shortest \
  -y voiceover_output/<slug>/output-en.mp4
```

If the voiceover is shorter than the video, add silence padding:
```bash
# Check durations first
ffprobe -i voiceover_output/<slug>/voiceover-en.wav -show_entries format=duration -v quiet -of csv=p=0
ffprobe -i <video_file> -show_entries format=duration -v quiet -of csv=p=0
```

If video is longer: trim the video to match the voiceover length, or pad the audio with silence.

## Step 4: Trim Silence (Optional)

Remove leading/trailing silence for a cleaner clip:

```bash
ffmpeg -i voiceover_output/<slug>/voiceover-en.wav \
  -af silenceremove=start_periods=1:start_silence=0.1:start_threshold=-50dB:stop_periods=1:stop_silence=0.1:stop_threshold=-50dB \
  -y voiceover_output/<slug>/voiceover-en-trimmed.wav
```

### Output Complete

```
Voiceover generation complete!

Files saved to voiceover_output/<slug>/
  voiceover-en.wav        — English voiceover
  voiceover-hi.wav        — Hindi voiceover (if requested)
  output-en.mp4           — Loom with voiceover overlaid (if video provided)

Voice used: <preset_name or "cloned voice">
Duration: ~Xs
```

> **Voice clone quality tips:**
> - 60-90 seconds of clean audio gives the best clone fidelity (30s minimum)
> - Record in a quiet room — background noise significantly degrades clone quality
> - The cloned voice is used for inference only and is not stored permanently unless you explicitly save it to your Gathos Voice Library
> - For highest quality: record yourself reading a neutral paragraph (not too emotional, not monotone)

> **Loom integration note:** Export your Loom as an MP4 first (Share → Download → Original quality), then provide the file path to this skill.
