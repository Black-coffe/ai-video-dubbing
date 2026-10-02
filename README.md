# ai-video-dubbing

Dub your YouTube video into another language **in your own voice**, from the command line.
You give it a video, it gives you a dubbed audio track that is exactly as long as the video, with the original music
and sounds kept. Upload it in YouTube Studio → Languages → Dub, or use the ready MP4 with both audio tracks.

Three ways to do the same job, so you can pick by quality, price and privacy:

| | What it is | Script | Cost per 20-min video (measured) | Trade-offs |
|---|---|---|---|---|
| **A** (recommended) | Demucs + faster-whisper on your GPU, translation by Claude with your glossary and a per-phrase length budget, voice by your ElevenLabs voice clone (`eleven_v4`), automatic time fitting | `dub.py` | **≈ $0.20** (Claude translation + `eleven_v4` voice on the ElevenLabs Starter plan at its promotional rate; ≈ $0.94 after that rate ends on 2026-10-12) | Best result in our listening test. Stops for you to review the translation. Needs two API keys and a voice clone. |
| **B** | ElevenLabs Dubbing API, one call does everything | `eldub.py` | **≈ $10–23** (≈ 3,000 credits per minute of video: ~$10 pay-as-you-go, ~$23 as a share of the Pro plan) | Easiest. 50–100× pricier than A, no control over the translation, sounded worst of the three. Good as a baseline. |
| **C** | Fully local and free: translation by MamayLM 12B in Ollama, voice cloned by OmniVoice from a 1–2 min sample | `variant_local.py` + `tts_omnivoice.py` | **≈ $0.01** (electricity only, ≈ 0.05 kWh) | Nothing leaves your PC. Clean sound, but in our Russian → Ukrainian test the voice kept a Russian intonation and put stress on wrong syllables. **OmniVoice weights are non-commercial (CC-BY-NC): not for a monetized channel.** |

Costs were measured on 2026-10-01 on a real 20-minute video project (12-minute and 3-minute runs scaled to 20 minutes).
Prices of the services change, so treat them as an order of magnitude.

Built and tested on **Russian → Ukrainian**. Other language pairs work through `--src` / `--tgt`, see
[Other languages](#other-languages) for what to adjust.

This is the pipeline built live in a video of the YouTube channel **AI pod rabotu**:
[watch the video](https://youtu.be/IT9yzqFiF9c) (in Russian), channel [youtube.com/@andrievskii](https://www.youtube.com/@andrievskii). All materials of the episodes:
[andrievskii.site/materials](https://andrievskii.site/materials).

---

## How variant A works

```
video.mp4
  └─ extract     ffmpeg + Demucs: original audio → vocals.wav + background.wav            (local GPU)
  └─ transcribe  faster-whisper large-v3: phrases with word timestamps                    (local GPU)
  └─ translate   Claude: glossary fixes, translation adapted to each phrase's time budget (API)
       ⏸  STOP: you review phrases_<tgt>.txt / translation.json, then run again
  └─ synth       ElevenLabs voice clone, phrase by phrase; too long → shorten text (≤ 2 rounds),
                 then speed up ≤ 1.10× (Rubber Band) and start earlier into the pause    (API + CPU)
  └─ mix         dubbed voice at the original voice's loudness over the background,
                 whole track at the original loudness, exactly the video's length         (CPU)
  └─ mux         output/<name>_<tgt>.mp4 (dub + original tracks) and <name>_<tgt>.m4a     (CPU)
```

Every step is resumable and cached: approved translations are reused by their source text, synthesized phrases are
cached by text, so a rerun only pays for what changed.

## Requirements

- **Python 3.12** (tested on 3.12.10).
- **ffmpeg with the `rubberband` filter** in `PATH`. Check: `ffmpeg -hide_banner -filters | grep rubberband`
  (Windows: `findstr` instead of `grep`). On Windows the "full" builds from gyan.dev include it.
- **An NVIDIA GPU is strongly recommended.** Tested on an RTX 3090 24 GB: Demucs + Whisper take ≈ 1.5 min for a
  20-minute video (≈ 15 s for a 3-minute fragment) and use a few GB of VRAM. A CPU-only PC works for A and B but is
  slower (≈ 20 min for the local steps of a 20-minute video). Variant C needs a GPU.
- **Windows + NVIDIA:** faster-whisper (CTranslate2) needs the cuBLAS and cuDNN 12 DLLs. `requirements.txt`
  installs `nvidia-cublas-cu12` and `nvidia-cudnn-cu12`, and `dub.py` adds their folders to the DLL path itself.
- **Mac (Apple Silicon):** Demucs runs on MPS, faster-whisper falls back to the CPU (no Metal backend). Not tested
  end to end.
- First run downloads the models: Whisper large-v3 (≈ 3 GB) and htdemucs.

## Install

```bash
git clone https://github.com/Black-coffe/ai-video-dubbing
cd ai-video-dubbing
python -m venv .venv
# Windows: .venv\Scripts\activate      Linux/Mac: source .venv/bin/activate

# 1. PyTorch with GPU support: take the exact command for your CUDA version from
#    https://pytorch.org/get-started/locally/  (tested: torch 2.14 + CUDA 13.0)
pip install torch torchaudio --index-url https://download.pytorch.org/whl/cu130

# 2. Everything else
pip install -r requirements.txt
```

## Configure keys

```bash
cp .env.example .env      # Windows: copy .env.example .env
```

Fill in `.env` (it is git-ignored; real environment variables take priority over it):

| Variable | Needed for | Where to get it |
|---|---|---|
| `ANTHROPIC_API_KEY` | A (translation) | console.anthropic.com → API keys |
| `ELEVENLABS_API_KEY` | A, B | elevenlabs.io → Developers → API keys |
| `ELEVENLABS_VOICE_ID` | A | the ID of your cloned voice, see [Make your voice clone](#make-your-voice-clone) |
| `OMNI_PY` | C (optional) | python of the OmniVoice venv, default `omni-env/Scripts/python.exe` |

No key is ever written to disk by the scripts. If you use Claude Code in the same terminal, don't export
`ANTHROPIC_API_KEY` globally there: Claude Code would then bill that key instead of your subscription. The `.env`
file is read only by these scripts.

## Run variant A (recommended)

Put your video in `input/` (the folder is git-ignored).

```bash
python dub.py run input/my-video.mp4                     # Russian → Ukrainian (default)
python dub.py run input/my-video.mp4 --src en --tgt es   # English → Spanish
```

1. The first run separates the voice, transcribes, translates and **stops**:
   `STOP: 120 phrases were machine-translated — review work/my-video/phrases_uk.txt`.
2. Read `work/my-video/phrases_<src>.txt` and `phrases_<tgt>.txt` side by side. Fix wording in
   `work/my-video/translation.json` (the `tgt` field of each phrase). Recurring fixes belong in `glossary.json`.
3. Run the same command again. It reuses your reviewed translation, synthesizes, fits, mixes and muxes.
4. Listen. If a phrase overlaps the next one, the run prints it; decide in `work/my-video/fit.json` and run again
   (only changed phrases are re-synthesized):

   ```json
   { "17": { "speed": 1.08, "shift": -0.3, "src": "exact source text of phrase 17" } }
   ```
   `speed` is the tempo (1.08 = 8% faster, pitch kept), `shift` moves the phrase start in seconds,
   `src` pins the decision to that phrase so it is skipped if a re-transcription renumbers phrases.
   A phrase with a decision is yours: it is never shortened or auto-fitted.

Single steps: `python dub.py <extract|transcribe|translate|synth|mix|mux> input/my-video.mp4`.
Useful flags: `--fresh` (redo extract + transcribe), `--no-review` (don't stop after translation),
`--manual file.json` (take translations `{"n": "text"}` from a file instead of the API),
`--device cpu|cuda|mps`.

Measured on a 12-minute video: 181 phrases, 8 minutes for the whole run, translation ≈ $0.09,
7,000 characters of synthesis.

## Run variant B (ElevenLabs Dubbing API, for comparison)

```bash
python eldub.py input/my-video.mp4                     # ru → uk
python eldub.py input/my-video.mp4 --src en --tgt es
```

Uploads the video, waits (up to 30 min), downloads the dub → `output/my-video_eldub.m4a`. The dubbing id is kept in
`work/my-video/eldub.json`, so a rerun resumes or re-downloads instead of paying again.

## Run variant C (fully local, free, non-commercial)

1. Install [Ollama](https://ollama.com) and pull the translation model used in `variant_local.py`:
   `ollama pull hf.co/INSAIT-Institute/MamayLM-Gemma-3-12B-IT-v2.0-GGUF:Q5_K_M`
   (MamayLM is tuned for Ukrainian; for another target language change `LLM` to a model that knows it).
2. Create a **separate** venv for OmniVoice (it pins its own torch):
   ```bash
   python -m venv omni-env
   omni-env/Scripts/pip install torch torchaudio --index-url https://download.pytorch.org/whl/cu128
   omni-env/Scripts/pip install -r requirements-omnivoice.txt
   ```
   (Linux/Mac: `omni-env/bin/...` and set `OMNI_PY=omni-env/bin/python` in `.env`.)
3. Put 1–2 minutes of clean speech of your voice, in the source language, at `input/voice-sample.wav`.
4. Run A's first two steps, then C:
   ```bash
   python dub.py extract input/my-video.mp4
   python dub.py transcribe input/my-video.mp4
   python variant_local.py input/my-video.mp4
   ```
   → `output/my-video_local.m4a`. Measured on a 3-minute fragment: translation 41 s, 29 phrases voiced in 37 s,
   2.1 GB VRAM (first model load ≈ 2.5 min).

## Outputs

| Path | What |
|---|---|
| `output/<name>_<tgt>.m4a` | A: the dub track for YouTube Studio → Languages → Add language → Dub |
| `output/<name>_<tgt>.mp4` | A: the video with the dub as the default track and the original as the second track |
| `output/<name>_eldub.m4a`, `output/<name>_local.m4a` | B and C tracks |
| `work/<name>/transcript.json` | recognized phrases with word timestamps |
| `work/<name>/translation.json`, `phrases_<lang>.txt` | translation per phrase, who made it (model / manual), length budget |
| `work/<name>/fit.json` | your per-phrase timing decisions |
| `work/<name>/timing_report.json` | per phrase: raw length, speed, shift, overlap |
| `work/<name>/timings.json` | time, tokens, cache hits and $ of every step |
| `work/<name>/tts/` | cached synthesized phrases |

## Glossary

One `glossary.json` for all your videos. The shipped entries are the real Russian → Ukrainian ones from the channel:
replace them with yours.

```json
{
  "asr_prompt": "Claude Code, MCP, Anthropic, API.",
  "asr_fixes":  [{"pattern": "\\bклод[- ]код\\w*", "replace": "Claude Code"}],
  "keep":       ["Claude Code", "MCP", "YouTube"],
  "terms":      [{"source": "скилл", "target": "скіл", "note": "do not translate as «навичка»"}],
  "drop_phrases": ["Субтитры сделал"],
  "pronounce":  [{"text": "Claude Code", "say": "клод код"}]
}
```

| Field | Applied | Effect |
|---|---|---|
| `asr_prompt` | recognition | hint for Whisper with words it tends to misspell |
| `asr_fixes` | after recognition | regex replacements in the source text (case-insensitive) |
| `keep` | translation | names never translated or transliterated |
| `terms` | translation | fixed translations of specific terms |
| `drop_phrases` | after recognition | segments containing these (Whisper hallucinations on music) are removed |
| `pronounce` | synthesis only | the translation keeps `text`, the voice receives `say` (fix mispronounced names) |

## Make your voice clone

Variant A speaks with an ElevenLabs clone of your voice. Clone only a voice you have the rights to (your own).

- **Instant Voice Clone**: from a 1–2 minute sample, available from the Starter plan (as of 2026-09). Good enough
  to try the pipeline; the likeness is noticeably weaker.
- **Professional Voice Clone**: trained on your recordings, much closer to you, available from the Creator plan
  (as of 2026-09). Upload at least **~30 minutes** of clean speech; **2–3 hours** is recommended.
- **Record the training audio in the target language** if you can speak it: the clone learns its accent and
  stress from what you give it, so a clone trained on your Russian speaks Ukrainian with a Russian accent.
- Clean audio matters more than length: one microphone, no music, no room echo, no long silences.
- When it is ready, open the voice in ElevenLabs → Voices, copy its voice ID into `ELEVENLABS_VOICE_ID`.

The synthesis model is set in `dub.py` (`TTS_MODEL = "eleven_v4"`, the newest ElevenLabs model when this was built).
To compare with the older one, set it to `eleven_multilingual_v2`; takes are cached per model, so both coexist.
Variant B uses whatever model the Dubbing API picks.

## Other languages

`--src` / `--tgt` (ISO 639-1 codes) set the Whisper language, the language names in the translation prompt, the
ElevenLabs Dubbing languages and the audio-track tags. Only Russian → Ukrainian was tested end to end. For another
pair, check these places:

| Where | What to adjust |
|---|---|
| `glossary.json` | replace all entries: they are Russian → Ukrainian examples |
| `dub.py` → `translation_prompt()` | the two examples are Russian → Ukrainian; the informal-address rule is tuned for Ukrainian. Works as is for other pairs, better with examples in your languages |
| `dub.py` → `ALLOWED` | the glitch filter allows Latin and Cyrillic only. For Greek, Arabic, CJK, etc. add the Unicode range, or every phrase is flagged |
| `dub.py` → `LANGS` | language names and 3-letter tags; unknown codes fall back to the code itself |
| `dub.py` → `DEFAULT_VOICE_RATE` | 14.6 chars/s, measured on a Ukrainian clone; replaced automatically after the first `synth` |
| `variant_local.py` → `LLM` | MamayLM is a Ukrainian model; pick an Ollama model for your target language |

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Could not load library cudnn_ops64_9.dll` / `cublas64_12.dll` (Windows) | `pip install nvidia-cublas-cu12 nvidia-cudnn-cu12` in the same venv; `dub.py` adds them to the path |
| `No such filter: 'rubberband'` | your ffmpeg build lacks librubberband: install a full build |
| `STOP: ELEVENLABS_VOICE_ID is not set` | put your voice ID in `.env` |
| `STOP: ... ANTHROPIC_API_KEY is not set` | set the key, or translate yourself and pass `--manual translations.json` |
| `STOP: phrases [..] look glitched` | the model returned another script or JSON scraps twice; fix those phrases in `translation.json` (or extend `ALLOWED` for your script) |
| `ElevenLabs HTTP 401 / 402 / 429` | wrong key, out of credits, or rate limit: check the ElevenLabs usage page and rerun (finished phrases are cached) |
| Voice runs over the next phrase | see `timing_report.json`; shorten the text in `translation.json` or decide speed/shift in `fit.json`. Speed-ups above ~1.15× are audible |
| A term is mispronounced | add a `pronounce` entry with a phonetic spelling in the target language |
| Phrases split differently after moving to another PC | keep one video on one machine: a different Whisper setup splits phrases differently, and approved translations stop matching |
| `HF Hub unauthenticated requests` warning | harmless; set `HF_TOKEN` for faster model downloads |
| OmniVoice step fails to start | check `OMNI_PY` points to the python of `omni-env` |

## Licenses

The code of this repository is MIT (see [LICENSE](LICENSE)). The models and services it calls keep their own
licenses and terms; check them before you publish:

| Component | License / terms | Commercial use |
|---|---|---|
| Demucs (htdemucs) | MIT | yes |
| faster-whisper, Whisper weights | MIT | yes |
| Claude API | Anthropic commercial terms | yes, under your account |
| ElevenLabs TTS and Dubbing | ElevenLabs terms; clone only voices you have rights to | per your plan |
| MamayLM (variant C) | see the model card on Hugging Face | check the card |
| **OmniVoice (variant C)** | code Apache-2.0, **weights CC-BY-NC** | **no**: tests and personal use only |
| ffmpeg, Rubber Band | LGPL/GPL depending on the build; Rubber Band GPL | fine for processing your own files |

More on why each component was chosen: [docs/DECISIONS.md](docs/DECISIONS.md). The original brief:
[docs/TASK.md](docs/TASK.md).

---

## Use it with Claude Code or Codex

Open the repository folder in Claude Code (or Codex) and paste one of these. Replace the parts in `<...>`.

**Set up the environment**
```text
Read README.md. Set this repo up on my machine: Python 3.12 venv in .venv, the GPU build of PyTorch that matches
my NVIDIA driver, then requirements.txt. Check that ffmpeg is in PATH and has the rubberband filter. Copy
.env.example to .env and tell me which keys to fill in; never print key values back to me.
Finish by running `python dub.py --help` and a GPU check.
```

**Dub my video with my voice**
```text
Dub input/<my-video.mp4> from <English> to <Spanish> with variant A (dub.py, --src <en> --tgt <es>), using my
ElevenLabs voice from ELEVENLABS_VOICE_ID and the eleven_v4 model. When it stops after translation, show me the
source and the translation side by side, flag anything that sounds unnatural or changed meaning, and wait for my
OK. Then finish the run and list the phrases that were shortened, sped up or still overlap.
```

**Add a glossary term**
```text
Add to glossary.json: keep "<Kubernetes>" untranslated; translate "<term>" as "<translation>"; Whisper writes it
as "<misheard form>", add an asr_fix; the voice mispronounces "<name>", add a pronounce entry. Then rerun
translate for input/<my-video.mp4> and show me only the phrases that changed.
```

**Switch to the free local variant**
```text
Set up variant C from README.md: Ollama with the model from variant_local.py (or one that knows <target language>),
a separate omni-env venv with requirements-omnivoice.txt, and check input/voice-sample.wav exists. Run extract,
transcribe and variant_local.py on input/<my-video.mp4>. Remind me that OmniVoice weights are non-commercial.
```
