# H.E.N.R.Y.

An always-on, self-hosted voice desk assistant for Raspberry Pi with a graph-based memory of your habits and context.

## About

H.E.N.R.Y. is a conversational desk companion you talk to continuously while you work. Instead of one-shot commands, it keeps a persistent, graph-based understanding of your ideas, todos, schedule, and preferences, so it has context the next time you speak. It runs on your own hardware in a distributed setup: the heavy compute (LLM, speech-to-text, graph database) lives on a home server, and a Raspberry Pi handles the always-on voice interface and GUI.

## Install

H.E.N.R.Y. uses a distributed architecture. The **home server** runs the backend in Docker; the **Raspberry Pi** runs the voice + GUI client under systemd. Both are managed from a single Poetry project via optional extras (`server`, `pi`).

### Prerequisites

- **Server**: Linux host with Docker + Docker Compose, plus reachable [Neo4j](https://neo4j.com/) and [Ollama](https://ollama.com/) instances. Whisper runs inside the container.
- **Raspberry Pi**: Pi 4B / Pi 5 running Raspberry Pi OS, Python 3.9–3.11 (not 3.12+, required by OpenWakeWord's `tflite-runtime`), and configured audio in/out.
- A shared network between them ([Tailscale](https://tailscale.com/) recommended).

### Local development

```bash
git clone https://github.com/pranavkannepalli/H.E.N.R.Y..git
cd H.E.N.R.Y.

# Install Poetry, then project deps
curl -sSL https://install.python-poetry.org | python3 -
poetry install

# Configure environment
cp .env.example .env.local   # edit Neo4j / Ollama URLs, AUDIO_ENABLED, wake word, etc.

# Run the API dev server (auto-reload, graceful shutdown)
poetry run python scripts/dev_server.py

# Or start the full local stack (API + combined GUI/voice app)
bash scripts/dev_run_all.sh
```

### Server deployment (Docker)

```bash
cp .env.server.example .env.server   # STT_ENGINE=whisper, Neo4j + Ollama connection
cp .env.deploy.example .env.deploy   # SERVER_USER / SERVER_HOST / SERVER_PATH

bash scripts/deploy_to_server.sh
```

The script rsyncs the project to the server, then runs `docker compose build` / `up -d` (service `henry-server`, port `8000`, `restart: unless-stopped`, with a `/health` healthcheck).

### Raspberry Pi deployment (systemd)

```bash
# On the Pi's .env: STT_ENGINE=remote, STT_SERVER_URL=http://<server>:8000, AUDIO_ENABLED=True
cp .env.deploy.example .env.deploy   # PI_USER / PI_HOST / PI_PATH

bash scripts/deploy_to_pi.sh
```

This rsyncs the project (excluding Docker files), runs `poetry install --extras pi`, and installs/enables the `henry.service` systemd unit that launches `scripts/henry_app.py` (combined GUI + voice loop) on boot. Start the server before the Pi service.

## Usage / Quickstart

Run the combined GUI + voice client directly (also how the Pi service launches it):

```bash
poetry run python scripts/henry_app.py
```

The newer GTK4/libadwaita shell with voice-runtime controls:

```bash
API_BASE_URL=http://127.0.0.1:8000 poetry run python scripts/henry_gtk_app.py
```

Once running, say the wake word ("Hey HENRY" by default) and talk. You can also drive the backend directly over the REST API:

```bash
# Text conversation
curl -X POST http://localhost:8000/conversation/chat \
  -H 'Content-Type: application/json' \
  -d '{"text": "add milk to my todos"}'

# Start a Pomodoro timer
curl -X POST http://localhost:8000/productivity/timer/start

# Service health (checks Ollama + Neo4j)
curl http://localhost:8000/health
```

Selected endpoints: `POST /conversation/chat`, `GET /conversation/history`, `GET /conversation/ui/state`, `POST /stt/transcribe`, `POST /productivity/timer/{start,stop,pause}`, `GET /productivity/timer/status`, `GET|POST|PUT /productivity/{ideas,todos}`, `GET /calendar/events`, and the voice-runtime controls under `/voice-runtime/*`.

Run the test suite with `pytest`.

## How it works

1. **Wake word** — On the Pi, OpenWakeWord listens continuously and triggers on the wake phrase using the bundled `model/hey_henry.onnx` / `.tflite` model.
2. **Voice loop** — After the trigger, audio is captured (with VAD) and sent to the server, where Whisper transcribes it. `ConversationService` routes the utterance: intent recognition → tool execution → `PersonalityService` → Ollama LLM response.
3. **Graph memory** — Ideas, todos, calendar events, and preferences are stored in a Neo4j knowledge graph via `KnowledgeService`, giving H.E.N.R.Y. persistent context across conversations. (A NetworkX fallback exists for graphless runs.)
4. **Tools & GUI** — Tools inherit from `BaseTool` and register with a plugin registry; the same actions are exposed over the REST API and the GUI. `ScreenManager` is the single source of truth for UI state, driving the Pi's display (timer, todos, ideas, calendar, and the animated face) and TTS output via Piper.

## Features

- ✅ **Always-on voice loop** — wake word detection, capture, remote STT, LLM response, TTS
- ✅ **Knowledge graph memory** — Neo4j-backed store for ideas, todos, preferences, and events (`KnowledgeService`), with a NetworkX fallback
- ✅ **Todo management** — categories, status tracking, GUI + API
- ✅ **Local calendar** — create / update / list / recurring events, upcoming and today views, stored in the graph
- ✅ **Pomodoro timer** — work/break phases, auto-transitions, voice + GUI control
- ✅ **Idea capture** — store and refine ideas before committing them to the graph
- ✅ **Pygame GUI** — timer, todo list, idea notebook, calendar view, sidebar, animated face
- 🚧 **GTK4 / libadwaita shell** — newer app shell (`henry_gtk_app.py`) with voice-runtime controls, alongside the Pygame GUI
- 📋 **Google Calendar sync** — external bidirectional sync is a placeholder (`CalendarSyncManager` raises `NotImplementedError`); only the local graph calendar is live today
- 📋 **Beeper / Matrix, n8n, home automation** — external integrations not yet implemented
- 📋 **Physical robot** — motor and sensor control planned, no hardware code yet
- 📋 **Mobile companion app** — Flutter / Swift clients planned

## Roadmap

- 📋 **External calendar sync** — implement `CalendarSyncManager` for Google Calendar (API v3) and Apple/iCloud (CalDAV), with two-way sync and conflict resolution
- 📋 **Integrations** — Beeper/Matrix messaging, n8n workflow automation, home-management/smart-home control
- 📋 **Robot features** — GPIO motor control, sensors, and physical personality expression
- 📋 **Companion app** — Flutter (cross-platform) and native Swift iOS clients with remote sync
- 🚧 **GUI convergence** — mature the GTK4 shell toward parity with the Pygame interface

---

Local-first by design: H.E.N.R.Y. runs on your own network, keeping your data and voice under your control.
