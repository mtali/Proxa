# CLAUDE.md

Guidance for Claude Code when working in the **Proxa** repository.

@CLAUDE.local.md

> **Git commits:** never add `Co-Authored-By:` or any AI attribution; keep commit messages
> straightforward and simple (concise one-line subject). See `CLAUDE.local.md`.

## What this is — and how to work in it

**Proxa** (from *proximity*) is a **learning project**: a modern, Kotlin/Compose rebuild of the ideas
behind [Briar](https://briarproject.org) — a serverless, peer-to-peer, **offline messenger** —
reimagined with current Android architecture. Two phones
message each other with **no server**, first over **classic Bluetooth (RFCOMM)**, later Wi-Fi/LAN and
eventually Tor. Everything is **local-first**: a message is saved on the device first, then synced to
a peer when a connection exists.

**This is for learning, so the bar is understanding, not speed.** When working here:
- **Explain every bit.** For each new concept (repository pattern, UDF, Room Flows, RFCOMM, foreground
  services, etc.) explain *what it is and why*, not just the code.
- **Plan before code.** Present the plan and confirm scope before scaffolding or writing
  implementation. Don't jump ahead phases.
- **One phase at a time.** Build, explain, verify on a device, then continue.

## Source of truth: the roadmap

📍 **`docs/PLAN.md`** is the authoritative plan. It defines Part A (Phases 0–6, the shippable
MVP) and Part B (milestones M1–M9 toward fuller Briar), each phase's *Build / Concepts / NIA parallel
/ Done-when*. **Read it before starting any work.** Keep it updated when the plan changes.

## Reference codebases (read these to learn the patterns)

| What | Local path |
|---|---|
| **Real Briar** (offline architecture we modernize) | `/Users/mtali/AndroidStudioProjects/briar` |
| **Now in Android** (architecture we follow) | `/Users/mtali/AndroidStudioProjects/nowinandroid` |

When in doubt about a transport/sync detail, read Briar. When in doubt about app architecture, read NIA.

## Architecture

Follows **Now in Android (NIA)** conventions, but **package-based in a single `:app` module** for now
(NOT multi-module). Package boundaries are kept as strict as if they were modules, so promoting each
`core/*` and `feature/*` package to a real Gradle module later (Milestone M9) is mechanical.

**Target package layout** (under `app/src/main/java/com/mtali/proxa/` — note the wizard uses `java/`,
not `kotlin/`):

```
ProxaApplication.kt      # @HiltAndroidApp (added in Phase 0/1)
MainActivity.kt
navigation/              # ProxaNavHost, destinations            (NIA: app + core:navigation)
core/
  model/                 # pure Kotlin domain models             (NIA: core:model)
  common/                # dispatchers, Result                   (NIA: core:common)
  data/                  # repositories (offline-first)          (NIA: core:data)
  database/              # Room: db, entities, DAOs              (NIA: core:database)
  datastore/             # DataStore: identity & settings        (NIA: core:datastore)
  domain/                # use cases (when needed)               (NIA: core:domain)
  designsystem/          # ProxaTheme + dumb components          (NIA: core:designsystem)
  ui/                    # composite components that know models (NIA: core:ui)
  transport/             # ★ the Briar Transport seam — replaces NIA core:network
    bluetooth/           #   BluetoothTransport
feature/                 # one package per screen: UI + ViewModel + UiState
  identity/  contacts/  chat/
sync/                    # ★ connection lifecycle + foreground service (NIA: sync = WorkManager)
```

**Two deliberate deviations from NIA:**
- `core/transport/` sits in NIA's `core:network` slot — a radio (`Transport`/`Connection` interface,
  Briar's `DuplexPlugin` idea) instead of Retrofit.
- `sync/` keeps a Bluetooth connection alive via a **foreground service**, instead of WorkManager.

### Conventions (NIA)
- **UDF / MVVM:** each feature has a `ViewModel` exposing one immutable `UiState` (sealed interface:
  `Loading / Success / Error`) as `StateFlow`; the Composable is a pure function of state.
- **Offline-first repositories:** UI reads from Room via `Flow`; the transport only *updates* the
  local store. UI never touches a data source directly.
- **`designsystem` vs `ui`:** `designsystem` = theme + reusable dumb widgets (no domain knowledge);
  `ui` = composite components that take domain models (e.g. `MessageBubble(message: Message)`).
- **DI:** Hilt (+ KSP). **Deps:** version catalog (`gradle/libs.versions.toml`) only — no hardcoded
  versions. **Serialization:** kotlinx.serialization.
- **Keep packages clean** — no cross-package shortcuts, or later modularization breaks.

## Current state (as of project creation)

Fresh Android Studio wizard scaffold — **Phase 0 not done yet**:
- AGP 9.2.1, Kotlin 2.2.10, Compose BOM 2026.02.01, compileSdk 36 (minor 1), minSdk 24, targetSdk 36.
- `applicationId` / `namespace` = `com.mtali.proxa`.
- Only `MainActivity` + `ui/theme/` exist. **Not yet added:** Hilt, Room, DataStore, Navigation,
  kotlinx.serialization, the NIA package layout, release signing config.
- Setting these up is **Phase 0** in `docs/PLAN.md`.

## Build & test commands

- `./gradlew :app:assembleDebug` — build debug APK
- `./gradlew :app:bundleRelease` — build release AAB (Phase 6 adds signing)
- `./gradlew test` — unit tests (JVM)
- `./gradlew connectedAndroidTest` — instrumentation tests (needs a device)
- `./gradlew lint` — lint checks

## Testing & device notes

- **Two physical Android phones are required** to test Bluetooth — emulators can't do classic
  Bluetooth (RFCOMM). Keep one "A", one "B".
- Phase 3 introduces a fake **`LoopbackTransport`** so sync/protocol logic is unit-testable with **no
  hardware** — write those tests first (TDD).
- Test stack to adopt: JUnit + Truth + Turbine (Flow) + Robolectric; Hilt test runner.
- Always **test the R8/minified release build on-device** before any Play upload — minification can
  break Room/serialization reflection without keep rules.

## Workflow (spec + TDD)

**Follow `docs/TDD.md` for changes that carry real logic — do not jump straight to production code.**
In short: `branch → spec in docs/specs/pending/ → explain the concept → failing test (red) → minimum
code (green) → refactor → verify → move spec to docs/specs/done/`. Specs map to a Phase/Milestone in
`docs/PLAN.md`.

**Test when it adds value, not for its own sake.** Test logic with edge cases (framing, sync,
repositories, mappers, `UiState` transitions); skip tests for scaffolding/DI wiring, pure UI, and
theme — verify those by running the app. Drive transport/sync logic through the fake
`LoopbackTransport` so it's testable without hardware; verify radio/permission/service behavior on
two physical phones. A skipped test is a deliberate, noted call — not an oversight.

@docs/TDD.md
