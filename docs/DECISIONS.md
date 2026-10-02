# Why these components

Decided on 2026-09-29 from a research pass over ~75 services and models (translated and condensed from Russian).
Prices below are the **pre-build estimates** from public price pages on that date; the **measured** costs after
the build are in the README and came out much lower. Prices change: check them yourself before you commit.

## Summary

| Block | Chosen | Runs where | Estimated $ per 20-min video |
|---|---|---|---|
| Voice / background separation | **Demucs v4** (htdemucs) | local GPU | $0 |
| Speech recognition | **faster-whisper** (Whisper large-v3) | local GPU | $0 |
| Translation | **Claude Sonnet** via API | Anthropic cloud | $0.72 (Batch: $0.36) |
| Voice | **ElevenLabs**, clone of the author's voice | ElevenLabs cloud | $2.75 base / $4.00 with re-takes (Creator plan) |
| Time fitting and mix | **ffmpeg + Rubber Band** | local CPU | $0 |

## Separation: Demucs v4
- Free, MIT license, no conflict with monetization. Runs on Windows with CUDA.
- Only needed for videos without a separate voice track; if you can export "voice" and "everything but voice"
  from your editor, you can skip it.
- Risk: the main repository was archived on 2025-01-01; a maintained fork exists. The models were made for
  music, so the background is "everything except vocals".
- Rejected: LALAL.AI, AudioShake, Music.AI, MVSEP (paid, upload to cloud); ElevenLabs Voice Isolator (gives the
  voice only, no background); UVR / RoFormer community weights (licenses often unstated); Spleeter (older, worse).

## Recognition: faster-whisper
- Free, MIT (code and Whisper weights). Word timestamps. Terms can be hinted with `initial_prompt`;
  the glossary fixes the rest after recognition.
- Rejected: cloud STT (cheap, $0.01–0.32 per video, but uploads audio and gives nothing extra when a local GPU is
  available); transcription APIs without word timestamps; WhisperX (heavier setup, keep as fallback).

## Translation: Claude Sonnet
- The prompt carries the glossary, the informal address, context and the per-phrase length budget.
  Classic machine translation only does glossaries.
- A whole 20-minute transcript fits in one request. Prompt caching makes the "shorten" rounds cheap.
- Rejected: Opus (2× the price), Haiku (risk on nuance and terms), DeepL / Google / Azure / Amazon
  (no context, no length budget), local LLMs (tried later as variant C), NLLB (non-commercial license).

## Voice: ElevenLabs voice clone
- Ukrainian is in all current models (v4, v3, Multilingual v2, Flash). Neighbouring phrases can be passed for
  smooth intonation. The pipeline uses `eleven_v4`; in one side-by-side listening test its intonation was preferred
  over `eleven_multilingual_v2` (one test, not a benchmark: compare both on your own voice).
- Instant clone (from a 1–2 minute sample) vs Professional Voice Clone (needs 30 minutes to a few hours of clean
  speech, noticeably closer to the original). See the README for how to make one.
- Rejected for publishing: local models whose weights forbid commercial use (OmniVoice, Fish S2 Pro, XTTS-v2,
  F5-TTS) or lack Ukrainian; providers without cloning or without Ukrainian.

## Fitting and mix: ffmpeg + Rubber Band
- Free. Rubber Band stretches speech better than `atempo`, which matters at +10%.
- `ebur128` / volume match the loudness to the original; `amix` puts the voice over the background.

## Rejected as a whole: ready-made dubbing services
ElevenLabs Dubbing, HeyGen, Rask AI and similar: little control over review stops and per-phrase decisions,
and $6–44 per video. ElevenLabs Dubbing is still included as variant B so you can compare by ear and by price.
