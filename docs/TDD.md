# Development workflow (spec + TDD) — Proxa

**This is the default process for features that carry real logic. Follow it in order — do not jump
straight to production code.** Specs live in `docs/specs/` as markdown: `docs/specs/pending/` for
in-progress, `docs/specs/done/` for finished. Use a snake/kebab-case filename describing the change
(e.g. `bluetooth_rfcomm_connect.md`).

> **Proxa is a learning project.** TDD here is not just for correctness — writing the test first
> forces you to articulate *what a piece does and why* before building it. Lean into that for the
> parts that have real behavior (see step 2b).

## When to test — and when not to
**Test when a test would actually catch a regression or clarify a design.** Don't write tests just to
have them. Use judgement:

- ✅ **Worth testing:** message framing/parsing, the sync protocol, repositories & entity↔model
  mappers, `Transport` logic via `LoopbackTransport`, ViewModel `UiState` transitions, anything with
  branching/edge cases.
- ⏭️ **Skip (or defer) tests:** project scaffolding & DI wiring, pure Compose UI/layout, theme,
  trivial data classes, one-line glue, throwaway spikes. Verify these by **running the app on a
  device** instead.
- 🔌 **Can't unit-test:** the radio, permissions, the foreground service → verify on **two physical
  phones**. (Push the testable logic behind the `Transport` interface so only the thin radio layer is
  left untested.)

A spec is still worth writing for any non-trivial change even when it ships with few or no tests —
it's where the design and the concept get articulated. Skipping tests is a deliberate call, not the
absence of one; note it in the spec's test plan ("no unit tests — UI-only, verified on device").

## 0. First time only — initialise the repo
Proxa is not yet a git repo. As part of **Phase 0** (`docs/PLAN.md`): `git init`, add a `.gitignore`,
make the first commit on `main`. After that, the loop below applies to every change.

## 1. Branch
Branch off `main` with a descriptive name, no ticket prefix (`feat/...`, `fix/...`,
`chore/...`). Commit any existing work first; never start new work on a dirty tree.

## 2. Spec first
Write `docs/specs/pending/<name>.md` covering:
- **Phase** — which Phase/Milestone in `docs/PLAN.md` this belongs to.
- **Context / motivation** — what and why.
- **Concept to learn** — the idea(s) this introduces (e.g. "Room `Flow` queries", "RFCOMM service
  UUID", "UDF `UiState`"), in a few sentences. *(Proxa-specific — this is the learning step.)*
- **Approach** — the design, and the NIA parallel / Briar reference where relevant.
- **Exact files to touch** — using the NIA package layout (`core/...`, `feature/...`, `sync/...`).
- **Test plan** — the behaviors to encode as tests, and which are unit-testable vs need a device.
- **Verification** — how we'll confirm it works (commands + on-device check if hardware is involved).

Get approval, then **commit the spec before writing any code**.

## 2b. Explain before you build *(Proxa-specific)*
Before the first red test, give a short plain-language explanation of the concept being introduced.
For non-trivial edits, show **before/after** code and explain the *why*, then apply. The goal is that
you could re-derive the code from understanding, not copy it.

## 3. TDD loop (red → green → refactor), one behavior at a time
For each behavior in the spec:

1. **RED — write the test first.** Add a test that encodes the behavior and watch it fail. Confirm it
   fails *for the right reason* (assertion, not an unrelated/compile error). Kotlin needs symbols to
   compile, so add the *minimal* API surface (signatures/stubs) only — never the real logic yet.
   - **Test against interfaces, not hardware.** Transport/sync/protocol logic must be driven through
     the `Transport`/`Connection` interface using the fake **`LoopbackTransport`** (Phase 3), so it's
     fully unit-testable with **no Bluetooth and no second phone**.
   - If a runtime can't reproduce a real condition (e.g. Robolectric can't open a real RFCOMM
     socket), assert the *invariant* instead (e.g. "framing writes a 4-byte length prefix",
     "repository saves PENDING before the transport is called") so the test is genuinely red.
2. **GREEN — implement the minimum** to make the test pass. Run it; confirm green.
3. **REFACTOR** with the test green; re-run. Reuse existing patterns/utilities (and NIA idioms)
   rather than inventing new ones.
4. Repeat for the next behavior. Proceed one step at a time.

## 4. Verify
Run and paste the results:
- `./gradlew :app:testDebugUnitTest` — full module green, including the new tests.
- `./gradlew :app:compileDebugKotlin` — clean compile.

For anything touching the **radio, permissions, or the foreground service**, unit tests are not
enough — also verify on **two physical phones** (A ↔ B) and note the result in the spec. (Emulators
can't do classic Bluetooth.) Before any Play upload, also verify the **R8/minified release build**
on-device.

Then commit with a single-line conventional subject (`feat(transport): add LoopbackTransport`,
`feat(chat): length-prefixed framing`, …).

## 5. Close the spec
`git mv docs/specs/pending/<name>.md docs/specs/done/<name>.md` and commit (keep history).

## Test stack (NIA-style — add deps when first needed)
JUnit + **Truth** (assertions) + **Turbine** (testing `Flow`) + **Robolectric** (Android APIs on the
JVM); **Hilt** test runner for DI in tests; fakes/test-doubles (the `LoopbackTransport`, fake
repositories) over mocks where practical.
