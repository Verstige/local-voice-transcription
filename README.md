# Local Voice Transcription: No Cloud, No Upload

> Transcribe voice notes and recordings on the machine itself. Nothing leaves the device, and it works when the network does not.

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg) ![Claude Code](https://img.shields.io/badge/Claude_Code-skill-blueviolet.svg) [![Available on Agensi](https://img.shields.io/badge/Agensi-%245-ff5a5f.svg)](https://agensi.io/skills/local-voice-transcription-no-cloud-no-upload)

## What it does

Transcribe voice messages and recordings on the machine itself, with no cloud round trip and no upload.

Local speech-to-text is faster than the cloud path, works with no
network, and keeps the audio on the machine. The usual failure is a
provider configured for local inference with the model runtime missing,
which surfaces as an "unavailable" error rather than anything obvious.

## The failure it prevents

Working this by hand means rediscovering the same edge cases every time. They are written down here instead, with the fix for each one.

## When to use it

- transcribe a voice message with no network
- batch transcribe recordings on the machine
- keep audio from leaving the device

## What's inside

```
plugins/
└── local-voice-transcription/
    ├── .claude-plugin/plugin.json
    └── skills/local-voice-transcription/
        ├── SKILL.md
```

## Install

Add this repo as a Claude Code marketplace, then install the skill:

```bash
claude plugin marketplace add Verstige/local-voice-transcription
claude plugin install local-voice-transcription@local-voice-transcription
```

Or copy the skill straight into your skills directory:

```bash
git clone https://github.com/Verstige/local-voice-transcription.git
cp -R local-voice-transcription/plugins/local-voice-transcription/skills/local-voice-transcription ~/.claude/skills/
```

## Usage

The skill activates on the work it describes. Ask for the outcome, and it
supplies the method:

```
> "transcribe a voice message with no network"
```

## License

MIT. See [LICENSE](LICENSE).

---

**Also available as a paid skill on Agensi** — $5. Install it with one command, no clone required:

[agensi.io/skills/local-voice-transcription-no-cloud-no-upload](https://agensi.io/skills/local-voice-transcription-no-cloud-no-upload)

Part of the [Verstige](https://github.com/Verstige) skills portfolio. More at [github.com/Verstige](https://github.com/Verstige).
