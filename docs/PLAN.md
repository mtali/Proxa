# Proxa — Learning Roadmap

**Proxa** (from *proximity* — people near you) is a modern, native-Android take on serverless,
peer-to-peer messaging — reimagining the offline-first architecture pioneered by
[Briar](https://briarproject.org) with today's Kotlin/Compose stack. It follows the architecture and
conventions of Google's [**Now in Android (NIA)**](https://github.com/android/nowinandroid) sample —
the current reference for modern Android — so everything you learn here is industry-standard and
transfers directly to real work.

- **Package / applicationId:** `com.mtali.proxa`
- **Type:** standalone learning project (separate codebase).

### Reference codebases (read these as you build)
| What | Local path |
|---|---|
| **Real Briar** (offline architecture we modernize) | `/Users/mtali/AndroidStudioProjects/briar` |
| **Now in Android** (architecture we follow) | `/Users/mtali/AndroidStudioProjects/nowinandroid` |
| **This project (Proxa)** | `/Users/mtali/AndroidStudioProjects/Proxa` |

> **This is a learning project.** Each phase explains not just *what* to build but *why* — the
> concept behind each piece, and the NIA parallel. We build one phase at a time and confirm before
> moving on.

> **Single module now, modules later — on purpose.** NIA is heavily *multi-module* (`core:model`,
> `feature:foryou`, …). For a solo project that's premature ceremony. Proxa starts as **one `:app`
> module organised into NIA's exact package layout**, with package boundaries kept as strict as if
> they were modules (no reaching across a package's "public surface"). When the project is big
> enough to justify it, each `core/*` and `feature/*` package promotes to a Gradle module almost
> mechanically — and *that's* when NIA's `build-logic` convention plugins earn their keep.

---

## North star

Two phones message each other with **no server** — first over Bluetooth, later Wi-Fi/LAN, and
eventually Tor. Everything is **local-first** (NIA calls it *offline-first*): a message is saved on
your device first, then synced to peers when a connection exists.

**Part A (the MVP) is a real, shippable app: it ends on the Google Play Store.**

## The one idea to keep from Briar

> **Hide the radio behind a thin byte-stream interface. Keep all chat/sync logic
> transport-agnostic.**

In Briar this is `DuplexTransportConnection` = `{ reader: InputStream, writer: OutputStream }`, and
each radio (Bluetooth, LAN, Tor) is a *plugin* implementing the same contract. The sync layer never
knows which radio it runs on. Add a radio → write one plugin → everything above works unchanged. In
Proxa this is the `Transport` interface (Phase 3), living where NIA puts `core:network`.

---

## Architecture — Now in Android conventions, package-based

NIA's layers (we copy the layout as **packages**, mapping 1:1 to future modules):

```
app/src/main/kotlin/com/mtali/proxa/
├── ProxaApplication.kt   # @HiltAndroidApp
├── MainActivity.kt
├── navigation/           # ProxaNavHost, TopLevelDestination     (NIA: app + core:navigation)
├── core/
│   ├── model/            # pure Kotlin domain models             (NIA: core:model)
│   ├── common/           # dispatchers, Result, Dispatcher qualifiers (NIA: core:common)
│   ├── data/             # repositories (offline-first) + data sources  (NIA: core:data)
│   ├── database/         # Room: db, entities, DAOs               (NIA: core:database)
│   ├── datastore/        # DataStore: identity & settings         (NIA: core:datastore)
│   ├── domain/           # use cases (optional until needed)      (NIA: core:domain)
│   ├── designsystem/     # ProxaTheme + dumb components (buttons, etc.) (NIA: core:designsystem)
│   ├── ui/               # composite components that know models  (NIA: core:ui)
│   └── transport/        # ★ the Briar Transport seam — replaces  (NIA: core:network)
│       └── bluetooth/    #   BluetoothTransport
├── feature/              # one package per screen: UI + ViewModel + UiState
│   ├── identity/         #   set your display name
│   ├── contacts/         #   discover/list devices, pick a peer
│   └── chat/             #   the conversation screen
└── sync/                 # ★ connection lifecycle + foreground service  (NIA: sync = WorkManager)
```

Two layers differ from NIA on purpose:
- **`core/transport/`** replaces NIA's `core:network`. NIA fetches from a server via Retrofit; Proxa
  connects to *peers* via a radio. Same architectural slot (the "remote data" boundary), different
  mechanism — a great thing to internalise.
- **`sync/`** in NIA is WorkManager pulling from a backend. In Proxa it means *keeping a Bluetooth
  connection alive* (a foreground service). Same role (background data movement), different mechanism.

### NIA conventions Proxa adopts
- **UDF / MVVM:** each feature has a `ViewModel` exposing a single immutable `UiState` (a sealed
  interface: `Loading / Success / Error`) as `StateFlow`; the Composable is a pure function of state.
- **Offline-first repositories:** the UI reads from Room via `Flow`; the network/transport only
  *updates* the local store. UI never talks to a data source directly.
- **`designsystem` vs `ui`:** `designsystem` = theme + reusable "dumb" widgets that know nothing about
  domain models; `ui` = composite components that *do* (e.g. a `MessageBubble(message: Message)`).
- **DI with Hilt**, modules per layer; **KSP** for Room/Hilt codegen.
- **Version catalog** (`gradle/libs.versions.toml`) for all deps/plugins.
- **Strict package boundaries** so modularization later is mechanical.

### Tech stack

| Concern | Choice | NIA uses it? |
|---|---|---|
| Language / UI | Kotlin + Jetpack Compose + Material3 | ✅ |
| DI | **Hilt** (+ KSP) | ✅ |
| Local DB | **Room** | ✅ |
| Settings/identity | **DataStore** | ✅ |
| Async | **Coroutines + Flow** | ✅ |
| Navigation | **Navigation Compose** (type-safe) | ✅ |
| State | **UDF** (ViewModel + `UiState` StateFlow) | ✅ |
| Deps | **Version catalog** | ✅ |
| Serialization | **kotlinx.serialization** | ✅ |
| Remote boundary | ❌ Retrofit (`core:network`) → replaced by `core/transport` | (NIA uses Retrofit) |
| Modularization | **deferred** — packages now, modules later (+ `build-logic`) | (NIA is multi-module now) |
| Transport (MVP) | **Classic Bluetooth RFCOMM** | n/a |

**Out of scope for the MVP** (later milestones): Tor, end-to-end crypto, groups/forums, KMP/iOS,
offline mailbox relay.

### Working method
**branch → short spec → failing test (red) → make it pass (green) → refactor → verify on a device.**

---

# Part A — MVP: a shippable Bluetooth messenger (Phases 0–6)

Goal of Part A: **two phones pair over Bluetooth and exchange text messages that persist locally —
and the app is published to the Play Store.** Play Store constraints are baked into the phases below.

### Phase 0 — Project scaffold (release-ready from day one)
- **Build:** single `:app` module; Gradle KTS; `libs.versions.toml`; Hilt + KSP + Compose plugins;
  the NIA package skeleton above; `ProxaApplication` + a hello-world `MainActivity`; a minimal
  `core/designsystem` `ProxaTheme`.
- **Play-readiness baked in now (cheap early, painful late):**
  - `applicationId = "com.mtali.proxa"` (permanent once published — choose once).
  - `targetSdk` = current Play requirement (35 as of 2024/25; bump as Play raises the floor).
    `minSdk = 24`.
  - `versionCode` / `versionName` scheme (`versionName` `0.1.0`).
  - `debug` vs `release` build types; `release` has **R8/minify + shrinkResources** on.
  - A **signing config** wired to a `keystore.properties` file that is **git-ignored** (never commit
    keystores or passwords).
- **Concepts:** version catalog wiring; `@HiltAndroidApp`; NIA's layered packages & strict
  boundaries; `designsystem` vs `ui`; what `applicationId` permanently commits you to; debug vs
  release builds; app signing basics.
- **NIA parallel:** `app/` (`NiaApplication`, `MainActivity`), `core:designsystem`,
  `gradle/libs.versions.toml`.
- **Done when:** `:app:assembleDebug` succeeds **and** `:app:bundleRelease` produces a signed `.aab`.

### Phase 1 — Identity & contact model
- **Build:** `core/model` (`Contact`, `LocalIdentity`); `core/datastore` persisting identity;
  `core/data` `IdentityRepository`; `feature/identity` screen (ViewModel + `UiState`) to set name.
- **Concepts:** model vs entity vs UiState; repository pattern (UI never touches a data source);
  DataStore vs SharedPreferences; UDF — ViewModel exposes `StateFlow<UiState>`.
- **NIA parallel:** `core:model`, `core:datastore` (`NiaPreferencesDataSource`), `core:data`
  (`UserDataRepository`), `feature:settings`.
- **Done when:** app shows "I am <name>" and persists it across restarts.

### Phase 2 — Local message store (Room)
- **Build:** `core/database` (`ProxaDatabase`, `MessageEntity`, `MessageDao` returning `Flow`);
  entity↔model mappers; `core/data` `MessageRepository`.
- **Concepts:** Room entities/DAOs/migrations; why DAOs return `Flow` (offline-first reactive UI);
  the entity↔model boundary; local-first (save before you send).
- **NIA parallel:** `core:database` (`NiaDatabase`, DAOs), `core:data` repositories.
- **Done when:** inserting a message streams to a test collector via Flow.

### Phase 3 — Transport abstraction (the architectural keystone) ★
- **Build:** interfaces in `core/transport`:
  ```kotlin
  interface Connection {
      val input: InputStream
      val output: OutputStream
      fun close()
  }
  interface Transport {
      val id: String
      fun startListening(onConnection: (Connection) -> Unit)  // incoming peers
      suspend fun connect(address: String): Connection         // outgoing
      fun stop()
  }
  ```
  Plus a fake `LoopbackTransport` (in-memory pipe) so logic can be tested with **no hardware**.
- **Concepts:** programming to an interface; dependency inversion (chat depends on `Transport`, not
  Bluetooth); fakes enabling TDD; this *is* Briar's `DuplexPlugin`/`DuplexTransportConnection`, and
  it sits exactly where NIA's `core:network` boundary sits.
- **NIA parallel:** `core:network` interfaces + `core:data`'s fake/test data sources.
- **Done when:** a unit test sends bytes A→B through `LoopbackTransport`.

### Phase 4 — Bluetooth transport (RFCOMM)
- **Build:** `BluetoothTransport : Transport` in `core/transport/bluetooth`.
  - **Listen:** `listenUsingRfcommWithServiceRecord(name, APP_UUID)` → `accept()` loop on IO.
  - **Connect:** bonded/discovered device → `createRfcommSocketToServiceRecord(APP_UUID)` →
    `connect()`.
  - **Permissions:** runtime `BLUETOOTH_CONNECT` / `BLUETOOTH_SCAN` (Android 12+), legacy
    `BLUETOOTH` / `BLUETOOTH_ADMIN` + location for discovery on older devices. Clear
    **permission-rationale UI** (Play review looks for this).
- **Concepts:** classic Bluetooth vs BLE; RFCOMM & service UUIDs; server vs client sockets; blocking
  socket I/O wrapped in coroutines (`withContext(io)`); the Android 12 permission model.
- **NIA parallel:** the concrete `core:network` implementation (Retrofit) — here it's the radio;
  real Briar's `AbstractBluetoothPlugin` + `AndroidBluetoothPlugin`.
- **Done when:** two phones open an RFCOMM connection and pass raw bytes both ways.

### Phase 5 — Chat UI + message protocol + background service
- **Build:**
  - **Framing:** `[4-byte length][UTF-8 JSON payload]`, payload = `{senderName, body, timestamp}`.
  - **Wiring:** incoming → parse → `MessageRepository.save(RECEIVED)` → Flow → UI; outgoing → save
    `PENDING` → write → mark `SENT`.
  - **UI:** `feature/contacts` (device list / connect) and `feature/chat` (bubbles + input);
    `MessageBubble` lives in `core/ui`, theme/atoms in `core/designsystem`; wired via `navigation/`.
  - **Background:** a `sync/` **foreground service** holding the server socket open, declared with
    `android:foregroundServiceType="connectedDevice"` (Android 14+ requires the type **and** a Play
    Console foreground-service declaration with a video showing why it's needed).
- **Concepts:** message framing on a raw stream; kotlinx.serialization; UDF end-to-end (Flow →
  UiState → Compose); `designsystem` vs `ui` split in practice; foreground services & their types.
- **NIA parallel:** `feature:*` (UDF screens), `core:ui`/`core:designsystem`, `sync` (WorkManager),
  `core:navigation` (`NiaNavHost`).
- **Done when:** type on phone A → appears and persists on phone B, including app backgrounded.

### Phase 6 — Release & Play Store ★ (the part that makes Part A *shippable*)
- **App identity & assets:** adaptive launcher icon, app label, polished theme; store listing —
  name, short + full description, **screenshots** (2+), feature graphic.
- **Signing & build:** **upload keystore**; finalize `release` signing from `keystore.properties`
  (git-ignored); enable Play **App Signing**; signed **AAB** (`bundleRelease`); confirm R8/minify
  keeps the app working (test the release build on-device).
- **Privacy policy (required):** Bluetooth / connected-device permissions require a **privacy policy
  URL**. Proxa stores messages **only locally**, **no server** — so the policy is short and honest.
  Host it anywhere stable.
- **Play Console declarations:** **Data safety** form (no off-device data → simple but mandatory);
  **permissions** rationale for Bluetooth + the **foreground service type** (`connectedDevice`)
  declaration; **app content** (content rating, target audience, no ads).
- **Target API compliance:** `targetSdk` must meet Play's current minimum.
- **Testing tracks (real constraint):** new personal developer accounts must run **closed testing
  with ≥12 testers for ≥14 days** before promoting to production — start a closed track early.
- **Concepts:** upload vs app-signing keys; AAB vs APK; what Play reviews; privacy/data-safety even
  for a local-only app; the testing-track promotion gate.
- **Done when:** a signed AAB is on a Play **closed testing** track and installs from Play on a real
  device. (Production follows the 12-tester/14-day gate.)

**✅ End of MVP = live on Play (closed track).** A serverless offline Bluetooth messenger built with
NIA conventions, actually shipped — plus a transport seam the rest of the roadmap plugs into.

---

# Part B — Milestones toward a fuller Briar

Each milestone builds on the stable, shipped MVP. Rough order by value/effort.

### M1 — Reliability & UX polish
Auto-reconnect; online/offline state; retry `PENDING` on reconnect; delivery **acks** (status ticks
pending→sent→delivered); hardened foreground service.
*Briar ref:* `ConnectionManager`, plugin lifecycle, event bus.

### M2 — Second transport: Wi-Fi / LAN TCP ★ payoff check
`LanTcpTransport : Transport` (ServerSocket/Socket; discovery via NSD/mDNS). **Chat, Room, and UI
stay unchanged — only a new `Transport`.** This is where Phase 3 pays off.
*Briar ref:* `plugin/tcp` `LanTcpPlugin`, `PluginManager`.

### M3 — Real cryptography (end-to-end)
Key pairs per identity (X25519 + Ed25519); handshake to derive a shared secret; encrypt every
payload (libsodium/NaCl box); encrypt the Room DB (SQLCipher); QR-code contact exchange.
*Briar ref:* `bramble/crypto`, `keyagreement`, key rotation, `qrcode`.

### M4 — Group messaging (forums / private groups)
A `Group` with members; messages addressed to a group; **store-and-forward gossip** — exchange the
messages each side lacks when connected to any member. The heart of Briar's sync.
*Briar ref:* `briar/forum`, `briar/privategroup`, `bramble/sync`.

### M5 — Robust sync protocol
Replace ad-hoc framing with OFFER / REQUEST / MESSAGE / ACK records, versioned; exchange only what's
missing; handle multi-hop, out-of-order, duplicates.
*Briar ref:* Bramble Synchronisation Protocol (`bramble/sync`).

### M6 — Internet transport via Tor
`TorTransport : Transport` using embedded Tor + a hidden service per identity — still just another
plugin.
*Briar ref:* `plugin/tor`, `onionwrapper`/`lyrebird`.

### M7 — Offline relay (mailbox)
An always-on mailbox holding messages for an offline contact until they reconnect.
*Briar ref:* `bramble/mailbox`, `briar-mailbox`.

### M8 — Contact introduction & sharing
Introduce two contacts to each other; share forums/blogs/feeds.
*Briar ref:* `briar/introduction`, `briar/sharing`, `briar/blog`, `briar/feed`.

### M9 — Modularize + cross-platform (KMP / iOS)
**First** promote the `core/*` and `feature/*` packages to real Gradle modules (add NIA-style
`build-logic` convention plugins) — the boundaries are already clean. **Then** extract the
transport-agnostic core into KMP `commonMain`; radios in `androidMain`/`iosMain` via
`expect`/`actual`. *iOS caveat:* no classic RFCOMM for third-party apps — iOS gets **BLE
(CoreBluetooth)** with a smaller-MTU model; Tor/LAN port more cleanly.

---

## How Proxa maps to real Briar

| Proxa piece | Real Briar |
|---|---|
| `core/transport` `Transport` / `Connection` | `DuplexPlugin` / `DuplexTransportConnection` |
| `core/transport/bluetooth` | `AbstractBluetoothPlugin` + `AndroidBluetoothPlugin` |
| `LanTcpTransport` (M2) | `plugin/tcp` `LanTcpPlugin` |
| `TorTransport` (M6) | `plugin/tor` |
| message framing → sync (M5) | Bramble Synchronisation Protocol (`bramble/sync`) |
| `core/database` + `MessageRepository` | encrypted H2 DB + `ConversationManager` |
| identity/contacts + crypto (M3) | `bramble/identity`, `contact`, `crypto`, `keyagreement` |
| groups (M4) | `briar/forum`, `briar/privategroup` |
| `sync/` foreground service | `ConnectionManager` + Android service plumbing |
| `feature/*` Compose UI | `briar-android` |

## Testing strategy (NIA-style)
- **Phase 3 `LoopbackTransport`** lets you unit-test sync/protocol logic with **no hardware**.
- **Bluetooth/Wi-Fi** need **two physical Android phones** (emulators can't do classic Bluetooth).
- Test stack: JUnit + Truth + Turbine (Flow) + Robolectric; Hilt test runner. NIA also keeps
  *test doubles* (fake repositories/data sources) — mirror that with the fake transport.
- **Test the release (R8) build on-device** before uploading — minification can break reflection
  (Room/serialization) without keep rules.
- Add an in-app debug log of connection events early.

## Key risks / watch-items
- **Bluetooth permissions** vary a lot across Android versions (esp. 12+) — budget time.
- **RFCOMM UUID** must match on both sides; one fixed app UUID for the MVP.
- **Foreground service type** (`connectedDevice`) + Play declaration is mandatory on Android 14+.
- **Play testing gate:** 12 testers / 14 days for new personal accounts before production.
- **`applicationId` is permanent** once published — `com.mtali.proxa` is the commitment.
- **Keep packages clean** — no cross-package shortcuts, or modularization later won't be mechanical.
- **Don't roll your own crypto** — at M3 use a vetted library and the simplest correct handshake.

## Suggested rhythm
Phases 0–2 = NIA fundamentals (Phase 0 already release-shaped). Phase 3 = the architectural keystone.
Phases 4–5 = the working demo. Phase 6 = ship it. Each Part B milestone is a mini-project on the
shipped MVP. Build one phase at a time, understand it fully, verify on a device.
