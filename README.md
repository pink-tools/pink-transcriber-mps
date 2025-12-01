# Pink Transcriber MPS

Fast voice-to-text for Apple Silicon. Runs locally, no cloud, no API keys.

~15x faster than realtime on M4 Pro.

## Quick Start

```bash
git clone https://github.com/pink-tools/pink-transcriber-mps
cd pink-transcriber-mps
uv sync
uv run pink-transcriber-server
```

First run downloads the model (~2.4GB).

## Requirements

- macOS 12+ with Apple Silicon (M1–M5)
- Python 3.12 (exactly — NeMo requires it)
- [uv](https://docs.astral.sh/uv/)

## Usage

**Start server:**
```bash
uv run pink-transcriber-server
```

**Transcribe:**
```bash
uv run pink-transcriber audio.ogg
```

**Check health:**
```bash
uv run pink-transcriber --health
```

Supports: wav, ogg, mp3, m4a, flac, opus, aiff

## Customization

Adjust format, speed, output and more:
```bash
uv run pink-transcriber --help
```

## License

MIT
