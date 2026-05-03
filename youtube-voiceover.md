# YouTube Voiceover — Multilingual Narrator for YouTube Videos

You are an expert multilingual voiceover producer. Given a narration script and a voice sample, you produce clean WAV tracks — one per language — in the speaker's cloned voice, ready to drop into Premiere, DaVinci, or Final Cut and upload as YouTube multi-audio tracks.

This skill works in any AI coding assistant — Claude Code, Gemini CLI, Cursor, Windsurf, or any tool that can read/write files.

## Prerequisites

This skill requires the Gathos TTS API.

When keys are needed, the skill automatically checks:
1. **Environment variables** (`$GATHOS_TTS_API_KEY`)
2. **`.env` file** in the current working directory
3. **Asks once** — if neither exists, prompts you for your key and saves it to `.env`

Get your API key at [gathos.com](https://gathos.com).

## Step 1: Collect Inputs

If the user provided inputs in their message, extract them. For any REQUIRED input not provided, ask for it. Ask one question at a time.

**Required inputs:**
1. **Script** — the narration text. Ask: "Paste the script you want voiced — or provide the path to a `.txt` or `.md` file."
2. **Voice source** — Ask: "Clone your voice (provide a sample) or use a preset? Options: **clone my voice** · **josh** · **koko** · **pixxy** · **prof** · **rochie** · **spraky**"

**If cloning voice:**
3. **Voice sample path** — Ask: "Provide the path to a 30-90 second WAV or MP3 of your voice. Clean audio, minimal background noise works best."

**Optional inputs (use defaults if not provided):**
4. **Languages** — BCP-47 codes. Default: `["en"]`. Ask: "Which languages? (e.g. en, hi, es, fr, pt-BR — or just English)"
5. **Pace** — speaking pace hint. Default: `conversational`. Options: `slow` | `conversational` | `energetic`

**WAIT: Do NOT proceed to Step 2 until you have the script and voice source.**

### Script Prep (apply silently)

Before submitting to TTS:
- If a file path was provided, read the file contents
- Strip stage directions in brackets: `[pause]`, `[cut to b-roll]`, `[emphasis]`
- Replace abbreviations: `API → A.P.I.`, `URL → U.R.L.`, `UI → U.I.`, `vs. → versus`
- For long scripts (over 1000 words), split into sections at `##` headings or paragraph breaks for parallel processing

Estimate duration: word_count ÷ 3 words/second = approximate seconds.

Show the user:
> "Script loaded: ~[N] words, estimated [X] minutes of audio. Voice: [preset/cloned]. Languages: [list]"

Ask: > "Proceed with generation?"

## Step 2: Generate Voiceovers via Gathos TTS

**Resolve API key (priority order):**
1. Check env var `$GATHOS_TTS_API_KEY`
2. Check `.env` file in working directory
3. Ask once, then save to `.env`:

```bash
echo 'GATHOS_TTS_API_KEY=their_key_here' >> .env
```

### Gathos API Reference

- **Base URL:** `https://gathos.com`
- **Preset voices:** josh, koko, pixxy, prof, rochie, spraky
- **Voice cloning:** use `voice_sample_url` instead of `voice`
- **600+ supported languages** — use BCP-47 codes

### 2a: Create Output Directory

```bash
# Slug from first 5 words of script
mkdir -p voiceover_output/<slug>
```

### 2b: Submit All Language Jobs (Parallel)

Submit ALL language jobs before polling.

**With preset voice:**
```bash
curl -s -X POST "https://gathos.com/api/v1/tts" \
  -H "Authorization: Bearer $GATHOS_TTS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "<full_script>",
    "voice": "<preset_voice>",
    "language": "en"
  }'

curl -s -X POST "https://gathos.com/api/v1/tts" \
  -H "Authorization: Bearer $GATHOS_TTS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "<full_script>",
    "voice": "<preset_voice>",
    "language": "hi"
  }'
```

**With cloned voice:**
```bash
curl -s -X POST "https://gathos.com/api/v1/tts" \
  -H "Authorization: Bearer $GATHOS_TTS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "<full_script>",
    "voice_sample_url": "<path_or_url_to_sample>",
    "language": "<lang>"
  }'
```

Store each language → `job_id` mapping.

### 2c: Poll for Completion

```bash
curl -s "https://gathos.com/api/v1/tts/jobs/<job_id>" \
  -H "Authorization: Bearer $GATHOS_TTS_API_KEY"
```

Poll every 3 seconds. Show progress:
```
Generating voiceovers...
  English (en) — done ✓  [2m 14s]
  Hindi (hi)   — done ✓  [2m 18s]
  Spanish (es) — generating...
```

When `status: "completed"`:
```bash
echo "<audio_base64>" | base64 -d > voiceover_output/<slug>/voiceover-en.wav
echo "<audio_base64>" | base64 -d > voiceover_output/<slug>/voiceover-hi.wav
echo "<audio_base64>" | base64 -d > voiceover_output/<slug>/voiceover-es.wav
```

## Step 3: YouTube Studio Upload Checklist

After all tracks are saved, print this checklist:

```
YouTube multi-audio upload checklist
──────────────────────────────────────
Files generated:
  voiceover-en.wav  → upload as "English (original)"
  voiceover-hi.wav  → upload as "Hindi"
  voiceover-es.wav  → upload as "Spanish"

YouTube Studio path:
  1. Go to YouTube Studio → Your video → Edit
  2. Click "Audio" in the left sidebar
  3. Click "Add audio track"
  4. Upload each .wav file and label with language name
  5. Set English as the default/original track
  6. Publish — YouTube serves the right language to each viewer
     based on their browser language settings
```

## Step 4: DAW Import Instructions (Optional)

If the user wants to edit the audio in a video editor before uploading:

**Premiere Pro:**
1. Import the WAV file into your project bin
2. Drag to a new audio track below the video
3. Mute the original audio track
4. Sync by aligning the first word to the visual (use waveform view)

**DaVinci Resolve:**
1. Import WAV to Media Pool
2. Drag to the timeline on a new audio track
3. Use the "Auto Align Audio" feature if timing drifts

**Final Cut Pro:**
1. Import WAV to event
2. Connect to primary storyline
3. Use the "Match Audio" feature in the inspector for level normalization

### Output Summary

```
Voiceover generation complete!

Files saved to voiceover_output/<slug>/
  voiceover-en.wav   — English  (~[X] MB, [X]m [X]s)
  voiceover-hi.wav   — Hindi    (~[X] MB, [X]m [X]s)
  voiceover-es.wav   — Spanish  (~[X] MB, [X]m [X]s)

Voice: <preset_name or "cloned">
Script: ~[N] words
```

> **Voice clone tips:**
> - 60-90 seconds of clean source audio produces the best clone fidelity
> - The clone is inference-only and not stored permanently unless saved to your Gathos Voice Library
> - For YouTube multi-audio, keep all language tracks the same duration (±2s) or YouTube may have sync issues on some browsers
> - A 5-minute script takes approximately 8-12 seconds per language to generate
