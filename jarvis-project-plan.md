# J.A.R.V.I.S. — Project Plan v3

**Single-user, self-hosted, real-time voice assistant.**
Thin clients, fat server, pluggable everything.

This document supersedes v1 and v2. It locks the architecture decisions, defines
the swap seams, and sequences the build so there is something usable on day one.

---

## 0. How to use this document

Work phase by phase. Do not start a phase until the previous phase's acceptance
criteria pass. Each phase is scoped to be finishable in a sitting or two.

To hand a phase to Claude Code:

```bash
claude "Read jarvis-project-plan.md. Implement Phase 0 only. \
Do not start Phase 1. Stop when the Phase 0 acceptance criteria pass."
```

Constrain it to one phase at a time. Given the whole document it will try to
build everything and you will get a large amount of untested code.

---

## 1. Architecture decisions (locked)

These were argued through and settled. Do not re-litigate them mid-build.

| Decision | Choice | Why |
|---|---|---|
| Topology | Thin client, fat server | One deployment target; any WebRTC client works |
| Server location | Home lab VM (Proxmox) | Vault stays on hardware you own |
| Media transport | LiveKit Cloud | No TURN/SFU ops; agent dials out, no port forwarding |
| STT (v1) | Deepgram (streaming) | Partial transcripts; local Whisper is batch-only |
| TTS (v1) | Cartesia (streaming) | ~100–150ms first chunk |
| Brain | API only, via provider registry | Hot-swap providers by config |
| Tools | MCP servers | Config-line capability adds; portable across clients |
| Network / auth | Tailscale | Gives TLS + access control + zero public exposure |
| Persistence | SQLite, co-located with agent | Agent owns all writes |
| Wake word | Client-side `openWakeWord` | Cannot stream mic to server 24/7 |
| Client form | PWA everywhere; Tauri shell on desktop | One UI codebase, native OS hooks where needed |
| Users | Exactly one | No user table, no multi-tenancy, ever |

### Deliberate non-goals

- Local model inference of any kind in v1 (STT, TTS, and LLM are all remote)
- Speech-to-speech models (kills the pluggable-brain principle)
- Multi-user support
- Public internet exposure
- Offline operation (the brain is an API; offline means no Jarvis regardless)

### Honest privacy statement

Use this wording. Do not claim "zero data leakage."

> Conversation content is sent as text to the configured LLM provider. Audio is
> relayed (encrypted) through LiveKit Cloud. The Obsidian vault never leaves the
> home lab — only the specific note excerpts returned by tool calls are sent to
> the model. Self-hosted STT/TTS (Phase 5) would keep the raw waveform local but
> does not change what the LLM provider sees.

---

## 2. Process topology

Every process, and where it runs. Get this right before writing code.

| Process | Host | Notes |
|---|---|---|
| Agent worker (Python) | Home lab VM | Dials out to LiveKit Cloud |
| Token server + dashboard | Home lab VM | Next.js or similar |
| SQLite | Home lab VM | Same filesystem as agent |
| Obsidian vault (canonical) | Home lab VM | Syncthing to laptop/phone |
| MCP servers | Home lab VM | stdio subprocesses of the agent |
| LiveKit SFU | LiveKit Cloud | Managed |
| Desktop app (Tauri + Python sidecar) | Each desktop client | Owns the mic; wake word + hotkey + UI in one artifact |
| PWA | Phones, or any device | Installable web app; push-to-talk + dashboard |

**The vault lives on the server.** Syncthing keeps your laptop and phone copies
in sync bidirectionally. A network mount over WAN will be too slow and breaks
when you are away from home.

### Client contract (frozen)

**Clients may:** capture microphone, play audio, run wake-word/hotkey detection,
render session state from the LiveKit data channel, send settings RPCs.

**Clients may not:** run model inference, access the vault, hold API keys,
define providers or MCP servers.

Any device satisfying this contract is a valid client — browser tab, installed
PWA, Tauri desktop app, terminal app, an ESP32 puck with a mic and an LED ring,
CarPlay.

**One UI codebase.** The Phase 3 frontend is the *only* interface ever written.
It runs as a browser tab, as an installed PWA on phone and desktop, and inside
the Phase 4 Tauri shell — unchanged in all three. Platform differences live in
the shell around it, never in the UI. Protect this property; it is what keeps
the client side cheap.

---

## 3. The four swap seams

Flexibility comes from these being config-driven. **No provider name may appear
in agent logic.** This is the single most important architectural constraint in
the document.

### 3.1 LLM — provider registry

`config/providers.yaml`:

```yaml
providers:
  anthropic:
    base_url: https://api.anthropic.com/v1/
    api_key_env: ANTHROPIC_API_KEY
    supports_tools: true
    models: [claude-opus-5, claude-sonnet-5, claude-haiku-4-5]
  openai:
    base_url: https://api.openai.com/v1/
    api_key_env: OPENAI_API_KEY
    supports_tools: true
    models: [gpt-5, gpt-5-mini]
  openrouter:
    base_url: https://openrouter.ai/api/v1/
    api_key_env: OPENROUTER_API_KEY
    supports_tools: true
    models: []            # empty = query /v1/models at runtime

default_provider: anthropic
default_model: claude-sonnet-5
fallback_provider: openai   # used when default returns 5xx/429
fallback_model: gpt-5-mini
```

Clients select any registered `provider` + `model` freely. **API keys resolve
server-side from env vars and are never sent to clients.** Adding a provider =
one YAML block + one line in `.env`. Clients never supply raw base URLs (that is
an SSRF primitive pointed at your own LAN).

### 3.2 and 3.3 STT / TTS — adapter classes

There is no OpenAI-compatible equivalent for streaming speech. LiveKit's
abstraction is a Python ABC (`stt.STT`, `tts.TTS`), so swapping engines without
a first-party plugin means writing an adapter.

Providers with first-party plugins (Deepgram, Cartesia, ElevenLabs, AssemblyAI)
are one-line swaps. For anything else, write the adapter against **your own
stable internal WebSocket contract** so the engine behind it becomes a URL swap:

```
engines:
  stt:
    active: deepgram
    deepgram: { plugin: livekit-plugins-deepgram, model: nova-3 }
    local_ws: { endpoint: ws://localhost:9090/transcribe }
  tts:
    active: cartesia
    cartesia: { plugin: livekit-plugins-cartesia, voice: <id> }
    local_ws: { endpoint: ws://localhost:9091/synthesize }
```

**TTS adapter requirements** (both custom and plugin-backed):
- Accept a token stream; buffer to sentence boundaries; begin synthesis on the
  first complete sentence — never wait for the full response.
- Support immediate cancellation for barge-in. Without cancellation propagating
  through, interruption "works" in the UI while audio keeps generating.

### 3.4 Tools — MCP

```python
mcp_servers=[
    mcp.MCPServerStdio(command="npx", args=["-y", "obsidian-mcp", VAULT_PATH]),
    # add more here; each is one line
]
```

Guardrails:
- **Tool sprawl degrades selection.** Past ~15 tools, models start picking wrong
  ones. Gate by session or group behind a router tool.
- **MCP servers are code you are trusting** with vault access. Vet third-party
  servers or wrap them yourself.
- MCP servers are defined server-side. Clients may toggle them on/off, not
  define new ones.

---

## 4. Phases

### Phase 0 — Talk to it today

**Objective:** A working voice conversation. Deliberately unimpressive.
Everything else in this document is an upgrade to a running system rather than a
prerequisite to a first conversation.

Scope: cloud STT, cloud TTS, one hardcoded model, no vault, no daemon, no
settings UI, browser only.

Tasks:
1. Python project via `uv`. Deps: `livekit-agents` (pin the current 1.6.x line),
   plugins for Deepgram/Cartesia/Silero/turn-detector, `python-dotenv`.
2. `JarvisAssistant(Agent)` with a baseline system prompt — concise, dry, formal.
   Explicitly instruct: short spoken answers, no markdown, no lists.
3. `AgentSession` wiring STT → LLM → TTS + Silero VAD + semantic turn detection.
4. Connect via the LiveKit agents playground. No custom frontend yet.

**Verify the API surface against live docs before generating.** The
`Agent`/`AgentSession`/`@function_tool` structure changed substantially between
0.x and 1.x, and training data is full of the old shape.

**Acceptance:**
- [ ] Speak a sentence in the playground, get a spoken reply
- [ ] Interrupt mid-sentence — TTS stops immediately
- [ ] Pause mid-sentence for ~1s — it does **not** cut you off
- [ ] End-to-end round trip under ~1s

---

### Phase 1 — Make it configurable and observable

**Objective:** Replace hardcoded choices with the registry pattern, and see
where time goes.

Tasks:
1. Implement `providers.yaml` + engine config from §3. Refactor Phase 0 so no
   provider name appears in agent logic.
2. Runtime provider/model selection with fallback on 5xx/429.
3. **Per-turn latency instrumentation:** log `vad_fire → stt_final →
   llm_first_token → tts_first_chunk → audio_out`. Emit as structured JSON.
4. **Spoken error handling.** Every external call gets a timeout and a spoken
   fallback. Silence is not an acceptable failure mode — you cannot tell whether
   it heard you.
5. Inject current datetime + timezone into the system prompt each turn.
   Without this, "remind me tomorrow" and "what did I write last week" break
   silently.

**Acceptance:**
- [ ] Switch Anthropic → OpenAI by editing config only, no code change
- [ ] Kill network mid-turn; hear a spoken error, not silence
- [ ] Latency breakdown printed per turn
- [ ] Ask "what time is it" — correct answer

---

### Phase 2 — Vault via MCP

**Objective:** Contextual awareness of the Obsidian vault.

Tasks:
1. Vault canonical copy on the server; Syncthing to other devices.
2. Wire an Obsidian MCP server. Evaluate existing ones first — a decent one
   collapses most of this phase into configuration. Only hand-roll
   `@function_tool`s if nothing suitable exists.
3. **Speakable-output normalizer** between tool result and TTS: strip markdown,
   code blocks, `[[wikilinks]]`, frontmatter, URLs, and file paths; expand dates
   and long numbers. Raw markdown read aloud is unusable.
4. **Thinking sound.** On tool-call start, immediately synthesize a short
   acknowledgement ("checking your notes") while the search runs. Two seconds of
   silence feels broken; two seconds after "checking" feels normal.
5. **Cancellation through tool calls.** Saying "stop" mid-search must cancel the
   tool future, not let it complete and reply to an abandoned question. This is
   structural — do not defer it.

Decide before starting: **read-only, or can Jarvis create notes by voice?**
(Voice capture is arguably the killer feature. It also means an LLM with write
access to your notes.)

**Acceptance:**
- [ ] Ask about a specific note; get a clean spoken summary with no markdown
      artifacts
- [ ] Interrupt during a search; it stops and does not reply afterwards
- [ ] Ask about a note that does not exist; get a graceful spoken answer

---

### Phase 3 — Installable web app, persistence, settings

**Objective:** A visual client and durable state. The frontend built here is the
**single UI codebase for every platform** — it is reused unchanged inside the
desktop shell in Phase 4. Do not build anything phase-4-specific into it.

Tasks:
1. Next.js + Tailwind frontend, mobile-friendly. `livekit-client` for room
   connection; token endpoint server-side.
1b. **Make it an installable PWA** — web app manifest (name, icons,
   `display: standalone`) plus a minimal service worker. Gets a home-screen
   icon and its own window with no address bar, on both desktop and phone, for
   roughly an hour of work.
   **Verify microphone access works in the installed PWA on iOS specifically.**
   It works in recent versions, but Apple has been inconsistent about which
   capabilities survive the move from Safari tab to home-screen app. Test this
   early — a failure here invalidates the phone client entirely, and the
   fallback is simply to use it as a normal Safari tab.
   A PWA does **not** provide global hotkeys, background wake word, or tray
   integration. That is Phase 4's job.
2. **TLS is a hard prerequisite** — browsers refuse microphone access over plain
   HTTP. `http://192.168.x.x:3000` will not prompt for the mic on your phone.
   Use Tailscale Serve (real cert, no port forwarding, works on cellular).
3. **Auth: none at the app layer.** Entire surface lives inside the tailnet.
   Write into the code comments: *the token endpoint must never be exposed to
   the public internet.*
4. SQLite: transcripts, session metadata, global settings. Agent owns all
   writes; frontend reads via a small API, never by opening the file.
5. UI state: idle / listening / thinking / speaking. Plus transcript history and
   the latency numbers from Phase 1.
6. **Settings, three tiers:**
   - *Session* (volume, rate, PTT vs VAD) — instant, no persistence
   - *Device* (wake word on/off, hotkey, input device) — client local storage
   - *Global* (provider, model, voice, system prompt, MCP toggles) — SQLite
   In-session changes travel over LiveKit RPC; server validates, persists, and
   broadcasts to all connected clients. Validate server-side regardless of
   tailnet — reject unknown model IDs, clamp ranges, whitelist voices.
7. Expose a safe subset as `@function_tool`s so "use a slower voice" works by
   voice. Confirm verbally — a silent config change is disorienting.
8. **Text input path.** Same agent, typed input over the data channel. Nearly
   free, and far faster for debugging prompts than talking to your laptop
   repeatedly.
9. System prompt editing with versioning + one-click revert to default. It is
   the most useful knob and the fastest way to break tool calling.

Decide here: **conversation lifecycle.** When does a session end (silence
timeout / "goodbye" / hotkey)? Does a new session start fresh or continue
yesterday's context? Suggested default: persist all transcripts, load last N
turns on session start, expose a "forget that" tool. This determines the schema,
so decide before writing it.

Model-swap policy: apply at next session start, **or** apply immediately and
clear context. Pick one and have the UI say which. Inheriting a persona the new
model did not produce is audibly weird.

**Acceptance:**
- [ ] Load dashboard on your phone over Tailscale; mic permission granted
- [ ] Install to home screen; **mic still works in the standalone window** (iOS
      and Android)
- [ ] Install on desktop; opens in its own window with no browser chrome
- [ ] Full voice conversation from the phone
- [ ] Change voice on laptop; phone UI reflects it without reload
- [ ] Review yesterday's transcript
- [ ] Type a message and get a spoken reply

---

### Phase 4 — Desktop app: wake word and global trigger

**Objective:** A real desktop application that can be summoned from anywhere on
the OS.

**Decide first: which OS?** macOS needs Accessibility + Microphone entitlements
and fights background mic access; Windows and Linux differ substantially. This
phase is not cross-platform for free.

**Shape: Tauri shell + Python sidecar.** Wrap the *same* Phase 3 frontend in a
[Tauri](https://tauri.app) application, with the Python wake-word/LiveKit client
running as a sidecar process. This makes the dashboard and the daemon one
artifact rather than two, which resolves two problems at once:

- Mic ownership — one process owns the input device; there is no browser tab
  competing for it.
- State visibility — a headless daemon has no UI, so you would otherwise have to
  build a tray indicator separately just to see whether it is listening. Here
  the UI already exists.

Tauri over Electron: it uses the system webview rather than bundling Chromium,
so the binary is a fraction of the size. Costs to budget for: a Rust toolchain
in the build, app packaging, and code signing (**macOS notarization is a genuine
annoyance** — allow an afternoon).

Tasks:
1. Tauri shell loading the Phase 3 frontend unchanged. Window hides to tray
   rather than quitting on close.
2. Python sidecar owns the mic permanently, runs `openWakeWord`, and opens its
   own LiveKit connection on trigger. Communicates state to the shell over local
   IPC so the UI reflects idle/listening/thinking/speaking even when hidden.
3. `openWakeWord` ships a pretrained **"hey jarvis"** model. No training needed.
4. Tunable confidence threshold + a spoken/visual cancel path for false wakes.
   It *will* fire on TV dialogue. An assistant that activates itself during a
   film gets disabled permanently within a week.
5. Global hotkey as the reliable alternative path. Audio cue on activation.
6. Tray icon reflecting current state; click to show the window.

The PWA remains the phone client permanently — no background daemon is practical
on iOS or Android, so phones are push-to-talk plus dashboard.

**Acceptance:**
- [ ] Say "hey jarvis" with the app window hidden; session activates with an
      audio cue
- [ ] Hotkey does the same
- [ ] Tray icon reflects state while the window is closed
- [ ] Leave it running through an evening of TV; false-wake rate tolerable
- [ ] Open the PWA on your phone while the desktop app runs — both work, no
      conflict
- [ ] App survives a reboot / launches at login

---

### Phase 5 — Optional experiments

Only after Phases 0–4 are stable. Each is independent.

- **Local TTS.** Kokoro (better quality, needs GPU) or Piper (fast on CPU,
  sounds like a 2010 satnav). TTS is the better local candidate — less
  latency-sensitive than STT, and Kokoro is competitive with paid services.
- **Local STT.** Only via a *streaming* server (WhisperLive / whisper-streaming)
  exposing rolling partials. Batch `faster-whisper` blocks the pipeline for
  600–1500ms and is a downgrade. If you want GPU: PCIe passthrough into the
  Proxmox VM (IOMMU on, host driver blacklisted, GPU in its own group) and
  pinned matching cuDNN/cuBLAS versions — the most common install failure.
- **Client action seam.** If Jarvis should open apps, control media, or paste
  text, that needs a small local RPC service on each client, exposed to the LLM
  as an MCP server. This changes the frozen client contract — decide
  deliberately. Note that Phase 4's Python sidecar is the natural host for it,
  so this becomes considerably cheaper once the desktop app exists.
- **Cost visibility.** Per-session token counts in the dashboard. A runaway tool
  loop on an expensive model is better seen than discovered on a bill.
- **Deterministic local shortcuts.** Regex match on "stop", "cancel", "louder"
  before any inference. Zero latency, 100% reliable — this is what makes Siri
  feel instant on its common cases.

---

## 5. Open decisions

Fill these in before Phase 0. Each blocks something downstream.

| # | Decision | Blocks |
|---|---|---|
| 1 | Desktop OS target | Phase 4 entirely |
| 2 | Vault read-only or read-write | Phase 2 scope |
| 3 | Session end condition | Phase 3 schema |
| 4 | Context across sessions: fresh or continuous | Phase 3 schema |
| 5 | Model swap: immediate + clear, or next session | Phase 3 UI |
| 6 | GPU in the home lab or not | Phase 5 feasibility |

---

## 6. Standing instructions for Claude Code

Include these in the prompt for every phase.

1. Implement **only** the named phase. Stop at its acceptance criteria.
2. **No provider name in agent logic.** Everything through the registry.
3. Verify the `livekit-agents` API against live documentation before generating.
   Do not trust training data for the `Agent`/`AgentSession`/`@function_tool`
   shape.
4. Every external call: explicit timeout + spoken fallback. Never fail silently.
5. Secrets from server env only. Never sent to, stored on, or accepted from
   clients.
6. Validate all client input server-side even though this is single-user on a
   tailnet.
7. Write acceptance-criteria tests where testable, and say plainly which
   criteria were verified and which were not.

---

## 7. Operational notes

- **Backups.** SQLite on a Proxmox VM with no snapshot policy is a regret
  waiting to happen. Set this up in Phase 3, not later.
- **Secrets.** One `.env` on the server, never committed. Rotate if it ever
  touches a client.
- **Latency budget (v1 target).** STT partial ~100ms → LLM TTFT 300–600ms →
  TTS first chunk 100–150ms → WebRTC ~50–100ms. **Total ~600–950ms.** If you
  measure worse, use the Phase 1 instrumentation to find the stage rather than
  guessing.
- **Costs.** At personal usage (100–300 min/month), Deepgram + Cartesia lands
  around **$3–6/month**. LLM cost dominates and depends on model choice.
