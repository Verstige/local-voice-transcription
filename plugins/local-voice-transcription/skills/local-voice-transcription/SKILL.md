---
name: local-voice-transcription
description: "Transcribe voice messages and recordings on the machine itself, with no cloud round trip and no upload."
version: 1.0.0
license: MIT
author: "the maintainers"
---
# Local voice transcription (faster-whisper)

Local speech-to-text is faster than the cloud path, works with no
network, and keeps the audio on the machine. The usual failure is a
provider configured for local inference with the model runtime missing,
which surfaces as an "unavailable" error rather than anything obvious.

## Fast path (proven)

```bash
VENV=$HOME/.cache/fw-venv
uv venv --python 3.12 $VENV
uv pip install --python $VENV/bin/python3 faster-whisper
uv pip install --python $VENV/bin/python3 'av==17.0.1'   # CRITICAL pin
$VENV/bin/python3 $TX_SCRIPT base.en <audio.ogg>
```

Transcribe script (save as `$TX_SCRIPT`):

```python
import sys
from faster_whisper import WhisperModel
m = WhisperModel(sys.argv[1], device="cpu", compute_type="int8")
segs, info = m.transcribe(sys.argv[2], beam_size=5, vad_filter=True)
print("LANG:", info.language, round(info.language_probability,3))
for s in segs:
    print(f"[{s.start:.1f}-{s.end:.1f}] {s.text.strip()}")
```

## Pitfalls (each one cost a round-trip)

1. **`av` MUST be pinned to `==17.0.1`.** `faster-whisper` 1.2.1 calls
   `av.open(..., metadata_errors="ignore")`. `av` 19.0.2 removed that kwarg
   → `TypeError: open() got an unexpected keyword argument 'metadata_errors'`.
   `uv pip install av` grabs 19 and breaks it. Pin 17.0.1.
2. **`av` is not a direct dep of faster-whisper** in some uv resolutions —
   it must be installed explicitly or `ModuleNotFoundError: No module named 'av'`.
3. **Do not try PYTHONPATH-into-uv-cache.** The `faster_whisper` dir exists in
   `~/.cache/uv/archive-v0/*/` but its `av` sibling is not resolvable that way.
   A real venv is faster than debugging it.
4. **Model:** `base.en` for English. The HF cache already has
   `models--Systran--faster-whisper-base`. On CPU use `compute_type="int8"`.
5. **`whisper-cli` exists on this box** (`/usr/local/bin/`) but the only GGML
   model is `for-tests-ggml-tiny.bin` (575 KB, not a real model) — don't go
   down the whisper.cpp path.

## The real fix: wire it into Hermes as `local_command`

Hermes' built-in `local` provider imports faster-whisper INSIDE the Hermes
Python runtime (which doesn't have it). The documented escape hatch is
`HERMES_LOCAL_STT_COMMAND` + `stt.provider: local_command`.

**Contract** (see `tools/transcription_local.py`):
- Hermes formats your template with `{input_path}`, `{output_dir}`,
  `{language}`, `{model}` — so the placeholders are substituted by HERMES,
  not by bash. Your script receives them as `$1..$4`.
- Hermes then reads back the FIRST `*.txt` in `{output_dir}`. You must write
  a `.txt` there or it errors with "did not produce a .txt transcript".

**Files (already installed on this machine):**
- `~ maintainer/.hermes/bin/fw-transcribe.sh` — the command script
- `~ maintainer/.hermes/bin/fw-transcribe.py` — the faster-whisper runner
- `<hermes-home>/.env` → `HERMES_LOCAL_STT_COMMAND=~ maintainer/.hermes/bin/fw-transcribe.sh {input_path} {output_dir} {language} {model}`
- `<hermes-home>/config.yaml` → `stt.provider: local_command`

**CRITICAL:** in the wrapper, `unset PYTHONPATH` before invoking the venv
python. Hermes exports a PYTHONPATH of its own uv dependency archives, whose
numpy is built for a different interpreter — it leaks in and breaks
`import numpy._core._multiarray_umath` inside the venv.

## Reporting

Show the transcript with timestamps, then answer the question it contains.
Don't make the user re-ask.

## The cached audio is ephemeral — transcribe at arrival

The path in a `[voice message could not be transcribed automatically]` notice
points into `<hermes-home>/cache/documents/native-inputs/<hash>/audio_<id>.ogg`.
That directory is **pruned**. A message you could not transcribe an hour ago
may have no audio left by the time you go back for it — the notice survives
in the transcript, the file does not.

Rules that follow:

- **Transcribe the voice message in the turn it arrives.** Do not defer it to
  "when the user asks about it later" — the audio may be gone by then, and
  the content becomes permanently unrecoverable.
- **Before running the transcriber, confirm the file still exists** and
  report the size. The wrapper is silent on a missing input and exits
  non-zero with no message, which reads as a broken transcriber rather than a
  missing file.
- **When the audio is gone, say that plainly.** "The audio for that message
  was pruned from the cache before I could transcribe it" is honest and
  actionable (ask them to resend). Claiming transcription failure because
  the script produced nothing is a false diagnosis that will send the next
  session chasing a working tool.
- **A voice message whose transcript you never obtained still exists as
  `[voice message could not be transcribed automatically]` in the session
  history.** Find it with `session_search` when the user references something
  they sent by voice, and say which part is missing rather than guessing at
  the content.


## Related

After transcribing, if the content references Hermes behavior, verify against
the source in `<hermes-home>/hermes-agent` before answering — don't trust the
video the user watched.
