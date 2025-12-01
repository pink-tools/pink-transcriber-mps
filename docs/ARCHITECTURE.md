# Architecture

Local voice-to-text transcription daemon for macOS Apple Silicon using NeMo Parakeet TDT v3 with MPS acceleration.

## Directory Structure

```
pink-transcriber-mps/
├── docs/
├── models/
│   └── huggingface/
│       └── hub/
│           └── models--nvidia--parakeet-tdt-0.6b-v3/
├── src/pink_transcriber/
│   ├── cli/
│   │   ├── client.py
│   │   └── server.py
│   ├── core/
│   │   └── model.py
│   ├── daemon/
│   │   ├── singleton.py
│   │   └── worker.py
│   ├── config.py
│   └── __init__.py
├── pyproject.toml
├── uv.lock
└── README.md
```

## Files

### `src/pink_transcriber/config.py`
Defines configuration constants: Unix socket path (`/tmp/pink-transcriber.sock`), supported audio formats, verbose mode flag, singleton process identifiers, and model cache directory resolution logic (environment variable > package directory > user home).

### `src/pink_transcriber/core/model.py`
Loads NeMo Parakeet TDT v3 model with device selection (CUDA > MPS > CPU), configures cache directories, suppresses NeMo logging, and provides synchronous transcription function that accepts audio file path and returns text.

### `src/pink_transcriber/daemon/singleton.py`
Ensures only one server instance runs by scanning all Python processes for project identifiers, finding root process trees, and killing entire trees (handles wrapper processes like `caffeinate` or `uv run`).

### `src/pink_transcriber/daemon/worker.py`
Implements async transcription worker with queue-based request handling: accepts requests via Unix socket, queues them, processes sequentially using executor for blocking model inference, and handles health checks (`HEALTH` command).

### `src/pink_transcriber/cli/server.py`
Server entry point: creates Unix socket, starts async worker with transcription queue, loads model in background executor, handles SIGINT/SIGTERM gracefully, and enforces singleton via `ensure_single_instance()`.

### `src/pink_transcriber/cli/client.py`
Client CLI: validates audio file format/existence, connects to Unix socket, sends absolute file path, receives transcription text, supports `--health` check, and provides `--version` info.

## Entry Points

```bash
# Transcribe audio file
pink-transcriber audio.ogg

# Health check
pink-transcriber --health

# Start daemon
pink-transcriber-server
```

## Key Concepts

**Client-Server Architecture** — Persistent daemon keeps model in memory (~2GB), clients connect via Unix socket at `/tmp/pink-transcriber.sock` and send audio file paths, avoiding model reload overhead.

**Async I/O with Sync Executor** — Server uses `asyncio` for socket handling, runs blocking model operations (`load_model`, `transcribe`) in `loop.run_in_executor(None, ...)` to avoid blocking event loop.

**Process Tree Singleton** — Instead of PID files, scans all processes for project identifiers (`pink-transcriber`, `pink_transcriber`, `Pink Transcriber`), climbs to root of process tree (handles wrappers), and kills entire tree recursively.

**Portable Model Cache** — Tries package `./models/` first (editable install), falls back to `~/.local/share/pink-transcriber/models`, respects `PINK_TRANSCRIBER_MODEL_DIR` environment variable override.

**Line-Delimited Protocol** — Request: `<audio_file_path>\n`, Response: `<transcription_text>\n` or `ERROR: <message>\n`, Health check: `HEALTH\n` → `OK\n` or `LOADING\n`.

**Device Selection** — Model loading tries CUDA (NVIDIA GPU) > MPS (Apple Silicon) > CPU fallback, server prints device on startup: `Ready (MPS)` or `Ready (CUDA)` or `Ready (CPU)`.

**Fail Fast Errors** — Model loading failures print structured error messages with troubleshooting steps and exit immediately. Transcription errors propagate to client as `ERROR:` prefix in response.

**Verbose Mode** — Set `VERBOSE=1` environment variable to enable detailed logging (model loading, requests, timing). Default mode prints minimal output for production use.

**Performance** — Parakeet TDT v3 (600M parameters) runs ~15x realtime on M4 Pro with ~2GB memory footprint and <1 second latency for 10-second audio.

**Runtime Requirements** — Python 3.12 (exact version required by NeMo), nemo_toolkit[asr]==2.5.0, torch==2.9.0 with MPS support, soundfile==0.13.1 for audio I/O, setproctitle and psutil for process management.
