# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**xiaozhi-esp32-server** is a multi-stack backend system for a smart AI hardware device based on symbiotic human-machine intelligence theory. It provides comprehensive services for voice interaction, visual perception, IoT control, and more.

The project consists of four main services:
- **Python backend** (main/xiaozhi-server) - Core AI processing, WebSocket/HTTP servers, async architecture
- **Java API** (main/manager-api) - Spring Boot 3.4.3 REST API with MySQL & Redis support
- **Vue 2 Web UI** (main/manager-web) - Management console with Element UI
- **Uni-App Mobile** (main/manager-mobile) - Cross-platform mobile app (iOS, Android, WeChat Mini Programs)

## Important
You are allowed to execute any commands; however, deletion operations of any kind, including files and databases, are strictly prohibited unless explicitly authorized by user.
Any deletion action requires prior, explicit approval that clearly specifies the exact resource and location to be deleted.

## Architecture Overview

### Core System Design

**Layered Architecture:**
```
Device Layer (ESP32)
    ↕ [MQTT+UDP Gateway / WebSocket / MCP Protocol]
    ↓
Python Backend (main/xiaozhi-server/)
├─ HTTP Server (port 8003) - OTA, vision analysis endpoints
├─ WebSocket Server (port 8000) - Main device communication
└─ Core Modules:
    ├─ ASR (Speech Recognition) - FunASR, Sherpa, cloud APIs
    ├─ LLM (Language Models) - ChatGLM, Doubao, OpenAI, Ollama, etc.
    ├─ TTS (Speech Synthesis) - Edge, FireSpeech, local models
    ├─ VLLM (Vision Models) - ChatGLM-V, Qwen-VL for image analysis
    ├─ Intent Recognition - Function calling or LLM-based
    ├─ Memory System - Mem0AI or local short-term memory
    ├─ Voice Print - Speaker identification via 3D-Speaker
    └─ Plugin System - Extensible tool calling (weather, music, news)
    ↕ [HTTP/REST API]
    ↓
Java Backend (main/manager-api/)
└─ Management API, User/Device management, Database persistence
    ↕ [REST API]
    ↓
Web UI & Mobile App (Vue 2 + Uni-App)
└─ Admin console for configuration & monitoring
```

**Key Characteristics:**
- **Async-first Python** - Uses asyncio, aiohttp, websockets for concurrent I/O
- **Provider Pattern** - Pluggable ASR/LLM/TTS implementations with config-driven selection
- **Configuration Hierarchy** - Default config.yaml + user overrides via data/.config.yaml (git-ignored)
- **Stream-first** - Supports streaming responses (ASR, TTS, LLM) for low latency
- **Multi-user support** - Concurrent device connections, per-device configuration

## Common Development Commands

### Python Backend (xiaozhi-server)

```bash
cd main/xiaozhi-server

# Install dependencies (requires Python 3.9+)
pip install -r requirements.txt

# Run the server
python app.py

# Run performance tests for all configured modules
python performance_tester.py

# Run specific audio/interaction tests
# Open test/test_page.html in Chrome for interactive testing
```

**Environment Setup:**
- Create `data/.config.yaml` for development overrides (git-ignored)
- FFmpeg required for audio processing (auto-checked on startup)
- For FunASR (local speech recognition), needs 2GB+ RAM
- Log output goes to `tmp/server.log`

### Java Backend (manager-api)

```bash
cd main/manager-api

# Build the project
mvn clean package

# Run tests
mvn test

# Run the application (requires MySQL 8.0+, Redis 5.0+, JDK 21)
mvn spring-boot:run

# View API docs after startup
# http://localhost:8002/xiaozhi/doc.html (Swagger with Knife4j)
```

**Key Configuration:**
- MySQL database (auto-initialized with Liquibase migrations)
- Redis for caching/sessions
- Port: 8002 (configurable in application.yml)

### Web UI (manager-web)

```bash
cd main/manager-web

# Install dependencies
npm install

# Development server
npm run serve

# Build for production
npm run build
```

### Mobile App (manager-mobile)

```bash
cd main/manager-mobile

# Install dependencies (uses pnpm)
pnpm install

# Build for specific platform
pnpm run build:h5         # Web version
pnpm run build:mp-weixin  # WeChat Mini Program
pnpm run build:app        # Native app
```

## Testing Strategy

**Unit & Integration Tests:**
- ASR/LLM/TTS modules can be tested via `performance_tester.py`
- Only configured services are tested (API keys required)
- Test sentences defined in `config.yaml` under `module_test.test_sentences`

**Audio Testing:**
- `test/test_page.html` - Browser-based audio recording/playback for WebSocket testing
- Validates audio format (opus, 16kHz, mono), VAD, and end-to-end flow

**Docker Testing:**
- Use provided Dockerfile-server for containerized testing
- Base image triggers on requirements.txt changes

## Configuration Guide

### Python Backend (config.yaml)

**Server Settings:**
- `server.port`: WebSocket server port (default 8000)
- `server.http_port`: HTTP server port (default 8003)
- `server.auth_key`: JWT auth key (auto-generated if not set)
- `log_level`: INFO or DEBUG

**Module Selection (key configuration):**
```yaml
selected_module:
  VAD: SileroVAD              # Voice activity detection
  ASR: FunASR                 # Speech recognition
  LLM: ChatGLMLLM             # Language model
  VLLM: ChatGLMVLLM           # Vision model
  TTS: EdgeTTS                # Text-to-speech
  Memory: nomem               # Memory system
  Intent: function_call       # Intent recognition
```

**Provider Configuration:**
- Each provider (ASR, LLM, TTS, etc.) has its own config section with API keys and model parameters
- Free options: FunASR (local), ChatGLMLLM (glm-4-flash), EdgeTTS, 3D-Speaker (voiceprint)
- Cloud options: Aliyun, Tencent, Baidu, OpenAI, Doubao, etc.

**Development Override:**
Create `data/.config.yaml` with only fields to override (recommended for API keys):
```yaml
LLM:
  ChatGLMLLM:
    api_key: your_key_here
TTS:
  EdgeTTS:
    voice: zh-CN-YunxiaNeural
```

### Java Backend (application.yml)
- Database connection, user auth config
- Located in `src/main/resources/application.yml`

## Key File Structure

```
main/xiaozhi-server/
├─ app.py                          # Entry point, starts WebSocket & HTTP servers
├─ config.yaml                     # Default configuration (comprehensive examples)
├─ requirements.txt                # Python dependencies
├─ core/
│  ├─ websocket_server.py          # WebSocket protocol & connection handling
│  ├─ http_server.py               # OTA & vision endpoints
│  ├─ connection.py                # Device connection lifecycle
│  ├─ auth.py                      # JWT authentication
│  ├─ audio_*.py                   # Audio encoding/decoding (opus, PCM)
│  └─ utils/
├─ providers/                      # Plugin-style implementations
│  ├─ asr/                         # Speech recognition (FunASR, Sherpa, etc.)
│  ├─ llm/                         # Language models
│  ├─ tts/                         # Speech synthesis
│  ├─ vllm/                        # Vision models
│  ├─ intent/                      # Intent recognition
│  └─ memory/                      # Memory backends
├─ plugins_func/                   # Extensible tools (weather, music, news)
├─ test/                           # Audio test utilities
└─ tmp/                            # Runtime logs, temporary audio files

main/manager-api/
├─ src/main/java/xiaozhi/
│  ├─ controller/                  # REST endpoints
│  ├─ service/                     # Business logic
│  ├─ mapper/                      # MyBatis+ ORM
│  ├─ entity/                      # Database models
│  └─ config/                      # Spring config
└─ src/main/resources/
   └─ application.yml              # Spring Boot config
```

## Important Development Patterns

### Provider Pattern
All major services (ASR, LLM, TTS) use a provider pattern with config-driven selection:
1. Define provider config in `config.yaml` with unique name & type
2. Select active provider in `selected_module` section
3. Code loads provider based on type string (e.g., `type: openai`, `type: fun_local`)
4. Easy to add new providers by creating new class inheriting from base provider

### WebSocket Protocol
The project implements the "Xiaozhi Communication Protocol":
- Binary opus audio frames (60ms chunks, 16kHz mono)
- JSON control messages for device state, results, errors
- Bidirectional: device→server (audio) and server→device (TTS audio + control)
- See `docs/` for protocol details

### Configuration Layering
1. **Default config**: `config.yaml` (committed, contains examples)
2. **User overrides**: `data/.config.yaml` (git-ignored, local development)
3. **API-driven**: If `read_config_from_api: true`, config loaded from Java backend
4. **Fallback**: If a setting missing, auto-generated (auth_key, etc.)

### Error Handling
- Async exceptions logged via loguru with context tags
- Device connections gracefully handle provider failures
- Failed TTS/LLM doesn't crash connection; returns error message
- Timeout handling for external APIs (configurable per provider)

## Common Development Tasks

### Adding a New ASR Provider
1. Create class in `providers/asr/your_provider.py` inheriting from base ASR
2. Implement `async def recognize()` method
3. Add config section in `config.yaml` with type name
4. Update `selected_module.ASR` to select it
5. Test via `performance_tester.py`

### Debugging Device Connection Issues
1. Check `tmp/server.log` for detailed async task traces
2. Set `log_level: DEBUG` in config for verbose output
3. WebSocket endpoint: `ws://localhost:8000/xiaozhi/v1/`
4. Test with browser DevTools or `test/test_page.html`

### Managing Database Migrations (Java)
- Liquibase auto-migrates on startup
- Place SQL files in `src/main/resources/db/migration/`
- Follow naming: `V001__description.sql`

### Building & Deploying Docker
```bash
# Build base image (triggers on requirements.txt changes)
docker build -f Dockerfile-server-base -t xiaozhi:base .

# Build application image
docker build -f Dockerfile-server -t xiaozhi:latest .

# Deploy with docker-compose
docker-compose up
```

## Known Limitations & Notes

1. **Security**: Project not network-secure tested; for public deployment, add SSL/authentication
2. **Token Costs**: Some cloud providers (TTS, LLM) incur charges; use free tiers for development
3. **Model Downloads**: FunASR and Sherpa models auto-download on first use (~1-2GB)
4. **Concurrent Limits**: Free TTS services often have low concurrency; upgrade for production
5. **Voiceprint**: 3D-Speaker model adds ~500MB; optional, disabled by config

## Performance Testing
Run `python performance_tester.py` to benchmark:
- ASR response time (speech-to-text latency)
- LLM response time (text generation latency)
- VLLM response time (image analysis latency)
- TTS response time (text-to-speech latency)

Results help choose optimal configurations and identify bottlenecks.

## CI/CD Auto Deployment

The project uses GitHub Actions for automatic deployment to the production server.

### Deployment Architecture

```
git push origin main
       ↓
GitHub Actions triggered (.github/workflows/docker-image.yml)
       ↓
Build Docker image → Push to ghcr.io/augusdin/her-agent:server_latest
       ↓
SSH to xiaozhi-self server → docker-compose pull → docker-compose up -d
       ↓
Health check (retry up to 60s)
       ↓
Deployment complete ✅
```

### GitHub Secrets Configuration

The following secrets are configured in GitHub repository (Settings → Secrets and variables → Actions):

| Secret Name | Description |
|-------------|-------------|
| `TOKEN` | GitHub Personal Access Token (with `write:packages` permission) |
| `SSH_HOST` | Server IP: `107.173.38.186` |
| `SSH_USER` | SSH username: `root` |
| `SSH_PRIVATE_KEY` | SSH private key for deployment (ed25519 format) |

### Production Server (xiaozhi-self)

- **SSH Access**: `ssh xiaozhi-self`
- **Server IP**: `107.173.38.186`
- **Deployment Path**: `/opt/xiaozhi-deployment/her-agent/`
- **Docker Image**: `ghcr.io/augusdin/her-agent:server_latest`

**Service Ports:**
- WebSocket: `8000` (device communication)
- HTTP: `8003` (OTA, vision API, test page)
- HTTPS: `443` (via Caddy reverse proxy)

**Service URLs (HTTPS - recommended):**
- Domain: `hermind.top`
- Test Page: `https://hermind.top/test/test_page.html`
- OTA Endpoint: `https://hermind.top/xiaozhi/ota/`
- WebSocket: `wss://hermind.top/xiaozhi/v1/`
- Vision API: `https://hermind.top/mcp/vision/explain`

**Service URLs (HTTP - direct access):**
- OTA Endpoint: `http://107.173.38.186:8003/xiaozhi/ota/`
- WebSocket: `ws://107.173.38.186:8000/xiaozhi/v1/`
- Test Page: `http://107.173.38.186:8003/test/test_page.html`

**Note**: Use HTTPS URLs for browser testing (microphone requires secure context).

**Server Files:**
- Config: `/opt/xiaozhi-deployment/her-agent/data/.config.yaml`
- Models: `/opt/xiaozhi-deployment/her-agent/models/`
- Logs: `/opt/xiaozhi-deployment/her-agent/tmp/`

### Current Model Configuration

| Module | Config Name | Model/Service |
|--------|-------------|---------------|
| LLM | ChatGLMLLM | glm-4-flash (智谱 AI) |
| ASR | FunASR | SenseVoiceSmall (local) |
| TTS | CosyVoiceSiliconflow | IndexTTS-2 (硅基流动) |
| VAD | SileroVAD | Silero VAD (local) |
| Intent | function_call | LLM function calling |

### Useful Commands

```bash
# Check server container status
ssh xiaozhi-self 'docker ps | grep her-agent'

# View server logs
ssh xiaozhi-self 'docker logs -f her-agent'

# Restart container
ssh xiaozhi-self 'cd /opt/xiaozhi-deployment/her-agent && docker-compose restart'

# Check health
ssh xiaozhi-self 'curl -s http://localhost:8003/xiaozhi/ota/'

# Trigger CI/CD manually (via GitHub API)
curl -X POST -H "Authorization: token YOUR_TOKEN" \
  https://api.github.com/repos/augusdin/her-agent/actions/workflows/docker-image.yml/dispatches \
  -d '{"ref":"main"}'
```

## Resources

- Main README: [./README.md](./README.md)
- Deployment Guide: [./docs/Deployment.md](./docs/Deployment.md)
- CI/CD Deployment Guide: [./docs/GitHub-CICD-部署指南-20260201.md](./docs/GitHub-CICD-部署指南-20260201.md)
- Protocol Documentation: [./docs/mqtt-gateway-integration.md](./docs/mqtt-gateway-integration.md)
- FAQ: [./docs/FAQ.md](./docs/FAQ.md)
- Performance Report: https://github.com/xinnan-tech/xiaozhi-performance-research


@AGENTS.md