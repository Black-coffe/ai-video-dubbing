# The task this pipeline was built for

This is the brief Claude Code received at the start (translated from Russian). It explains why the
pipeline stops where it stops and why the timing logic looks the way it does.

## What we build

A command-line tool that takes a video with the author's Russian speech and produces a Ukrainian audio
track in the author's own voice. The track is uploaded to the same video: YouTube Studio → Languages → dub.

Only the author's voice is translated. Music, sounds and effects stay as in the original. There are no other
speakers in the videos.

## Input

| File | What it is | Parameters |
|---|---|---|
| `input/video.mp4` | A finished video (voice already mixed with the background) | e.g. 1920×1080, H.264; AAC stereo 48 kHz |
| `input/voice-sample.wav` | A sample of the author's voice (variant C) | 1–2 min of clean speech, the same microphone as in the videos |

Future videos: about 20 minutes each, two a week. No script and no subtitles: the text comes from speech
recognition.

## Output

- A full dubbed audio track: dubbed voice + original background. Exactly as long as the video, loudness
  matched to the original. Ready to upload to YouTube as a dub track.
- Intermediate files in the video's work folder: transcript, translation file, synthesized phrases,
  timing report.

Out of scope: subtitles, title and description, lip sync (no face on screen), changes to the picture.

## Requirements

### Translation
- Address the viewer informally ("ти" in Ukrainian).
- Never translated: Claude Code, MCP, skills, names of programs and services. The list lives in the glossary.
- Topic: AI at work, IT.
- The author reviews the translation. **The run must stop before synthesis** until the author has checked it.

### Voice and intonation
- The voice must be recognizably the author's.
- **Intonation matters a lot**: it must not sound like monotone reading.
- The expected hard part: IT terms and English words, both in recognition and in pronunciation.

### Timing
- Fitting to time is decided phrase by phrase: add a pause here, speed up slightly there. The author decides.
- A single phrase can be re-synthesized without touching the rest.

### Voice engines
- Main option: **ElevenLabs**.
- Must be compared with a **local option on an RTX 3090**, with the same translation, by ear.

### Environment
- Command line, the author's computer: Windows 11, Python 3.12, ffmpeg, NVIDIA RTX 3090 24 GB.

## Proposed scheme (as given to Claude Code)

1. **Separate the voice from the background** on the GPU (Demucs) for videos without a separate voice track.
2. **Speech recognition** with word timestamps (Whisper large-v3 locally). English terms written in Cyrillic
   are fixed by the glossary.
3. **Translation with Claude**: with the glossary and the duration of each phrase.
4. **Pronunciation dictionary**: the reviewed text keeps terms in Latin, synthesis gets a phonetic spelling
   ("Claude Code" → "клод код").
5. **Synthesis** with the neighbouring phrases passed in for smooth intonation.
6. **Time fitting**: automatic speed-up without pitch change and use of the neighbouring pause; what still
   doesn't fit goes to a report and the author decides. Decisions are saved and re-applied on rerun.
7. **Mix**: background + dubbed voice, loudness matched to the original, duration exactly as the video.
