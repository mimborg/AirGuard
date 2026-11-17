# AirGuard

Native iOS app that keeps vehicle drivers safe on airport airside. Drivers check in at the gate, the app streams their GPS position to Dr. Liu's ACL server, and ACL pushes collision and restricted-zone alerts back to the phone. Alerts have severity levels, and Critical alerts must reach the driver even on silent. While driving, the UI works like Google Maps: very visible but limited, with large text, few buttons, no typing, and spoken alerts.

The repo is new. The structure and commands below are the plan; update this file as they become real.

## Core rules

- **Never fail silently.** If the ACL connection, GPS, or a required permission drops, the app must switch to "Not protected" and warn the driver loudly. Treat any code path that can hide a lost connection as a bug.
- **Live connection first, push as backup.** Keep a WebSocket to ACL open during an active session. APNs delivery time is not guaranteed, so push is only a fallback.
- **Severity levels:** Critical, Warning, Caution, Info, and Not protected. Each has a fixed color, sound, vibration, and spoken message. Keep them in one place (`AlertSeverity`) so they stay consistent.
- **Driving mode:** no text entry, large touch targets, and speech carries the message.

## Stack

- Swift and SwiftUI, Xcode 26.x
- Minimum iOS version: **TBD**
- Apple frameworks only for the prototype: Core Location (background location, geofences), Core Image (check-in QR code), URLSession WebSocket (ACL link), UserNotifications and APNs (Time Sensitive and Critical Alerts), AVSpeechSynthesizer (spoken alerts), MapKit (zones and nearby vehicles), ActivityKit (lock screen Live Activity)
- No third-party dependencies without a reason in the PR. If any are added, use Swift Package Manager.
- A small fake ACL server in Python (`tools/fake-acl/`) sends test alerts, heartbeats, and fake vehicles so the app can be built before the real server is ready.

## Project structure (planned)

```
AirGuard.xcodeproj
AirGuard/
  App/            app entry point, app state, environment
  Features/       one folder per screen: Onboarding, CheckIn, Status, Map, Settings
  Services/       ACLClient (WebSocket), LocationService, NotificationService, SpeechService
  Models/         Codable message types, AlertSeverity, Session, Profile
  DesignSystem/   colors, fonts, severity styles, reusable views
  Resources/      assets, sounds, Info.plist strings
AirGuardTests/    unit tests (Swift Testing)
AirGuardUITests/  UI tests (XCTest)
tools/fake-acl/   Python fake ACL server
tools/gpx/        recorded GPS routes for the Simulator
```

## Build and test

Run from the repo root on the MacBook. Pick a simulator that exists with `xcrun simctl list devices available`.

```sh
# Build
xcodebuild -project AirGuard.xcodeproj -scheme AirGuard \
  -destination 'platform=iOS Simulator,name=iPhone 17' build

# Unit and UI tests
xcodebuild -project AirGuard.xcodeproj -scheme AirGuard \
  -destination 'platform=iOS Simulator,name=iPhone 17' test

# Unit tests only
xcodebuild ... test -only-testing:AirGuardTests

# Send a test push to the booted simulator
xcrun simctl push booted <bundle-id> tools/push/critical.apns

# Start the fake ACL server
python3 tools/fake-acl/server.py
```

The Simulator can't test screen-off or background behavior, battery drain, Critical Alerts overriding silent mode, or real LTE drop-outs. Those need a real iPhone. Use GPX routes in `tools/gpx/` to simulate driving past zones.

## Xcode tooling for agents

Agents run on mal's MacBook and drive Xcode through MCP, using either Apple's Xcode MCP server (`xcrun mcpbridge`) or XcodeBuildMCP. Prefer the MCP tools for building, running tests, booting simulators, and reading build errors. Fall back to the `xcodebuild` commands above if neither is available. Builds and tests can't run in a Linux cloud session.

## Coding conventions

- SwiftUI views stay small. Logic lives in `@Observable` view models or services, not in views.
- Use Swift concurrency (`async`/`await`, actors). No completion handlers in new code. Turn on strict concurrency checking.
- Services sit behind protocols so tests and previews can use fakes (for example `ACLClientProtocol` and `FakeACLClient`).
- ACL messages are `Codable` structs in `Models/`, decoded in one place. Unknown message types are logged, never crash.
- Every view has a `#Preview`, including each severity state.
- User-facing strings go in a String Catalog. Support Dynamic Type and VoiceOver.
- Naming follows the Swift API Design Guidelines. One main type per file, with the file named after it.
- Never commit secrets, APNs keys, or provisioning profiles.

## Agent roles and handoffs

Three Claude agents share this repo. Each works on its own branch and hands off through pull requests.

| Role | Owns | Doesn't touch without a handoff |
|---|---|---|
| **Frontend** | `Features/`, `DesignSystem/`, previews, accessibility | `Services/` internals |
| **Backend** | `Services/`, `Models/`, `tools/fake-acl/`, ACL message format, notifications | view layout |
| **Testing** | `AirGuardTests/`, `AirGuardUITests/`, `tools/gpx/`, CI | production code, except small testability fixes |

How work moves:

1. Branch names: `frontend/<topic>`, `backend/<topic>`, `test/<topic>`.
2. Open a PR into `main` that says what changed, how it was tested, and what the next role needs to do.
3. When a change crosses roles (for example a new `Models/` type the UI needs), the backend PR lands the protocol and a fake first. Frontend then builds against the fake.
4. The testing agent reviews every PR, adds or updates tests, and confirms `xcodebuild test` passes before mal merges.
5. Only mal merges to `main`.
6. Keep PRs small: one feature or fix each.

## Open questions

- **Backend choice:** Firebase, Supabase, or a custom API, for profiles, check-in records, and trip history. ACL is Dr. Liu's server and handles alerts either way.
- **Outside data APIs:** which ones (for example airport maps, aircraft positions, or weather), and their terms.
- **ACL protocol:** transport (WebSocket, HTTP, or MQTT), message format, heartbeat rate, and target alert latency. The 2 Hz GPS ask may not be reachable, since iPhones usually report about 1 Hz.
- **Minimum iOS version.**
- **Onboarding:** geofence check-in, QR scan, or both.
- **Critical Alerts entitlement:** needs a paid Apple Developer account and Apple's approval. Until then, use Time Sensitive notifications.
