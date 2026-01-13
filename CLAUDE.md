# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AI-XiaoPi is an open-source AI livestream robot system that integrates Douyin (TikTok China) danmaku (bullet comments/chat) with LLM responses and TTS audio output. It supports ESP32 hardware devices for physical robot interactions.

## Common Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Run the bot (main entry point)
python run_bot.py

# Alternative startup scripts
./tools/start_danmaku.sh   # Linux/macOS
tools/start_danmaku.bat    # Windows

# Test tools (in tools/ directory)
python tools/test_client.py          # Basic client test
python tools/test_device_client.py   # Hardware device client test
python tools/test_edge_tts.py        # TTS test
python tools/test_full_flow.py       # Full pipeline test
python tools/test_ota.py             # OTA update test
```

## Architecture

### Core Pipeline Flow
```
Danmaku Source → Collector → Handler → LLM → TTS → Device Manager → ESP32 Hardware
```

### Key Components

**Entry Point:**
- `run_bot.py` - Main entry, loads `configs/danmaku_config.yaml` and starts `DanmakuService`

**Core Services (`core/`):**
- `danmaku/service.py` - Main service orchestrator, manages WebSocket server (port 8001), HTTP server (port 8003 for OTA), and component lifecycle
- `danmaku/handler.py` - Processes danmaku messages serially, manages LLM calls and TTS queue
- `danmaku/device_manager.py` - Manages connected ESP32 devices and audio broadcast
- `live/douyin_proxy_collector.py` - Connects to DouyinBarrageGrab for real danmaku
- `live/douyin_collector.py` - Direct/mock danmaku collection

**Provider System (`core/providers/`):**
- `llm/` - LLM adapters (OpenAI, ChatGLM, Gemini, Ollama, Dify, FastGPT, Coze)
- `tts/` - TTS adapters (EdgeTTS, Aliyun, Doubao, custom)
- `asr/` - ASR adapters (Aliyun, Baidu, Doubao, Vosk, etc.)
- `memory/` - Memory providers (mem0ai, local short-term)
- `intent/` - Intent recognition (function call, LLM-based)
- `tools/` - MCP tools and device IoT integration

**Adding New Providers:**
Extend `LLMProviderBase` (llm/base.py) or `TTSProviderBase` (tts/base.py). Key methods:
- LLM: `response(session_id, dialogue)` - generator yielding text chunks
- TTS: `text_to_speak(text, output_file)` - async method returning audio bytes

### Configuration

**Main config:** `configs/danmaku_config.yaml`
- `danmaku.use_mock` - Use simulated danmaku (dev mode)
- `danmaku.use_proxy` - Use DouyinBarrageGrab proxy (recommended for production)
- `danmaku.proxy_ws_url` - DouyinBarrageGrab WebSocket URL
- `danmaku.flow_control_enabled` - Enable rate limiting (recommended)
- `selected_module.LLM` / `selected_module.TTS` - Active provider names
- `LLM.<name>` / `TTS.<name>` - Provider-specific configs

### Message Processing

The danmaku handler uses serial processing with flow control:
1. Danmaku enters queue via `add_danmaku()`
2. `_process_danmaku()` calls LLM with dialogue history
3. LLM response chunks are sent to TTS queue as `TTSMessageDTO`
4. TTS generates Opus audio frames
5. `AudioRateController` manages playback timing
6. Audio broadcasts to all connected devices via `DeviceManager`

### Hardware Protocol

ESP32 devices connect via WebSocket to `/danmaku/?device-id=xxx`. The server sends:
- `hello` response with audio params (Opus, 16kHz, mono, 60ms frames)
- TTS control messages (`start`, `sentence_start`)
- Binary Opus audio frames

### Directory Structure

```
configs/          - Configuration files and loaders
core/
  danmaku/        - Danmaku service and handlers
  handle/         - Message handlers (text, audio, abort)
  live/           - Livestream collectors
  providers/      - Pluggable AI service providers
  utils/          - Utilities (audio, text, dialogue)
  api/            - HTTP API handlers
plugins/          - Function plugins for LLM
tools/            - Development and test utilities
data/             - Runtime data (requires .config.yaml)
tmp/              - Temporary audio files
```
