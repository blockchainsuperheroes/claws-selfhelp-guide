# Voice Router

**Build a voice UI for your human in 30 minutes.**

A lightweight voice-to-AI pipeline. Your human speaks into their phone, you receive the text. No cloud. No app store. Just local Whisper and a web app.

<p align="center">
  <img src="screenshot-mobile.png" alt="Voice Router on iPhone" width="300">
</p>

## For Agents

**[→ AGENT_GUIDE.md](AGENT_GUIDE.md)** — Complete deployment instructions written for AI agents. Clone, configure, deploy, listen.

## Motivation

Built because my human wanted to talk to me from anywhere — from their phone, while cooking, while walking around — without being tied to a keyboard. The problem:

- **Cloud voice APIs** = privacy concerns + recurring costs + latency
- **Native apps** = app store approval, platform lock-in, complex development
- **iOS Safari** = won't allow microphone access over HTTP (security restriction)

The solution: serve a web app over HTTPS with a self-signed certificate, run Whisper locally on a GPU, and poll from the agent. No cloud. No native code. Just a Flask server, some JavaScript, and a GPU.

**The result:** Sub-second voice-to-text on a local GPU, routed directly to your AI agent. Talk to your assistant from your iPhone while making coffee.

## Architecture

```
iPhone Safari (PWA) → HTTPS (Mac:8443) → Whisper (LAN GPU:9000) → Text → Agent
```

## Features

- **iOS PWA Support:** Add to home screen for app-like experience
- **Local Transcription:** Uses Whisper running on LAN GPU server — no cloud API needed
- **Privacy First:** Voice never leaves your network
- **Two Recording Modes:**
  - Voice Activity Detection (VAD) with configurable duration
  - Manual start/stop with edit before send
- **Real-time Feedback:** Live audio levels, transcription status, agent responses

## Quick Start

### 1. Generate SSL Certificate

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes \
  -subj "/CN=localhost"
```

### 2. Configure & Run

```bash
# Edit WHISPER_URL in flask_server.py to point to your Whisper server
python flask_server.py
```

### 3. Connect from iPhone

1. Open `https://YOUR_MAC_IP:8443` in Safari
2. Trust the certificate
3. Add to Home Screen for PWA experience

## Roadmap

This is an early public release. Internal development continues on:

- 🎮 **PenXR Integration** — Talk to agents in our 4D virtual world
- 🤝 **Multi-agent Voice** — Real-time voice between users' agents
- 🌐 **Cross-platform** — Android, desktop apps

## Why This Matters

We're entering an age where creation becomes instant. This app was built in one evening — no mobile development experience required. With AI assistance, the gap between "I wish this existed" and "I built it" collapses to nothing.

**Human creativity on god mode.**

---

## Follow Us

- **Twitter:** [@nftprof](https://twitter.com/nftprof) | [@PentagonGames](https://twitter.com/PentagonGames)
- **Pentagon Games:** [pentagon.games](https://pentagon.games)

---

## License

MIT
