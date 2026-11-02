# Yoto ↔ Audiobookshelf Bridge

> [!NOTE]
> **Fork Notice & Upstream Credits:**  
> This repository is an enhanced fork of the original [mylacc/yoto-audiobookshelf-bridge](https://github.com/mylacc/yoto-audiobookshelf-bridge). It extends the original bridge architecture with a comprehensive machine-to-machine REST API (`/api/*`), dual-layer token authentication (`api_key` & `ui_pin`), external ESP32 middleware broker integration, decoupled per-item 16×16 pixel art caching, multi-player MQTT telemetry streaming, native Discord notification webhooks with 48-hour rolling pruning, and pre-built GHCR Docker images.

A lightweight, high-performance bridge daemon that connects your self-hosted [Audiobookshelf](https://www.audiobookshelf.org/) server with physical [Yoto Players](https://my.yotoplay.com/). It allows physical Make Your Own (MYO) cards to play your personal audiobook and podcast library directly on Yoto players with seamless, bidirectional progress synchronization between the Yoto and your mobile Audiobookshelf apps.

- **Physical MYO Cards:** Tap an NFC card to play an Audiobookshelf book from wherever you left off, whether on the Yoto or in the ABS phone app. Pull the card out and your listening position saves back to Audiobookshelf.
- **Phone Web App:** Save the mobile-optimized web UI to your phone's home screen. Browse "Continue listening" and your entire library. Tap **"▶ <player>"** to start any book on any player instantly without needing a physical card.
- **Audio Streams from Your Server:** Players fetch audio directly from your bridge on your local LAN or through a secured public HTTPS address. Audio is never stored on third-party servers. Yoto's developer cloud is used solely to link cards to playlists, issue play/pause commands, and receive MQTT position telemetry.

---

## 📚 Documentation & Official Wiki

Comprehensive documentation, architectural breakdowns, setup walkthroughs, and troubleshooting guides are maintained in the [Yoto-Audiobookshelf-Bridge GitHub Wiki](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki):

### Getting Started & Installation
- **[Wiki Home](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki)**: High-level architectural overview, system diagrams, and core concepts.
- **[Installation & Docker Compose](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/Installation-and-Docker-Compose)**: Container deployment, volume mounts, `config.yaml` options, and pre-built GHCR images.
- **[Audiobookshelf Configuration Guide](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/Audiobookshelf-Configuration)**: Master API keys, multi-account per-child segregation (`devices:`), and library access control.
- **[Yoto Developer App & OAuth Setup](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/Yoto-Developer-App-&-OAuth-Setup)**: Registering on `dashboard.yoto.dev`, configuring RFC 8252 callback URLs, and OAuth PKCE handshakes.

### Playback, Transcoding & Advanced Cards
- **[Series & Collection Cards](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/Series-and-Collection-Cards)**: Continuous multi-book series progression, collection playback, and dynamic book selection.
- **[Audio Transcoding, Speed & Quiet-Cut Snapping](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/Audio-Transcoding-&-Quiet-Cut-Snapping)**: On-the-fly `ffmpeg` transcoding (64k CBR mono MP3), speech pause snapping (`quiet.json`), pitch-preserved speed adjustments (`atempo`), and bedtime cards that keep their own place (`card_progress.json`).
- **[Chapter Markers & Audible Lookup Engine](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/Chapter-Markers-and-Audible-Lookup)**: Automated library scanning, Audible ASIN matching, and Whisper STT speech-to-text chapter extraction (`tools/audio_chapters.py`).
- **[Offline Cards & Storage Architecture](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/Offline-Cards-&-Cache-Management)**: Cloud-uploaded offline copies for travel (car trips, flights) and in-memory cloud uploading.

### Networking, Pixel Art & Integrations
- **[Reverse Proxy & SSL Publishing](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/Reverse-Proxy-&-SSL-Publishing)**: Publishing `/s/` securely over HTTPS via NGINX Proxy Manager or Cloudflare Tunnels for remote Wi-Fi streaming, buffering optimization, and LAN isolation.
- **[16x16 Pixel Art Engine](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/16x16-Pixel-Art-Engine)**: Item-level artwork caching (`data/icons/items/<item_id>/`), dynamic chapter icons, retro themes, and AI image synthesis (Claude / Whisper / Ollama).
- **[REST & Streaming API Specification](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/REST-API-Specification)**: Full endpoint reference (`/s/*`, `/api/*`) for external brokers (ESP32), Home Assistant automations, and third-party dashboards.
- **[Production Operations & Troubleshooting](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/Production-Operations-and-Troubleshooting)**: Diagnostic logs, moving old cards with uploaded audio, Discord notifications (48h rolling log), token recovery, and known limits.
- **[Fork Comparison & Architectural Enhancements](https://github.com/alicia86/yoto-audiobookshelf-bridge/wiki/Fork-Comparison-&-Enhancements)**: Detailed technical comparison against upstream (`mylacc`), feature breakdown matrix, and deep architectural modifications.

---

## ⚡ Quick Start

### 1. Requirements
- Docker host (Linux/Debian, Raspberry Pi, Unraid, TrueNAS, etc.)
- Audiobookshelf server (v2.26+ recommended)
- Yoto Developer Account ([dashboard.yoto.dev](https://dashboard.yoto.dev))

### 2. Docker Compose Deployment
```yaml
services:
  yoto-bridge:
    image: ghcr.io/alicia86/yoto-audiobookshelf-bridge:latest
    container_name: yoto-bridge
    restart: unless-stopped
    ports:
      - "8090:8090"
    environment:
      - TZ=America/New_York
      - LOG_LEVEL=INFO
    volumes:
      - ./data:/app/data
```

### 3. Basic Configuration (`data/config.yaml`)
```yaml
abs_url: "http://your-abs-server:13378"
abs_api_key: "YOUR_ABS_MASTER_API_KEY"
public_url: "https://yoto.yourdomain.com" # Or http://<lan-ip>:8090
yoto_client_id: "YOUR_YOTO_DEVELOPER_CLIENT_ID"

# Security & API Protection
api_key: "CHANGE_ME_TO_A_32_CHAR_HEX_KEY"
ui_pin: "1234" # Optional PIN protection for card edits in the phone web UI
```

Start the container:
```sh
docker compose up -d
```
Then visit `http://<host-ip>:8090` in your web browser and click **Connect Yoto** to pair your player account.

---

## 🔐 Dual-Layer Security Model

The bridge implements dual-layer security tailored for both local family browsing and automated smart home clients:
- **Master API Key (`api_key`)**: Protects all `/api/*` management endpoints against unauthorized external requests. External automations (Home Assistant, ESP32 Broker, scripts) authenticate via `X-Api-Key: <key>`, `Authorization: Bearer <key>`, or `?apikey=<key>`.
- **Phone Web UI PIN (`ui_pin`)**: When set, prompts for a PIN on destructive actions in the phone browser (reassigning cards, wiping slots, initiating offline copies) while permitting smooth playback browsing.
- **Stream Slot Protection (`/s/`)**: Stream and icon URLs are keyed with random 72-bit cryptographic slot identifiers, allowing physical players to fetch streams without exposing management interfaces.

---

## 🛠️ Development & Local Testing

The frontend is built with React, Vite, Tailwind CSS 4, and shadcn/ui components (`web/`), compiling directly into static assets served by the FastAPI/Python backend:

```sh
# Run pytest test suite (requires ffmpeg on PATH):
pip install -r requirements.txt pytest pytest-asyncio
python -m pytest

# Run locally in debug mode:
LOG_LEVEL=DEBUG BRIDGE_CONFIG=data/config.yaml python -m yotobridge
```

---

## 🤖 AI Development Notice

This repository and its architectural enhancements utilize artificial intelligence (AI) tools, including Anthropic Claude and Google Gemini / Antigravity, to assist in REST API engineering, security token middleware, pixel art algorithms, asynchronous streaming pipelines, and documentation. All AI-assisted changes are verified against test suites, mock servers, and physical Yoto hardware.

---

## ⚖️ Legal & Disclaimer

**Unofficial.** Not affiliated with or endorsed by Yoto. Works with Yoto via the official [developer API](https://yoto.dev). Each user registers their own client app and accepts Yoto's developer terms. Audiobooks and media remain the property of their respective creators; only stream audio you have the legal right to access.
