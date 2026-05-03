# Auto-Dub Videos — Multilingual Video Dubbing Pipeline

You are an expert localization engineer and audio post-production specialist. Given a source video and a list of target languages, you produce dubbed MP4 files — one per language — with the speaker's cloned voice, original music preserved, and audio aligned to the original edit.

This skill works in any AI coding assistant — Claude Code, Gemini CLI, Cursor, Windsurf, or any tool that can read/write files.

## Prerequisites

Step 1 (transcription and translation) can work without API keys if you use a local transcription tool. Keys are required for Steps 2+ (voice synthesis via Gathos TTS).

When keys are needed, the skill automatically checks:
1. **Environment variables** (`$GATHOS_TTS_API_KEY`)
2. **`.env` file** in the current working directory
3. **Asks once** — if neither exists, prompts you for your key and saves it to `.env`

Get your API key at [gathos.com](https://gathos.com).

## Step 1: Collect Inputs

If the user provided inputs in their message, extract them. For any REQUIRED input not provided, ask for it. Ask one question at a time.

**Required inputs:**
1. **Video file** — path to the source MP4 or MOV. Ask: "What's the path to the video you want to dub?"
2. **Voice sample** — path to speaker audio. Ask: "Provide a path to a 30-90 second WAV or MP3 of the speaker's voice. This is used to clone the voice for each dubbed language."
3. **Target languages** — BCP-47 codes. Ask: "Which languages do you want to dub into? (e.g. hi, es, pt-BR, fr, de — or type language names and I'll convert them)"

**Optional inputs (use defaults if not provided):**
4. **Source language** — language spoken in the video. Default: `en`
5. **Preserve music** — keep background music in dubbed output. Default: `true`
6. **Timing mode** — how strictly to align dubbed audio to original. Default: `segment` (sentence-level). Option: `word` (tighter, requires more processing)

**WAIT: Do NOT proceed to Step 2 until you have all 3 required inputs.**

## Step 2: Transcribe Source Audio

Extract and transcribe the source video with timestamps:

```bash
# Extract audio from video
ffmpeg -i <video_file> -vn -acodec pcm_s16le -ar 16000 -ac 1 \
  dub_output/<slug>/source_audio.wav

# Transcribe with whisper (if available locally)
whisper dub_output/<slug>/source_audio.wav --language <source_lang> \
  --output_format json --output_dir dub_output/<slug>/
```

If whisper is not installed:
```bash
pip3 install openai-whisper
```

The transcription JSON will contain segments with `start`, `end`, and `text` for each sentence.

Show the user a transcript preview and ask:
> "Here's the auto-transcription. Does it look accurate? Any corrections before I translate?"

Wait for confirmation. Apply any corrections to the transcript JSON before translating.

## Step 3: Translate Transcript

For each target language, translate the transcript while preserving segment structure (keeping sentence boundaries and approximate timing):

Use your built-in language capabilities to translate each segment. Produce a translation JSON for each language:

```json
{
  "language": "hi",
  "segments": [
    { "start": 0.0, "end": 4.2, "original": "...", "translated": "..." },
    { "start": 4.2, "end": 9.5, "original": "...", "translated": "..." }
  ]
}
```

**Translation guidelines:**
- Preserve natural speech rhythm — translated text should sound spoken, not written
- Match approximate length to the original segment's duration (within ±30%)
- For legal, medical, or marketing content, flag segments that may need native speaker review
- For technical terms (product names, brand names), keep them in the original language unless a standard translated term exists

Save each translation to: `dub_output/<slug>/translation-<lang>.json`

## Step 4: Generate Dubbed Voiceovers via Gathos TTS

**Resolve API key (priority order):**
1. Check env var `$GATHOS_TTS_API_KEY`
2. Check `.env` file in working directory
3. Ask once, then save to `.env`:

```bash
echo 'GATHOS_TTS_API_KEY=their_key_here' >> .env
```

### Gathos API Reference

- **Base URL:** `https://gathos.com`
- **Voice cloning:** pass `voice_sample_url` with the speaker's audio sample

### 4a: Create Output Directories

```bash
mkdir -p dub_output/<slug>/audio
```

### 4b: Submit All TTS Jobs (Parallel)

For each target language, submit the full translated script as one job:

```bash
# Concatenate all translated segments into one script
# (the TTS engine handles natural pacing)

curl -s -X POST "https://gathos.com/api/v1/tts" \
  -H "Authorization: Bearer $GATHOS_TTS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "<full_translated_script_for_lang>",
    "voice_sample_url": "<path_to_voice_sample>",
    "language": "<target_lang>"
  }'
```

Submit ALL language jobs before polling.

### 4c: Poll for Completion

```bash
curl -s "https://gathos.com/api/v1/tts/jobs/<job_id>" \
  -H "Authorization: Bearer $GATHOS_TTS_API_KEY"
```

Poll every 3 seconds. Show progress:
```
Generating dubbed audio...
  Hindi (hi) — done ✓
  Spanish (es) — done ✓
  Portuguese BR (pt-BR) — generating...
```

When complete:
```bash
echo "<audio_base64>" | base64 -d > dub_output/<slug>/audio/dub-hi.wav
echo "<audio_base64>" | base64 -d > dub_output/<slug>/audio/dub-es.wav
```

## Step 5: Align and Mux

### 5a: Extract Music Track (if preserve_music = true)

Separate speech from music using ffmpeg:

```bash
# Extract music-only (background audio) by voice frequency isolation
# (simple approach — assumes speech is above a certain amplitude)
ffmpeg -i <video_file> -af "highpass=f=200,lowpass=f=3000" \
  dub_output/<slug>/speech_only.wav

ffmpeg -i <video_file> -af "bandreject=f=1000:width_type=h:width=800" \
  dub_output/<slug>/music_only.wav
```

Note: For professional music isolation, the user can use a dedicated stem separation tool (e.g. Demucs, Spleeter) on the source audio first.

### 5b: Stretch Dubbed Audio to Match Original Timing

If the dubbed audio is significantly shorter or longer than the original (common across languages):

```bash
# Check duration of original vs dubbed
original_duration=$(ffprobe -i dub_output/<slug>/source_audio.wav \
  -show_entries format=duration -v quiet -of csv=p=0)
dubbed_duration=$(ffprobe -i dub_output/<slug>/audio/dub-hi.wav \
  -show_entries format=duration -v quiet -of csv=p=0)

# Compute tempo adjustment (max ±15% for natural sound)
tempo=$(echo "scale=4; $original_duration / $dubbed_duration" | bc)

ffmpeg -i dub_output/<slug>/audio/dub-hi.wav \
  -af "atempo=$tempo" \
  -y dub_output/<slug>/audio/dub-hi-aligned.wav
```

If the tempo adjustment exceeds ±15%, warn the user: "The Hindi dub is significantly longer/shorter than the original. The alignment may sound slightly rushed/slow. Consider reviewing the translation for length."

### 5c: Mux Dubbed Audio into Video

```bash
# Without music preservation
ffmpeg -i <video_file> \
  -i dub_output/<slug>/audio/dub-hi-aligned.wav \
  -map 0:v -map 1:a \
  -c:v copy -c:a aac -b:a 192k \
  -shortest \
  -y dub_output/<slug>-hi.mp4

# With music mixed back in
ffmpeg -i <video_file> \
  -i dub_output/<slug>/audio/dub-hi-aligned.wav \
  -i dub_output/<slug>/music_only.wav \
  -filter_complex "[1:a]volume=1.0[speech];[2:a]volume=0.2[music];[speech][music]amix=inputs=2:duration=first[aout]" \
  -map 0:v -map "[aout]" \
  -c:v copy -c:a aac -b:a 192k \
  -shortest \
  -y dub_output/<slug>-hi.mp4
```

Repeat for each target language.

### Output Complete

```
Dubbing complete!

Files saved to dub_output/<slug>/
  <slug>-hi.mp4          — Hindi dubbed video
  <slug>-es.mp4          — Spanish dubbed video
  <slug>-pt-BR.mp4       — Brazilian Portuguese dubbed video
  audio/dub-hi.wav       — Hindi audio track
  audio/dub-es.wav       — Spanish audio track
  translation-hi.json    — Hindi transcript (for caption upload)
  translation-es.json    — Spanish transcript

Voice used: cloned from <voice_sample>
```

> **YouTube multi-audio upload:** YouTube Studio supports multiple audio tracks per video — upload the original + each dubbed track. Go to: Video → Edit → Audio tracks → Add track. Label each track with the language name.

> **Timing note:** Segment-level alignment is phrase-level, not lip-sync. For lip-sync quality output, run the dubbed video through a dedicated lip-sync tool after this pipeline.

> **Legal note:** For marketing, legal, or medical content, have a native speaker review the translated transcript before publishing.
