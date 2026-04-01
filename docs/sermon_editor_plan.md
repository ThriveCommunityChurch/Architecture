# 🎬 Sermon Video Editor Agent — Technical Plan

> **Project**: `Sermon_Video_Editor`
> **Location**: `C:\Users\wyatt\Documents\GitHub\Thrive\Sermon_Video_Editor\`
> **Pattern**: LangGraph pipeline — mirrors `Sermon_Summarization_Agent` and `Sermon-Shorts`
> **Runtime**: Local Windows machine with NVIDIA GPU
> **Author**: Thrive Community Church Media Team

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Source Material](#2-source-material)
3. [Architecture Overview](#3-architecture-overview)
4. [Pipeline Stages](#4-pipeline-stages)
5. [Project Structure](#5-project-structure)
6. [Agent State](#6-agent-state)
7. [Node Specifications](#7-node-specifications)
8. [FFmpeg Filter Chains](#8-ffmpeg-filter-chains)
9. [Sermon Boundary Detection — Prompt Engineering](#9-sermon-boundary-detection--prompt-engineering)
10. [Configuration & Environment](#10-configuration--environment)
11. [CLI Interface](#11-cli-interface)
12. [Error Handling & Recovery](#12-error-handling--recovery)
13. [Dependencies](#13-dependencies)
14. [Development Phases](#14-development-phases)
15. [Verification & Definition of Done](#15-verification--definition-of-done)

---

## 1. Problem Statement

| Metric | Current | Target |
|--------|---------|--------|
| Weekly editing time | ~1 hour | < 5 minutes (review only) |
| Monthly editing time (graphics change) | ~3 hours | ~30 minutes (graphics only) |
| Manual steps eliminated | 0 | Sync, trim, color, audio, render |
| Backlog | 2-3 sermons | 0 |

**The agent automates everything except creative decisions** (new graphics, special one-off edits). The human reviews the agent's output and swaps in updated graphics when needed.

---

## 2. Source Material

### 2.1 Input Files

Every week, two video files are produced from the **same physical audio/video source** — they are split before processing:

| Property | 📹 Raw Recording | 📡 Broadcast Recording |
|----------|-------------------|------------------------|
| **Video** | ✅ **USE** — clean, no watermarks, no lower-thirds | ❌ Has overlaid watermarks & lower-third animations |
| **Audio** | ❌ Unprocessed soundboard @ ~-12 dB (safety/reference) | ✅ **USE** — Waves-mastered, -16 LUFS, peaks near -1 dB |
| **Start time** | Arbitrary — recording starts/stops independently | Arbitrary — different from Raw |
| **Timecode** | ❌ None | ❌ None |
| **Format** | `.mp4` or `.mov` with embedded audio | `.mp4` or `.mov` with embedded audio |

### 2.2 The Sync Problem

Both files originate from the same source but:
- Start and stop recording at **different times**
- Have **no timecode** embedded
- Have **different audio processing** (raw -12 dB vs. Waves-mastered -16 LUFS)

**Solution**: Audio cross-correlation. The waveform *shape* is identical despite gain/compression differences. Numpy cross-correlation will find the sample-accurate offset.

### 2.3 What the Agent Must Produce

1. **Synced composite**: Raw video (clean) + Broadcast audio (mastered)
2. **Trimmed to sermon**: Worship, announcements, and pre-sermon content removed
3. **Intro prepended**: "You are listening to a message from Thrive Community Church..." + graphics
4. **Color graded**: LUT + Lumetri-equivalent adjustments applied
5. **Audio finalized**: Light normalization on the already-mastered broadcast audio
6. **Rendered**: GPU-accelerated H.264 via NVENC

### 2.4 Training Data — 500 Transcripts

**Location**: `C:\Users\wyatt\Documents\GitHub\Thrive\AWSLambdas\scripts\.transcript_cache\`
**Count**: 500 JSON files
**Format**: Each file contains a `.transcript` field with the full text of a **finished, edited** sermon.

> ⚠️ **Critical Note**: These transcripts are from *post-production* — the "You are listening to..." intro was **added during editing** and is NOT in the raw footage. The agent must learn what sermon content looks like from the text *after* that intro, and use that knowledge to find the sermon start in the *raw, unedited* Whisper output.

**Observed sermon-start patterns** (from the 500 transcripts):

| Pattern | Example |
|---------|---------|
| Series introduction | "So we are in the second week of the Red Letter Challenge..." |
| Opening prayer | "Lord God, we thank you for this time of the year..." |
| Anecdote/hook | "I don't know if this has ever happened to you. Have you ever been going around the house..." |
| Scripture reading | "We're going to be reading in Jonah chapter 3..." |
| Guest speaker intro | "My name is Hunter Kessler. I am the Campus Minister..." |
| Direct topic statement | "This week, we're going to be focusing on forgiveness..." |

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Sermon Video Editor Agent                        │
│                    (LangGraph StateGraph)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐       │
│   │  Node 1  │──▶│  Node 2  │──▶│  Node 3  │──▶│  Node 4  │       │
│   │Transcribe│   │  Sync    │   │ Detect   │   │  Trim    │       │
│   │ (Whisper)│   │ (Numpy)  │   │Boundaries│   │ + Intro  │       │
│   └──────────┘   └──────────┘   │ (LLM)    │   └────┬─────┘       │
│                                  └──────────┘        │              │
│                                                      ▼              │
│                                  ┌──────────┐   ┌──────────┐       │
│                                  │  Node 6  │◀──│  Node 5  │       │
│                                  │  Render  │   │  Color + │       │
│                                  │ (NVENC)  │   │  Audio   │       │
│                                  └──────────┘   └──────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Execution model**: Deterministic linear — no LLM-driven routing. Each node runs in sequence, same as the `Sermon_Summarization_Agent`'s `process_single_file()` pattern:

```python
transcribe_node.invoke({})
sync_node.invoke({})
detect_boundaries_node.invoke({})
trim_and_intro_node.invoke({})
color_audio_node.invoke({})
render_node.invoke({})
```



---

## 4. Pipeline Stages — Detailed

### Stage 1: Transcribe Both Files (Whisper + CUDA)

**Input**: Two video files — Raw and Broadcast
**Output**: Two sets of timestamped transcript segments (JSON)
**Tool**: OpenAI Whisper `small.en` model, local GPU

```
For each input file:
  1. Extract audio via FFmpeg → PCM WAV (48kHz, mono, 16-bit)
  2. Load Whisper model (cached after first load — same as Summarization Agent)
  3. Transcribe with fp16=True on CUDA
  4. Write transcription_segments.json with start/end/text per segment
```

**Why transcribe both?** The Broadcast file has the Waves-processed audio which may transcribe more cleanly. But we transcribe the Raw file too because the LLM needs timestamps that align with the Raw video's timeline for trimming. After sync offset is known, we can map between the two.

**Whisper output format** (per segment):
```json
{
  "start": 125.4,
  "end": 131.2,
  "start_str": "02:05",
  "end_str": "02:11",
  "text": "So we are in the second week of the Red Letter Challenge."
}
```

### Stage 2: Audio Cross-Correlation Sync

**Input**: Extracted WAV from Raw + extracted WAV from Broadcast
**Output**: `sync_offset_seconds` (float) — how many seconds the Raw leads or lags the Broadcast
**Tool**: Numpy + Librosa

**Algorithm**:
```python
import numpy as np
import librosa

def find_sync_offset(raw_wav, broadcast_wav, sr=16000):
    # Load both files at same sample rate (downsample for speed)
    raw, _ = librosa.load(raw_wav, sr=sr, mono=True)
    bcast, _ = librosa.load(broadcast_wav, sr=sr, mono=True)

    # Use 60s chunk from middle of each file
    chunk = 60 * sr
    raw_c = raw[len(raw)//2 - chunk//2 : len(raw)//2 + chunk//2]
    bcast_c = bcast[len(bcast)//2 - chunk//2 : len(bcast)//2 + chunk//2]

    # Normalize
    raw_c = raw_c / (np.max(np.abs(raw_c)) + 1e-8)
    bcast_c = bcast_c / (np.max(np.abs(bcast_c)) + 1e-8)

    # Cross-correlate and find peak
    corr = np.correlate(raw_c, bcast_c, mode='full')
    peak = np.argmax(np.abs(corr))
    offset_samples = peak - (len(bcast_c) - 1)

    # Account for midpoint difference
    mid_offset = (len(raw)//2 - len(bcast)//2) / sr
    return (offset_samples / sr) + mid_offset
```

**Why this works**: Same source audio. Waves changes gain/dynamics but NOT timing. Correlation peak will be razor-sharp. Confidence check: peak value > 0.7 or warn.

### Stage 3: Sermon Boundary Detection (LLM)

**Input**: Timestamped transcript from Whisper
**Output**: `sermon_start_time` and `sermon_end_time` (seconds)
**Tool**: GPT-4o-mini via LangChain

The LLM reads the timestamped transcript and identifies:
1. **Sermon start** — transition from announcements/worship into sermon content
2. **Sermon end** — closing prayer, altar call, or final "amen"

See [Section 9](#9-sermon-boundary-detection--prompt-engineering) for full prompt design.

### Stage 4: Trim + Intro Prepend

**Input**: Synced composite (Raw video + Broadcast audio), sermon boundaries, intro assets
**Output**: Trimmed video file with intro prepended
**Tool**: FFmpeg

```
Steps:
  1. Map sermon_start/end from Broadcast timeline → Raw timeline via sync_offset
  2. FFmpeg inputs:
     - Input 0: Intro graphic/video (with "You are listening to..." audio)
     - Input 1: Raw video file     → extract video stream (v:0)
     - Input 2: Broadcast file     → extract audio stream (a:0)
  3. Trim video from Input 1 and audio from Input 2 to sermon boundaries
  4. Concat: [intro] + [trimmed sermon]
```

### Stage 5: Color Grade + Audio Processing

**Video filters** (applied in order):
```
lut3d=file='path/to/church.cube'       # Apply .cube LUT
eq=contrast=1.05:saturation=1.1        # Contrast + saturation (placeholder values)
colorbalance=rs=0.02:gs=-0.01:bs=0.03  # Shadow tweaks (placeholder)
```

**Audio filters** (minimal — Broadcast audio is already Waves-mastered):
```
loudnorm=I=-16:TP=-1.5:LRA=11         # EBU R128 to -16 LUFS target
```

**Fallback audio chain** (if Raw audio must be used):
```
compand → agate → mcompand → adeclick → loudnorm → alimiter
```

### Stage 6: Render (NVENC)

```bash
ffmpeg -hwaccel cuda -hwaccel_device 0 \
  -i processed.mp4 \
  -c:v h264_nvenc -preset p6 -cq 20 -rc vbr \
  -c:a aac -b:a 192k -movflags +faststart \
  -y "Sermon_Final.mp4"
```

| Parameter | Value | Reason |
|-----------|-------|--------|
| Encoder | `h264_nvenc` | GPU, 3-5x faster than libx264 |
| Preset | `p6` | High quality, still fast on GPU |
| CQ | `20` | Visually lossless for talking-head content |
| Audio | `aac @ 192k` | Universal compatibility |
| `+faststart` | — | Moves moov atom for web streaming |

---

## 5. Project Structure

```
Sermon_Video_Editor/
├── .env                          # API keys + file paths (git-ignored)
├── .env.example                  # Template for .env
├── .gitignore
├── requirements.txt
├── agent.py                      # Entry point — builds & runs the LangGraph pipeline
│
├── classes/
│   └── agent_state.py            # TypedDict defining the pipeline state
│
├── nodes/
│   ├── transcription_node.py     # Stage 1 — Whisper transcription of both files
│   ├── sync_node.py              # Stage 2 — Audio cross-correlation sync
│   ├── boundary_detection_node.py # Stage 3 — LLM sermon start/end detection
│   ├── trim_node.py              # Stage 4 — FFmpeg trim + intro prepend
│   ├── color_audio_node.py       # Stage 5 — LUT + audio normalization
│   └── render_node.py            # Stage 6 — NVENC final render
│
├── utils/
│   ├── ffmpeg_helpers.py         # Shared FFmpeg subprocess wrappers
│   ├── gpu_detect.py             # CUDA/NVENC availability check
│   └── audio_sync.py             # Cross-correlation logic (extracted for testing)
│
├── assets/
│   ├── intro.mp4                 # "You are listening to..." intro video
│   └── church.cube               # Color grading LUT file
│
└── output/                       # Generated files (git-ignored)
    ├── raw_audio.wav
    ├── broadcast_audio.wav
    ├── raw_transcription.json
    ├── broadcast_transcription.json
    ├── sync_report.json
    ├── boundary_report.json
    ├── trimmed_with_intro.mp4
    ├── color_graded.mp4
    └── Sermon_Final.mp4
```

---

## 6. Agent State

```python
from typing import TypedDict, Optional, List

class AgentState(TypedDict):
    # ── Inputs ──
    raw_video_path: str                    # Path to raw recording (clean video)
    broadcast_video_path: str              # Path to broadcast recording (mastered audio)
    intro_video_path: str                  # Path to intro graphic/video
    lut_path: str                          # Path to .cube LUT file
    output_dir: str                        # Directory for all intermediate + final files

    # ── Stage 1: Transcription ──
    raw_audio_path: Optional[str]          # Extracted WAV from raw
    broadcast_audio_path: Optional[str]    # Extracted WAV from broadcast
    raw_segments_path: Optional[str]       # JSON transcript segments (raw timeline)
    broadcast_segments_path: Optional[str] # JSON transcript segments (broadcast timeline)

    # ── Stage 2: Sync ──
    sync_offset_seconds: Optional[float]   # Raw leads broadcast by this many seconds
    sync_confidence: Optional[float]       # Cross-correlation peak value (0-1)
    overlap_start_raw: Optional[float]     # Start of overlapping region on raw timeline
    overlap_end_raw: Optional[float]       # End of overlapping region on raw timeline

    # ── Stage 3: Boundary Detection ──
    sermon_start_broadcast: Optional[float] # Sermon start on broadcast timeline (seconds)
    sermon_end_broadcast: Optional[float]   # Sermon end on broadcast timeline (seconds)
    sermon_start_raw: Optional[float]       # Mapped to raw timeline via offset
    sermon_end_raw: Optional[float]         # Mapped to raw timeline via offset
    boundary_reasoning: Optional[str]       # LLM's explanation of why it chose these points

    # ── Stage 4: Trim ──
    trimmed_video_path: Optional[str]      # Output of trim + intro concat

    # ── Stage 5: Color + Audio ──
    processed_video_path: Optional[str]    # Output of color grade + audio processing

    # ── Stage 6: Render ──
    final_output_path: Optional[str]       # Final rendered .mp4
    render_duration_seconds: Optional[float]
    render_fps: Optional[float]

    # ── Metadata ──
    status: str                            # "pending" | "transcribing" | "syncing" | etc.
    errors: List[str]                      # Accumulated error messages
    warnings: List[str]                    # Non-fatal warnings (e.g. low sync confidence)
```

---

## 7. Node Specifications

### 7.1 `transcription_node.py`

| Property | Value |
|----------|-------|
| **Reads from state** | `raw_video_path`, `broadcast_video_path`, `output_dir` |
| **Writes to state** | `raw_audio_path`, `broadcast_audio_path`, `raw_segments_path`, `broadcast_segments_path` |
| **External tools** | FFmpeg (audio extraction), Whisper (transcription) |
| **GPU required** | Yes — CUDA for Whisper inference |

**Logic**:
```python
for source in ["raw", "broadcast"]:
    video_path = state[f"{source}_video_path"]

    # 1. Extract audio → WAV
    wav_path = extract_audio(video_path, output_dir, label=source)

    # 2. Load/reuse cached Whisper model
    model = get_or_load_whisper("small.en", device="cuda")

    # 3. Transcribe
    result = model.transcribe(str(wav_path), fp16=True, language="English")

    # 4. Write segments JSON
    segments = format_segments(result["segments"])
    segments_path = write_segments_json(segments, output_dir, label=source)

    # 5. Update state
    state[f"{source}_audio_path"] = str(wav_path)
    state[f"{source}_segments_path"] = str(segments_path)
```

### 7.2 `sync_node.py`

| Property | Value |
|----------|-------|
| **Reads from state** | `raw_audio_path`, `broadcast_audio_path` |
| **Writes to state** | `sync_offset_seconds`, `sync_confidence`, `overlap_start_raw`, `overlap_end_raw` |
| **External tools** | Numpy, Librosa |
| **GPU required** | No |

**Logic**:
```python
# 1. Run cross-correlation (see Stage 2 algorithm above)
offset = find_sync_offset(raw_wav, broadcast_wav, sr=16000)

# 2. Calculate overlap region
raw_duration = librosa.get_duration(filename=raw_wav)
bcast_duration = librosa.get_duration(filename=broadcast_wav)

if offset >= 0:  # Raw starts first
    overlap_start_raw = offset
    overlap_end_raw = min(raw_duration, offset + bcast_duration)
else:  # Broadcast starts first
    overlap_start_raw = 0.0
    overlap_end_raw = min(raw_duration, bcast_duration + offset)

# 3. Validate — overlap should be substantial (>80% of shorter file)
shorter = min(raw_duration, bcast_duration)
overlap_len = overlap_end_raw - overlap_start_raw
if overlap_len / shorter < 0.8:
    warnings.append(f"Low overlap: {overlap_len:.1f}s of {shorter:.1f}s")

# 4. Write sync_report.json with offset, confidence, overlap
```

### 7.3 `boundary_detection_node.py`

| Property | Value |
|----------|-------|
| **Reads from state** | `broadcast_segments_path`, `sync_offset_seconds` |
| **Writes to state** | `sermon_start_broadcast`, `sermon_end_broadcast`, `sermon_start_raw`, `sermon_end_raw`, `boundary_reasoning` |
| **External tools** | OpenAI GPT-4o-mini via LangChain |
| **GPU required** | No |

**Logic**:
```python
# 1. Load timestamped segments from broadcast transcription
segments = load_json(broadcast_segments_path)

# 2. Format segments as numbered lines for the LLM
transcript_text = format_for_llm(segments)  # "[00:00-00:15] Welcome everyone..."

# 3. Call GPT-4o-mini with boundary detection prompt (see Section 9)
response = llm.invoke(boundary_prompt.format(transcript=transcript_text))

# 4. Parse response → sermon_start_broadcast, sermon_end_broadcast
boundaries = parse_boundary_response(response)

# 5. Map to raw timeline
state["sermon_start_raw"] = boundaries.start + sync_offset_seconds
state["sermon_end_raw"] = boundaries.end + sync_offset_seconds
state["boundary_reasoning"] = boundaries.reasoning
```

### 7.4 `trim_node.py`

| Property | Value |
|----------|-------|
| **Reads from state** | `raw_video_path`, `broadcast_video_path`, `sermon_start_raw`, `sermon_end_raw`, `sermon_start_broadcast`, `sermon_end_broadcast`, `intro_video_path` |
| **Writes to state** | `trimmed_video_path` |
| **External tools** | FFmpeg |
| **GPU required** | No (copy streams, no re-encode) |

**Logic**:
```python
# 1. Create intermediate: trimmed sermon body (raw video + broadcast audio)
#    Uses -ss and -to for frame-accurate seeking
#    -c copy for no re-encoding at this stage
cmd_body = [
    "ffmpeg",
    "-ss", str(sermon_start_raw), "-to", str(sermon_end_raw),
    "-i", raw_video_path,           # Input 0: Raw video
    "-ss", str(sermon_start_broadcast), "-to", str(sermon_end_broadcast),
    "-i", broadcast_video_path,     # Input 1: Broadcast audio
    "-map", "0:v:0",               # Take video from raw
    "-map", "1:a:0",               # Take audio from broadcast
    "-c", "copy",                  # No re-encode
    "-y", trimmed_body_path
]

# 2. Concat intro + body using FFmpeg concat demuxer
#    Write concat list file:
#    file 'intro.mp4'
#    file 'trimmed_body.mp4'
cmd_concat = [
    "ffmpeg",
    "-f", "concat", "-safe", "0",
    "-i", concat_list_path,
    "-c", "copy",
    "-y", trimmed_video_path
]
```

> ⚠️ **Note on concat**: The intro file must match the sermon body's codec, resolution, and frame rate for `-c copy` concat to work. If they differ, the render node (Stage 6) will handle re-encoding. Alternatively, Stage 4 can re-encode the intro to match.

### 7.5 `color_audio_node.py`

| Property | Value |
|----------|-------|
| **Reads from state** | `trimmed_video_path`, `lut_path` |
| **Writes to state** | `processed_video_path` |
| **External tools** | FFmpeg |
| **GPU required** | Optional (can use GPU for decode/encode, but filters run on CPU) |

**Logic**:
```python
# Build video filter chain
vf = ";".join([
    f"lut3d=file='{lut_path}'",
    "eq=contrast=1.05:saturation=1.1",
    "colorbalance=rs=0.02:gs=-0.01:bs=0.03"
])

# Build audio filter chain (minimal for Waves-mastered audio)
af = "loudnorm=I=-16:TP=-1.5:LRA=11"

cmd = [
    "ffmpeg",
    "-i", trimmed_video_path,
    "-vf", vf,
    "-af", af,
    "-c:v", "libx264", "-crf", "18",  # High quality intermediate
    "-c:a", "pcm_s16le",              # Lossless audio intermediate
    "-y", processed_video_path
]
```

### 7.6 `render_node.py`

| Property | Value |
|----------|-------|
| **Reads from state** | `processed_video_path`, `output_dir` |
| **Writes to state** | `final_output_path`, `render_duration_seconds`, `render_fps` |
| **External tools** | FFmpeg with NVENC |
| **GPU required** | Yes — NVENC for H.264 encoding |

**Logic**:
```python
# Detect GPU capability
has_nvenc = check_nvenc_available()

if has_nvenc:
    codec_args = ["-c:v", "h264_nvenc", "-preset", "p6", "-cq", "20", "-rc", "vbr"]
    hwaccel_args = ["-hwaccel", "cuda", "-hwaccel_device", "0"]
else:
    codec_args = ["-c:v", "libx264", "-crf", "20", "-preset", "slow"]
    hwaccel_args = []
    warnings.append("NVENC not available — falling back to CPU encoding (slower)")

cmd = [
    "ffmpeg", *hwaccel_args,
    "-i", processed_video_path,
    *codec_args,
    "-c:a", "aac", "-b:a", "192k",
    "-movflags", "+faststart",
    "-y", final_output_path
]
```

---

## 8. FFmpeg Filter Chains — Reference

### 8.1 Audio Extraction (used in Stage 1)

```bash
ffmpeg -i input.mp4 -vn -acodec pcm_s16le -ar 48000 -ac 1 -y output.wav
```

| Flag | Purpose |
|------|---------|
| `-vn` | Discard video |
| `-acodec pcm_s16le` | Uncompressed 16-bit PCM |
| `-ar 48000` | 48kHz sample rate (matches source) |
| `-ac 1` | Downmix to mono (for Whisper + correlation) |

### 8.2 Dual-Source Marriage (used in Stage 4)

```bash
ffmpeg \
  -ss 125.4 -to 3456.7 -i raw_video.mp4 \
  -ss 118.2 -to 3449.5 -i broadcast.mp4 \
  -map 0:v:0 -map 1:a:0 \
  -c copy \
  -y married_output.mp4
```

**Key**: `-ss` *before* `-i` enables fast seeking. `-map 0:v:0` takes video from first input, `-map 1:a:0` takes audio from second input.

### 8.3 Concat with Intro (used in Stage 4)

```bash
# concat_list.txt:
# file 'assets/intro.mp4'
# file 'output/married_output.mp4'

ffmpeg -f concat -safe 0 -i concat_list.txt -c copy -y output/trimmed_with_intro.mp4
```

### 8.4 Color Grading + Audio (used in Stage 5)

```bash
ffmpeg -i trimmed_with_intro.mp4 \
  -vf "lut3d=file='assets/church.cube',eq=contrast=1.05:saturation=1.1,colorbalance=rs=0.02:gs=-0.01:bs=0.03" \
  -af "loudnorm=I=-16:TP=-1.5:LRA=11" \
  -c:v libx264 -crf 18 \
  -c:a pcm_s16le \
  -y output/color_graded.mp4
```

### 8.5 Final Render with NVENC (used in Stage 6)

```bash
ffmpeg -hwaccel cuda -hwaccel_device 0 \
  -i output/color_graded.mp4 \
  -c:v h264_nvenc -preset p6 -cq 20 -rc vbr \
  -c:a aac -b:a 192k \
  -movflags +faststart \
  -y "output/Sermon_Final.mp4"
```

### 8.6 Fallback Audio Mastering Chain (if Raw audio used)

```bash
ffmpeg -i input.mp4 -af " \
  compand=attacks=0.01:decays=0.3:points=-80/-80|-45/-30|-27/-20|0/-10, \
  agate=threshold=0.01:ratio=2:attack=25:release=150, \
  mcompand=0.005 0.1 -47/-40|-34/-34|-17/-33 100 | \
          0.003 0.05 -34/-34|-17/-33 100 | \
          0.000 0.02 -34/-34|-17/-33 100, \
  adeclick=window=55:overlap=75, \
  loudnorm=I=-16:TP=-1.5:LRA=11, \
  alimiter=limit=0.95:attack=5:release=50 \
" -c:a pcm_s16le -y output_mastered.wav
```

| Filter | Purpose |
|--------|---------|
| `compand` | Dynamic range compression (bring quiet up, loud down) |
| `agate` | Noise gate — suppress low-level background noise |
| `mcompand` | Multi-band compression — tame frequency-specific dynamics |
| `adeclick` | Remove clicks/pops from the recording |
| `loudnorm` | EBU R128 loudness normalization to -16 LUFS |
| `alimiter` | Brick-wall limiter preventing clipping above -0.5 dB |

---

## 9. Sermon Boundary Detection — Prompt Engineering

### 9.1 System Prompt

```
You are an expert at analyzing church service transcripts. Your job is to identify
the exact timestamps where the SERMON begins and ends within a full church service
recording.

A typical church service includes (in rough order):
  - Pre-service music / countdown
  - Welcome & greeting from host or worship leader
  - Worship songs (singing, music)
  - Announcements (upcoming events, giving, community info)
  - Transition prayer or intro from host
  - **SERMON START** ← Find this
  - Sermon body (teaching, scripture reading, illustrations, application)
  - Closing prayer or altar call
  - **SERMON END** ← Find this
  - Post-sermon announcements or dismissal

CRITICAL RULES:
1. The sermon typically starts when the PASTOR (not worship leader, not host)
   begins their teaching. Look for these patterns:
   - A prayer before diving into content ("Lord God, we thank you...")
   - A series/topic introduction ("We are in week three of...")
   - An opening anecdote ("Has this ever happened to you...")
   - A scripture reading ("Turn in your Bibles to...")
   - A direct topic statement ("Today we're talking about...")

2. The sermon ends when:
   - The pastor gives a closing prayer ("Lord, I pray that...")
   - There's an altar call ("If you want to accept Christ...")
   - The pastor says a clear conclusion ("Let's pray" after the teaching)
   - There's a final "amen" followed by dismissal

3. Do NOT confuse pre-sermon prayers (during announcements/transition) with
   the pastor's opening prayer that kicks off the sermon.

4. Do NOT include worship music, announcements, or offering segments.

5. When in doubt, err on the side of INCLUDING content rather than cutting it.
   It's better to include 30 seconds of transition than to cut the first
   sentence of the sermon.

6. IGNORE any "You are listening to a message from..." intros — these are
   added in post-production and will NOT appear in raw recordings.
```

### 9.2 User Prompt Template

```
Below is a timestamped transcript of a church service recording. Each line shows
the time range and the spoken text.

Identify the SERMON START and SERMON END timestamps.

TRANSCRIPT:
{transcript}

Respond in this exact JSON format:
{
  "sermon_start_seconds": <float>,
  "sermon_end_seconds": <float>,
  "sermon_start_timestamp": "<MM:SS>",
  "sermon_end_timestamp": "<MM:SS>",
  "reasoning": "<1-2 sentences explaining why you chose these boundaries>",
  "confidence": "<high|medium|low>"
}
```

### 9.3 Example LLM Input/Output

**Input** (abbreviated):
```
[00:00-00:15] Welcome everyone to Thrive Community Church.
[00:15-00:45] Let's stand and worship together...
[05:30-06:00] ...thank you worship team. A few announcements...
[08:15-08:30] ...and now let's welcome Pastor Mike.
[08:30-09:00] Thank you. Let's pray. Lord God, we thank you for
              this opportunity to open your Word together...
[09:00-09:30] So we are in the second week of the Red Letter Challenge.
              If you remember last week, we talked about...
...
[52:00-52:30] ...and I pray that this week you would take that step.
              Lord, seal this word in our hearts. In Jesus' name, amen.
[52:30-53:00] Thank you Pastor Mike. Hey everyone, don't forget...
```

**Output**:
```json
{
  "sermon_start_seconds": 510.0,
  "sermon_end_seconds": 3150.0,
  "sermon_start_timestamp": "08:30",
  "sermon_end_timestamp": "52:30",
  "reasoning": "The sermon begins at 08:30 when the pastor starts with a prayer before diving into the Red Letter Challenge series. It ends at 52:30 after the closing prayer and 'amen', before the host returns with announcements.",
  "confidence": "high"
}
```

### 9.4 Fallback Strategy

If the LLM returns `"confidence": "low"`:
1. Log a warning with the reasoning
2. Write the boundary report with a `needs_review: true` flag
3. The human operator reviews and adjusts before rendering continues
4. Pipeline pauses at this stage until confirmation (CLI prompt or config override)

---

## 10. Configuration & Environment

### 10.1 `.env` Variables

```bash
# ── API Keys ──
OPENAI_API_KEY=sk-...                    # For GPT-4o-mini boundary detection

# ── File Paths ──
RAW_VIDEO_PATH=C:\Users\Videos\raw.mp4
BROADCAST_VIDEO_PATH=C:\Users\Videos\broadcast.mp4
INTRO_VIDEO_PATH=C:\Users\Videos\assets\intro.mp4
LUT_PATH=C:\Users\Videos\assets\church.cube
OUTPUT_DIR=C:\Users\Videos\output

# ── Whisper ──
WHISPER_MODEL=small.en                   # Options: tiny.en, base.en, small.en, medium.en
WHISPER_FORCE_CPU=false                  # Set true to skip GPU even if available

# ── Rendering ──
NVENC_PRESET=p6                          # p1 (fastest) to p7 (best quality)
CQ_VALUE=20                              # Constant quality (lower = better, 18-23 typical)
TARGET_LUFS=-16                          # Audio loudness target

# ── Behavior ──
AUTO_RENDER=true                         # false = pause after boundary detection for review
SKIP_COLOR_GRADE=false                   # true = skip LUT + color adjustments
SKIP_AUDIO_NORMALIZE=false               # true = use broadcast audio as-is
```

### 10.2 `.env.example`

Same as above with placeholder values and comments explaining each option.

---

## 11. CLI Interface

```bash
# Basic usage — process a pair of recordings
python agent.py --raw "path/to/raw.mp4" --broadcast "path/to/broadcast.mp4"

# With all options
python agent.py \
  --raw "path/to/raw.mp4" \
  --broadcast "path/to/broadcast.mp4" \
  --intro "assets/intro.mp4" \
  --lut "assets/church.cube" \
  --output "output/" \
  --review                              # Pause after boundary detection for manual review

# Override env vars via CLI
python agent.py \
  --raw "path/to/raw.mp4" \
  --broadcast "path/to/broadcast.mp4" \
  --whisper-model medium.en \
  --no-color                            # Skip color grading
```

**CLI arguments** (all override `.env`):

| Argument | Short | Required | Default | Description |
|----------|-------|----------|---------|-------------|
| `--raw` | `-r` | Yes | env `RAW_VIDEO_PATH` | Path to raw video file |
| `--broadcast` | `-b` | Yes | env `BROADCAST_VIDEO_PATH` | Path to broadcast video file |
| `--intro` | `-i` | No | env `INTRO_VIDEO_PATH` | Path to intro video/graphic |
| `--lut` | `-l` | No | env `LUT_PATH` | Path to `.cube` LUT file |
| `--output` | `-o` | No | env `OUTPUT_DIR` | Output directory |
| `--review` | | No | `false` | Pause after boundary detection |
| `--whisper-model` | | No | `small.en` | Whisper model size |
| `--no-color` | | No | `false` | Skip color grading |
| `--no-audio` | | No | `false` | Skip audio normalization |
| `--cpu-only` | | No | `false` | Force CPU for all operations |

---

## 12. Error Handling & Recovery

### 12.1 Strategy

Each node writes a **checkpoint file** (JSON) to the output directory before completing. If the pipeline crashes, it can resume from the last successful checkpoint.

```python
# At the end of each node:
write_checkpoint(output_dir, stage="sync", state=current_state)

# At pipeline start:
checkpoint = load_latest_checkpoint(output_dir)
if checkpoint:
    print(f"Resuming from stage: {checkpoint['stage']}")
    start_from = checkpoint["stage"]
```

### 12.2 Node-Level Error Handling

| Node | Failure Mode | Recovery |
|------|-------------|----------|
| **Transcription** | Whisper OOM on GPU | Retry with `force_cpu=True` |
| **Transcription** | FFmpeg not found | Clear error message with install instructions |
| **Sync** | Low correlation (<0.5) | Warning + ask user to verify files are same source |
| **Sync** | Files don't overlap | Fatal error — wrong files provided |
| **Boundary Detection** | LLM returns invalid JSON | Retry up to 3 times with stricter prompt |
| **Boundary Detection** | Low confidence | Pause for human review (if `--review`) |
| **Trim** | Codec mismatch (intro vs body) | Re-encode intro to match body format |
| **Color/Audio** | LUT file not found | Skip color grade, warn user |
| **Render** | NVENC not available | Fall back to libx264 CPU encoding |
| **Render** | Disk full | Fatal error with disk space info |

### 12.3 Logging

```python
import logging

# File + console logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    handlers=[
        logging.FileHandler(output_dir / "pipeline.log"),
        logging.StreamHandler()
    ]
)
```

---

## 13. Dependencies

### 13.1 `requirements.txt`

```
# ── Core Pipeline ──
langgraph>=0.2.0
langchain>=0.3.0
langchain-openai>=0.2.0

# ── Transcription ──
openai-whisper>=20231117
torch>=2.0.0                  # CUDA-enabled build for GPU inference

# ── Audio Processing ──
numpy>=1.24.0
librosa>=0.10.0
soundfile>=0.12.0             # Required by librosa for WAV I/O

# ── Environment ──
python-dotenv>=1.0.0

# ── Utilities ──
tqdm>=4.65.0                  # Progress bars for long operations
```

### 13.2 System Requirements

| Requirement | Version | Purpose |
|-------------|---------|---------|
| **Python** | 3.10+ | Runtime |
| **FFmpeg** | 6.0+ | Audio extraction, video processing, rendering |
| **NVIDIA Driver** | 535+ | CUDA support |
| **CUDA Toolkit** | 12.0+ | Whisper GPU inference |
| **NVENC** | (included in driver) | GPU-accelerated H.264 encoding |
| **Windows** | 10/11 | Target OS |

### 13.3 FFmpeg Build Requirements

FFmpeg must be compiled with:
- `--enable-cuda-llvm` — CUDA hardware acceleration
- `--enable-nvenc` — NVIDIA encoder
- `--enable-libx264` — CPU fallback encoder
- `--enable-filter=lut3d` — LUT color grading

> 💡 The [BtbN FFmpeg builds](https://github.com/BtbN/FFmpeg-Builds/releases) include all of these. Download the `ffmpeg-master-latest-win64-gpl` release.

### 13.4 Python Virtual Environment (Required)

> ⚠️ **All Python packages MUST be installed inside a virtual environment. Never install to global Python.**

```bash
# ── Create the venv (once, from project root) ──
cd C:\Users\wyatt\Documents\GitHub\Thrive\Sermon_Video_Editor
python -m venv venv

# ── Activate (every session) ──
# PowerShell:
.\venv\Scripts\Activate.ps1

# CMD:
venv\Scripts\activate.bat

# ── Install dependencies ──
pip install -r requirements.txt

# ── For CUDA-enabled PyTorch (must use the CUDA wheel index) ──
pip install torch --index-url https://download.pytorch.org/whl/cu121

# ── Verify isolation ──
where python          # Should point to venv\Scripts\python.exe
pip list              # Should only show project dependencies
```

**Rules**:
1. The `venv/` directory is **gitignored** — never committed
2. Every terminal session must activate the venv before running `agent.py`
3. No `pip install --user` or `pip install` outside the venv
4. The `requirements.txt` is the single source of truth for dependencies
5. If a new package is needed, `pip install <pkg>` inside the venv, then `pip freeze > requirements.txt` to update

---

## 14. Development Phases

### Phase 1: Foundation 🏗️
**Goal**: Scaffold project, get Whisper transcription working on both files

| Task | Details |
|------|---------|
| Create project directory | `Sermon_Video_Editor/` with structure from Section 5 |
| Set up `.env` + `.gitignore` | Copy pattern from `Sermon_Summarization_Agent` |
| Write `agent_state.py` | TypedDict from Section 6 |
| Write `transcription_node.py` | Port from Summarization Agent, add dual-file support |
| Write `agent.py` skeleton | Linear execution, just Stage 1 |
| **Test**: Transcribe both files | Verify segments JSON output for both Raw + Broadcast |

### Phase 2: Sync + Detect 🔍
**Goal**: Audio sync working, LLM boundary detection returning valid timestamps

| Task | Details |
|------|---------|
| Write `audio_sync.py` utility | Cross-correlation algorithm from Section 4.2 |
| Write `sync_node.py` | Integrate sync utility, write report JSON |
| Write `boundary_detection_node.py` | LLM prompt from Section 9, JSON parsing |
| **Test**: Sync on real files | Verify offset is consistent, confidence is high |
| **Test**: Boundary detection | Compare LLM output against manually identified boundaries |

### Phase 3: Trim + Assemble ✂️
**Goal**: Sermon trimmed with intro prepended, dual-source marriage working

| Task | Details |
|------|---------|
| Write `ffmpeg_helpers.py` | Shared subprocess wrappers, error handling |
| Write `trim_node.py` | Dual-source marriage + concat logic |
| Prepare `assets/intro.mp4` | Encode intro graphic to match sermon specs |
| **Test**: End-to-end trim | Verify output plays correctly with intro → sermon |

### Phase 4: Polish + Render 🎨
**Goal**: Color grading, audio normalization, GPU rendering

| Task | Details |
|------|---------|
| Write `gpu_detect.py` | NVENC availability check |
| Write `color_audio_node.py` | LUT + eq + loudnorm filters |
| Write `render_node.py` | NVENC encoding with CPU fallback |
| Obtain `.cube` LUT file | Export from Premiere/DaVinci or create custom |
| **Test**: Full pipeline | Raw + Broadcast → Final rendered MP4 |

### Phase 5: Harden + CLI 🔧
**Goal**: Production-ready with error handling, checkpoints, and polished CLI

| Task | Details |
|------|---------|
| Add checkpoint/resume logic | JSON checkpoints after each stage |
| Add CLI argument parsing | `argparse` with all options from Section 11 |
| Add `--review` mode | Pause after boundary detection, show preview |
| Add comprehensive logging | File + console logger |
| Write error recovery for each node | Per Section 12.2 |
| **Test**: Crash recovery | Kill mid-pipeline, verify resume works |
| **Test**: Edge cases | Missing files, wrong format, no GPU |

---

## 15. Verification & Definition of Done

### 15.1 Acceptance Criteria — Per Stage

Each stage has a **pass/fail gate** that must be met before the pipeline is considered functional.

| Stage | ✅ Pass Criteria | ❌ Fail Criteria |
|-------|-----------------|-----------------|
| **1 — Transcribe** | Both Raw and Broadcast produce valid `segments.json` files with >0 segments. Timestamps are monotonically increasing. Text is non-empty English. | Empty segments, garbled text, Whisper crash, or only one file transcribed. |
| **2 — Sync** | `sync_offset_seconds` is a finite float. `sync_confidence` ≥ 0.7. Overlap region covers ≥80% of the shorter file's duration. | Confidence < 0.5, NaN/infinite offset, or overlap < 50% (wrong files). |
| **3 — Detect** | LLM returns valid JSON with `sermon_start_seconds` < `sermon_end_seconds`. Sermon duration is between 15 and 90 minutes. `confidence` is `high` or `medium`. | Invalid JSON, start ≥ end, duration outside 15-90 min range, or `confidence: low` without human override. |
| **4 — Trim** | Output file plays in VLC/media player. Video stream is from Raw (no watermarks). Audio stream is from Broadcast (mastered quality). Intro plays first, then sermon. Duration ≈ `sermon_end - sermon_start + intro_duration` (±2 sec). | Black frames, no audio, wrong audio source, intro missing, duration way off. |
| **5 — Color/Audio** | LUT is visibly applied (A/B frame comparison). Audio measures between -18 and -14 LUFS (measured via `ffmpeg -i file -af loudnorm=print_format=json -f null -`). No clipping. | No visible color change, audio outside LUFS range, clipping detected. |
| **6 — Render** | Final `.mp4` plays in VLC, YouTube upload test, and browser. File size is reasonable (500MB-3GB for 30-60 min sermon). `moov` atom is at front (`+faststart` verified). Codec is H.264/AAC. | Corrupt file, won't play, codec rejected by YouTube, massive/tiny file size. |

### 15.2 End-to-End Acceptance Test

The project is **done** when this single test passes:

```
GIVEN:  One real Raw recording + one real Broadcast recording from the same service
WHEN:   python agent.py --raw "raw.mp4" --broadcast "broadcast.mp4"
THEN:
  1. Pipeline completes without human intervention (no --review pause)
  2. Output file exists at the expected path
  3. Output contains ONLY the sermon (no worship, no announcements)
  4. Video is clean (from Raw — no watermarks or lower thirds)
  5. Audio is mastered (from Broadcast — Waves-processed quality)
  6. Intro "You are listening to..." plays before the sermon
  7. Color grading is applied (LUT visible)
  8. Audio loudness is -16 LUFS ± 2 dB
  9. File plays correctly in VLC and uploads to YouTube without errors
  10. Total wall-clock time from start to final file: < 15 minutes
```

### 15.3 Validation Checklist (Human Review)

After the first 3 real sermons processed, the operator confirms:

- [ ] Sermon start point is accurate (within ~5 seconds of where you'd manually cut)
- [ ] Sermon end point is accurate (within ~5 seconds)
- [ ] Audio/video sync is tight — no visible lip-sync drift
- [ ] No audio glitches at the Raw→Broadcast splice point
- [ ] Color grade matches the look you'd apply in Premiere/DaVinci
- [ ] Intro graphic transitions cleanly into sermon content
- [ ] Final file is YouTube-ready with no additional editing needed
- [ ] You'd be comfortable publishing this to the church channel as-is

### 15.4 Automated Verification (Built into Pipeline)

These checks run automatically at the end of every execution:

```python
def verify_output(state: AgentState) -> dict:
    """Post-pipeline verification. Returns pass/fail with details."""
    checks = {}

    # 1. File exists and is non-trivial
    final = Path(state["final_output_path"])
    checks["file_exists"] = final.exists()
    checks["file_size_mb"] = final.stat().st_size / (1024 * 1024) if final.exists() else 0
    checks["size_reasonable"] = 500 < checks["file_size_mb"] < 4000

    # 2. Duration matches expected sermon length
    probe = ffprobe_duration(final)  # ffprobe wrapper
    expected = (state["sermon_end_raw"] - state["sermon_start_raw"]) + intro_duration
    checks["duration_seconds"] = probe
    checks["duration_match"] = abs(probe - expected) < 5.0

    # 3. Audio loudness
    lufs = measure_loudness(final)  # ffmpeg loudnorm print_format=json
    checks["loudness_lufs"] = lufs
    checks["loudness_pass"] = -18.0 <= lufs <= -14.0

    # 4. Codec verification
    codec_info = ffprobe_codecs(final)
    checks["video_codec"] = codec_info["video"]  # expect "h264"
    checks["audio_codec"] = codec_info["audio"]  # expect "aac"
    checks["codec_pass"] = codec_info["video"] == "h264" and codec_info["audio"] == "aac"

    # 5. Faststart (moov atom at front)
    checks["faststart"] = check_moov_position(final)

    # 6. Sync confidence from earlier stage
    checks["sync_confidence"] = state["sync_confidence"]
    checks["sync_pass"] = state["sync_confidence"] >= 0.7

    # 7. Boundary confidence
    checks["boundary_confidence"] = state.get("boundary_confidence", "unknown")

    # Overall
    checks["all_passed"] = all([
        checks["file_exists"], checks["size_reasonable"],
        checks["duration_match"], checks["loudness_pass"],
        checks["codec_pass"], checks["faststart"], checks["sync_pass"]
    ])

    return checks
```

### 15.5 Definition of Done — Project Level

The project is **shipped** when:

| # | Criterion | How to verify |
|---|-----------|---------------|
| 1 | Pipeline processes a real sermon pair end-to-end with zero manual intervention | Run `agent.py` and walk away |
| 2 | Output passes all automated verification checks (§15.4) | `all_passed == True` in verification report |
| 3 | Operator approves 3 consecutive sermons without re-editing | Human checklist (§15.3) signed off 3x |
| 4 | Wall-clock time is under 15 minutes on target hardware | Timed run logged in `pipeline.log` |
| 5 | Pipeline recovers from a mid-run crash and resumes correctly | Kill during Stage 4, restart, verify it picks up from checkpoint |
| 6 | `--review` mode pauses and accepts human boundary overrides | Test with intentional low-confidence transcript |
| 7 | CPU fallback works when GPU is unavailable | Run with `--cpu-only`, verify output quality matches |

> 🏁 **Ship criteria**: Items 1-4 are **required**. Items 5-7 are **hardening** — nice to have for v1, required for v1.1.

---

## 📋 Summary

| What | How |
|------|-----|
| **Orchestration** | LangGraph `StateGraph`, deterministic linear pipeline |
| **Transcription** | Whisper `small.en` on CUDA, dual-file |
| **Sync** | Numpy cross-correlation on extracted audio |
| **Boundary Detection** | GPT-4o-mini with structured prompt, JSON response |
| **Trim + Intro** | FFmpeg `-map` for dual-source, concat demuxer for intro |
| **Color** | FFmpeg `lut3d` + `eq` + `colorbalance` |
| **Audio** | FFmpeg `loudnorm` (broadcast) or full chain (raw fallback) |
| **Render** | FFmpeg `h264_nvenc` GPU encoding, `-cq 20`, `+faststart` |
| **Recovery** | JSON checkpoints, per-node error handling, `--review` mode |
| **Target** | < 5 minutes end-to-end (review only) |