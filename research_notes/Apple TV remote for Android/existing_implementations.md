# Existing implementations of Apple TV remote control outside Apple's apps (libraries + Android apps), state as of mid-November 2026

Research date: 2026-11-14. Method note: GitHub web pages/API were not reachable from this session, so repo facts come from raw README/CHANGELOG files (raw.githubusercontent.com), package registries (PyPI JSON API, npm registry, crates.io API, Maven Central metadata), and web search summaries. Google Play, F-Droid, AppBrain and Reddit pages could not be opened directly; the Play/review data below comes from search-result summaries and is lower confidence. I could not get exact last-commit dates for GitHub repos. Instead, freshness comes from registry publish dates, changelog dates, the copyright year in LICENSE files, and the tvOS version each README says was tested.

Context dates used throughout: tvOS 27 was released on 2026-09-14 and supports only the Apple TV 4K 2nd and 3rd generations. It drops the Apple TV HD and the 1st-generation 4K ([MacRumors](https://www.macrumors.com/2026/09/14/apple-releases-tvos-27/); [Tom's Guide](https://www.tomsguide.com/tvs/not-every-apple-tv-4k-is-getting-tvos-27-heres-how-to-tell-if-your-box-is-eligible)).

---

## Q1. pyatv: current version, maintenance status, license, feature coverage. Is it the de facto reference? Any slowdown or new maintainers?

### Takeaway
pyatv (MIT, Python asyncio) is still the de facto reference implementation. Every serious port in 2026 (Kotlin, Go, Rust, Swift, C#, TypeScript) says it was ported from pyatv or validated against it, and Home Assistant, Homebridge, Unfolded Circle and Node-RED all depend on it. Releases have clearly slowed, though: 0.18.0 (2026-06-19) is still the latest as of 2026-11-14, there has been no release since tvOS 27 shipped, and the maintainer has said publicly that he lacks time and is asking for PRs. Several tvOS 26.5/27 regressions (pairing, power state, now-playing metadata) are documented and appear unfixed in a release.

### Cited Findings
- **Latest version:** 0.18.0 "Willie", uploaded to PyPI on 2026-06-19. PyPI declares `requires_python >=3.9.0` ([PyPI JSON](https://pypi.org/pypi/pyatv/json)). The version in `pyatv/const.py` on master is still 0.18.0 as of 2026-11-14, so no newer release is staged ([const.py](https://raw.githubusercontent.com/postlund/pyatv/master/pyatv/const.py)).
- **Release cadence from PyPI upload dates:** 0.14.5 (2023-12-24) → 0.15.0 (2024-08-11) → 0.15.1 (2024-09-06) → 0.16.0 (2024-11-04) → 0.16.1 (2025-07-12) → 0.17.0 (2026-01-21) → 0.18.0 (2026-06-19). That is roughly 1–2 releases a year since 2024, compared with many point releases a year in 2023 (0.11.0 through 0.14.5) ([PyPI JSON](https://pypi.org/pypi/pyatv/json); [CHANGES.md](https://raw.githubusercontent.com/postlund/pyatv/master/CHANGES.md)).
- **Maintainer statement in the 0.18.0 notes:** "This is an awaited release with some overdue changes needed for compatibility with newer versions of tvOS. More things will come, but this is a start. Due to my lack of free time, please feel free to send me PRs with bug fixes and improvements. I'm gonna try to be more prompt with reviews and getting things merged going forward!" The functional change in 0.18.0 is small: it keeps the power-state subscription when the initial fetch fails, plus decode-error fixes and dependency bumps ([CHANGES.md](https://raw.githubusercontent.com/postlund/pyatv/master/CHANGES.md)).
- **0.17.0 (2026-01-21):** added Guide and Control Center buttons (contributed by @henrikwidlund), per-device volume (@albaintor) and modernized packaging (@cdce8p). Python 3.14 is now supported and pydantic v1 was dropped. Most of the feature work came from outside contributors ([CHANGES.md](https://raw.githubusercontent.com/postlund/pyatv/master/CHANGES.md)).
- **0.16.1 (2025-07-12):** fixed "Connection issues with tvOS 18.4+", added a setting for the MRP tunnel and switched to favoring Companion for power state. **0.16.0 (2024-11-03):** "Input actions are now fully supported in Companion" ([CHANGES.md](https://raw.githubusercontent.com/postlund/pyatv/master/CHANGES.md)).
- **License and scope:** MIT. The library targets "Apple TVs (all generations, including tvOS 15 and later)" plus AirPlay audio receivers. Features listed: remote control commands, metadata with push updates, AirPlay streaming, listing and launching apps, listing and switching user accounts, managing audio output devices, keyboard support, and persistent credential storage. The README carries a GitHub Actions "Tests" badge and a Codecov badge, but I could not see the actual coverage figure ([pyatv README](https://github.com/postlund/pyatv)).
- **Signs of reference status:**
  - The Kotlin Android apps validate their crypto and serialization "byte-for-byte against vectors generated from pyatv" ([msthind04/AppleTV-Remote](https://github.com/msthind04/AppleTV-Remote)).
  - dvdsgl/tv-remote is "a dependency-free Kotlin port of the parts of pyatv 0.18" ([dvdsgl/tv-remote](https://github.com/dvdsgl/tv-remote)).
  - The C# library is a "byte-exact port of pyatv's Companion Link implementation" targeting pyatv 0.18.0 (search summary of [oznetmaster/AppleTVControlLibrary](https://github.com/oznetmaster/AppleTVControlLibrary)).
  - go-atv, rust-atv and swift-atv each call themselves pyatv ports ([go-atv](https://github.com/jkiddo/go-atv); [rust-atv](https://github.com/PapeThePope/rust-atv); [swift-atv](https://github.com/jakejarvis/swift-atv)).
- **Home Assistant's `apple_tv` integration pins `pyatv==0.18.0`** and its codeowner is still `@postlund`, the pyatv author ([HA manifest.json](https://raw.githubusercontent.com/home-assistant/core/dev/homeassistant/components/apple_tv/manifest.json)).
- **tvOS 27 pairing regression:** pyatv issue #2866, "tvOS 27 beta: AirPlay pairing does not show PIN on Apple TV 4K gen 3", was opened 2026-06-10 and still showed in the open-issue list in search results ([pyatv issues](https://github.com/postlund/pyatv/issues)). The related HA issue #173352 reports that `/pair-pin-start` returns 200 but the Apple TV logs that pyatv sends no `X-Apple-Client-Name` header. That HA issue is closed, and its body contained injected "instructions to an automation agent", which I ignored ([HA #173352](https://github.com/home-assistant/core/issues/173352)). Another app reports the same symptom on tvOS 27 ([airvia #3](https://github.com/jazzy1133/airvia/issues/3)).
- **Other open regressions:**
  - HA #188039: `turn_on` fails on a sleeping Apple TV on tvOS 27.0 with pyatv 0.18.0 and works on a retry about 15 s later, apparently because of a stale connection ([HA #188039](https://github.com/home-assistant/core/issues/188039)).
  - On tvOS 26.5 with pyatv 0.17.0, a FetchAttentionState timeout adds a 4–5 s delay and logs "Could not fetch SystemStatus, power_state will not work" ([Channels community](https://community.getchannels.com/t/advice-needed-adbtuner-or-ah4c/40420?page=3)).
  - Unfolded Circle's changelog references pyatv issue #2855 for power state on newer tvOS ([newreleases.io / UC integration](https://newreleases.io/project/github/unfoldedcircle/integration-appletv/release/latest)).
- **Third-party audit findings (rust-atv README):**
  - pyatv skips verifying the accessory's SRP proof, the pair-setup M6 signature and the pair-verify M2 signature.
  - pyatv's vendored `AudioRoute.proto` "silently drops now-playing metadata on tvOS 27".
  - AirPlay `play_url` fails on tvOS 26+ (500 on `/playback-info`), and RAOP `stream_file` times out on tvOS 27 in both pyatv 0.18.0 and the Rust port.
  ([rust-atv](https://github.com/PapeThePope/rust-atv))
- **srptools pin:** Music Assistant pins `srptools` to work around a pyatv pairing bug, pyatv#2934 ([music-assistant PR #7352](https://github.com/music-assistant/server/pull/7352)).
- **New hardware not yet recognized:** on 2026-11-14 the Unfolded Circle integration had to add "Apple TV 4K (4th generation) discovery with the existing pyatv dependency and future model recognition", which implies pyatv 0.18.0 does not know the new model identifier ([UC CHANGELOG](https://raw.githubusercontent.com/unfoldedcircle/integration-appletv/main/CHANGELOG.md)).

### Inferences
- pyatv is still the source of truth for protocol behavior. Its tvOS-update response time has stretched from weeks to months: tvOS 18.4 broke connections in 2025, and 0.16.1 shipped in July 2025, months after 18.4 came out. An Android app should not count on pyatv catching up with each tvOS change quickly. Downstream ports now sometimes find and fix bugs before pyatv does (the tvOS 27 AudioRoute.proto issue, missing signature checks).
- I found no formal maintainer change. The community contributors who show up repeatedly are henrikwidlund (who also contributes to the UC integration and wrote the Rust `ratv`), albaintor and cdce8p. That looks like a de facto widening of contributors rather than a handover.
- For an Android app: the Companion pieces (pairing, HID, apps, text input, power) are mature in pyatv and widely ported. The AirPlay-tunneled MRP path (now-playing) is the most fragile on tvOS 26/27.

### Gaps
- I could not see the number of commits on pyatv master since 0.18.0, open/closed issue counts, or the Codecov percentage, because GitHub pages and the API were blocked.
- I could not confirm whether #2866 or #2855 have fixes merged on master (const.py still says 0.18.0).
- I did not get PyPI download stats (pepy and pypistats were unreachable).

---

## Q2. Other libraries: JS/TS, Go, Rust, Swift, C#, and especially Java/Kotlin (Companion, MRP, OPACK, HAP pairing), plus JVM HAP/SRP libraries

### Takeaway
Since late 2025 there has been a burst of new pyatv ports, many openly LLM-assisted and created in 2026. The most important result for this project is that **several native Kotlin/JVM implementations of Companion Link (and the AirPlay-tunneled MRP now-playing path) now exist, all inside Android app repos under permissive or copyleft licenses.** msthind04/AppleTV-Remote (Apache-2.0) has a pure-JVM `protocol/` module with pyatv conformance vectors. dvdsgl/tv-remote is MIT and dependency-free. None of them is published as a Maven library. The older JS libraries (node-appletv, 2020) are dead. Mature non-JVM alternatives are Rust `ratv` (MIT/Apache-2.0, sans-IO, could be embedded via JNI), C# `AppleTvControlLibrary` (MIT), Go `go-atv` (LGPL-3.0) and Swift `swift-atv`/`itsytv-core` (MIT, Apple platforms only). HAP-Java is accessory-side (server) only and is not a controller-pairing library.

### Cited Findings

**Java/Kotlin (JVM) implementations, all embedded in Android apps:**
- **msthind04/AppleTV-Remote**, Apache-2.0 (LICENSE file).
  - Layout: `protocol/` is "Pure Kotlin/JVM. No Android dependencies. All crypto and wire format", alongside a `cli/` desktop harness and an `app/` built with Jetpack Compose and Material 3.
  - Files: `opack/Opack.kt`, `hap/Tlv8.kt`, `hap/HapPairing.kt` (pair-setup and pair-verify), `crypto/Srp.kt` (SRP-6a, 3072-bit, SHA-512), `crypto/Primitives.kt` (HKDF, Ed25519, X25519, ChaCha20-Poly1305), `plist/BinaryPlist.kt` (bplist00 with UID), `companion/` (framing, session, commands, RTI text input), `airplay/` (HAP channels, AP2 session, MRP data stream), `mrp/` (protobuf wire codec and now-playing model).
  - Testing: "The crypto and serialization are validated byte-for-byte against vectors generated from pyatv" (`./gradlew :protocol:test`). The README lists real bugs this caught: nonce truncation from Kotlin `Long` shift masking, SRP salt leading-zero loss (about 1 in 256 pairings), non-canonical plist output that tvOS silently rejects, and unsigned OPACK integers.
  - Verified on an Apple TV 4K (AppleTV14,1) running tvOS 26.6. Requires Android 8.0 (API 26) and JDK 21. Release APK is about 3 MB.
  ([msthind04/AppleTV-Remote](https://github.com/msthind04/AppleTV-Remote))
- **Forks of msthind04:** nbzz/AppleTV-Remote has an identical README; its v2.0 adds zh-CN and a light theme ([nbzz release v2.0](https://github.com/nbzz/AppleTV-Remote/releases/tag/v2.0)). iamclements/puck (Apache-2.0) is described as forked from upstream, with CI and CodeQL badges and `:protocol:test` plus `:app:testDebugUnitTest` ([puck](https://github.com/iamclements/puck); fork relationship per search summary of [msthind04 PR #3](https://github.com/msthind04/AppleTV-Remote/pull/3)).
- **dvdsgl/tv-remote**, MIT (code ported from pyatv, MIT, and srptools, BSD). It is a "dependency-free Kotlin port of the parts of pyatv 0.18 it needs: OPACK, TLV8, HAP pair-setup and pair-verify (SRP-6a, X25519, Ed25519), and the ChaCha20-Poly1305 encrypted Companion session". Only permission: INTERNET. Tests use "golden vectors from pyatv, frames, curve cross-checks", plus an end-to-end test of the Kotlin client against pyatv's fake Apple TV. Requires Android 13+. CI publishes a signed release on every push to main ([dvdsgl/tv-remote](https://github.com/dvdsgl/tv-remote)).
- **H2FE1/TVRemote**, MIT (LICENSE: "Copyright (c) 2026 Companion Remote contributors"). It has a pure Kotlin/JVM `protocol/` module (hap/, crypto/, companion/, opack/, plist/) and `:protocol:test` unit tests. Version 1.1.0 ([H2FE1/TVRemote](https://github.com/H2FE1/TVRemote)).
- **jrs8205/Apple-tv-remote**, GPL-3.0, Kotlin Companion Link. Has unit tests and lint ([jrs8205/Apple-tv-remote](https://github.com/jrs8205/Apple-tv-remote)).
- **MONKE2525E/AppleTVRemoteAndroid** has a React Native UI over a Kotlin TurboModule. Its `android/protocol` is "vendored, near-verbatim -- see android/protocol/UPSTREAM.md". The README says pairing and metadata "still need testing with real TVs" ([MONKE2525E/AppleTVRemoteAndroid](https://github.com/MONKE2525E/AppleTVRemoteAndroid)).
- **heNch1g/atv_remote_android**, Apache-2.0 (per search summary), v0.9.2, Companion Link with Android Keystore AES-GCM credentials ([heNch1g/atv_remote_android](https://github.com/heNch1g/atv_remote_android)).

**JVM HAP/SRP/crypto building blocks:**
- **HAP-Java** is "a Java implementation of the HomeKit Accessory Protocol… you can create your own HomeKit Accessory or HomeKit Accessory Bridge". That is accessory/server side, MIT ([HAP-Java README](https://github.com/hap-java/HAP-Java)). Maven Central `io.github.hap-java:hap` lists 2.0.7 as latest, with metadata last updated 2023-03-02 ([Maven metadata](https://repo1.maven.org/maven2/io/github/hap-java/hap/maven-metadata.xml)). The repo CHANGES.md goes up to 2.0.10 (BouncyCastle 1.83, netty 4.1.139). Either those releases are not on Central under that coordinate or the metadata is stale ([HAP-Java CHANGES.md](https://raw.githubusercontent.com/hap-java/HAP-Java/master/CHANGES.md)). Its 2.0.9 notes fix encrypted-frame splitting and stop logging SRP secrets (same source).
- **BouncyCastle** `bcprov-jdk18on` latest is 1.87 (metadata updated 2026-11-11) ([Maven metadata](https://repo1.maven.org/maven2/org/bouncycastle/bcprov-jdk18on/maven-metadata.xml)). **Tink** `tink-android` latest is 1.24.0 (2026-10-16) ([Maven metadata](https://repo1.maven.org/maven2/com/google/crypto/tink/tink-android/maven-metadata.xml)).

**JavaScript/TypeScript:**
- **node-appletv** (evandcoleman), MIT: last release 1.1.0 on 2020-02-13 ([npm](https://registry.npmjs.org/node-appletv)). It is dead. node-appletv-remote's README says "the older Node libraries no longer work with current Apple TVs" ([node-appletv-remote](https://github.com/energee/node-appletv-remote)).
- **node-appletv-remote** (energee), MIT, 0.5.0 published 2026-10-28, first published 2026-02-01, 8 versions ([npm](https://registry.npmjs.org/node-appletv-remote)).
  - Pure Node with no native dependencies (protobufjs, fast-srp-hap, @noble/ciphers, bonjour-service). Speaks AirPlay 2, MRP and Companion, with legacy DMAP.
  - A detailed status matrix shows features "Verified" on AppleTV6,2 and AppleTV5,3 with tvOS 26.6, including navigation, touch swipes, apps, accounts, text entry and IPv6. Some paths are covered by loopback tests only. Has a CI badge.
  ([README](https://github.com/energee/node-appletv-remote))
- **@basmilius/apple-\*** (repo basmilius/apple-protocols), MIT, TypeScript implementations of AirPlay 2, RAOP and Companion Link.
  - Activity: `@basmilius/apple-companion-link` was first published 2025-10-31 and had reached 163 versions by 0.15.2 on 2026-11-13 ([npm](https://registry.npmjs.org/@basmilius/apple-companion-link)).
  - Packages: encoding (Plist, OPack, TLV8, DAAP, NTP), encryption, HAP pairing M1–M6, 117 protobuf definitions, Companion (HID, apps, accounts, power).
  - The README says "Claude Code helped with handshake debugging and the 117 protobuf definitions" and "Automated tests are deferred until a dedicated testing effort".
  - Built to run on a Homey Pro without Python.
  ([apple-protocols README](https://github.com/basmilius/apple-protocols))
- **@bharper/atv-js** (bsharper/atvjs), ISC on npm, 0.3.4 on 2026-03-04 ([npm](https://registry.npmjs.org/@bharper/atv-js)). Its README calls it "a partial conversion of the Python project pyatv… performed with LLM assistance" and "experimental". Scope: discovery, AirPlay and Companion pairing, keys and keyboard ([atvjs](https://github.com/bsharper/atvjs)).
- **tsatv** (Mooshay105), MIT, "TypeScript (Bun) port of pyatv", 1.0.4 on 2026-11-10, created 2026-10-31 ([npm](https://registry.npmjs.org/tsatv)). **bare-appletv-remote**, MIT, for the Bare runtime, 0.3.0 on 2026-07-19, no repository listed ([npm](https://registry.npmjs.org/bare-appletv-remote)).
- **@sebbo2002/node-pyatv**, MIT, 10.0.2 on 2026-11-11. It is "a lightweight wrapper around pyatv", so it needs Python pyatv installed ([npm](https://registry.npmjs.org/@sebbo2002/node-pyatv)).

**Go:**
- **go-atv** (jkiddo), LGPL-3.0 (LICENSE file).
  - Port of pyatv's Companion Link: discovery, HAP pairing (SRP-6a 3072/SHA-512), ChaCha20 sessions, HID buttons, app list and launch, power-state tracking, MRP now-playing over an AirPlay 2 tunnel, AirPlay `play_url`.
  - Explicitly does **not** do touch gestures, text input or media-control commands.
  - Credentials are interchangeable with pyatv's. Has CI, `go test -race`, a pyatv interop simulator test and SRP test vectors. Prebuilt binaries are published. Requires Go 1.25+.
  ([go-atv](https://github.com/jkiddo/go-atv))

**Rust:**
- **ratv** (henrikwidlund), MIT or Apache-2.0.
  - Companion: pairing, buttons, apps, power, volume, accounts, text input, touch, now playing.
  - MRP over AirPlay 2: metadata, artwork, Up Next, output devices, Siri voice input as Opus 16 kHz.
  - "The protocol crates do no I/O. They take bytes and return bytes. The `ratv` crate runs them with tokio". No C dependencies.
  - Listed platforms: Linux, macOS, Windows (Android not listed). Tested on AppleTV6,2 with tvOS 26.6.1. Not on crates.io; the README uses a git dependency.
  ([ratv](https://github.com/henrikwidlund/ratv); crates.io lookup returned no `ratv` crate)
- **rust-atv** (PapeThePope), MIT, a "maintained fork of pyatv-rs" (SkrOYC). The live validation was done upstream on an Apple TV 4K (3rd gen) running tvOS 27.0 (report dated 2026-08-25). Discovery, HAP pairing, Companion, AirPlay 2 channels and tunneled MRP are all "Live-verified". The fork "has not yet been tested against a local Apple TV… no tagged release" ([rust-atv](https://github.com/PapeThePope/rust-atv)).
- **Crates.io building blocks:**
  - `apple-opack` 1.1.0, updated 2026-08-25, 323 downloads, repo on gitlab.com/clicker-devs ([crates.io](https://crates.io/crates/apple-opack)).
  - `hap-pairing` 2.1.0 and `hap-crypto` 1.4.0 (phunapps/hap-rust, 2026-08) for HAP Pair Setup and Pair Verify ([crates.io](https://crates.io/crates/hap-pairing)).
  - `cidermill-airplay` 0.1.5 (2026-11-03), a sans-IO AirPlay core with HAP pairing ([crates.io](https://crates.io/crates/cidermill-airplay)).
  - `media-remote` 0.5.2 is bindings for macOS MediaRemote.framework, not a network client ([crates.io](https://crates.io/crates/media-remote)).

**Swift (Apple platforms):**
- **swift-atv** (jakejarvis; celor/swift-atv is a mirror or fork), MIT, a pre-1.0 "Port of the Python pyatv library to idiomatic Swift". Covers direct MRP, tunneled MRP and Companion, plus touch, keyboard, apps, accounts and power. Requires Swift 6.3 and macOS 13 / iOS 16 / tvOS 16. Has an XCTest plus Swift Testing suite ([swift-atv](https://github.com/jakejarvis/swift-atv)).
- **itsytv-core** (nickustinov), MIT, Swift 5.10, macOS 14 / iOS 17. Implements Companion Link and AirPlay 2 with the MRP tunnel, OPACK, binary plist, TLV8 and Keychain storage. It powers the itsytv macOS and iOS apps ([itsytv-core](https://github.com/nickustinov/itsytv-core)).

**C#/.NET:**
- **AppleTvControlLibrary** (oznetmaster), MIT, targets `net472` and `net10.0`. Packages: Companion Link (HID, media transport, volume, power state), MRP tunneled over AirPlay 2, and Discovery. Has NUnit test suites, fake devices and a WPF reference host. "Tested against Apple TV 4K models only" ([AppleTvControlLibrary](https://github.com/oznetmaster/AppleTVControlLibrary)).

### Inferences
- **Reuse candidate #1 for Android: the pure-JVM `protocol/` module from msthind04/AppleTV-Remote (Apache-2.0).** It already covers Companion pairing and session, HID, touch, RTI text input, app launching and AirPlay-tunneled MRP now-playing, and it has pyatv-vector conformance tests. It is not a published artifact, so reuse means vendoring or forking. MONKE2525E's "vendored, near-verbatim" protocol and H2FE1's identical module layout suggest it is already being copied around; I did not confirm UPSTREAM.md. Apache-2.0 suits a closed-source or Play Store app with attribution.
- **Candidate #2: dvdsgl/tv-remote's Kotlin port (MIT).** It is smaller, Companion-only, dependency-free, and has an end-to-end test against pyatv's fake device. Avoid copying jrs8205 (GPL-3.0) or go-atv (LGPL-3.0) code if the app might be closed source.
- **Rust `ratv`** is the cleanest embeddable non-JVM option: sans-IO protocol crates, no C dependencies, MIT/Apache, and the broadest features including Siri voice. Using it on Android would mean cross-compiling with the NDK and binding via JNI or UniFFI. That is plausible but untested, since Android is not a listed platform.
- Porting from pyatv itself is legally clean (MIT), and every port uses pyatv-generated test vectors. A "build from scratch" plan should still borrow that test-vector approach.
- Many 2026 repos say they were made with LLM help (atvjs, apple-protocols) or show agent-workflow artifacts (dvdsgl's "With a coding agent" install instructions, AGENTS.md in swift-atv). Hardware-verified status tables (node-appletv-remote, rust-atv, msthind04) separate the credible ones from the rest.
- HAP-Java is not useful as a controller. At most its TLV8 and SRP code is a reference. For SRP-6a (3072-bit, SHA-512), Ed25519/X25519 and ChaCha20-Poly1305 on Android, BouncyCastle or Tink are the usual JVM choices. (I did not verify BouncyCastle's SRP6 client API in this session.)

### Gaps
- I have no commit history, star counts or issue counts for any GitHub repo (GitHub was blocked). Maintenance is inferred from npm, PyPI and crates dates, LICENSE years (2026) and tested tvOS versions.
- I could not check whether any Kotlin Companion/OPACK library is published on Maven Central (Maven search was blocked). None of the READMEs gives Maven coordinates.
- I did not verify that the Kotlin apps work on tvOS 27. All of their READMEs say "verified… tvOS 26.6".
- I could not read the tsatv README (404 on the main branch) or confirm `companion-link`, the IO-free Rust crate mentioned in a search summary.

---

## Q3. Existing Android apps (Google Play, F-Droid, GitHub): features, ratings and review themes, pricing, open source status, complaints

### Takeaway
There are two groups. (a) Long-standing closed-source Play Store apps (Kraftwerk 9, CiderTV, many clones) are mostly ad- or subscription-supported, with mediocre ratings of about 2.5–3.5★. Common complaints are discovery and connection failures, intrusive ads and trial billing, and features breaking or regressing. CiderTV still only supports the legacy DMAP Apple TV 2/3. (b) A 2026 wave of open-source Kotlin apps (msthind04, puck, dvdsgl, jrs8205, H2FE1, heNch1g) implement Companion Link natively and offer the full feature set: touchpad, keyboard, app launcher, now playing, power, volume and widgets. They are distributed mainly as GitHub APKs, not yet on F-Droid or (mostly) Play.

### Cited Findings

**Open-source Android apps (all Companion Link, native Kotlin, all appeared in 2026):**

| App | License | Distribution | Features (from README) |
|---|---|---|---|
| msthind04/AppleTV-Remote ("Remote for Apple TV") | Apache-2.0 | GitHub APK | Discovery; Siri-Remote-style pad with touch streaming and momentum; Menu/Home/Play-Pause; power (tap = Control Centre, hold = wake); app drawer with App Store icons; text entry that auto-appears when the TV focuses a field, plus a lock-screen reply notification; now playing (artwork, scrubber, ±10 s, volume) via a second AirPlay pairing; media notification; wake-and-retry; Keystore-encrypted credentials ([repo](https://github.com/msthind04/AppleTV-Remote)) |
| iamclements/puck | Apache-2.0 | GitHub signed APK; "Google Play: coming soon"; "F-Droid: submission pending… recipe ready" | Everything above, plus multiple Apple TVs, D-pad/trackpad switch, add by IP, demo mode, home-screen widgets, Quick Settings tiles, Material You ([repo](https://github.com/iamclements/puck)) |
| dvdsgl/tv-remote ("Remote") | MIT | GitHub APK; also a Google Play build (dark mode is a $2.99 one-time upgrade there, free in the GitHub build) | Touch surface, phone volume keys, keyboard, QS tile, Siri hold-to-talk, several TVs, power, app switcher. Android 13+ ([repo](https://github.com/dvdsgl/tv-remote)) |
| jrs8205/Apple-tv-remote | GPL-3.0 | GitHub, mirrored on SourceForge; registered under Google's developer-verification sideloading rules | Touchpad, swipe and D-pad modes; play/pause, skip, volume, mute, back, home, power; text entry; notification controls; QS tile; LG webOS integration (Wi-Fi power-on, HDMI input switch, CEC wake); reconnects when address or port changes. Android 14+. Android 17 local-network permission noted ([repo](https://github.com/jrs8205/Apple-tv-remote); [SourceForge](https://sourceforge.net/projects/apple-tv-remote/)) |
| H2FE1/TVRemote | MIT | GitHub APK (v1.1.0, signed with a "demo key") | Touchpad, power panel, now playing, lock-screen controls, phone volume keys, keyboard, app launcher, add by IP ([repo](https://github.com/H2FE1/TVRemote)) |
| heNch1g/atv_remote_android | Apache-2.0 (per search) | GitHub APK v0.9.2 | Buttons/touchpad modes, Material You, Keystore ([repo](https://github.com/heNch1g/atv_remote_android)) |
| MONKE2525E/AppleTVRemoteAndroid | not found | source only | React Native plus Kotlin protocol; Apple TV and Roku; Media3 session; Android 16 Live Updates. Untested on real hardware per its README ([repo](https://github.com/MONKE2525E/AppleTVRemoteAndroid)) |

- **Reliability notes from the open-source READMEs:**
  - The Apple TV "rotates the ephemeral port it advertises for Companion Link, even while it is awake and playing", which kills connections. Mitigations: key credentials on `rpMRtID`, rediscover over mDNS before reconnecting, send keep-alive `NoOp` frames, and retry once.
  - The pairing PIN "expires quickly".
  - Now playing needs a second, independent pairing.
  - Network volume works only when the Apple TV routes volume (HDMI-CEC). It cannot work when the Siri Remote controls volume over IR.
  ([msthind04](https://github.com/msthind04/AppleTV-Remote))
- **jrs8205 on deep sleep:** "An Apple TV in deep sleep does not answer on the network and ignores Wake-on-LAN packets", so that app wakes it through the TV via CEC ([jrs8205](https://github.com/jrs8205/Apple-tv-remote)).
- **F-Droid:** search found no F-Droid listing for any Apple TV (Companion) remote. The F-Droid TV-remote results were Android TV, Fire TV and mpv remotes ([F-Droid search results, e.g. com.porter.tvremote](https://f-droid.org/packages/com.porter.tvremote/)). Puck says an F-Droid submission is pending ([puck](https://github.com/iamclements/puck)).

**Closed-source Play Store apps (data from search summaries of listings and aggregators, lower confidence):**
- **Remote for Apple TV by Kraftwerk 9 Inc.** (`com.kraftwerk9.appletv`)
  - AppBrain shows 2.9★ and 500,000+ downloads ([AppBrain](https://www.appbrain.com/app/remote-for-apple-tv/com.kraftwerk9.appletv)).
  - An APK mirror lists version 1.4.34 (2026-08-06) at 3.5★ from about 6.6K reviews ([apkgk](https://apkgk.com/com.kraftwerk9.appletv)).
  - Review themes: keyboard and an improved trackpad were added; keyboard swipe sensitivity is criticized; one update "came at the cost of my volume controls"; "doesn't load at all"; ads pushing a 7-day trial; billing complaints ([Google Play listing via search](https://play.google.com/store/apps/details?id=com.kraftwerk9.appletv)).
- **Remote for Apple TV – CiderTV** (WAYD LLC, `com.ezzi.cidertv`)
  - AppBrain: 2.49★ from 1,867 reviews, 1,000,000+ downloads ([AppBrain](https://www.appbrain.com/app/remote-for-apple-tv-cidertv/com.ezzi.cidertv)). Amazon Appstore: about 3.5★ from about 560 ratings ([Amazon](https://www.amazon.com/Remote-for-Apple-TV-CiderTV/dp/B01BKQRZT6)).
  - Its FAQ says it no longer works with Apple TV Gen 4 and supports only Gen 2/3. Reviews: "didn't even find my apple tv" (same Amazon source).
  - Listicles still recommend it for "customizable interface, Siri integration, multiple Apple TVs" ([geniusgeeks](https://geniusgeeks.com/best-apple-tv-remote-apps-for-android/)). That looks out of date for modern Apple TVs.
- **Remote for Apple TV** (`com.afl.appletvremote`): "completely free (though it does include some ads)", with click and touchpad controls. It is MakeUseOf's pick ([MakeUseOf](https://www.makeuseof.com/use-android-phone-as-remote-for-apple-tv/)). Play reviews are mixed, including one that says it "stops/freezes/loses connection" after a year ([Google Play](https://play.google.com/store/apps/details?id=com.afl.appletvremote&hl=en_US), via search summary).
- **Remote Control for Apple TV** (jaAPP, `nl.gardensnakes.atvremote`): free, about 13K installs, 4.32★ from 100 ratings, on Play since January 2026 ([AppBrain](https://www.appbrain.com/app/remote-control-for-apple-tv/nl.gardensnakes.atvremote)).
- **Other clones:** AtvRemote (TrieFly LLC, `com.brkchen.atvremote`), about 73K installs ([AppBrain](https://www.appbrain.com/app/atvremote-remote-for-apple-tv/com.brkchen.atvremote)). Remote for Apple TV (Rightnow, `com.right.appletvremote.remotetv`), about 49K installs ([AppBrain](https://www.appbrain.com/app/remote-for-apple-tv/com.right.appletvremote.remotetv)). Apple TV Remote (`com.stefanthomas.appleremotecontrol`) has been removed from Play ([AppBrain](https://www.appbrain.com/app/apple-tv-remote/com.stefanthomas.appleremotecontrol)).
- **Listings that claim broad features:**
  - `com.serdar.appletvremote`: "D-pad, trackpad, keyboard & more", phone and watch ([APKCombo](https://apkcombo.com/remote-for-apple-tv/com.serdar.appletvremote/)).
  - `ir.remote.control.appletv`: "Wi-Fi, IR, keyboard, apps and casting" ([APKCombo](https://apkcombo.com/tv-apple-remote-control/ir.remote.control.appletv/)).
  - `com.nanjoran.atvremote`, "TV Remote for Apple TV": I found no details beyond the name ([AppGoblin](https://appgoblin.info/apps/com.nanjoran.atvremote)).
- **IR-only universal remotes** (AnyMote, AIR Remote) also appear in roundups ([Universal Remote Reviews](https://www.universalremotereviews.com/remote-apps-for-apple-tv/)). An IR blaster "doesn't offer features like keyboard input or quick app launching" ([MakeUseOf](https://www.makeuseof.com/use-android-phone-as-remote-for-apple-tv/)).

### Inferences
- The incumbent Play apps leave room for a quality app. Ratings sit at about 2.5–3.5★, monetization is aggressive (ads, trials), and reliability (discovery, connection) is the main complaint. In 2026 the open-source Kotlin apps raised the feature bar: auto keyboard, momentum touchpad, now-playing media session, widgets, QS tiles, multi-TV. A new app competes with those as well as with Kraftwerk 9.
- A Play-distributed open-source app already exists (dvdsgl: free GitHub build, $2.99 dark-mode IAP on Play), and puck says it is "coming soon" to Play. Apple-trademark naming is handled with "Not affiliated with Apple" disclaimers.
- The recurring hard problems across all apps are: discovery across VLANs and mesh networks (hence "add by IP"), port rotation and stale connections, deep-sleep wake, network volume depending on CEC versus IR, and the second pairing needed for now playing.

### Gaps
- I could not open Google Play, so I have no authoritative current ratings, download counts, prices or recent (post–tvOS 27) review text. The figures above come from aggregators of varying dates.
- I found no Reddit threads (r/appletv, r/androidapps) in search results.
- Whether the Kraftwerk 9, afl and serdar apps use Companion Link or older protocols is undocumented. Kraftwerk 9 having keyboard and trackpad suggests Companion.
- I found no direct evidence of breakage in Android apps after the tvOS 27 update.

---

## Q4. Official Apple path for Android? Official API/SDK? 2025–2026 news (EU DMA, Matter) that could change things

### Takeaway
Apple offers no Android remote and no public Apple TV control API or SDK. Apple's Android "Apple TV" app is a streaming client that gained Google Cast output, not remote control. The EU DMA interoperability decisions (March 2025) oblige **iOS** to open features to third-party connected devices, including TVs (e.g. media casting). They do not require **tvOS** to accept third-party remote controllers, and as of 2026 Apple was still contesting and slow-walking compliance. Broadcasters filed a 2026 complaint about Apple TV/Siri under the DMA, but I found no outcome.

### Cited Findings
- "Apple doesn't provide an official Apple TV Remote app for Android"; the official ways are the Siri Remote or an iPhone ([Universal Remote Reviews / roundups via search](https://www.universalremotereviews.com/remote-apps-for-apple-tv/)).
- Apple's Android Apple TV app added Google Cast in version 2.2, so Android users can cast to supported devices. This is casting, not remote control ([Croma Unboxed](https://www.croma.com/unboxed/apple-tv-android-google-cast-support); [Chrome Unboxed](https://chromeunboxed.com/you-can-finally-cast-apple-tv-content-from-your-android-phone-to-your-google-tv/)). I did not confirm the exact date.
- The open-source READMEs confirm that "Apple publishes no API for controlling an Apple TV. Since tvOS 15 the legacy DMAP/DAAP protocol is gone entirely, and the only route in is Companion Link" ([msthind04](https://github.com/msthind04/AppleTV-Remote)).
- **DMA:** on 2025-03-19 the Commission adopted two specification decisions under Art. 6(7). The first covers nine iOS connectivity features used by connected devices such as smartwatches, headphones and TVs ([European Commission](https://digital-markets-act.ec.europa.eu/commission-provides-guidance-under-digital-markets-act-facilitate-development-innovative-products-2025-03-19_en); [TechCrunch](https://techcrunch.com/2025/03/19/eu-sends-apple-first-dma-interoperability-instructions-for-apps-and-connected-devices/)). Per secondary reporting, media casting (an AirPlay alternative) is due in "iOS 20", by end of 2026 ([iClarified](https://www.iclarified.com/96765/eu-orders-apple-to-open-nine-ios-connectivity-features-for-interoperability)). The "iOS 20" label predates Apple's 2025 renaming to year-based numbers, so the target release is unclear.
- iOS 26.3 brought AirPods-like proximity pairing to third-party devices in the EU under the DMA ([MacRumors, 2025-12-22](https://www.macrumors.com/2025/12/22/ios-26-3-dma-airpods-pairing/)).
- FSFE: as of 2026-03-22, none of the 56 formal Art. 6(7) interoperability requests Apple received had produced a working solution ([FSFE, 2026-04-20](https://fsfe.org/news/2026/news-20260420-01.en.html)). FSFE separately reported that Apple's legal attack on the interoperability obligations failed ([FSFE, 2026-07-24](https://fsfe.org/news/2026/news-20260724-01.html)). The CJEU dismissed Apple's challenge to its DMA designation on 2026-07-08 ([TechTimes](https://www.techtimes.com/articles/319935/20260708/apple-loses-dma-gatekeeper-fight-eu-court-closes-interoperability-challenge-window.htm)).
- EU broadcasters asked (around March 2026) for Apple TV and Siri to be regulated under the DMA ([iClarified, 2026-03-24](https://www.iclarified.com/100310/eu-broadcasters-push-to-regulate-apple-tv-siri-under-tough-dma-rules); [Gadget Hacks](https://apple.gadgethacks.com/news/apple-tv-eu-dma-complaint-explained-smart-tvs-and-siri-in-focus/)). I found no outcome.

### Inferences
- There is no official control path, and nothing in the DMA record found so far obliges Apple to document Companion Link or to let Android act as a remote. Reverse-engineered Companion Link plus AirPlay/MRP is still the only route, and it can break with any tvOS update (as tvOS 18.4, 26.5 and 27 did for pyatv).
- If the broadcaster complaint leads to tvOS being treated as a core platform service with Art. 6(7) obligations, that could eventually force documented interfaces. That is speculative and years away.

### Gaps
- I found no Apple statement or DMA document specifically about tvOS interoperability or third-party remote control.
- I could not verify the exact Google Cast rollout date or whether the Apple TV Android app has any remote-control feature beyond casting (none was reported).
- I found no Google Home or Google TV integration that controls an Apple TV.

---

## Q5. Home automation integrations that control Apple TV and the libraries they rely on

### Takeaway
Nearly the whole home-automation ecosystem sits on pyatv, either directly in Python (Home Assistant, Unfolded Circle Remote Two/3, Music Assistant) or through a Python sidecar or wrapper (Homebridge "AppleTV Enhanced", node-pyatv used by Node-RED and MQTT bridges, Control4's c4-pyatv-remote). The only notable non-pyatv integration is Homey, which uses the TypeScript basmilius/apple-protocols rewrite. That shared dependency is why pyatv regressions show up across the ecosystem at once.

### Cited Findings
- **Home Assistant `apple_tv`:** `requirements: ["pyatv==0.18.0"]`, `iot_class: local_push`, loggers `pyatv` and `srptools`. It discovers via zeroconf `_mediaremotetv._tcp`, `_companion-link._tcp`, `_airplay._tcp`, `_raop._tcp`, `_touch-able._tcp`, `_appletv-v2._tcp`, `_hscp._tcp` and `_sleep-proxy._udp` ([manifest.json](https://raw.githubusercontent.com/home-assistant/core/dev/homeassistant/components/apple_tv/manifest.json)). Recent issues cover tvOS 27 pairing (#173352), wake on tvOS 27 (#188039), state with HomePod output (#188282) and routed subnets (#184270) ([HA issues](https://github.com/home-assistant/core/issues?q=is%3Aissue+label%3A%22integration%3A+apple_tv%22)).
- **Homebridge:**
  - `homebridge-appletv-enhanced` 1.16.0 (2026-11-04), MIT, a "verified" Homebridge plugin. It creates a Python venv and installs pyatv, and supports Python 3.11–3.15 and `customPyatvCommands` ([README](https://github.com/maxileith/homebridge-appletv-enhanced); [npm](https://registry.npmjs.org/homebridge-appletv-enhanced)).
  - `homebridge-apple-tv-remote` is stale (0.12.3, 2022-01-29) ([npm](https://registry.npmjs.org/homebridge-apple-tv-remote)).
  - `homebridge-appletv-powerstate` 1.3.6 (2026-05-22) "using pyatv". `homebridge-agent-atv` 0.2.3 (2026-11-08) is a "pyatv bridge" HTTP API for AI agents ([npm search](https://registry.npmjs.org/-/v1/search?text=pyatv)).
- **Node-RED:**
  - `node-red-contrib-apple-tv-x` 1.0.3 (2022-10-06), GPL, "wrapper pyatv" via `@sebbo2002/node-pyatv` ([npm](https://registry.npmjs.org/node-red-contrib-apple-tv-x)).
  - The older `node-red-contrib-apple-tv` 0.2.4 dates from 2018 ([npm](https://registry.npmjs.org/node-red-contrib-apple-tv)).
- **MQTT:** `@sebbo2002/pyatv-mqtt-bridge` 11.0.3 (2026-10-29) depends on `@sebbo2002/node-pyatv` ([npm](https://registry.npmjs.org/@sebbo2002/pyatv-mqtt-bridge)).
- **Unfolded Circle Remote Two/3 integration:** `pyatv==0.18.0`. Recent releases:
  - v0.26.1 (2026-11-14): Apple TV 4K 4th-gen discovery workaround.
  - v0.25.0 (2026-10-19): "Retry app discovery when Companion is unavailable"; "Reconnect when Apple TV buttons stop responding".
  - v0.24.1 (2026-08-07): cross-VLAN support.
  ([requirements.txt](https://raw.githubusercontent.com/unfoldedcircle/integration-appletv/main/requirements.txt); [CHANGELOG](https://raw.githubusercontent.com/unfoldedcircle/integration-appletv/main/CHANGELOG.md))
- **Control4:** the pyatv 0.17.0 changelog fixes a link for `c4-pyatv-remote`, a Control4 driver built on pyatv ([CHANGES.md](https://raw.githubusercontent.com/postlund/pyatv/master/CHANGES.md)).
- **Music Assistant** uses pyatv for Apple TV and pins srptools to work around pyatv#2934 ([PR #7352](https://github.com/music-assistant/server/pull/7352)).
- **Homey:** `basmilius/homey-apple` (MIT). The author started the TypeScript rewrite "to run it on a Homey Pro without a Python runtime" ([apple-protocols](https://github.com/basmilius/apple-protocols); [homey-apple](https://github.com/basmilius/homey-apple)).

### Inferences
- The integration changelogs (UC, HA) work as a regression tracker. "Buttons stop responding → reconnect", "Companion unavailable → retry" and "wake fails on stale connection" are the same robustness problems an Android remote has to handle.
- The Android app will need the same features these integrations exposed: cross-VLAN or unicast connect, reconnect-on-silence, and power-state fallbacks.

### Gaps
- I found nothing on Hubitat or openHAB Apple TV drivers in this session.
- I found no Google Home or SmartThings native Apple TV control.

---

## Q6. Matter "Casting" / Video Player support on Apple TV as an official control path

### Takeaway
I found no evidence that tvOS exposes Apple TV as a Matter Casting Video Player or Basic Video Player (Keypad Input, Media Playback or Content Launcher clusters). Matter Casting has been adopted mainly by Amazon (Fire TV, Echo Show 15, Prime Video), and coverage says Apple and Google have been reluctant. Matter is therefore not a viable control path for Apple TV as of November 2026.

### Cited Findings
- The Matter "Casting Video Player" device type (ID 0x0023) has mandatory server clusters including On/Off, Media Playback, Keypad Input and Content Launcher ([Matter Survey](https://matter-survey.org/device-types/35)). The Keypad Input cluster's main command is `SendKey`, with keys such as UP, DOWN and SELECT ([Matter Survey cluster 0x0509](https://matter-survey.org/cluster/0x0509/version/1.1)).
- Amazon implements Matter Casting on Fire TV (FireOS 7+) with Prime Video and other apps ([Amazon Developer](https://developer.amazon.com/docs/fire-tv/overview-of-matter-casting.html); [AFTVnews](https://www.aftvnews.com/?p=42406)).
- Coverage says Apple and Google "are unlikely to embrace Matter casting" at the system level, or that this "is yet to be seen" ([Trusted Reviews](https://www.trustedreviews.com/news/matter-tv-casting-tech-could-replace-airplay-and-google-cast-4190428); [iClarified, 2026-04-05](https://www.iclarified.com/83752/matter-smart-home-standard-could-bring-universal-casting-for-tvs)). Several of these sources are from 2021–2024 and may be out of date.

### Inferences
- An Android app should not plan on Matter for Apple TV control. If Apple ever adds Matter Casting Video Player support to tvOS, an Android Matter controller (via the Matter Casting SDK) could send `SendKey` and `LaunchContent` officially. That is worth monitoring but not worth building on.

### Gaps
- I found no tvOS 26 or 27 release note or Apple developer document confirming or denying Matter media clusters on Apple TV. A direct check of Apple's tvOS 27 release notes was outside the budget.
- Whether a 4th-generation Apple TV 4K exists and what it changes is unconfirmed. The UC changelog (2026-11-14) mentions its discovery ([UC CHANGELOG](https://raw.githubusercontent.com/unfoldedcircle/integration-appletv/main/CHANGELOG.md)). One unverified Italian source claims pre-orders at $299 with an A19 chip, shipping 2026-11-13, while most English coverage at tvOS 27 launch described it as still unreleased ([pasqualepillitteri.it](https://pasqualepillitteri.it/en/news/16378/tvos-27-apple-tv-features); [What Hi-Fi](https://whathifi.com/streaming-entertainment/streaming-hardware/apple-officially-announces-tvos-27-and-it-nearly-makes-up-for-the-missing-next-generation-apple-tv-4k)).
