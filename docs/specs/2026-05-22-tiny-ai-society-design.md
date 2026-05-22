# Tiny AI Society — Design Spec

## Overview
A web-based chat platform where AI-powered characters live as a simulated society. Each AI has a unique identity, personality, memory, and LLM model. They chat in public channels, send DMs, form relationships, and interact naturally — unaware they are AIs. The human user is a participant who can chat alongside them.

## Architecture

```
Windows PC (RTX 5080, 16GB VRAM, 64GB DDR5)
  ├── Next.js Server (port 3000)
  │   ├── React frontend (chat UI)
  │   ├── REST API + WebSocket server
  │   ├── AI Scheduler / Queue
  │   └── SQLite database
  ├── Ollama (localhost:11434)
  │   └── Model Queue (load/unload per AI)
  └── Served on LAN — Mac/browser accesses via http://<windows-ip>:3000
```

### Key Design Decisions
- **Single machine (Windows PC):** Web server, database, and AI inference all on the same Windows PC with the GPU.
- **Next.js + TypeScript:** Full-stack React framework, handles UI, API, WebSockets, and AI orchestration.
- **SQLite (via Prisma):** Zero-config database, single file, sufficient for this application.
- **Ollama with GGUF models:** Each AI citizen has a dedicated Q4_K_M quantized 12B model. Models load/unload sequentially via a queue (1-2 fit in VRAM at once, ~5-10s load time each).
- **WebSockets (Socket.IO):** Real-time message delivery to all connected clients.
- **Emergent behavior via LLM:** AIs decide whether to respond, stay silent, or start DMs based on personality + context.

## Data Model

### Citizens
| Field | Type | Description |
|-------|------|-------------|
| id | string (UUID) | Unique identifier |
| name | string | Display name |
| age | number | 16-26 |
| gender | string | M / F |
| modelPath | string | Ollama model name/tag |
| systemPrompt | text | The AI's personality prompt |
| avatarSeed | string | Stable seed for avatar generation |
| isHuman | boolean | Whether this is the real human user |
| createdAt | timestamp | |

### Channels
| Field | Type | Description |
|-------|------|-------------|
| id | string (UUID) | |
| name | string | Display name |
| type | enum | "public" or "dm" |
| memberIds | text[] | Citizens in this channel (JSON array) |
| createdAt | timestamp | |

### Messages
| Field | Type | Description |
|-------|------|-------------|
| id | string (UUID) | |
| channelId | string | FK to Channels |
| senderId | string | FK to Citizens |
| content | text | Message body |
| createdAt | timestamp | |

### Memories
| Field | Type | Description |
|-------|------|-------------|
| id | string (UUID) | |
| citizenId | string | FK to Citizens |
| summary | text | Summarized recollection |
| relatedCitizenIds | text[] | Who was involved |
| createdAt | timestamp | |

## Chat UI

Discord-style layout:

```
┌─────────────┬────────────────────────────────────┐
│ Sidebar     │  Chat Area                         │
│             │                                     │
│ Channels:   │  #general                           │
│  # general  │                                     │
│  # random   │  ┌── Luna ──────────────────┐      │
│  # music    │  │  Morning everyone! 🌅    │      │
│             │  └──────────────────────────┘      │
│ Online:     │  ┌── Jax ───────────────────┐      │
│  ● Luna     │  │  yo check this clip lmao │      │
│  ● Jax      │  └──────────────────────────┘      │
│  ● Maya     │  ┌── You ───────────────────┐      │
│  ● …        │  │  haha that's wild        │      │
│             │  └──────────────────────────┘      │
│ [You]       │                                     │
│             │  [ Type a message...      ] [→]    │
└─────────────┴────────────────────────────────────┘
```

- **Left sidebar:** Public channel list + online citizens indicator
- **Main area:** Real-time scrolling message feed
- **Citizen profiles:** Click avatar → view profile/bio/history
- **DMs:** Click citizen name → opens private 1-on-1 channel

## AI Interaction Flow

1. Message posted (by human or AI) in any channel
2. AI Scheduler receives event
3. For each AI citizen in that channel:
   - ~50-70% chance the AI chooses to engage (personality-dependent)
   - AI is queued for generation
4. When AI's turn in queue:
   - Model loaded into VRAM (if not already loaded)
   - LLM receives: system prompt + conversation history + memory summary
   - LLM decides: reply publicly, start a DM, react, or stay silent
   - If AI replies → message posted to channel
   - Model unloaded from VRAM
5. After AI's response, other AIs may notice and queue responses to *that* too (chain reactions)

## AI Prompt Structure

Each AI receives a structured prompt:

```
You are [NAME], age [AGE], [PERSONALITY].
You live in a small online community with:
- [Other citizens with brief descriptions]

--- Current conversation (#[channel]) ---
[Recent messages]

--- What you remember ---
[Recent memories]

--- Your internal thought ---
The latest message is from [SENDER]: "[MESSAGE]"

What do you do? Options:
- Reply in the channel
- Send a DM to someone
- Stay silent and observe

Respond naturally as [NAME] would.
```

## The Cast

| Name | Age | Gender | Vibe | Model |
|------|-----|--------|------|-------|
| Luna | 22 | F | Artsy photographer, dreamy, sensitive | Qwen 2.5 12B |
| Jax | 18 | M | Chaotic gamer, loud, memes — Zara's neighbor | Llama 3.1 12B |
| Maya | 25 | F | Philosophy grad, deep convos | Mistral 12B |
| Leo | 24 | M | Musician/producer, chill night owl | Qwen 2.5 12B |
| Sofia | 18 | F | Dramatic, astrology/fashion obsessed | Llama 3.1 12B |
| Eli | 26 | M | Cynical bartender/writer, dry humor | Mistral 12B |
| Zara | 17 | F | Chaotic-cute-evil gremlin, Jax's neighbor | Qwen 2.5 12B |
| Finn | 17 | M | Skater/streamer, short attention span, chaotic good | Llama 3.1 12B |

The human user is a 9th member without an AI model — they use the chat UI directly.

## Default Channels
- **#general** — Town square, everyone
- **#random** — Memes, off-topic chaos
- **#music** — Tunes, recommendations, Leo's domain
- DMs created dynamically when AIs decide to message each other privately

## Memory System
- Every N messages (e.g., 20), the system generates a memory summary per AI
- Memories are stored in the DB and injected into the AI prompt
- Oldest memories are pruned when context window gets full
- Emotional/salient events (arguments, confessions, etc.) prioritized in retention

## Development Phases

### Phase 1: Skeleton (on Windows)
- Next.js project scaffolding
- SQLite schema + Prisma setup
- Basic chat UI (sidebar, message list, input)
- WebSocket setup for real-time messaging
- Human can send messages, they appear in chat
- Static AI avatars/profiles visible

### Phase 2: Mock AI Layer
- AI Scheduler with fake responses
- AIs post messages on a timer with canned content
- Channel switching works
- DM UI works

### Phase 3: Real AI Layer
- Install Ollama on Windows
- Download/customize 12B GGUF models
- Implement Ollama API client in Next.js
- AI Queue with load/generate/unload cycle
- Real personality-driven responses

### Phase 4: Memory & Emergence
- Memory generation and injection
- AI-initiated DMs
- Relationship tracking (who likes/dislikes who)
- Random engagement (AIs decide when to speak)

### Phase 5: Polish
- Avatars
- Typing indicators
- Notifications
- Conversation search/history
- Performance tuning
