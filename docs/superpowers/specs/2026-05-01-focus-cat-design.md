# badcatsociety — Design Spec

**Date:** 2026-05-01
**Status:** Design draft, pending implementation plan
**Working name:** badcatsociety
**Mascot working name:** Whiskers (placeholder — open question)

## 1. What this is

A cross-platform desktop app that runs user-initiated focus sessions. During a session, an animated **2D cat** lives in a small always-on-top overlay window. The cat runs two complementary tracks: a **negative escalation ladder** (when the user drifts to distractions, escalating from a subtle stare to a full slap and an end-of-session crying-cat) and a **positive reward ladder** (when the user stays focused for extended periods, the cat does encouraging pat-pats and celebrations). Each session ends with a shareable result card with one-click share buttons for X, Reddit, and Discord.

> **Spec amendment (2026-05-01):** mascot switched from 3D to 2D — simpler pipeline, faster ship, lighter runtime. Positive reward ladder added. Sharing channels elevated from generic share-sheet to explicit X / Reddit / Discord buttons.

The product positioning is a polished side project ("viral side project, B-now-with-C-alive" path): ship fast for the launch wave, but architect so a paid Pro tier with webcam-based eye tracking can drop in later without rework.

## 2. Goals and non-goals

### Goals

- A focus tool that is *fun*, not nagging — every interaction is meant to be screenshotable.
- **Both punishment and praise** — a slap ladder for drift, a reward ladder for sustained focus. Pure-negative reinforcement makes a sad product; the cat should be rooting for you.
- Detection that respects passive work (reading, watching lectures, taking handwritten notes).
- 2D animated mascot that references famous cat memes through animation, not by reproducing meme images directly.
- One-click share to X, Reddit, and Discord — not a generic OS share sheet.
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

## 6. Reaction ladders (slap *and* praise)

The cat runs two ladders concurrently during a session: a **negative escalation ladder** for drift, and a **positive reward ladder** for sustained focus. They share the same overlay surface — only one cat animation plays at a time — but they're triggered independently.

### Negative ladder — during-session tiers (Tier 0–5)

| Tier | Name | Trigger | Cat behavior | Sound | Meme reference |
|---|---|---|---|---|---|
| 0 | Napping | In WORK app or idle 5+ min | Curled in corner, slow breathing, occasional ear twitch | Soft purr (if unmuted) | None |
| 1 | Alert | First UNCLASSIFIED app, or borderline title | Head pops up, eyes open, ears perk forward | Short "mrrp?" | Concerned-cat silhouette |
| 2 | Stare | 5–10s in DISTRACTION, or Tier 1 ignored | Sits upright, narrowed eyes, tail swishes | Low growl | Judgement-cat / business-cat |
| 3 | Paw tap | 15s+ in DISTRACTION, or 2nd Tier 2 ignored | Leans in, paw tap on screen edge | Soft tap, gentle meow | Tapping cat / "ahem" |
| 4 | SLAP | 30s+ in DISTRACTION, or 3rd repeat | Leaps to center, full slap motion across screen, motion blur, "SLAP!" comic-text overlay, brief screen-shake | Whip-crack + angry meow | **Chom-chom slap cat** |
| 5 | Outrage | 4+ infractions in one session | Sprints across screen, hissing, fur puffed, "STOP IT" / "i am once again asking" overlay | Hiss + scream-meow | Woman-yelling-at-cat overlay + screaming cat |

Tier 5 is the maximum during-session response. The "crying cat" is reserved for the end-of-session post-mortem, below.

### Positive ladder — reward events during a session

The cat actively cheers you on when you sustain focus. Reward events fire based on cumulative *clean* time during the session (time spent in WORK apps, excluding idle and distraction).

| Reward tier | Trigger | Cat reaction | Sound |
|---|---|---|---|
| R1 — Pat-pat | 5 min of cumulative clean focus | Cat reaches over and gently pats screen edge (encouraging, friendly) | Soft chirrup |
| R2 — Thumbs-up | 15 min cumulative clean focus | Cat sits up, gives a paw thumbs-up, brief sparkle | Bright meow |
| R3 — Celebration | 30 min cumulative clean focus | Cat does a happy spin / wiggle, "you got this 🌟" overlay | Triumphant meow |
| R4 — Reverence | 60 min cumulative clean focus | Cat bows respectfully, "absolute legend" overlay | Slow purr |

Reward events are **not blocked** by escalation events — they fire on their own clean-time threshold. But: clean-time accumulator **resets on any SLAP-tier infraction** (not on lower-tier drift). Logic: drifting briefly is human; getting actually slapped means you broke focus. After a SLAP, you start earning rewards over again.

This creates a clear emotional loop: drift → cat slaps → recover → keep going → cat rewards. Both directions are screenshotable.

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

## 7. Mascot pipeline (2D, free stack)

The mascot is a **2D animated character**, authored as a single Rive state-machine file (`.riv`) with all reactions as named states/animations. Rendered in the overlay window via `@rive-app/canvas` or `@rive-app/react-canvas`.

### Why Rive

| Tool | Verdict |
|---|---|
| **Rive** | ✅ **Pick.** Free authoring tool, free runtime, native web playback, single `.riv` asset bundles all states + transitions, smooth skeletal 2D animation, ~50–500 KB per character. Built for product mascots. |
| Lottie | ✅ Solid alternative — author in After Effects (paid) or LottieFiles' free editor; single JSON per animation, but state transitions less elegant than Rive. |
| SVG + CSS animations | ✅ Absolute zero-tool path. Hand-author cat in SVG, animate state changes with CSS transitions/keyframes. Most flexible, hardest to make smooth. |
| Sprite sheets (PNG frames) | 🟡 Works, but big files and frame-by-frame authoring is tedious. |
| Live2D Cubism | 🟡 VTuber-grade rigging, more expressive but more authoring work. Overkill for v1. |
| GIF | ❌ Heavy, no interactivity, dated feel. |

**Pick: Rive** for v1. Authoring tool is genuinely good (free), runtime is web-native (`@rive-app/react-canvas`), and the state-machine model maps 1:1 to our reaction tiers (negative ladder + positive ladder = states; transitions = inputs from the engine).

### Animation list (single Rive file)

The `.riv` file contains all of these as named animations, driven by inputs from the engine:

| State | Source | Notes |
|---|---|---|
| `idle_breath` | Negative Tier 0 (default) | Loop. Cat sitting calmly, ear twitch, slow blink. |
| `alert` | Negative Tier 1 | Quick head-up, eyes open, ears perk. |
| `stare` | Negative Tier 2 | Sit upright, narrowed eyes, tail swish loop. |
| `paw_tap` | Negative Tier 3 | Lean in, soft screen tap. |
| `slap` | Negative Tier 4 | Chom-chom slap motion. The big one. |
| `outrage` | Negative Tier 5 | Hiss + sprint, fur puffed. |
| `pat_pat` | Reward R1 | Reach over, gentle screen pat. Encouraging. |
| `thumbs_up` | Reward R2 | Sit up, paw thumbs-up, sparkle. |
| `celebration` | Reward R3 | Happy spin / wiggle. |
| `reverence` | Reward R4 | Slow respectful bow. |
| `nap` | Idle 5+ min | Curled up, slow breathing. |
| `stretch` | End-of-session: clean | Big content stretch. |
| `headbutt` | End-of-session: decent | "You tried" headbutt. |
| `side_eye` | End-of-session: rough | Judgmental side-eye, tail flick. |
| `crying` | End-of-session: disaster | Single tear, slow head-down. (Crying cat reference.) |

15 animations in one file. Realistic to author in Rive in ~3–5 days for a competent illustrator/animator, faster with reused base motion.

### Bootstrap path (free)

1. Design the cat character on paper or in Procreate / Figma — single canonical pose, color, vibe. (Half-day.)
2. Open Rive (free at rive.app), import the canonical drawing as bones / vector layers.
3. Build the state machine and 15 named animations. Reuse base motion across related states (e.g., `slap` and `outrage` share approach motion).
4. Export `.riv`, drop into `assets/cat/whiskers.riv`.
5. In React: `<RiveCanvas src="whiskers.riv" stateMachine="cat" />`, drive state via inputs that the engine sends.

If you don't want to author yourself: commission a Rive cat on Fiverr / Behance for $100–400 with full source `.riv` file.

### Meme references in 2D

Memes referenced through *animation poses* and the static end-screen card art — not by republishing third-party meme images.

- `slap` references chom-chom slap cat silhouette and pose timing.
- `outrage` text overlay quotes "i am once again asking" / woman-yelling-at-cat phrasing.
- `crying` mimics crying-cat photo silhouette.
- `side_eye` references the side-eye cat meme.

### Tools (all free)

| Stage | Tool |
|---|---|
| Character design | Procreate ($13 one-time) or Figma (free) or Inkscape (free) |
| Authoring | **Rive** (free editor, free runtime) |
| Asset format | `.riv` file (~50–500 KB total) |
| Audio | royalty-free meow / hiss / slap / pat / chirrup SFX (freesound.org) |
| Frontend integration | `@rive-app/react-canvas` (free, open source) |

### Mascot effort estimate

- Day 1: design canonical cat character, finalize palette + proportions.
- Day 2–3: build Rive state machine + bones, author idle / alert / stare / nap base loops.
- Day 4–5: author slap / paw_tap / outrage + the four reward states.
- Day 6–7: author end-of-session states + share-card art.
- Day 8: polish, integration into React, smoke-test all transitions.

Total: ~1 to 1.5 weeks of focused work. Significantly less than the 3D pipeline (3+ weeks).

## 8. Tech stack

| Layer | Pick |
|---|---|
| Desktop shell | Tauri 2.x (Rust backend, native webview) |
| Frontend framework | Vite + React + TypeScript |
| **Mascot rendering** | **Rive (`@rive-app/react-canvas`)** — single `.riv` state-machine file |
| State (frontend) | Zustand |
| Active-window detection | `active-win-pos-rs` Rust crate |
| Window-title via Accessibility | macOS: `objc2` bridge to `AXUIElement`; Windows: `GetWindowTextW` |
| Idle detection | `user-idle` Rust crate |
| Local storage | SQLite via direct `sqlx` use (Rust-owned) |
| Audio | HTMLAudioElement in webview |
| Share-card rendering | HTML `<canvas>` + `html-to-image` (or pure canvas drawing) → PNG blob |
| Sharing | Direct deep-links — `twitter.com/intent/tweet`, `reddit.com/submit`, Discord webhook URL (user-configured) |
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
- **`Settings`** — app classifications editor, mute/snooze, sound preferences, mascot tweaks (positioning corner, scale), Discord webhook URL.
- **`CatOverlay`** — the transparent always-on-top window's contents: a `<RiveCanvas>` containing the cat + an `EffectsLayer` for text overlays (e.g., the "SLAP!" / "you got this 🌟" comic-text).
- **`CatRive`** — wraps `useRive`, holds inputs that the engine drives (negative tier, reward tier, end-state), drives the named animations.
- **`ShareCard`** — `<canvas>` PNG composer for end-of-session, with three explicit share buttons (X / Reddit / Discord) plus copy/save fallbacks.

### IPC events (renderer ← engine)

- `session:started` — duration, start time
- `session:tick` — current negative tier, current reward tier, elapsed, infraction count, clean-time accumulator
- `cat:react` — negative tier number, reason (which signal triggered)
- `cat:reward` — reward tier number (R1–R4), reason
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

CREATE TABLE rewards (
  id INTEGER PRIMARY KEY,
  session_id INTEGER NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  occurred_at INTEGER NOT NULL,
  reward_tier INTEGER NOT NULL CHECK (reward_tier BETWEEN 1 AND 4)
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

## 11. Privacy, permissions, and sharing

- **All data stays on-device by default.** No telemetry, no cloud, no analytics in v1.
- **Sharing is explicit user action only.** When the user clicks "Share to X / Reddit / Discord", the app:
  - Renders the share-card PNG locally in a `<canvas>`.
  - Copies the PNG to the system clipboard.
  - Opens the relevant intent URL in the user's default browser:
    - **X:** `https://twitter.com/intent/tweet?text=<encoded-caption>&url=<badcatsociety-site>`
    - **Reddit:** `https://www.reddit.com/submit?title=<encoded-title>&url=<badcatsociety-site>`
    - **Discord:** posts the PNG + caption to a user-configured Discord webhook URL (the only direct API call the app ever makes — and only when the user has explicitly set up a webhook in Settings).
  - The user manually attaches the (already-on-clipboard) PNG. App makes no other network calls.
- **macOS Accessibility permission** is requested once on first run, with a clear explanation: "Whiskers needs to read window titles to know when you've drifted." Without it, the app falls back to active-app-only detection (still functional, less precise).
- **Webcam (v2 only):** when implemented, runs ML on-device exclusively. No video frames or gaze data leave the machine.
- **Privacy policy** explicit on the site at launch even though there's nothing to disclose — sets the tone for the C-path Pro tier.

### Share-card visual

The share-card PNG (1080×1080, square — fits all three target platforms) renders client-side via `<canvas>`. It contains:
- Cat in the appropriate end-of-session pose (stretch / headbutt / side-eye / crying), exported as a PNG snapshot from the Rive runtime.
- Stats: duration, infractions, current streak, list of distracting apps caught.
- Verdict line ("I disappointed Whiskers today" / "Clean run! 🌟" etc.).
- Footer: `badcatsociety.app · 🐈‍⬛`.

A pre-canned caption is also generated for each platform with appropriate hashtags / formatting (e.g., on Reddit, suggests posting in r/productivity or r/getdisciplined).

## 12. Distribution

- **GitHub Releases** for direct downloads of `.dmg` (macOS), `.msi` (Windows), `.AppImage` (Linux best-effort).
- **macOS:** notarized via Apple Developer Program ($99/yr) — the one paid commitment, made before the launch wave to avoid scary Gatekeeper warnings for non-technical users.
- **Windows:** ship unsigned for v1 (SmartScreen warning is acceptable for an indie launch); revisit if launch goes well.
- **No app stores in v1.** Mac App Store has rules around always-on-top overlay windows and Accessibility permissions that complicate things; revisit post-launch.

## 13. v1 scope cuts (explicit non-goals)

What is *not* in v1, listed so the implementation plan can stay tight:

- 3D rendering of any kind (we picked 2D Rive)
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
- Programmatic posting to X or Reddit (would require user OAuth; v1 uses intent URLs + clipboard handoff instead). Discord webhook IS a direct post but only when the user explicitly configures a webhook URL.

## 14. Roadmap

**v1 (this spec) — viral launch.**
- Pomodoro focus sessions
- 6-tier negative escalation ladder + 4-tier positive reward ladder
- **2D animated cat (Rive)** with meme-referenced poses
- Active-app + window-title + idle detection
- Local SQLite, no cloud (except optional user-configured Discord webhook)
- One-click share to X / Reddit / Discord
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
- Reward thresholds — currently 5 / 15 / 30 / 60 min cumulative clean. These are guesses; may need tuning after first usage data.
- Share captions — auto-generated per result tier and per platform. Snarky? Earnest? Both? (Lean: snarky for negative results, earnest for clean runs.)
- Discord webhook setup UX — do we ship a guide, or assume users who configure webhooks know how?

## 16. Risks and mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Rive authoring takes longer than estimated for non-animator | Medium | Commission `.riv` from Fiverr ($100–400) as fallback; keep negotiable scope on number of distinct animations |
| Battery drain from Rive + 1Hz polling | Low | Rive runtime is light; pause animation when at idle nap state |
| Accessibility permission rejection on macOS | Medium | App still works (less precision); UI explicitly explains and points to System Settings |
| Cat feels too aggressive / off-putting | Medium | **Reward ladder mitigates this directly** — the cat is rooting for you, not just punishing. Default sound off; gradual de-escalation; snooze always available |
| Sharing intent URLs require manual image attach | Medium | Make the clipboard handoff loud and obvious — "Image copied! Paste in the post →" inline guidance after share click |
| Tauri ecosystem gap forces Rust deep-dive | Low | Pre-validated crates exist for every v1 signal; only the Accessibility bridge needs custom glue |

---

End of design spec. Implementation plan to follow via `superpowers:writing-plans`.
