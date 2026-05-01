# badcatsociety v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a working Tauri desktop app that runs user-declared focus sessions, detects active apps, classifies them, escalates a 6-tier reaction state, and shows the current tier in a small always-on-top overlay window. Uses a placeholder visual (colored tier badge) instead of the 3D cat — that ships in Plan 2.

**Architecture:** Rust backend owns the focus engine: a tokio broadcast `SignalBus` collects observations from pluggable detection signal implementations (`ForegroundSignal`, `IdleSignal`), an `AppClassifier` buckets each observation into WORK / DISTRACTION / UNCLASSIFIED, and an `EscalationStateMachine` advances tiers per the spec's rules. The `SessionController` owns the lifecycle (start / tick / snooze / end) and persists everything to SQLite via sqlx. Two Tauri windows: a popup for session config and a transparent always-on-top overlay for the tier badge. IPC is typed events emitted from Rust → React, plus typed `#[tauri::command]` calls React → Rust.

**Tech Stack:** Rust 2021 / Tauri 2.x / sqlx (sqlite) / tokio / active-win-pos-rs / user-idle / Vite + React 18 + TypeScript / Zustand.

**Scope deferred to follow-up plans:**
- 3D Three.js cat with Quaternius model (Plan 2)
- macOS Accessibility bridge for hard-to-read window titles (Plan 2)
- Audio (Plan 2)
- Default DISTRACTION seed list (Plan 3 — needs the settings UI to be useful)
- Share-card PNG generator (Plan 3)
- Settings UI for editing classifications, mute, scope=SESSION classifications, snooze button (Plan 3)
- Distribution / signing / GitHub Actions (Plan 4)

What this plan ships: a working desktop app where you start a focus session, the app polls your foreground app once a second, asks you "is this real work?" the first time it sees an unfamiliar app, slaps you (visually, via tier escalation in the overlay) when you switch to a distraction, and shows a session summary at the end.

---

## File structure

Created or modified by this plan:

```
badcatsociety/
├── package.json                       # frontend deps + tauri scripts
├── pnpm-lock.yaml
├── vite.config.ts                     # Vite build for popup + overlay entries
├── tsconfig.json
├── index.html                         # popup window entry
├── overlay.html                       # overlay window entry
├── src/                               # React frontend
│   ├── popup.tsx                      # popup entry + mount
│   ├── overlay.tsx                    # overlay entry + mount
│   ├── App.tsx                        # popup root component
│   ├── Overlay.tsx                    # overlay root component
│   ├── lib/
│   │   ├── ipc.ts                     # typed wrappers around invoke/emit/listen
│   │   └── store.ts                   # Zustand store
│   └── components/
│       ├── SessionConfig.tsx          # duration + work-app picker
│       ├── SessionTimer.tsx           # in-session display
│       ├── ClassificationPrompt.tsx   # "is this real work?" modal
│       ├── SessionSummary.tsx         # end-of-session result screen
│       └── TierBadge.tsx              # placeholder "cat" — colored emoji per tier
├── src-tauri/
│   ├── Cargo.toml
│   ├── tauri.conf.json                # two-window config
│   ├── build.rs
│   ├── icons/                         # default icons (from `tauri init`)
│   ├── migrations/
│   │   └── 001_initial.sql
│   └── src/
│       ├── main.rs                    # Tauri setup, app state, command registration
│       ├── ipc.rs                     # #[tauri::command] handlers + event types
│       ├── db.rs                      # SqlitePool setup + migrations
│       └── focus_engine/
│           ├── mod.rs                 # re-exports
│           ├── signals/
│           │   ├── mod.rs             # DetectionSignal trait, Observation, SignalBus
│           │   ├── foreground.rs      # ForegroundSignal (app + title)
│           │   └── idle.rs            # IdleSignal
│           ├── classifier.rs          # AppClassifier (Bucket enum)
│           ├── escalation.rs          # Tier enum + EscalationStateMachine
│           ├── repo.rs                # CRUD for app_classifications, sessions, infractions, streaks
│           ├── streak.rs              # streak update logic
│           └── session.rs             # SessionController (lifecycle + tick loop)
└── docs/
    └── superpowers/
        └── plans/2026-05-01-badcatsociety-v1.md   # this file
```

**Note on spec deviation:** the spec lists `ActiveAppSignal` and `WindowTitleSignal` as separate `DetectionSignal` plugins. In practice both come from one OS query (`active-win-pos-rs::get_active_window()` returns app and title together), so this plan implements them as a single `ForegroundSignal` emitting `{app_id, title}`. The classifier preserves the spec's logical separation by allowing rules to match on either field independently.

---

## Phase 1 — Project bootstrap

### Task 1: Initialize Tauri 2 project with React + TypeScript template

**Files:**
- Create: `package.json`, `vite.config.ts`, `tsconfig.json`, `index.html`, `src/popup.tsx`, `src/App.tsx`
- Create: `src-tauri/Cargo.toml`, `src-tauri/tauri.conf.json`, `src-tauri/build.rs`, `src-tauri/src/main.rs`, `src-tauri/icons/*`

- [ ] **Step 1: Run Tauri create-tauri-app**

```bash
cd /Users/yangli/badcatsociety
pnpm create tauri-app@latest . --template react-ts --manager pnpm --identifier app.badcatsociety
```

Answer prompts: project name `badcatsociety`, frontend `React + TypeScript`, package manager `pnpm`. If `pnpm` isn't installed: `npm install -g pnpm` first.

- [ ] **Step 2: Install dependencies**

Run: `pnpm install`
Then: `cd src-tauri && cargo build && cd ..`

Expected: both succeed (Tauri pulls a lot of crates; ~2-5 min on first build).

- [ ] **Step 3: Verify dev mode launches**

Run: `pnpm tauri dev`
Expected: a window opens showing the default Tauri+React landing page. Close it with Cmd+Q.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "chore: bootstrap Tauri 2 + React + TS project"
```

---

### Task 2: Add Rust backend dependencies

**Files:**
- Modify: `src-tauri/Cargo.toml`

- [ ] **Step 1: Add deps to Cargo.toml**

Open `src-tauri/Cargo.toml`, replace the `[dependencies]` block with:

```toml
[dependencies]
tauri = { version = "2", features = [] }
tauri-plugin-shell = "2"
tauri-plugin-opener = "2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["rt-multi-thread", "macros", "sync", "time"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite", "migrate", "macros"] }
active-win-pos-rs = "0.9"
user-idle = "0.6"
thiserror = "1"
chrono = { version = "0.4", features = ["serde"] }
tracing = "0.1"
tracing-subscriber = "0.3"

[dev-dependencies]
tokio = { version = "1", features = ["test-util", "macros", "rt"] }
```

- [ ] **Step 2: Verify it builds**

Run: `cd src-tauri && cargo build && cd ..`
Expected: success. Crate downloads will happen on first run.

- [ ] **Step 3: Commit**

```bash
git add src-tauri/Cargo.toml src-tauri/Cargo.lock
git commit -m "chore: add focus engine Rust dependencies"
```

---

### Task 3: Add frontend dependencies

**Files:**
- Modify: `package.json`

- [ ] **Step 1: Install frontend deps**

```bash
pnpm add zustand
pnpm add -D @types/node
```

- [ ] **Step 2: Verify**

Run: `pnpm build`
Expected: TypeScript build succeeds.

- [ ] **Step 3: Commit**

```bash
git add package.json pnpm-lock.yaml
git commit -m "chore: add Zustand for frontend state"
```

---

## Phase 2 — Database

### Task 4: SQLite migration + connection pool

**Files:**
- Create: `src-tauri/migrations/001_initial.sql`
- Create: `src-tauri/src/db.rs`
- Modify: `src-tauri/src/main.rs` to wire it up

- [ ] **Step 1: Write the migration SQL**

Create `src-tauri/migrations/001_initial.sql`:

```sql
CREATE TABLE app_classifications (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  app_id TEXT NOT NULL,
  title_pattern TEXT,
  bucket TEXT NOT NULL CHECK (bucket IN ('WORK', 'DISTRACTION')),
  scope TEXT NOT NULL CHECK (scope IN ('GLOBAL', 'SESSION')),
  created_at INTEGER NOT NULL,
  UNIQUE (app_id, title_pattern, scope)
);

CREATE TABLE sessions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  started_at INTEGER NOT NULL,
  ended_at INTEGER,
  duration_secs INTEGER NOT NULL,
  ended_early INTEGER NOT NULL DEFAULT 0,
  infractions INTEGER NOT NULL DEFAULT 0,
  result TEXT CHECK (result IN ('CLEAN','DECENT','ROUGH','DISASTER'))
);

CREATE TABLE infractions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
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
  id INTEGER PRIMARY KEY CHECK (id = 1),
  current INTEGER NOT NULL DEFAULT 0,
  best INTEGER NOT NULL DEFAULT 0,
  last_clean_session_at INTEGER
);

INSERT INTO streaks (id) VALUES (1);
```

- [ ] **Step 2: Write the db module with a failing test**

Create `src-tauri/src/db.rs`:

```rust
use sqlx::sqlite::{SqliteConnectOptions, SqlitePoolOptions};
use sqlx::SqlitePool;
use std::path::Path;
use std::str::FromStr;

pub type DbResult<T> = Result<T, sqlx::Error>;

pub async fn open(db_path: &Path) -> DbResult<SqlitePool> {
    let opts = SqliteConnectOptions::from_str(&format!("sqlite://{}", db_path.display()))?
        .create_if_missing(true);
    let pool = SqlitePoolOptions::new().max_connections(4).connect_with(opts).await?;
    sqlx::migrate!("./migrations").run(&pool).await?;
    Ok(pool)
}

pub async fn open_in_memory() -> DbResult<SqlitePool> {
    let opts = SqliteConnectOptions::from_str("sqlite::memory:")?;
    let pool = SqlitePoolOptions::new().max_connections(1).connect_with(opts).await?;
    sqlx::migrate!("./migrations").run(&pool).await?;
    Ok(pool)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn migrations_apply_on_fresh_in_memory_db() {
        let pool = open_in_memory().await.expect("open in-memory");
        let row: (i64,) = sqlx::query_as("SELECT current FROM streaks WHERE id = 1")
            .fetch_one(&pool).await.expect("seed row exists");
        assert_eq!(row.0, 0);
    }
}
```

- [ ] **Step 3: Run the test, expect FAIL until module is wired**

Add `mod db;` to `src-tauri/src/main.rs` (top of file, alongside existing modules).

Run: `cd src-tauri && cargo test --lib db -- --nocapture`
Expected: PASS. (If migrations file isn't found at compile time, double-check the relative path `./migrations` matches your `Cargo.toml` location.)

- [ ] **Step 4: Commit**

```bash
git add src-tauri/migrations src-tauri/src/db.rs src-tauri/src/main.rs
git commit -m "feat(db): SQLite schema migrations + in-memory test pool"
```

---

## Phase 3 — Detection signal infrastructure

### Task 5: Define DetectionSignal trait + Observation + SignalBus

**Files:**
- Create: `src-tauri/src/focus_engine/mod.rs`
- Create: `src-tauri/src/focus_engine/signals/mod.rs`
- Modify: `src-tauri/src/main.rs` (add `mod focus_engine;`)

- [ ] **Step 1: Add module declarations**

In `src-tauri/src/main.rs`, near the top:

```rust
mod db;
mod focus_engine;
```

Create `src-tauri/src/focus_engine/mod.rs`:

```rust
pub mod signals;
```

- [ ] **Step 2: Write failing test for SignalBus pub/sub**

Create `src-tauri/src/focus_engine/signals/mod.rs`:

```rust
use serde::{Deserialize, Serialize};
use tokio::sync::broadcast;

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct Observation {
    pub signal_id: String,
    pub app_id: Option<String>,
    pub title: Option<String>,
    pub idle_secs: Option<u64>,
    pub timestamp_ms: i64,
}

#[derive(Clone)]
pub struct SignalBus {
    sender: broadcast::Sender<Observation>,
}

impl SignalBus {
    pub fn new(capacity: usize) -> Self {
        let (sender, _) = broadcast::channel(capacity);
        Self { sender }
    }

    pub fn publish(&self, obs: Observation) {
        let _ = self.sender.send(obs);
    }

    pub fn subscribe(&self) -> broadcast::Receiver<Observation> {
        self.sender.subscribe()
    }
}

#[async_trait::async_trait]
pub trait DetectionSignal: Send + Sync {
    fn id(&self) -> &'static str;
    async fn poll(&mut self) -> Option<Observation>;
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn bus_delivers_published_observations_to_subscribers() {
        let bus = SignalBus::new(8);
        let mut rx = bus.subscribe();
        let obs = Observation {
            signal_id: "test".into(),
            app_id: Some("com.test".into()),
            title: Some("hello".into()),
            idle_secs: None,
            timestamp_ms: 1000,
        };
        bus.publish(obs.clone());
        let received = rx.recv().await.expect("receives");
        assert_eq!(received, obs);
    }
}
```

- [ ] **Step 3: Add `async-trait` to Cargo.toml**

Add to `[dependencies]`:

```toml
async-trait = "0.1"
```

- [ ] **Step 4: Run the test**

Run: `cd src-tauri && cargo test --lib signals`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src-tauri/Cargo.toml src-tauri/Cargo.lock src-tauri/src/main.rs src-tauri/src/focus_engine
git commit -m "feat(engine): DetectionSignal trait, Observation, SignalBus"
```

---

### Task 6: Implement IdleSignal (provider abstraction for testability)

**Files:**
- Create: `src-tauri/src/focus_engine/signals/idle.rs`
- Modify: `src-tauri/src/focus_engine/signals/mod.rs` (add `pub mod idle;`)

- [ ] **Step 1: Write the failing test**

Create `src-tauri/src/focus_engine/signals/idle.rs`:

```rust
use super::{DetectionSignal, Observation};
use async_trait::async_trait;
use chrono::Utc;

pub trait IdleProvider: Send + Sync {
    fn idle_secs(&self) -> u64;
}

pub struct SystemIdleProvider;

impl IdleProvider for SystemIdleProvider {
    fn idle_secs(&self) -> u64 {
        user_idle::UserIdle::get_time()
            .map(|d| d.as_seconds())
            .unwrap_or(0)
    }
}

pub struct IdleSignal<P: IdleProvider> {
    provider: P,
}

impl<P: IdleProvider> IdleSignal<P> {
    pub fn new(provider: P) -> Self { Self { provider } }
}

#[async_trait]
impl<P: IdleProvider + 'static> DetectionSignal for IdleSignal<P> {
    fn id(&self) -> &'static str { "idle" }

    async fn poll(&mut self) -> Option<Observation> {
        let secs = self.provider.idle_secs();
        Some(Observation {
            signal_id: "idle".into(),
            app_id: None,
            title: None,
            idle_secs: Some(secs),
            timestamp_ms: Utc::now().timestamp_millis(),
        })
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    struct FixedIdle(u64);
    impl IdleProvider for FixedIdle { fn idle_secs(&self) -> u64 { self.0 } }

    #[tokio::test]
    async fn idle_signal_emits_provider_value() {
        let mut sig = IdleSignal::new(FixedIdle(42));
        let obs = sig.poll().await.expect("emits");
        assert_eq!(obs.signal_id, "idle");
        assert_eq!(obs.idle_secs, Some(42));
    }
}
```

- [ ] **Step 2: Wire the module**

In `src-tauri/src/focus_engine/signals/mod.rs`, add at the top:

```rust
pub mod idle;
```

- [ ] **Step 3: Run the test**

Run: `cd src-tauri && cargo test --lib signals::idle`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/focus_engine/signals
git commit -m "feat(engine): IdleSignal with provider abstraction"
```

---

### Task 7: Implement ForegroundSignal (active app + title)

**Files:**
- Create: `src-tauri/src/focus_engine/signals/foreground.rs`
- Modify: `src-tauri/src/focus_engine/signals/mod.rs` (add `pub mod foreground;`)

- [ ] **Step 1: Write the failing test**

Create `src-tauri/src/focus_engine/signals/foreground.rs`:

```rust
use super::{DetectionSignal, Observation};
use async_trait::async_trait;
use chrono::Utc;

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct ForegroundWindow {
    pub app_id: String,
    pub title: String,
}

pub trait ForegroundProvider: Send + Sync {
    fn current(&self) -> Option<ForegroundWindow>;
}

pub struct SystemForegroundProvider;

impl ForegroundProvider for SystemForegroundProvider {
    fn current(&self) -> Option<ForegroundWindow> {
        active_win_pos_rs::get_active_window().ok().map(|w| ForegroundWindow {
            app_id: w.process_path.to_string_lossy().to_string(),
            title: w.title,
        })
    }
}

pub struct ForegroundSignal<P: ForegroundProvider> {
    provider: P,
}

impl<P: ForegroundProvider> ForegroundSignal<P> {
    pub fn new(provider: P) -> Self { Self { provider } }
}

#[async_trait]
impl<P: ForegroundProvider + 'static> DetectionSignal for ForegroundSignal<P> {
    fn id(&self) -> &'static str { "foreground" }

    async fn poll(&mut self) -> Option<Observation> {
        let fg = self.provider.current()?;
        Some(Observation {
            signal_id: "foreground".into(),
            app_id: Some(fg.app_id),
            title: Some(fg.title),
            idle_secs: None,
            timestamp_ms: Utc::now().timestamp_millis(),
        })
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    struct FixedFg(Option<ForegroundWindow>);
    impl ForegroundProvider for FixedFg {
        fn current(&self) -> Option<ForegroundWindow> { self.0.clone() }
    }

    #[tokio::test]
    async fn foreground_signal_emits_app_and_title() {
        let provider = FixedFg(Some(ForegroundWindow {
            app_id: "com.apple.Safari".into(),
            title: "Twitter / X".into(),
        }));
        let mut sig = ForegroundSignal::new(provider);
        let obs = sig.poll().await.expect("emits");
        assert_eq!(obs.app_id.as_deref(), Some("com.apple.Safari"));
        assert_eq!(obs.title.as_deref(), Some("Twitter / X"));
    }

    #[tokio::test]
    async fn foreground_signal_emits_none_when_no_window() {
        let mut sig = ForegroundSignal::new(FixedFg(None));
        assert!(sig.poll().await.is_none());
    }
}
```

- [ ] **Step 2: Wire the module**

In `src-tauri/src/focus_engine/signals/mod.rs`:

```rust
pub mod foreground;
```

- [ ] **Step 3: Run the test**

Run: `cd src-tauri && cargo test --lib signals::foreground`
Expected: both tests PASS.

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/focus_engine/signals
git commit -m "feat(engine): ForegroundSignal with provider abstraction"
```

---

## Phase 4 — Repository (SQLite CRUD)

### Task 8: Implement repo for app_classifications + sessions + infractions + streaks

**Files:**
- Create: `src-tauri/src/focus_engine/repo.rs`
- Modify: `src-tauri/src/focus_engine/mod.rs` (add `pub mod repo;`)

- [ ] **Step 1: Write the failing tests**

Create `src-tauri/src/focus_engine/repo.rs`:

```rust
use serde::{Deserialize, Serialize};
use sqlx::SqlitePool;

#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq, sqlx::Type)]
#[serde(rename_all = "UPPERCASE")]
#[sqlx(rename_all = "UPPERCASE")]
pub enum Bucket { Work, Distraction }

impl Bucket {
    pub fn as_str(&self) -> &'static str {
        match self { Bucket::Work => "WORK", Bucket::Distraction => "DISTRACTION" }
    }
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
pub enum Result_ { Clean, Decent, Rough, Disaster }

impl Result_ {
    pub fn as_str(&self) -> &'static str {
        match self {
            Result_::Clean => "CLEAN",
            Result_::Decent => "DECENT",
            Result_::Rough => "ROUGH",
            Result_::Disaster => "DISASTER",
        }
    }
    pub fn from_infraction_count(n: u32, ended_early: bool) -> Self {
        if ended_early || n >= 6 { Result_::Disaster }
        else if n >= 3 { Result_::Rough }
        else if n >= 1 { Result_::Decent }
        else { Result_::Clean }
    }
}

#[derive(Debug, Clone, Serialize)]
pub struct ClassificationRow {
    pub app_id: String,
    pub title_pattern: Option<String>,
    pub bucket: Bucket,
}

pub async fn upsert_classification(
    pool: &SqlitePool,
    app_id: &str,
    title_pattern: Option<&str>,
    bucket: Bucket,
) -> Result<(), sqlx::Error> {
    let now = chrono::Utc::now().timestamp_millis();
    sqlx::query(
        "INSERT INTO app_classifications (app_id, title_pattern, bucket, scope, created_at)
         VALUES (?, ?, ?, 'GLOBAL', ?)
         ON CONFLICT(app_id, title_pattern, scope) DO UPDATE SET bucket = excluded.bucket"
    )
    .bind(app_id)
    .bind(title_pattern)
    .bind(bucket.as_str())
    .bind(now)
    .execute(pool).await?;
    Ok(())
}

pub async fn list_classifications(pool: &SqlitePool) -> Result<Vec<ClassificationRow>, sqlx::Error> {
    let rows = sqlx::query_as::<_, (String, Option<String>, String)>(
        "SELECT app_id, title_pattern, bucket FROM app_classifications WHERE scope = 'GLOBAL'"
    )
    .fetch_all(pool).await?;
    Ok(rows.into_iter().map(|(a, t, b)| ClassificationRow {
        app_id: a,
        title_pattern: t,
        bucket: if b == "WORK" { Bucket::Work } else { Bucket::Distraction },
    }).collect())
}

pub async fn create_session(pool: &SqlitePool, started_at: i64, duration_secs: i64) -> Result<i64, sqlx::Error> {
    let res = sqlx::query("INSERT INTO sessions (started_at, duration_secs) VALUES (?, ?)")
        .bind(started_at).bind(duration_secs).execute(pool).await?;
    Ok(res.last_insert_rowid())
}

pub async fn finalize_session(
    pool: &SqlitePool,
    id: i64,
    ended_at: i64,
    ended_early: bool,
    infractions: u32,
    result: Result_,
) -> Result<(), sqlx::Error> {
    sqlx::query(
        "UPDATE sessions SET ended_at = ?, ended_early = ?, infractions = ?, result = ? WHERE id = ?"
    )
    .bind(ended_at)
    .bind(ended_early as i64)
    .bind(infractions as i64)
    .bind(result.as_str())
    .bind(id)
    .execute(pool).await?;
    Ok(())
}

pub async fn record_infraction(
    pool: &SqlitePool,
    session_id: i64,
    occurred_at: i64,
    app_id: &str,
    title: Option<&str>,
    tier_reached: i64,
) -> Result<(), sqlx::Error> {
    sqlx::query(
        "INSERT INTO infractions (session_id, occurred_at, app_id, title, tier_reached) VALUES (?, ?, ?, ?, ?)"
    )
    .bind(session_id).bind(occurred_at).bind(app_id).bind(title).bind(tier_reached)
    .execute(pool).await?;
    Ok(())
}

pub async fn current_streak(pool: &SqlitePool) -> Result<(i64, i64), sqlx::Error> {
    let row: (i64, i64) = sqlx::query_as("SELECT current, best FROM streaks WHERE id = 1")
        .fetch_one(pool).await?;
    Ok(row)
}

pub async fn update_streak(pool: &SqlitePool, clean: bool) -> Result<(i64, i64), sqlx::Error> {
    if clean {
        sqlx::query(
            "UPDATE streaks SET current = current + 1,
                                best = MAX(best, current + 1),
                                last_clean_session_at = ?
             WHERE id = 1"
        ).bind(chrono::Utc::now().timestamp_millis()).execute(pool).await?;
    } else {
        sqlx::query("UPDATE streaks SET current = 0 WHERE id = 1").execute(pool).await?;
    }
    current_streak(pool).await
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::db;

    #[tokio::test]
    async fn classifications_upsert_then_list() {
        let pool = db::open_in_memory().await.unwrap();
        upsert_classification(&pool, "com.apple.Safari", None, Bucket::Distraction).await.unwrap();
        upsert_classification(&pool, "com.microsoft.VSCode", None, Bucket::Work).await.unwrap();
        let all = list_classifications(&pool).await.unwrap();
        assert_eq!(all.len(), 2);
    }

    #[tokio::test]
    async fn classifications_upsert_overwrites_bucket() {
        let pool = db::open_in_memory().await.unwrap();
        upsert_classification(&pool, "com.foo.App", None, Bucket::Work).await.unwrap();
        upsert_classification(&pool, "com.foo.App", None, Bucket::Distraction).await.unwrap();
        let all = list_classifications(&pool).await.unwrap();
        assert_eq!(all.len(), 1);
        assert_eq!(all[0].bucket, Bucket::Distraction);
    }

    #[tokio::test]
    async fn sessions_lifecycle() {
        let pool = db::open_in_memory().await.unwrap();
        let id = create_session(&pool, 1000, 1500).await.unwrap();
        record_infraction(&pool, id, 1100, "com.foo.App", Some("title"), 4).await.unwrap();
        finalize_session(&pool, id, 2500, false, 1, Result_::Decent).await.unwrap();
        let row: (i64, i64, i64, String) = sqlx::query_as(
            "SELECT id, infractions, ended_at, result FROM sessions WHERE id = ?"
        ).bind(id).fetch_one(&pool).await.unwrap();
        assert_eq!(row.1, 1);
        assert_eq!(row.2, 2500);
        assert_eq!(row.3, "DECENT");
    }

    #[tokio::test]
    async fn streak_increments_on_clean_resets_on_dirty() {
        let pool = db::open_in_memory().await.unwrap();
        let (s1, _) = update_streak(&pool, true).await.unwrap();
        let (s2, _) = update_streak(&pool, true).await.unwrap();
        let (s3, _) = update_streak(&pool, false).await.unwrap();
        assert_eq!((s1, s2, s3), (1, 2, 0));
    }

    #[test]
    fn result_classification_thresholds() {
        assert_eq!(Result_::from_infraction_count(0, false), Result_::Clean);
        assert_eq!(Result_::from_infraction_count(2, false), Result_::Decent);
        assert_eq!(Result_::from_infraction_count(5, false), Result_::Rough);
        assert_eq!(Result_::from_infraction_count(6, false), Result_::Disaster);
        assert_eq!(Result_::from_infraction_count(0, true), Result_::Disaster);
    }
}
```

- [ ] **Step 2: Wire the module**

In `src-tauri/src/focus_engine/mod.rs`:

```rust
pub mod signals;
pub mod repo;
```

- [ ] **Step 3: Run the tests**

Run: `cd src-tauri && cargo test --lib repo`
Expected: 5 tests PASS.

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/focus_engine
git commit -m "feat(engine): SQLite repo for classifications, sessions, infractions, streaks"
```

---

## Phase 5 — Classifier

### Task 9: Implement AppClassifier

**Files:**
- Create: `src-tauri/src/focus_engine/classifier.rs`
- Modify: `src-tauri/src/focus_engine/mod.rs` (add `pub mod classifier;`)

- [ ] **Step 1: Write failing tests**

Create `src-tauri/src/focus_engine/classifier.rs`:

```rust
use crate::focus_engine::repo::{Bucket, ClassificationRow};
use std::collections::HashMap;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Classification { Work, Distraction, Unclassified }

#[derive(Default)]
pub struct AppClassifier {
    by_app: HashMap<String, Bucket>,
    title_patterns: Vec<(String, String, Bucket)>,
}

impl AppClassifier {
    pub fn from_rows(rows: Vec<ClassificationRow>) -> Self {
        let mut c = AppClassifier::default();
        for r in rows {
            match r.title_pattern {
                Some(p) => c.title_patterns.push((r.app_id, p, r.bucket)),
                None => { c.by_app.insert(r.app_id, r.bucket); }
            }
        }
        c
    }

    pub fn classify(&self, app_id: &str, title: Option<&str>) -> Classification {
        if let Some(t) = title {
            for (pat_app, pat_title, bucket) in &self.title_patterns {
                if pat_app == app_id && t.to_lowercase().contains(&pat_title.to_lowercase()) {
                    return match bucket { Bucket::Work => Classification::Work, Bucket::Distraction => Classification::Distraction };
                }
            }
        }
        match self.by_app.get(app_id) {
            Some(Bucket::Work) => Classification::Work,
            Some(Bucket::Distraction) => Classification::Distraction,
            None => Classification::Unclassified,
        }
    }

    pub fn upsert_app(&mut self, app_id: String, bucket: Bucket) {
        self.by_app.insert(app_id, bucket);
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn classify_unknown_app_is_unclassified() {
        let c = AppClassifier::default();
        assert_eq!(c.classify("com.unknown.app", None), Classification::Unclassified);
    }

    #[test]
    fn classify_known_app_returns_bucket() {
        let mut c = AppClassifier::default();
        c.upsert_app("com.microsoft.VSCode".into(), Bucket::Work);
        c.upsert_app("com.tinyspeck.slackmacgap".into(), Bucket::Distraction);
        assert_eq!(c.classify("com.microsoft.VSCode", None), Classification::Work);
        assert_eq!(c.classify("com.tinyspeck.slackmacgap", None), Classification::Distraction);
    }

    #[test]
    fn classify_title_pattern_overrides_app_bucket() {
        let rows = vec![
            ClassificationRow { app_id: "com.google.Chrome".into(), title_pattern: None, bucket: Bucket::Work },
            ClassificationRow { app_id: "com.google.Chrome".into(), title_pattern: Some("twitter".into()), bucket: Bucket::Distraction },
        ];
        let c = AppClassifier::from_rows(rows);
        assert_eq!(c.classify("com.google.Chrome", Some("Docs - Google Chrome")), Classification::Work);
        assert_eq!(c.classify("com.google.Chrome", Some("Twitter / X — Google Chrome")), Classification::Distraction);
    }
}
```

- [ ] **Step 2: Wire the module**

In `src-tauri/src/focus_engine/mod.rs`:

```rust
pub mod signals;
pub mod repo;
pub mod classifier;
```

- [ ] **Step 3: Run the tests**

Run: `cd src-tauri && cargo test --lib classifier`
Expected: 3 tests PASS.

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/focus_engine
git commit -m "feat(engine): AppClassifier with title-pattern overrides"
```

---

## Phase 6 — Escalation state machine

### Task 10: Define Tier enum + EscalationStateMachine + tick logic

**Files:**
- Create: `src-tauri/src/focus_engine/escalation.rs`
- Modify: `src-tauri/src/focus_engine/mod.rs` (add `pub mod escalation;`)

- [ ] **Step 1: Write the failing tests**

Create `src-tauri/src/focus_engine/escalation.rs`:

```rust
use crate::focus_engine::classifier::Classification;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum Tier {
    Napping = 0,
    Alert = 1,
    Stare = 2,
    PawTap = 3,
    Slap = 4,
    Outrage = 5,
}

impl Tier {
    pub fn as_int(self) -> u8 { self as u8 }
    pub fn from_int(n: u8) -> Tier {
        match n {
            0 => Tier::Napping, 1 => Tier::Alert, 2 => Tier::Stare,
            3 => Tier::PawTap, 4 => Tier::Slap, _ => Tier::Outrage,
        }
    }
}

const ESCALATION_THRESHOLDS_SECS: [u64; 6] = [0, 0, 5, 15, 30, 60];
const DE_ESCALATE_AFTER_CLEAN_SECS: u64 = 30;
const IDLE_NAP_SECS: u64 = 5 * 60;

#[derive(Debug, Clone)]
pub struct EscalationState {
    pub tier: Tier,
    pub time_in_distraction_secs: u64,
    pub time_clean_secs: u64,
    pub session_infractions: u32,
}

impl Default for EscalationState {
    fn default() -> Self {
        Self {
            tier: Tier::Napping,
            time_in_distraction_secs: 0,
            time_clean_secs: 0,
            session_infractions: 0,
        }
    }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum TickOutcome {
    NoChange { tier: Tier },
    TierChanged { from: Tier, to: Tier, reason: TickReason },
    Infraction { tier: Tier },
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TickReason { Distraction, Unclassified, Idle, Clean }

pub fn tick(
    state: &mut EscalationState,
    classification: Classification,
    idle_secs: u64,
    elapsed_since_last_tick_secs: u64,
) -> TickOutcome {
    let prev_tier = state.tier;

    if idle_secs >= IDLE_NAP_SECS {
        state.time_in_distraction_secs = 0;
        state.time_clean_secs = state.time_clean_secs.saturating_add(elapsed_since_last_tick_secs);
        state.tier = Tier::Napping;
        return outcome(prev_tier, state.tier, TickReason::Idle);
    }

    match classification {
        Classification::Work => {
            state.time_in_distraction_secs = 0;
            state.time_clean_secs = state.time_clean_secs.saturating_add(elapsed_since_last_tick_secs);
            state.tier = de_escalate(state.tier, state.time_clean_secs);
            outcome(prev_tier, state.tier, TickReason::Clean)
        }
        Classification::Unclassified => {
            state.time_in_distraction_secs = 0;
            state.time_clean_secs = 0;
            if prev_tier == Tier::Napping {
                state.tier = Tier::Alert;
            }
            outcome(prev_tier, state.tier, TickReason::Unclassified)
        }
        Classification::Distraction => {
            state.time_in_distraction_secs = state.time_in_distraction_secs.saturating_add(elapsed_since_last_tick_secs);
            state.time_clean_secs = 0;
            let new_tier = tier_for_distraction_secs(state.time_in_distraction_secs, state.session_infractions);
            let was_lower = prev_tier.as_int() < Tier::Slap.as_int();
            state.tier = new_tier;
            let crossed_into_slap = was_lower && new_tier.as_int() >= Tier::Slap.as_int();
            if crossed_into_slap {
                state.session_infractions = state.session_infractions.saturating_add(1);
                return TickOutcome::Infraction { tier: new_tier };
            }
            outcome(prev_tier, state.tier, TickReason::Distraction)
        }
    }
}

fn outcome(from: Tier, to: Tier, reason: TickReason) -> TickOutcome {
    if from == to { TickOutcome::NoChange { tier: to } }
    else { TickOutcome::TierChanged { from, to, reason } }
}

fn tier_for_distraction_secs(secs: u64, prior_infractions: u32) -> Tier {
    if prior_infractions >= 4 { return Tier::Outrage; }
    let mut highest = Tier::Napping;
    for (i, threshold) in ESCALATION_THRESHOLDS_SECS.iter().enumerate() {
        if secs >= *threshold { highest = Tier::from_int(i as u8); }
    }
    highest
}

fn de_escalate(tier: Tier, clean_secs: u64) -> Tier {
    if clean_secs >= DE_ESCALATE_AFTER_CLEAN_SECS { return Tier::Napping; }
    let drops = (clean_secs / 5) as u8;
    let cur = tier.as_int();
    Tier::from_int(cur.saturating_sub(drops))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn work_keeps_napping_and_de_escalates() {
        let mut s = EscalationState { tier: Tier::Slap, time_in_distraction_secs: 30, time_clean_secs: 0, session_infractions: 1 };
        for _ in 0..6 { tick(&mut s, Classification::Work, 0, 5); } // 30s clean
        assert_eq!(s.tier, Tier::Napping);
    }

    #[test]
    fn distraction_escalates_through_tiers() {
        let mut s = EscalationState::default();
        tick(&mut s, Classification::Distraction, 0, 1);
        assert_eq!(s.tier, Tier::Alert);
        tick(&mut s, Classification::Distraction, 0, 5);  // total 6s
        assert_eq!(s.tier, Tier::Stare);
        tick(&mut s, Classification::Distraction, 0, 10); // total 16s
        assert_eq!(s.tier, Tier::PawTap);
        let outcome = tick(&mut s, Classification::Distraction, 0, 15); // total 31s → SLAP
        assert!(matches!(outcome, TickOutcome::Infraction { tier: Tier::Slap }));
        assert_eq!(s.session_infractions, 1);
    }

    #[test]
    fn fourth_infraction_jumps_to_outrage() {
        let mut s = EscalationState { tier: Tier::Napping, time_in_distraction_secs: 0, time_clean_secs: 0, session_infractions: 4 };
        tick(&mut s, Classification::Distraction, 0, 1);
        assert_eq!(s.tier, Tier::Outrage);
    }

    #[test]
    fn idle_5_min_naps_regardless_of_classification() {
        let mut s = EscalationState { tier: Tier::Slap, time_in_distraction_secs: 30, time_clean_secs: 0, session_infractions: 1 };
        let out = tick(&mut s, Classification::Distraction, 5 * 60, 1);
        assert!(matches!(out, TickOutcome::TierChanged { to: Tier::Napping, .. } | TickOutcome::NoChange { tier: Tier::Napping }));
        assert_eq!(s.tier, Tier::Napping);
    }

    #[test]
    fn unclassified_pops_to_alert_only_from_napping() {
        let mut s = EscalationState::default();
        tick(&mut s, Classification::Unclassified, 0, 1);
        assert_eq!(s.tier, Tier::Alert);
        // Already Alert: stays Alert (does not escalate just from unclassified)
        tick(&mut s, Classification::Unclassified, 0, 5);
        assert_eq!(s.tier, Tier::Alert);
    }
}
```

- [ ] **Step 2: Wire the module**

In `src-tauri/src/focus_engine/mod.rs`:

```rust
pub mod signals;
pub mod repo;
pub mod classifier;
pub mod escalation;
```

- [ ] **Step 3: Run the tests**

Run: `cd src-tauri && cargo test --lib escalation`
Expected: 5 tests PASS.

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/focus_engine
git commit -m "feat(engine): EscalationStateMachine with 6-tier ladder + de-escalation"
```

---

## Phase 7 — Session controller

### Task 11: SessionController lifecycle + tick loop

**Files:**
- Create: `src-tauri/src/focus_engine/session.rs`
- Modify: `src-tauri/src/focus_engine/mod.rs` (add `pub mod session;`)

- [ ] **Step 1: Write the failing tests**

Create `src-tauri/src/focus_engine/session.rs`:

```rust
use crate::focus_engine::classifier::{AppClassifier, Classification};
use crate::focus_engine::escalation::{tick, EscalationState, Tier, TickOutcome};
use crate::focus_engine::repo::{self, Result_};
use crate::focus_engine::signals::Observation;
use chrono::Utc;
use serde::Serialize;
use sqlx::SqlitePool;
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Debug, Clone, Serialize, PartialEq, Eq)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum SessionEvent {
    Started { session_id: i64, duration_secs: i64 },
    Tick { tier: u8, elapsed_secs: i64, infractions: u32 },
    AskClassify { app_id: String, title: Option<String> },
    Reaction { tier: u8, reason: String },
    Ended { result: String, infractions: u32, ended_early: bool, streak: i64 },
}

pub struct SessionController {
    pool: SqlitePool,
    inner: Arc<Mutex<Inner>>,
}

struct Inner {
    classifier: AppClassifier,
    escalation: EscalationState,
    session_id: Option<i64>,
    started_at_ms: i64,
    duration_secs: i64,
    last_observation: Option<Observation>,
    last_idle_secs: u64,
    pending_classify_app: Option<String>,
}

impl SessionController {
    pub async fn new(pool: SqlitePool) -> Result<Self, sqlx::Error> {
        let rows = repo::list_classifications(&pool).await?;
        Ok(Self {
            pool,
            inner: Arc::new(Mutex::new(Inner {
                classifier: AppClassifier::from_rows(rows),
                escalation: EscalationState::default(),
                session_id: None,
                started_at_ms: 0,
                duration_secs: 0,
                last_observation: None,
                last_idle_secs: 0,
                pending_classify_app: None,
            })),
        })
    }

    pub async fn start(&self, duration_secs: i64) -> Result<SessionEvent, sqlx::Error> {
        let now = Utc::now().timestamp_millis();
        let id = repo::create_session(&self.pool, now, duration_secs).await?;
        let mut inner = self.inner.lock().await;
        inner.session_id = Some(id);
        inner.started_at_ms = now;
        inner.duration_secs = duration_secs;
        inner.escalation = EscalationState::default();
        inner.pending_classify_app = None;
        Ok(SessionEvent::Started { session_id: id, duration_secs })
    }

    /// Apply one observation to the engine. Returns events to emit to the UI.
    pub async fn apply(&self, obs: Observation, elapsed_since_last_tick_secs: u64) -> Vec<SessionEvent> {
        let mut inner = self.inner.lock().await;
        let mut events = Vec::new();

        if obs.signal_id == "idle" {
            inner.last_idle_secs = obs.idle_secs.unwrap_or(0);
            return events;
        }

        let app_id = obs.app_id.clone().unwrap_or_default();
        let title = obs.title.clone();
        inner.last_observation = Some(obs.clone());

        let classification = inner.classifier.classify(&app_id, title.as_deref());

        if classification == Classification::Unclassified
            && inner.pending_classify_app.as_deref() != Some(&app_id)
        {
            inner.pending_classify_app = Some(app_id.clone());
            events.push(SessionEvent::AskClassify { app_id: app_id.clone(), title: title.clone() });
        }

        let outcome = tick(&mut inner.escalation, classification, inner.last_idle_secs, elapsed_since_last_tick_secs);
        let elapsed_secs = (Utc::now().timestamp_millis() - inner.started_at_ms) / 1000;

        match &outcome {
            TickOutcome::TierChanged { to, reason, .. } => {
                events.push(SessionEvent::Reaction { tier: to.as_int(), reason: format!("{:?}", reason) });
            }
            TickOutcome::Infraction { tier } => {
                events.push(SessionEvent::Reaction { tier: tier.as_int(), reason: "infraction".into() });
                if let Some(sid) = inner.session_id {
                    let _ = repo::record_infraction(
                        &self.pool, sid, Utc::now().timestamp_millis(),
                        &app_id, title.as_deref(), tier.as_int() as i64
                    ).await;
                }
            }
            TickOutcome::NoChange { .. } => {}
        }

        events.push(SessionEvent::Tick {
            tier: inner.escalation.tier.as_int(),
            elapsed_secs,
            infractions: inner.escalation.session_infractions,
        });

        events
    }

    pub async fn classify_pending(&self, app_id: String, bucket: repo::Bucket) -> Result<(), sqlx::Error> {
        let mut inner = self.inner.lock().await;
        inner.classifier.upsert_app(app_id.clone(), bucket);
        inner.pending_classify_app = None;
        drop(inner);
        repo::upsert_classification(&self.pool, &app_id, None, bucket).await
    }

    pub async fn end(&self, ended_early: bool) -> Result<SessionEvent, sqlx::Error> {
        let mut inner = self.inner.lock().await;
        let id = inner.session_id.expect("session running");
        let infractions = inner.escalation.session_infractions;
        let result = Result_::from_infraction_count(infractions, ended_early);
        let now = Utc::now().timestamp_millis();
        repo::finalize_session(&self.pool, id, now, ended_early, infractions, result).await?;
        let (current_streak, _best) = repo::update_streak(&self.pool, matches!(result, Result_::Clean)).await?;
        inner.session_id = None;
        Ok(SessionEvent::Ended {
            result: result.as_str().into(),
            infractions,
            ended_early,
            streak: current_streak,
        })
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::db;

    fn obs_app(app: &str, title: &str) -> Observation {
        Observation {
            signal_id: "foreground".into(),
            app_id: Some(app.into()),
            title: Some(title.into()),
            idle_secs: None,
            timestamp_ms: 0,
        }
    }

    #[tokio::test]
    async fn unclassified_app_emits_ask_event_once() {
        let pool = db::open_in_memory().await.unwrap();
        let ctl = SessionController::new(pool).await.unwrap();
        ctl.start(1500).await.unwrap();
        let evs = ctl.apply(obs_app("com.unknown.X", "doc"), 1).await;
        assert!(evs.iter().any(|e| matches!(e, SessionEvent::AskClassify { .. })));
        let evs2 = ctl.apply(obs_app("com.unknown.X", "doc"), 1).await;
        assert!(!evs2.iter().any(|e| matches!(e, SessionEvent::AskClassify { .. })));
    }

    #[tokio::test]
    async fn distraction_records_infraction_and_persists() {
        let pool = db::open_in_memory().await.unwrap();
        repo::upsert_classification(&pool, "com.distraction.X", None, repo::Bucket::Distraction).await.unwrap();
        let ctl = SessionController::new(pool.clone()).await.unwrap();
        ctl.start(1500).await.unwrap();
        // Push enough seconds in the distraction to cross SLAP (>=30s).
        ctl.apply(obs_app("com.distraction.X", "twit"), 35).await;
        let count: (i64,) = sqlx::query_as("SELECT COUNT(*) FROM infractions").fetch_one(&pool).await.unwrap();
        assert_eq!(count.0, 1);
    }

    #[tokio::test]
    async fn end_clean_increments_streak() {
        let pool = db::open_in_memory().await.unwrap();
        let ctl = SessionController::new(pool.clone()).await.unwrap();
        ctl.start(1500).await.unwrap();
        let ev = ctl.end(false).await.unwrap();
        if let SessionEvent::Ended { streak, result, .. } = ev {
            assert_eq!(streak, 1);
            assert_eq!(result, "CLEAN");
        } else { panic!("expected ended"); }
    }
}
```

- [ ] **Step 2: Wire the module**

In `src-tauri/src/focus_engine/mod.rs`:

```rust
pub mod signals;
pub mod repo;
pub mod classifier;
pub mod escalation;
pub mod session;
```

- [ ] **Step 3: Run the tests**

Run: `cd src-tauri && cargo test --lib session`
Expected: 3 tests PASS.

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/focus_engine
git commit -m "feat(engine): SessionController lifecycle + tick application"
```

---

### Task 12: Spawn the polling tick loop

**Files:**
- Create: `src-tauri/src/focus_engine/runner.rs`
- Modify: `src-tauri/src/focus_engine/mod.rs` (add `pub mod runner;`)

- [ ] **Step 1: Write the runner**

Create `src-tauri/src/focus_engine/runner.rs`:

```rust
use crate::focus_engine::session::{SessionController, SessionEvent};
use crate::focus_engine::signals::{
    foreground::{ForegroundSignal, SystemForegroundProvider},
    idle::{IdleSignal, SystemIdleProvider},
    DetectionSignal,
};
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::task::JoinHandle;
use tokio::time;

const TICK_INTERVAL: Duration = Duration::from_secs(1);

pub fn spawn_engine(controller: Arc<SessionController>) -> (JoinHandle<()>, mpsc::Receiver<SessionEvent>) {
    let (tx, rx) = mpsc::channel(64);
    let handle = tokio::spawn(async move {
        let mut fg = ForegroundSignal::new(SystemForegroundProvider);
        let mut idle = IdleSignal::new(SystemIdleProvider);
        let mut interval = time::interval(TICK_INTERVAL);
        loop {
            interval.tick().await;
            if let Some(idle_obs) = idle.poll().await {
                let _ = controller.apply(idle_obs, 1).await;
            }
            if let Some(fg_obs) = fg.poll().await {
                let evs = controller.apply(fg_obs, 1).await;
                for e in evs { let _ = tx.send(e).await; }
            }
        }
    });
    (handle, rx)
}
```

- [ ] **Step 2: Wire the module**

In `src-tauri/src/focus_engine/mod.rs`:

```rust
pub mod runner;
```

- [ ] **Step 3: Verify it builds**

Run: `cd src-tauri && cargo build`
Expected: success.

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/focus_engine
git commit -m "feat(engine): tick-loop runner that polls signals and feeds controller"
```

---

## Phase 8 — IPC layer

### Task 13: Tauri commands + event forwarding

**Files:**
- Create: `src-tauri/src/ipc.rs`
- Modify: `src-tauri/src/main.rs` (register commands, manage state, run engine)

- [ ] **Step 1: Write the IPC module**

Create `src-tauri/src/ipc.rs`:

```rust
use crate::focus_engine::repo::Bucket;
use crate::focus_engine::session::{SessionController, SessionEvent};
use std::sync::Arc;

pub struct AppState { pub controller: Arc<SessionController> }

#[tauri::command]
pub async fn start_session(state: tauri::State<'_, AppState>, duration_secs: i64) -> Result<SessionEvent, String> {
    state.controller.start(duration_secs).await.map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn end_session(state: tauri::State<'_, AppState>, ended_early: bool) -> Result<SessionEvent, String> {
    state.controller.end(ended_early).await.map_err(|e| e.to_string())
}

#[tauri::command]
pub async fn classify_app(state: tauri::State<'_, AppState>, app_id: String, bucket: String) -> Result<(), String> {
    let b = match bucket.as_str() {
        "WORK" => Bucket::Work, "DISTRACTION" => Bucket::Distraction,
        _ => return Err("invalid bucket".into()),
    };
    state.controller.classify_pending(app_id, b).await.map_err(|e| e.to_string())
}
```

- [ ] **Step 2: Wire main.rs**

Replace the contents of `src-tauri/src/main.rs` with:

```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

mod db;
mod focus_engine;
mod ipc;

use focus_engine::runner::spawn_engine;
use focus_engine::session::SessionController;
use ipc::AppState;
use std::sync::Arc;
use tauri::{Emitter, Manager};

fn main() {
    tauri::Builder::default()
        .plugin(tauri_plugin_shell::init())
        .plugin(tauri_plugin_opener::init())
        .setup(|app| {
            let app_handle = app.handle().clone();
            tauri::async_runtime::block_on(async move {
                let app_dir = app_handle.path().app_data_dir().expect("app data dir");
                std::fs::create_dir_all(&app_dir).ok();
                let pool = db::open(&app_dir.join("focus.db")).await.expect("open db");
                let controller = Arc::new(SessionController::new(pool).await.expect("controller"));
                let (_handle, mut rx) = spawn_engine(controller.clone());
                app_handle.manage(AppState { controller });

                // Forward engine events to renderer
                let emit_handle = app_handle.clone();
                tauri::async_runtime::spawn(async move {
                    while let Some(ev) = rx.recv().await {
                        let _ = emit_handle.emit("engine-event", ev);
                    }
                });
            });
            Ok(())
        })
        .invoke_handler(tauri::generate_handler![
            ipc::start_session,
            ipc::end_session,
            ipc::classify_app,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

- [ ] **Step 3: Verify it builds**

Run: `cd src-tauri && cargo build`
Expected: success.

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/ipc.rs src-tauri/src/main.rs
git commit -m "feat(ipc): typed Tauri commands + event forwarding from engine to renderer"
```

---

### Task 14: Configure two windows in tauri.conf.json

**Files:**
- Modify: `src-tauri/tauri.conf.json`
- Create: `overlay.html`

- [ ] **Step 1: Write the overlay HTML entry**

Create `overlay.html` at the repo root:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>badcatsociety overlay</title>
    <style>html,body,#root{margin:0;padding:0;background:transparent;width:100vw;height:100vh;overflow:hidden}</style>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/overlay.tsx"></script>
  </body>
</html>
```

- [ ] **Step 2: Update tauri.conf.json windows + add overlay entry to Vite**

Open `src-tauri/tauri.conf.json`. Replace the `app.windows` array with:

```json
"windows": [
  {
    "label": "popup",
    "title": "badcatsociety",
    "width": 360,
    "height": 480,
    "resizable": false,
    "fullscreen": false,
    "url": "index.html"
  },
  {
    "label": "overlay",
    "title": "cat",
    "width": 200,
    "height": 200,
    "x": 100,
    "y": 100,
    "resizable": false,
    "decorations": false,
    "transparent": true,
    "alwaysOnTop": true,
    "skipTaskbar": true,
    "url": "overlay.html"
  }
]
```

Open `vite.config.ts`. Add the multi-page input config:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { resolve } from "path";

export default defineConfig({
  plugins: [react()],
  clearScreen: false,
  server: { port: 1420, strictPort: true, host: "0.0.0.0" },
  build: {
    rollupOptions: {
      input: {
        popup: resolve(__dirname, "index.html"),
        overlay: resolve(__dirname, "overlay.html"),
      },
    },
  },
});
```

- [ ] **Step 3: Verify build still passes**

Run: `pnpm build`
Expected: builds two HTML entries (`dist/index.html`, `dist/overlay.html`).

- [ ] **Step 4: Commit**

```bash
git add overlay.html src-tauri/tauri.conf.json vite.config.ts
git commit -m "feat(shell): two windows — popup + transparent always-on-top overlay"
```

---

## Phase 9 — Frontend

### Task 15: Typed IPC wrapper + Zustand store

**Files:**
- Create: `src/lib/ipc.ts`
- Create: `src/lib/store.ts`

- [ ] **Step 1: Install Tauri JS plugin packages**

```bash
pnpm add @tauri-apps/api
```

- [ ] **Step 2: Write the IPC wrapper**

Create `src/lib/ipc.ts`:

```ts
import { invoke } from "@tauri-apps/api/core";
import { listen, type UnlistenFn } from "@tauri-apps/api/event";

export type SessionEvent =
  | { kind: "started"; session_id: number; duration_secs: number }
  | { kind: "tick"; tier: number; elapsed_secs: number; infractions: number }
  | { kind: "ask_classify"; app_id: string; title: string | null }
  | { kind: "reaction"; tier: number; reason: string }
  | { kind: "ended"; result: "CLEAN" | "DECENT" | "ROUGH" | "DISASTER"; infractions: number; ended_early: boolean; streak: number };

export const startSession = (durationSecs: number) =>
  invoke<SessionEvent>("start_session", { durationSecs });

export const endSession = (endedEarly: boolean) =>
  invoke<SessionEvent>("end_session", { endedEarly });

export const classifyApp = (appId: string, bucket: "WORK" | "DISTRACTION") =>
  invoke<void>("classify_app", { appId, bucket });

export const onEngineEvent = (cb: (e: SessionEvent) => void): Promise<UnlistenFn> =>
  listen<SessionEvent>("engine-event", (msg) => cb(msg.payload));
```

- [ ] **Step 3: Write the store**

Create `src/lib/store.ts`:

```ts
import { create } from "zustand";
import type { SessionEvent } from "./ipc";

type SessionStatus = "idle" | "running" | "ended";

interface State {
  status: SessionStatus;
  tier: number;
  elapsedSecs: number;
  durationSecs: number;
  infractions: number;
  pendingClassify: { appId: string; title: string | null } | null;
  lastResult: Extract<SessionEvent, { kind: "ended" }> | null;
  setStatus: (s: SessionStatus) => void;
  applyEvent: (e: SessionEvent) => void;
  clearPending: () => void;
}

export const useStore = create<State>((set) => ({
  status: "idle",
  tier: 0,
  elapsedSecs: 0,
  durationSecs: 0,
  infractions: 0,
  pendingClassify: null,
  lastResult: null,
  setStatus: (status) => set({ status }),
  applyEvent: (e) => set((s) => {
    switch (e.kind) {
      case "started": return { status: "running", durationSecs: e.duration_secs, tier: 0, elapsedSecs: 0, infractions: 0, pendingClassify: null };
      case "tick": return { tier: e.tier, elapsedSecs: e.elapsed_secs, infractions: e.infractions };
      case "ask_classify": return { pendingClassify: { appId: e.app_id, title: e.title } };
      case "reaction": return { tier: e.tier };
      case "ended": return { status: "ended", lastResult: e };
      default: return s;
    }
  }),
  clearPending: () => set({ pendingClassify: null }),
}));
```

- [ ] **Step 4: Commit**

```bash
git add package.json pnpm-lock.yaml src/lib
git commit -m "feat(ui): typed IPC wrapper + Zustand store"
```

---

### Task 16: Popup window — session config, in-session timer, end summary

**Files:**
- Create: `src/popup.tsx`
- Create: `src/App.tsx`
- Create: `src/components/SessionConfig.tsx`
- Create: `src/components/SessionTimer.tsx`
- Create: `src/components/SessionSummary.tsx`
- Create: `src/components/ClassificationPrompt.tsx`
- Modify: `index.html` (point script tag at `/src/popup.tsx`)

- [ ] **Step 1: Update the popup HTML entry**

Open `index.html` at repo root. Replace the body with:

```html
<body>
  <div id="root"></div>
  <script type="module" src="/src/popup.tsx"></script>
</body>
```

- [ ] **Step 2: Write the popup entry**

Create `src/popup.tsx`:

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import { onEngineEvent } from "./lib/ipc";
import { useStore } from "./lib/store";

onEngineEvent((e) => useStore.getState().applyEvent(e));

ReactDOM.createRoot(document.getElementById("root")!).render(<App />);
```

- [ ] **Step 3: Write the popup root**

Create `src/App.tsx`:

```tsx
import React from "react";
import { useStore } from "./lib/store";
import { SessionConfig } from "./components/SessionConfig";
import { SessionTimer } from "./components/SessionTimer";
import { SessionSummary } from "./components/SessionSummary";
import { ClassificationPrompt } from "./components/ClassificationPrompt";

export default function App() {
  const status = useStore((s) => s.status);
  const pending = useStore((s) => s.pendingClassify);
  return (
    <div style={{ padding: 20, fontFamily: "-apple-system, sans-serif", height: "100vh", boxSizing: "border-box" }}>
      {status === "idle" && <SessionConfig />}
      {status === "running" && <SessionTimer />}
      {status === "ended" && <SessionSummary />}
      {pending && <ClassificationPrompt />}
    </div>
  );
}
```

- [ ] **Step 4: Write SessionConfig**

Create `src/components/SessionConfig.tsx`:

```tsx
import React, { useState } from "react";
import { startSession } from "../lib/ipc";
import { useStore } from "../lib/store";

export function SessionConfig() {
  const [duration, setDuration] = useState(25);
  const apply = useStore((s) => s.applyEvent);

  async function start() {
    const ev = await startSession(duration * 60);
    apply(ev);
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16, height: "100%", justifyContent: "center" }}>
      <h2 style={{ margin: 0, fontSize: 20 }}>🐈‍⬛ Start a focus session</h2>
      <label>
        Duration: <strong>{duration} min</strong>
        <input type="range" min={5} max={90} value={duration} onChange={(e) => setDuration(Number(e.target.value))} style={{ width: "100%" }} />
      </label>
      <button onClick={start} style={{ padding: "12px 16px", background: "#1f2937", color: "#fff", border: "none", borderRadius: 8, fontWeight: 600, cursor: "pointer" }}>
        Start
      </button>
    </div>
  );
}
```

- [ ] **Step 5: Write SessionTimer**

Create `src/components/SessionTimer.tsx`:

```tsx
import React from "react";
import { endSession } from "../lib/ipc";
import { useStore } from "../lib/store";

const fmt = (s: number) => `${Math.floor(s / 60)}:${String(s % 60).padStart(2, "0")}`;

export function SessionTimer() {
  const elapsed = useStore((s) => s.elapsedSecs);
  const duration = useStore((s) => s.durationSecs);
  const tier = useStore((s) => s.tier);
  const infractions = useStore((s) => s.infractions);
  const apply = useStore((s) => s.applyEvent);
  const remaining = Math.max(0, duration - elapsed);

  async function endNow() {
    const ev = await endSession(true);
    apply(ev);
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 14, height: "100%", justifyContent: "center", textAlign: "center" }}>
      <div style={{ fontSize: 48, fontWeight: 700 }}>{fmt(remaining)}</div>
      <div style={{ color: "#666" }}>Tier <strong>{tier}</strong> · {infractions} infraction{infractions === 1 ? "" : "s"}</div>
      <button onClick={endNow} style={{ padding: 8, background: "transparent", color: "#dc2626", border: "1px solid #dc2626", borderRadius: 6, cursor: "pointer" }}>
        End session early
      </button>
    </div>
  );
}
```

- [ ] **Step 6: Write SessionSummary**

Create `src/components/SessionSummary.tsx`:

```tsx
import React from "react";
import { useStore } from "../lib/store";

const VERDICT: Record<string, { emoji: string; line: string }> = {
  CLEAN:    { emoji: "😸", line: "Clean run! Whiskers is proud." },
  DECENT:   { emoji: "🙂", line: "You tried." },
  ROUGH:    { emoji: "😾", line: "Whiskers is judging." },
  DISASTER: { emoji: "😿", line: "I disappointed Whiskers today." },
};

export function SessionSummary() {
  const result = useStore((s) => s.lastResult);
  const setStatus = useStore((s) => s.setStatus);
  if (!result) return null;
  const v = VERDICT[result.result];
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 14, height: "100%", justifyContent: "center", textAlign: "center" }}>
      <div style={{ fontSize: 64 }}>{v.emoji}</div>
      <h2 style={{ margin: 0 }}>{v.line}</h2>
      <div style={{ color: "#666" }}>{result.infractions} slap{result.infractions === 1 ? "" : "s"} · streak {result.streak}</div>
      <button onClick={() => setStatus("idle")} style={{ padding: "10px 14px", background: "#1f2937", color: "#fff", border: "none", borderRadius: 8, cursor: "pointer" }}>
        Start another
      </button>
    </div>
  );
}
```

- [ ] **Step 7: Write ClassificationPrompt**

Create `src/components/ClassificationPrompt.tsx`:

```tsx
import React from "react";
import { classifyApp } from "../lib/ipc";
import { useStore } from "../lib/store";

export function ClassificationPrompt() {
  const pending = useStore((s) => s.pendingClassify);
  const clear = useStore((s) => s.clearPending);
  if (!pending) return null;

  async function pick(bucket: "WORK" | "DISTRACTION") {
    await classifyApp(pending!.appId, bucket);
    clear();
  }

  return (
    <div style={{ position: "fixed", inset: 0, background: "rgba(0,0,0,0.6)", display: "flex", alignItems: "center", justifyContent: "center" }}>
      <div style={{ background: "#fff", padding: 20, borderRadius: 12, width: 280, textAlign: "center" }}>
        <div style={{ fontSize: 36 }}>😼</div>
        <h3 style={{ margin: "8px 0" }}>is this real work?</h3>
        <div style={{ fontSize: 12, color: "#666", marginBottom: 14, wordBreak: "break-all" }}>
          {pending.title || pending.appId}
        </div>
        <div style={{ display: "flex", gap: 10 }}>
          <button onClick={() => pick("WORK")} style={{ flex: 1, padding: 10, background: "#16a34a", color: "#fff", border: "none", borderRadius: 6, cursor: "pointer" }}>Yes 🌟</button>
          <button onClick={() => pick("DISTRACTION")} style={{ flex: 1, padding: 10, background: "#dc2626", color: "#fff", border: "none", borderRadius: 6, cursor: "pointer" }}>No 😾</button>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 8: Verify it builds**

Run: `pnpm build`
Expected: success.

- [ ] **Step 9: Commit**

```bash
git add index.html src/popup.tsx src/App.tsx src/components
git commit -m "feat(ui): popup with session config, timer, summary, classification prompt"
```

---

### Task 17: Overlay window — TierBadge placeholder

**Files:**
- Create: `src/overlay.tsx`
- Create: `src/Overlay.tsx`
- Create: `src/components/TierBadge.tsx`

- [ ] **Step 1: Write the overlay entry**

Create `src/overlay.tsx`:

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import Overlay from "./Overlay";
import { onEngineEvent } from "./lib/ipc";
import { useStore } from "./lib/store";

onEngineEvent((e) => useStore.getState().applyEvent(e));

ReactDOM.createRoot(document.getElementById("root")!).render(<Overlay />);
```

- [ ] **Step 2: Write the overlay root**

Create `src/Overlay.tsx`:

```tsx
import React from "react";
import { TierBadge } from "./components/TierBadge";

export default function Overlay() {
  return (
    <div style={{ width: "100%", height: "100%", display: "flex", alignItems: "center", justifyContent: "center" }}>
      <TierBadge />
    </div>
  );
}
```

- [ ] **Step 3: Write TierBadge**

Create `src/components/TierBadge.tsx`:

```tsx
import React from "react";
import { useStore } from "../lib/store";

const TIER_VISUAL: Record<number, { emoji: string; color: string; label: string }> = {
  0: { emoji: "😴", color: "#16a34a", label: "napping" },
  1: { emoji: "👀", color: "#eab308", label: "alert" },
  2: { emoji: "😼", color: "#f59e0b", label: "stare" },
  3: { emoji: "🐾", color: "#f97316", label: "paw tap" },
  4: { emoji: "🥊", color: "#ef4444", label: "SLAP" },
  5: { emoji: "🙀", color: "#db2777", label: "outrage" },
};

export function TierBadge() {
  const tier = useStore((s) => s.tier);
  const status = useStore((s) => s.status);
  if (status !== "running") return null;
  const v = TIER_VISUAL[tier] ?? TIER_VISUAL[0];
  return (
    <div style={{
      width: 160, height: 160, borderRadius: "50%",
      background: v.color, display: "flex", flexDirection: "column",
      alignItems: "center", justifyContent: "center",
      boxShadow: "0 12px 30px rgba(0,0,0,0.4)",
      transition: "background-color 250ms ease",
    }}>
      <div style={{ fontSize: 64 }}>{v.emoji}</div>
      <div style={{ color: "#fff", fontWeight: 700, fontSize: 12, letterSpacing: 1, textTransform: "uppercase" }}>{v.label}</div>
    </div>
  );
}
```

- [ ] **Step 4: Verify build**

Run: `pnpm build`
Expected: success.

- [ ] **Step 5: Commit**

```bash
git add src/overlay.tsx src/Overlay.tsx src/components/TierBadge.tsx
git commit -m "feat(ui): overlay window with placeholder TierBadge (cat substitute)"
```

---

## Phase 10 — End-to-end manual test

### Task 18: Run the app and verify the loop

**Files:** none

- [ ] **Step 1: Build and launch**

Run: `pnpm tauri dev`
Expected: two windows appear. Popup at top-left showing "Start a focus session". Overlay (transparent circle, currently empty) somewhere on screen.

- [ ] **Step 2: Manual scenario — clean run**

1. Set duration to 5 minutes.
2. Click Start.
3. Stay in the popup or any unclassified app and click "yes" if asked.
4. Wait ~30 seconds.
5. Click "End session early".
6. Expected: summary screen shows "I disappointed Whiskers today" because you ended early. Streak resets to 0.

- [ ] **Step 3: Manual scenario — escalation**

1. Restart `pnpm tauri dev` if needed.
2. Start a 5-minute session.
3. Switch to your browser. The popup will show the "is this real work?" prompt — click **No 😾**. The browser is now marked as DISTRACTION.
4. Stay in the browser. Watch the overlay badge progress over ~30 seconds: napping (😴 green) → alert (👀 yellow) → stare (😼 orange) → paw tap (🐾 deeper orange) → SLAP (🥊 red).
5. Switch back to your terminal/IDE (mark it as Yes/Work if prompted). Watch the badge de-escalate back toward napping over ~30 seconds.

- [ ] **Step 4: Manual scenario — unclassified prompt**

1. Start a session.
2. Switch to an app you've never opened with this app running (e.g., Notion, Calculator).
3. Expected: a modal appears in the popup asking "is this real work?". Click Yes/No. The next time you switch to that app, no prompt.

- [ ] **Step 5: Document any issues**

If anything is broken, note it in `docs/superpowers/notes/v1-manual-test-issues.md` (create the file). Do **not** fix mid-test — just document.

- [ ] **Step 6: Commit any test notes**

```bash
git add -A
git commit -m "docs: v1 manual test notes" --allow-empty
```

---

## Done — what you have now

After Phase 10, the app:

- Has a working menubar popup window for session config.
- Has a transparent always-on-top overlay window showing the current tier as a colored emoji badge (placeholder for the 3D cat).
- Polls foreground app + idle every second.
- Classifies apps WORK / DISTRACTION / UNCLASSIFIED, asks the user once when it sees something new, persists their answer.
- Escalates through 6 tiers per the spec rules (Napping → Alert → Stare → PawTap → Slap → Outrage), with proper de-escalation on clean behavior.
- Records infractions and full session results in SQLite.
- Tracks the cross-session streak.
- Ends sessions cleanly with a verdict screen.

What's still placeholder vs. spec:
- Cat is a colored circle with an emoji. **Plan 2** swaps in Three.js + Quaternius cat with animation clips per tier.
- No audio. **Plan 2** adds tier sounds.
- No share-card PNG. **Plan 3** adds a `<canvas>`-based generator + share buttons.
- No settings UI for editing classifications. **Plan 3** adds that.
- No macOS Accessibility bridge for hard-to-read window titles. **Plan 2** adds it.
- No code signing or installer polish. **Plan 4** does that for launch.
