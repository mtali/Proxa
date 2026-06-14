# Proxa

**Talk to the people around you — no internet, no servers, no accounts.**

Proxa is an offline, peer-to-peer messenger for Android. Messages travel **directly between phones**
over Bluetooth (with Wi-Fi and more on the way) — they never pass through a server, the cloud, or
anyone else's infrastructure. There's nothing to sign up for and nothing to leak: your conversations
live only on the devices taking part in them.

It's built for the moments connectivity isn't: a crowded venue, a remote field site, a power cut, a
trip off the grid — anywhere two people are near each other but the network isn't there.

## Why Proxa

- **Serverless by design.** No backend to run, trust, subpoena, or take down. Phones talk to each
  other and that's it.
- **Private by default.** Data stays local. No accounts, no phone numbers, no contact-list upload.
- **Works offline.** If you can reach someone over Bluetooth, you can message them — internet or not.
- **Local-first.** Every message is saved on your device the instant you send it, then delivered to
  your peer when a connection exists.

## How it works

Each device runs both sides of the conversation: it **listens** for nearby peers and can **reach out**
to them. When two Proxa devices connect, they open a direct link and exchange messages over it. The
messaging logic doesn't care *how* that link is made — Bluetooth today, Wi-Fi or other radios
tomorrow — because every connection looks the same to it: a simple two-way byte stream. Adding a new
way to connect means adding one small piece, and everything else keeps working.

## Status

🚧 **Early development.** Proxa is being built in the open, one focused milestone at a time. The first
goal (the MVP) is a polished one-to-one Bluetooth chat, shippable to the Play Store. Group
conversations, Wi-Fi/LAN transport, end-to-end encryption, and internet-capable private connections
are planned next.

The full build plan lives in [`docs/PLAN.md`](docs/PLAN.md).

## Tech

- **Kotlin** + **Jetpack Compose** (Material 3)
- **MVVM / unidirectional data flow**, **offline-first repositories**
- **Room** (local storage), **DataStore** (settings/identity), **Hilt** (DI), **Coroutines + Flow**
- Modern Android architecture following [Now in Android](https://github.com/android/nowinandroid)
  conventions — packaged as a single module today, structured to split into feature/core modules as
  it grows.

## Build & run

Requires Android Studio (latest) and a device or emulator.

```bash
./gradlew :app:assembleDebug      # build the debug APK
./gradlew :app:installDebug       # install on a connected device
./gradlew test                    # run unit tests
```

> **Testing Bluetooth needs two physical Android phones** — emulators don't support classic
> Bluetooth.

## Contributing & process

Development follows a lightweight spec + TDD workflow — see [`docs/TDD.md`](docs/TDD.md). Tests are
written where they earn their keep (message handling, sync, data layer); UI and wiring are verified by
running the app.

## License

TBD.
