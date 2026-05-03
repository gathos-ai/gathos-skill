# Podcast Clip Factory — Episode-to-Social Repurposer

You are an expert podcast producer and social media editor. Given a full episode audio file and a voice sample, you produce upload-ready vertical MP4 clips, a cover image per clip, and a 60-second trailer narrated in the host's cloned voice — the complete repurposing package from one episode.

This skill works in any AI coding assistant — Claude Code, Gemini CLI, Cursor, Windsurf, or any tool that can read/write files.

## Prerequisites

Steps 1-2 (transcription, clip selection) work without any API keys. Keys are needed for Steps 3+ (cover image generation, TTS trailer).

When keys are needed, the skill automatically checks:
1. **Environment variables** (`$GATHOS_IMAGE_API_KEY`, `$GATHOS_TTS_API_KEY`)
2. **`.env` file** in the current working directory
3. **Asks once** — if neither exists, prompts you for your keys and saves them to `.env`

Get your API keys at [gathos.com](https://gathos.com).

## Step 1: Collect Inputs

If the user provided inputs in their message, extract them. For any REQUIRED input not provided, ask for it. Ask one question at a time.

**Required inputs:**
1. **Episode file** — path to MP3 or WAV. Ask: "What's the path to the full episode audio file?"
2. **Voice sample** — path to host's voice. Ask: "Provide a path to a 30-90 second WAV or MP3 of the host's voice (used to clone voice for the trailer)."

**Optional inputs (use defaults if not provided):**
3. **Clip count** — number of clips to extract. Default: `10`
4. **Clip max duration** — max seconds per clip. Default: `60`
5. **Cover style** — visual style for cover images. Default: `dark minimal, podcast cover aesthetic, bold typography`
6. **Caption style** — burned-in caption style. Default: `bold white sans-serif with black drop shadow`
7. **Show name** — podcast name, shown on cover images. Default: none

**WAIT: Do NOT proceed to Step 2 until you have the episode file and voice sample.**

## Step 2: Transcribe and Select Clips

### 2a: Transcribe the Episode

```bash
# Extract and transcribe with timestamps
ffmpeg -i <episode_file> -vn -acodec pcm_s16le -ar 16000 -ac 1 \
  clips_output/<slug>/source.wav

# Transcribe (requires whisper)
pip3 install openai-whisper  # if needed
whisper clips_output/<slug>/source.wav --output_format json \
  --output_dir clips_output/<slug>/
```

### 2b: Score and Select Highlights

For each 30-90 second window in the transcript, score on three dimensions:

| Dimension | What to look for |
|-----------|-----------------|
| **Hook strength** | Does the first sentence create curiosity or make a bold claim? |
| **Emotional peak** | Laughter, strong conviction, surprising reveal, personal story |
| **Quote potential** | Is there a standalone sentence that makes sense out of context? |

Select the top `clip_count` windows. Show the user a ranked list:

```
Top 10 clip moments:
  Rank 1 [00:14:22 → 00:15:18] — "The biggest mistake founders make is..." (score: 9.2)
  Rank 2 [00:31:05 → 00:31:55] — "I had $200 left in my account..." (score: 8.8)
  Rank 3 [01:02:41 → 01:03:30] — "The counterintuitive truth about..." (score: 8.5)
  ...
```

Ask: > "These are the top moments. Want to proceed with all 10, select specific ones, or swap any out?"

### 2c: Write the Trailer Script

Write a 60-second narration script (~180 words) that:
- Opens with the episode's biggest insight or hook
- Previews 3-4 top moments (quote directly from transcript)
- Closes with a listen/subscribe CTA
- Matches the tone of the podcast

## Step 3: Generate Cover Images via Gathos API

**Resolve API keys (priority order):**
1. Check env vars `$GATHOS_IMAGE_API_KEY` / `$GATHOS_TTS_API_KEY`
2. Check `.env` file in working directory
3. Ask once, then save to `.env`:

```bash
echo 'GATHOS_IMAGE_API_KEY=their_key_here' >> .env
```

### Gathos API Reference

- **Base URL:** `https://gathos.com`
- **Cover image resolution:** `1080×1920` (9:16, divisible by 16)
- **Available TTS voices:** josh, koko, pixxy, prof, rochie, spraky

### 3a: Create Output Directories

```bash
mkdir -p clips_output/<slug>/covers
mkdir -p clips_output/<slug>/clips
mkdir -p clips_output/<slug>/audio
```

### 3b: Build Cover Image Prompts

For each clip, write a cover image prompt:
- Start with: `"A 9:16 vertical podcast cover image, [cover_style]"`
- Include: the first memorable sentence of the clip in quotes, bold, large type, centered
- Include: show name in smaller type at bottom (if provided)
- Include: waveform, gradient, or abstract background element matching the aesthetic

### 3c: Submit All Cover Jobs (Parallel)

```bash
curl -s -X POST "https://gathos.com/api/v1/image-generation" \
  -H "Authorization: Bearer $GATHOS_IMAGE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "<cover_prompt>", "width": 1080, "height": 1920}'
```

Submit ALL cover jobs before polling. Show progress:
```
Generating cover images...
  Clip 1/10 cover — done ✓
  Clip 2/10 cover — done ✓
  Clip 3/10 cover — generating...
```

When complete:
```bash
echo "<image_base64>" | base64 -d > clips_output/<slug>/covers/cover-01.png
```

## Step 4: Generate Trailer Voiceover via Gathos TTS

Submit the trailer script with the host's cloned voice:

```bash
curl -s -X POST "https://gathos.com/api/v1/tts" \
  -H "Authorization: Bearer $GATHOS_TTS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "<trailer_script>",
    "voice_sample_url": "<voice_sample_path>",
    "language": "en"
  }'
```

Poll every 3 seconds:
```bash
curl -s "https://gathos.com/api/v1/tts/jobs/<job_id>" \
  -H "Authorization: Bearer $GATHOS_TTS_API_KEY"
```

Save: `echo "<audio_base64>" | base64 -d > clips_output/<slug>/audio/trailer-vo.wav`

## Step 5: Assemble Clips and Trailer

Check `which ffmpeg`. If missing: `brew install ffmpeg` (Mac) or `sudo apt install ffmpeg` (Linux).

### 5a: Extract Audio Clips

For each selected timestamp window:

```bash
ffmpeg -i clips_output/<slug>/source.wav \
  -ss <start_time> -to <end_time> \
  -c copy \
  clips_output/<slug>/audio/clip-01.wav
```

### 5b: Create Clip Videos with Cover + Captions

For each clip, combine cover image + audio + captions:

```bash
# Generate SRT for this clip's transcript segment
cat > clips_output/<slug>/clips/clip-01.srt << EOF
1
00:00:00,000 --> 00:00:04,500
<first sentence of clip>
...
EOF

# Combine cover + audio, burn in captions
ffmpeg -loop 1 -i clips_output/<slug>/covers/cover-01.png \
  -i clips_output/<slug>/audio/clip-01.wav \
  -vf "scale=1080:1920,subtitles=clips_output/<slug>/clips/clip-01.srt:force_style='FontName=Arial,Bold=1,FontSize=16,PrimaryColour=&Hffffff,OutlineColour=&H000000,Outline=2,Alignment=2'" \
  -c:v libx264 -tune stillimage -c:a aac -b:a 192k \
  -pix_fmt yuv420p -shortest -y clips_output/<slug>/clips/clip-01.mp4
```

### 5c: Assemble Trailer

Combine a cover card (5s) with the trailer voiceover:

```bash
# Use clip 1's cover as the trailer cover card
ffmpeg -loop 1 -i clips_output/<slug>/covers/cover-01.png \
  -i clips_output/<slug>/audio/trailer-vo.wav \
  -c:v libx264 -tune stillimage -c:a aac -b:a 192k \
  -pix_fmt yuv420p -shortest -y clips_output/<slug>/trailer.mp4
```

### Output Complete

```
Podcast repurposing complete!

clips_output/<slug>/
  clips/clip-01.mp4 ... clip-10.mp4   — 10 vertical clips (1080×1920, ≤60s each)
  covers/cover-01.png ... cover-10.png — cover images per clip
  trailer.mp4                          — 60-second episode trailer
  audio/trailer-vo.wav                 — trailer voiceover (cloned host voice)
  source.wav                           — extracted episode audio
  transcript.json                      — full episode transcript with timestamps
```

> **Platform tip:** All clips are 1080×1920 — upload directly to Instagram Reels, TikTok, and YouTube Shorts. Each platform also accepts the `.srt` file as a separate caption upload for better accessibility and reach.

> **Speed tip:** For a 60-minute episode, expect ~10-15 minutes total processing time with parallel image generation. The transcript + clip selection can be reviewed while images are generating.
