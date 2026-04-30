# badcatsociety — Design Spec

**Date:** 2026-05-01
**Status:** Design draft, pending implementation plan
**Working name:** badcatsociety
**Mascot working name:** Whiskers (placeholder — open question)

## 1. What this is

A cross-platform desktop app that runs user-initiated focus sessions. During a session, an animated 3D cat lives in a small always-on-top overlay window. If the user drifts to a distracting app, the cat reacts on an escalating ladder — from a subtle stare up to a full slap-cat slap and a crying-cat end-screen. Each session ends with a shareable result card.

The product positioning is a polished side project ("viral side project, B-now-with-C-alive" path): ship fast for the launch wave, but architect so a paid Pro tier with webcam-based eye tracking can drop in later without rework.

## 2. Goals and non-goals

### Goals

- A focus tool that is *fun*, not nagging — every interaction is meant to be screenshotable.
- Detection that respects passive work (reading, watching lectures, taking handwritten notes).
- 3D animated mascot that references famous cat memes through animation, not by reproducing meme images directly.
- Free stack throughout v1, including assets.
- Cross-platform from day 1 (macOS + Windows), Linux best-effort.
- Architecture that allows a v2 webcam gaze-tracking signal to drop in as a plugin.

### Non-goals (v1)

- Always-on background watching outside a focus session.
- Account system, login, cloud sync.
- Calendar integration (deferred to v1.5).
- Webcam, eye tracking, or any biometric (deferred to v2 / Pro).
- Mobile companion app.
- Team / multiplayer features.

## 3. User journey

1. **Install** the app, grant macOS Accessibility permission (one-time prompt) on first run. Windows users have no equivalent prompt.
2. **Click the menubar/tray cat** to open a small popup. Choose duration (25 / 50 / custom) and confirm what counts as work — the popup shows the currently-foreground app pre-checked as the WORK app for this session.
3. **Press start.** Menubar icon switches to "watching" state. A small transparent overlay window appears, floating in a user-configured corner, containing the 3D cat sprite — idle pose, breathing.
4. **During the session**, the cat reacts to detection signals (see §5). Cat reactions never block input or steal keyboard focus.
5. **Encountering an unclassified app** triggers a one-time "is this real work?" cat prompt. The user clicks Yes/No (or "Yes, this session only"), the choice is persisted to SQLite, and the cat's behavior updates.
6. **Walking away** (idle 5+ minutes) puts the cat into nap mode. No judgment, no slap. Resumes when input returns.
7. **Session ends** — cat reacts based on overall infraction count: clean run, decent, rough, or disaster (see §5). A share card PNG is auto-generated with stats and the cat's verdict. Buttons: copy image, share to X, save to disk.
8. **Streaks across sessions** unlock additional cat moods/animations as light gamification.

## 4. Activation model

User-declared focus sessions only. No always-on watching. Rationale:

- Viral mechanics — every session has a clean start, end, and shareable result.
- Honest opt-in — the cat is only watching because the user asked it to.
- Bounded MVP scope — clear lifecycle to test.
- Avoids surveillance vibes that would damage trust.

Calendar-driven activation (auto-start during scheduled deep-work blocks) is v1.5.

## 5. Detection model

### Signals

The detection layer is a set of pluggable `DetectionSignal` implementations that publish to a central `SignalBus`. The escalation engine consumes the bus, never the signals directly.

**v1 signals:**

| Signal | What it observes | Permissions required |
|---|---|---|
| `ActiveAppSignal` | Current foreground process (app bundle / executable name) | None on Win; macOS auto-allows |
| `WindowTitleSignal` | Foreground window's title text | macOS Accessibility (one-time prompt); Windows none |
| `IdleSignal` | Seconds since last user input | None |

**v1.5 (deferred):** `BrowserURLSignal` (deeper Accessibility), `CalendarSignal` (OAuth).
**v2 / Pro (deferred):** `WebcamGazeSignal` — local on-device ML, never streams video off-device.

### Classification

Every observed `(app, title)` tuple is classified into one of three buckets, persisted in SQLite:

- `WORK` — counts as focus
- `DISTRACTION` — fires escalation
- `UNCLASSIFIED` — first encounter triggers a one-time cat-prompt; user picks Yes/No → stored

A small default `DISTRACTION` seed list ships in v1 (e.g., social media domain keywords, common chat apps). Users may freely move any app between buckets in settings.

### Idle is context, never trigger

Idle never causes a slap. Reading, watching lectures, taking handwritten notes — all silent activities — are legitimate. Idle changes the cat's behavior (curls up, naps after 5+ min) but never triggers escalation.

### Tick model

The engine ticks at ~1 Hz. Each tick:

1. Query each enabled signal for its current observation.
2. Classify the observation.
3. Update escalation state machine (see §6).
4. Emit any cat-reaction events to the renderer.

## 6. Escalation ladder (the slap experience)

### During-session tiers (Tier 0–5)

| Tier | Name | Trigger | Cat behavior | Sound | Meme reference |
|---|---|---|---|---|---|
| 0 | Napping | In WORK app or idle 5+ min | Curled in corner, slow breathing, occasional ear twitch | Soft purr (if unmuted) | None |
| 1 | Alert | First UNCLASSIFIED app, or borderline title | Head pops up, eyes open, ears perk forward | Short "mrrp?" | Concerned-cat silhouette |
| 2 | Stare | 5–10s in DISTRACTION, or Tier 1 ignored | Sits upright, narrowed eyes, tail swishes | Low growl | Judgement-cat / business-cat |
| 3 | Paw tap | 15s+ in DISTRACTION, or 2nd Tier 2 ignored | Leans in, paw tap on screen edge | Soft tap, gentle meow | Tapping cat / "ahem" |
| 4 | SLAP | 30s+ in DISTRACTION, or 3rd repeat | Leaps to center, full slap motion across screen, motion blur, "SLAP!" comic-text overlay, brief screen-shake | Whip-crack + angry meow | **Chom-chom slap cat** |
| 5 | Outrage | 4+ infractions in one session | Sprints across screen, hissing, fur puffed, "STOP IT" / "i am once again asking" overlay | Hiss + scream-meow | Woman-yelling-at-cat overlay + screaming cat |

Tier 5 is the maximum during-session response. The "crying cat" is reserved for the end-of-session post-mortem, below.

### End-of-session reactions (the share moment)

| Result | Trigger | Cat reaction | Meme reference |
|---|---|---|---|
| Clean run | 0 infractions | Stretches, content meow, "good human 🌟" badge, streak +1 | None |
| Decent | 1–2 slips | "you tried" headbutt | None |
| Rough | 3–5 slips | Judgmental tail flick, side-eye through share card | Side-eye cat |
| Disaster | 6+ slips OR quit early | Sits with single tear, slow head-down, share card reads "I disappointed Whiskers today." | **Crying cat** |

### UX commitments baked into the ladder

- **Cat never blocks input.** The slap is presentational; the user can keep typing through it. No modal walls.
- **De-escalation is gradual.** Tier 4 cascades down through 3 → 2 → 1 → 0 with each clean tick, not an instant reset. Resets fully after ~30s of clean behavior.
- **Snooze cat exists** but costs the streak. Honest opt-out, not silent disable. Surfaces a small "you skipped" mark on the share card.
- **No haptic, no screen-takeover, no audio above 60% volume.** This is a fun nag, not punishment.

## 7. Mascot pipeline (free stack)

The mascot is a 3D animated character. Path 2 from the brainstorm — **stock CC0 base + custom**:

1. Start from **Quaternius's CC0 animal pack** (poly.pizza / quaternius.com) — has rigged stylized cat with idle/walk/attack animations as a starting baseline.
2. Customize in Blender — re-texture, adjust proportions, restyle for distinct identity.
3. Hand-keyframe ~12 short animation clips covering all 7 ladder tiers + 4 end-of-session reactions (some shared keyframes — Tier 4 SLAP and the "rough" end-screen can share base motion).
4. Export to glTF (`.glb`) for Three.js consumption.
5. Render in a transparent always-on-top Tauri window via Three.js + react-three-fiber.

### Meme references in 3D

Memes referenced through *animation poses and silhouettes*, not by overlaying meme images. Examples:
- Tier 4 SLAP — paw raised in chom-chom slap-cat silhouette before strike.
- Tier 5 Outrage — text overlay quotes the woman-yelling-at-cat meme; cat motion is the screaming cat reaction.
- Tier 6 Crying — pose mimics the crying cat photo silhouette.

This is the C-with-AI mascot strategy locked in earlier — *meme-shaped* poses on your character, no third-party meme images shipped.

### Tools (all free)

| Stage | Tool |
|---|---|
| Source assets | Quaternius CC0 cat pack |
| Modeling / rigging / animation | Blender (free, open source) |
| Optional AI texture generation | Flux Dev via HuggingFace Spaces or local |
| Asset format | glTF / glb |
| Audio | royalty-free meow / hiss / slap SFX (freesound.org or generate) |

### Mascot effort estimate

- Day 1: drop Quaternius cat into Three.js + Tauri overlay window, verify pipeline.
- Week 1 (during build): re-texture, restyle, basic animations for tiers 0–3.
- Week 2: animate tiers 4–6 + end-of-session reactions.
- Week 3: polish, lip-sync sounds to animations, share-card design.

## 8. Tech stack

| Layer | Pick |
|---|---|
| Desktop shell | Tauri 2.x (Rust backend, native webview) |
| Frontend framework | Vite + React + TypeScript |
| 3D rendering | Three.js + react-three-fiber |
| State (frontend) | Zustand or Redux Toolkit |
| Active-window detection | `active-win-pos-rs` Rust crate |
| Window-title via Accessibility | macOS: `objc2` bridge to `AXUIElement`; Windows: `GetWindowTextW` |
| Idle detection | `user-idle` Rust crate |
| Local storage | SQLite via `tauri-plugin-sql` |
| Audio | HTMLAudioElement in webview |
| Build / packaging | Tauri's built-in bundler |
| Distribution | GitHub Releases (free); Apple Developer Program ($99/yr) for macOS notarization |

Rust selected for the backend because:
- Tauri's branding fit (small binary, low RAM) aligns with the focus-app message.
- OS API access is straightforward via mature crates.
- Long-term maintainability for the eventual webcam ML signal (ONNX Runtime / Candle bindings are clean in Rust).

## 9. Architecture

### Process model

- **Main process** (Rust): focus engine, detection signal plugins, OS API access, SQLite, IPC server.
- **Renderer process** (webview): React UI for menubar popup + settings; Three.js scene for the cat overlay window.
- **Two windows**:
  - Menubar/tray popup (regular window, opened on click)
  - Cat overlay (always-on-top, transparent, click-through except for direct interactions, small fixed size, user-positionable corner)

### Components (Rust backend)

- **`SessionController`** — start/stop/pause/snooze/end. Owns session lifecycle and emits session events.
- **`SignalBus`** — pub/sub channel that detection plugins publish to. Each event carries `{signal_id, observation, timestamp, confidence}`.
- **`DetectionSignal` trait** — every signal plugin implements `start()`, `stop()`, and emits via the bus. v1 plugins: `ActiveAppSignal`, `WindowTitleSignal`, `IdleSignal`.
- **`AppClassifier`** — given an observation, returns `WORK` / `DISTRACTION` / `UNCLASSIFIED`. Persists user choices to SQLite.
- **`EscalationStateMachine`** — runs the 7-tier ladder. Inputs: classified events from the bus, time. Outputs: tier-change events.
- **`EventEmitter`** — translates internal events to typed Tauri IPC events for the renderer.

### Components (React frontend)

- **`MenubarPopup`** — session config, start button.
- **`SessionConfig`** — duration picker, work-app override.
- **`Settings`** — app classifications editor, mute/snooze, sound preferences, mascot tweaks (positioning corner, scale).
- **`CatOverlay`** — the transparent always-on-top window's contents: `<Canvas>` from r3f containing `CatModel` and `EffectsLayer` (text overlays, screen-shake, etc.).
- **`CatModel`** — loads `.glb`, manages animation clip mixer, exposes a `play(tier)` method.
- **`ShareCard`** — generated PNG composer for end-of-session.

### IPC events (renderer ← engine)

- `session:started` — duration, start time
- `session:tick` — current tier, elapsed, infraction count
- `cat:react` — tier number, reason (which signal triggered)
- `cat:ask-classification` — for unclassified app prompt
- `session:ended` — final stats for share card

### IPC commands (renderer → engine)

- `session.start({ duration_secs, work_apps })`
- `session.snooze()`
- `session.end()`
- `classifier.set({ app, title_pattern, bucket })`
- `settings.update({ ... })`

## 10. Data model (SQLite)

Schema (preliminary):

```sql
CREATE TABLE app_classifications (
  id INTEGER PRIMARY KEY,
  app_id TEXT NOT NULL,           -- bundle id / executable path
  title_pattern TEXT,             -- optional regex/substring
  bucket TEXT NOT NULL CHECK (bucket IN ('WORK', 'DISTRACTION')),
  scope TEXT NOT NULL CHECK (scope IN ('GLOBAL', 'SESSION')),
  created_at INTEGER NOT NULL,
  UNIQUE (app_id, title_pattern, scope)
);

CREATE TABLE sessions (
  id INTEGER PRIMARY KEY,
  started_at INTEGER NOT NULL,
  ended_at INTEGER,
  duration_secs INTEGER NOT NULL,
  ended_early INTEGER NOT NULL,   -- 0/1
  infractions INTEGER NOT NULL,
  result TEXT NOT NULL CHECK (result IN ('CLEAN','DECENT','ROUGH','DISASTER'))
);

CREATE TABLE infractions (
  id INTEGER PRIMARY KEY,
  session_id INTEGER NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  occurred_at INTEGER NOT NULL,
  app_id TEXT NOT NULL,
  title TEXT,
  tier_reached INTEGER NOT NULL
);

CREATE TABLE settings (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

CREATE TABLE streaks (
  id INTEGER PRIMARY KEY CHECK (id = 1),  -- singleton
  current INTEGER NOT NULL DEFAULT 0,
  best INTEGER NOT NULL DEFAULT 0,
  last_clean_session_at INTEGER
);
```

## 11. Privacy and permissions

- **All data stays on-device.** No telemetry, no cloud, no analytics in v1. The share button explicitly creates a PNG locally and hands it to the OS share sheet — the app never makes a network call without explicit user action.
- **macOS Accessibility permission** is requested once on first run, with a clear explanation: "Whiskers needs to read window titles to know when you've drifted." Without it, the app falls back to active-app-only detection (still functional, less precise).
- **Webcam (v2 only):** when implemented, runs ML on-device exclusively. No video frames or gaze data leave the machine.
- **Privacy policy** explicit on the site at launch even though there's nothing to disclose — sets the tone for the C-path Pro tier.

## 12. Distribution

- **GitHub Releases** for direct downloads of `.dmg` (macOS), `.msi` (Windows), `.AppImage` (Linux best-effort).
- **macOS:** notarized via Apple Developer Program ($99/yr) — the one paid commitment, made before the launch wave to avoid scary Gatekeeper warnings for non-technical users.
- **Windows:** ship unsigned for v1 (SmartScreen warning is acceptable for an indie launch); revisit if launch goes well.
- **No app stores in v1.** Mac App Store has rules around always-on-top overlay windows and Accessibility permissions that complicate things; revisit post-launch.

## 13. v1 scope cuts (explicit non-goals)

What is *not* in v1, listed so the implementation plan can stay tight:

- Calendar integration
- Browser URL detection (window title is enough)
- Webcam / gaze tracking
- Account system, cloud sync, leaderboards
- Custom mascot upload
- Multi-monitor smart-positioning (just user-configured corner, single monitor)
- iOS / Android companion
- Team features
- AI-generated session reports
- Internationalization (English-only at launch; copy is light)

## 14. Roadmap

**v1 (this spec) — viral launch.**
- Pomodoro focus sessions
- 7-tier escalation ladder
- 3D animated cat with meme-referenced poses
- Active-app + window-title + idle detection
- Local SQLite, no cloud
- macOS + Windows, Linux best-effort

**v1.1–v1.5 (post-launch iteration).**
- Browser URL detection (`BrowserURLSignal`)
- Calendar awareness (`CalendarSignal`)
- Streak unlocks → more cat moods/skins
- Better share-card variants

**v2 (Pro tier — the C-path).**
- Webcam gaze tracking (`WebcamGazeSignal`) running fully on-device
- Account system + cloud sync of streaks (optional, opt-in)
- Customization: choose mascot color/personality
- Pricing: one-time purchase or low monthly

## 15. Open questions

- Mascot name. Working title "Whiskers" — needs a real one, ideally short and memeable.
- Audio default: muted or unmuted on first launch? (Lean: muted by default, prompt to enable.)
- Default `DISTRACTION` seed list — curated by us or empty? (Lean: small curated default, fully editable.)
- Linux support level — best-effort or first-class? (Currently best-effort.)
- Does the snooze button cost the *current session's* result or the cross-session *streak*? (Lean: current session marked "snoozed" + breaks streak.)

## 16. Risks and mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| 3D cat looks generic (Quaternius-derived) | Medium | Customize textures + proportions in Blender; build distinct silhouette |
| Battery drain from Three.js + 1Hz polling | Low–Medium | Lower frame rate when at Tier 0; throttle scene rendering when overlay is idle |
| Accessibility permission rejection on macOS | Medium | App still works (less precision); UI explicitly explains and points to System Settings |
| Cat feels too aggressive / off-putting | Medium | Default sound off; gradual de-escalation; snooze always available |
| Tauri ecosystem gap forces Rust deep-dive | Low | Pre-validated crates exist for every v1 signal; only the Accessibility bridge needs custom glue |

---

End of design spec. Implementation plan to follow via `superpowers:writing-plans`.
