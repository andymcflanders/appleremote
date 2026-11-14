# Android implementation options for an Apple TV remote app (Companion Link / MRP / AirPlay), as of November 2026

Research date: 2026-11-14. Scope: Android-side architecture, libraries, platform constraints and UX integration. The Apple protocols themselves, the feature catalog and the survey of existing apps are covered by other researchers. Open-source Android projects are cited here only for the engineering choices they made.

Method note: besides web sources, I shallow-cloned and read the source of Chaquopy (master at 2026-11-10), tink-java (main at 2026-11-13), corvofeng/atv-core (2026-10-07) and four open-source Kotlin Apple TV remotes (dvdsgl/tv-remote at 2026-10-28, msthind04/AppleTV-Remote at 2026-09-13, gavinsan33/tvremote at 2026-10-20, heNch1g/atv_remote_android at 2026-10-28). I also queried PyPI's JSON API live for wheel availability. Findings cited to GitHub files are what those files said on those dates. chaquo.com (the Chaquopy package index site) could not be reached from this session, so Chaquopy package availability comes from the recipe directory in its GitHub repo.

## 1. Architecture options and trade-offs (native Kotlin vs Chaquopy+pyatv vs Rust/Go core vs Kotlin Multiplatform)

### Takeaway
A native Kotlin implementation is now a proven path. At least four open-source Android apps (all updated Sept–Oct 2026) implement Companion Link pairing, the encrypted session, OPACK and (in two of them) MRP-over-AirPlay now-playing in plain Kotlin, at around 26–33 minSdk, with no Python or native code. Embedding pyatv through Chaquopy is technically possible but blocked in practice for pyatv 0.18.0 (June 2026): the pinned `cryptography`, `pydantic-core` and `zeroconf` have no ready Android wheels, and Chaquopy 17 no longer builds sdists. A Rust core via JNI has a precedent only on the receiver side (atv-core). I found no Go/gomobile precedent. KMP is a cheap add-on if the protocol core is written as a pure Kotlin/JVM module, but it adds little for an Android-only app.

### Cited Findings

**(a) Native Kotlin: precedents and how they are built**
- dvdsgl/tv-remote describes itself as "a dependency-free Kotlin port of the parts of pyatv 0.18 it needs: OPACK, TLV8, HAP pair-setup and pair-verify (SRP-6a, X25519, Ed25519), and the ChaCha20-Poly1305 encrypted Companion session. Its only permission is INTERNET." It requires Android 13 or later (minSdk 33, targetSdk 36). It ships test tooling that "run[s] pyatv itself to produce test vectors and a fake Apple TV for the end-to-end test." — [dvdsgl/tv-remote README](https://github.com/dvdsgl/tv-remote/blob/main/README.md); [build.gradle.kts](https://github.com/dvdsgl/tv-remote/blob/main/android/app/build.gradle.kts)
- msthind04/AppleTV-Remote is a "native Kotlin implementation of Apple's Companion Link protocol – no server or bridge required" (minSdk 26, targetSdk 35). It is split into a pure-Kotlin `protocol` module (companion, hap, crypto, opack, plist, mrp, airplay, discovery packages), an Android `app` module and a desktop `cli` module. Dependencies: `org.bouncycastle:bcprov-jdk18on:1.80`, kotlinx-coroutines, and `org.jmdns:jmdns:3.5.9`, which is used only by the desktop CLI; the Android app uses NsdManager. — [msthind04/AppleTV-Remote](https://github.com/msthind04/AppleTV-Remote); [cli JmdnsDiscovery.kt](https://github.com/msthind04/AppleTV-Remote/blob/main/cli/src/main/kotlin/dev/atvremote/cli/JmdnsDiscovery.kt)
- gavinsan33/tvremote has "a plain-Kotlin core module with HomeKit pair-setup/pair-verify, encrypted framing, OPACK, and remote commands", plus a CLI. It uses `bcprov-jdk18on:1.78.1` (minSdk 29, targetSdk 35). — [gavinsan33/tvremote](https://github.com/gavinsan33/tvremote)
- heNch1g/atv_remote_android (minSdk 26, BouncyCastle 1.80, Russian README) has the same package layout and file names as msthind04 (protocol/mrp/Protobuf.kt, airplay/Ap2Session.kt, NowPlayingService.kt). It appears to be a derivative. iamclements/puck is an explicit fork of msthind04 that adds multi-device support. — [heNch1g/atv_remote_android](https://github.com/heNch1g/atv_remote_android); [iamclements/puck](https://github.com/iamclements/puck)

**(b) Chaquopy + pyatv**
- pyatv 0.18.0 (uploaded 2026-06-19) is MIT-licensed and requires Python >=3.9. Its dependencies are `aiohttp>=3.8.3,<5`, `cryptography>=44.0.1`, `chacha20poly1305-reuseable>=0.13.2`, `ifaddr`, `miniaudio>=1.45`, `protobuf>=6.31.1`, `pydantic>=2.0.0`, `requests`, `srptools>=0.2.0`, `tabulate`, `tinytag` and `zeroconf>=0.129.0`. — [PyPI pyatv JSON](https://pypi.org/pypi/pyatv/json) / [pyatv on PyPI](https://pypi.org/project/pyatv/)
- Chaquopy is MIT-licensed ("Copyright (c) 2017-2025 Chaquo Ltd and contributors"). The current version is 17.2.x: 17.2.0 shipped 2026-10-26 and the repo's VERSION.txt reads 17.2.1. — [Chaquopy LICENSE.txt](https://github.com/chaquo/chaquopy/blob/master/LICENSE.txt); [changelog](https://github.com/chaquo/chaquopy/blob/master/product/runtime/docs/sphinx/changelog.rst)
- Chaquopy runtime Python versions are 3.10.22, 3.11.17, 3.12.15, 3.13.16, 3.14.8 and 3.15.0. The default is 3.10, and COMPILE_SDK_VERSION is "37.0". — [Common.java](https://github.com/chaquo/chaquopy/blob/master/product/buildSrc/src/main/java/com/chaquo/python/internal/Common.java)
- Chaquopy 17.0.0 (2025-12-01) changes: "Pip now uses the `--only-binary` option, so it will no longer install sdists from PyPI or other indexes. Installing a local path to an sdist or source directory is still possible, as long as it's a pure-Python package." Also: "Devices with 16 KB pages are now supported by Chaquopy itself. However, many existing Android wheels will still fail to load on 16 KB devices. For best compatibility with these devices, use Python 3.13 or later," and Python 3.14 "currently has very few Android wheels available." — [Chaquopy changelog](https://github.com/chaquo/chaquopy/blob/master/product/runtime/docs/sphinx/changelog.rst)
- Chaquopy 17.1.0 (2026-10-12): ".so files are now made read-only before loading them, in case this becomes compulsory in future versions of Android." — [Chaquopy changelog](https://github.com/chaquo/chaquopy/blob/master/product/runtime/docs/sphinx/changelog.rst). It did become compulsory: for apps targeting Android 17, "All native files loaded using `System.load()` must be marked as read-only. Otherwise, the system throws `UnsatisfiedLinkError`." — [Android 17 behavior changes](https://developer.android.com/about/versions/17/behavior-changes-17)
- Chaquopy needs minSdk ≥ 24. ABIs are arm64-v8a and x86_64; armeabi-v7a and x86 are only available for Python 3.11 and older. "Each ABI will add several MB to the size of the app, plus the size of any native requirements." The FAQ says APK splits and app bundles "won't help much" because of how Chaquopy packages native code, and recommends ABI product flavors instead. — [Chaquopy android.rst](https://github.com/chaquo/chaquopy/blob/master/product/runtime/docs/sphinx/android.rst); [faq.rst](https://github.com/chaquo/chaquopy/blob/master/product/runtime/docs/sphinx/faq.rst)
- Startup: the app must call `Python.start()`, either in `Application.onCreate()` (or via `PyApplication`) or lazily before first use. The docs give no startup-time numbers. — [Chaquopy android.rst](https://github.com/chaquo/chaquopy/blob/master/product/runtime/docs/sphinx/android.rst)
- Chaquopy's native-package recipes include cryptography **42.0.8** (built with Rust and OpenSSL), cffi 1.17.1, aiohttp 3.10.10, yarl 1.16.0, multidict 6.1.0, frozenlist 1.4.1 and miniaudio 1.61. There is **no recipe for pydantic-core, zeroconf or protobuf.** — [chaquopy/server/pypi/packages](https://github.com/chaquo/chaquopy/tree/master/server/pypi/packages); [cryptography meta.yaml](https://github.com/chaquo/chaquopy/blob/master/server/pypi/packages/cryptography/meta.yaml)
- Live PyPI check on 2026-11-14:
  - aiohttp 3.14.5 publishes official Android wheels (`cp313/cp314-android_24_arm64_v8a` and `x86_64`).
  - cryptography 50.0.2, pydantic-core 2.50.4, zeroconf 0.154.0 and miniaudio 1.73 publish no Android wheels. zeroconf 0.154.0 has 71 platform wheels but no pure-Python (`py3-none-any`) wheel.
  - protobuf 7.36.2, propcache 0.5.4, srptools 2.0.0, chacha20poly1305-reuseable 0.13.2, ifaddr 0.2.0, tinytag 2.3.2 and tabulate 0.10.0 have pure-Python wheels.
  - Sources: [aiohttp](https://pypi.org/project/aiohttp/), [cryptography](https://pypi.org/project/cryptography/), [pydantic-core](https://pypi.org/project/pydantic-core/), [zeroconf](https://pypi.org/project/zeroconf/), [protobuf](https://pypi.org/project/protobuf/), [srptools](https://pypi.org/project/srptools/)
- I found no public example of an Android app shipping pyatv through Chaquopy. A web search for "Chaquopy pyatv" returned nothing relevant. One Android project cites pyatv only as a porting reference ("follows pyatv (MIT) for the Companion protocol and OPACK"). — [gavinsan33/tvremote](https://github.com/gavinsan33/tvremote)

**(c) Rust / Go core**
- corvofeng/atv-core (MIT, last commit 2026-10-07) is a Rust workspace (`atv-core`, `atv-cli`, `atv-android`). It implements Companion Link and MRP on the **receiver** side ("Fake Apple TV": it lets the iPhone's built-in Apple TV Remote control an Android TV or Mac).
  - The Android crate is a `cdylib` built on `jni = "0.21"` and `tokio`, loaded with `System.loadLibrary("atv_android")` and Kotlin `external fun` declarations (no uniffi).
  - The release profile is tuned for size: `opt-level = "z"`, `lto = true`, `panic = "abort"`, `strip = true`.
  - Sources: [atv-core Cargo.toml](https://github.com/corvofeng/atv-core/blob/main/Cargo.toml); [atv-android Cargo.toml](https://github.com/corvofeng/atv-core/blob/main/crates/atv-android/Cargo.toml); [README](https://github.com/corvofeng/atv-core/blob/main/README.md)
- A search for a Rust crate implementing the Apple TV **controller** side of Companion/MRP pairing found no published crates.io library. Rust's `hap-crypto` crate implements HomeKit SRP (3072-bit group, g=5, k = H(N | PAD(g))). — [hap-crypto srp.rs](https://docs.rs/crate/hap-crypto/1.1.0/source/src/srp.rs)
- Non-JVM reference implementations exist: a .NET AppleTvControlLibrary (Companion Link and MRP with shared HAP code) and Node/TypeScript libraries (node-appletv-remote, BunATV). — [AppleTvControlLibrary on NuGet](https://www.nuget.org/packages/AppleTVControlLibrary)

**(d) Kotlin Multiplatform**
- msthind04 and gavinsan33 both keep the protocol in a platform-neutral Kotlin/JVM module that also runs as a desktop CLI. This is a JVM split, not KMP, but it shows the protocol layer has no Android dependencies. — [msthind04/AppleTV-Remote](https://github.com/msthind04/AppleTV-Remote); [gavinsan33/tvremote](https://github.com/gavinsan33/tvremote)
- whyoleg/cryptography-kotlin 0.6.0 added EdDSA (Ed25519/Ed448) and XDH (X25519/X448) on the JDK, OpenSSL, CryptoKit and WebCrypto providers, and ChaCha20-Poly1305 on JDK, OpenSSL and CryptoKit. The library delegates to platform libraries (JCA, OpenSSL, CryptoKit, WebCrypto) and does not implement primitives itself. — [cryptography-kotlin 0.6.0 release](https://newreleases.io/project/github/whyoleg/cryptography-kotlin/release/0.6.0); [cryptography-kotlin repo](https://www.github.com/whyoleg/cryptography-kotlin)

### Inferences
- **Recommended architecture: (a) native Kotlin**, structured as a pure-Kotlin/JVM `protocol` module (or a KMP `commonMain` module) plus an Android app module. This matches every working Android precedent, keeps the APK small (no CPython or Rust runtime), avoids the 16 KB-page and read-only-.so pitfalls, and lets the protocol be unit-tested on the JVM. dvdsgl's approach of generating golden test vectors from pyatv and running a fake Apple TV is worth copying.
- **Chaquopy + pyatv (b)** would need custom Android wheels for `cryptography>=44` (Rust + OpenSSL cross-build) and `pydantic-core` (Rust/PyO3), plus a locally built pure-Python `zeroconf` wheel. Alternatively, pyatv's dependency pins would have to be patched or vendored. Python 3.13+ is needed for 16 KB-page devices, and Chaquopy's index has few wheels for 3.13+. Expect a size penalty of "several MB" per ABI before packages, plus pyatv's own native dependencies. The approach makes sense only as a prototype or a desktop-parity tool. If used, discovery should be done with Android's NsdManager and pyatv given explicit hosts, because zeroconf's raw multicast sockets need a multicast lock on older devices and `ACCESS_LOCAL_NETWORK` under Android 17 (see section 4).
- **Rust (c)** is viable: atv-core shows a tokio + jni-rs cdylib running Companion/MRP code on Android. But a controller-side Rust library does not exist yet, so this route means writing the protocol from scratch in Rust. It only pays off if the same core must also ship on iOS, desktop or Wear. uniffi would be the more ergonomic binding generator; atv-core uses hand-written JNI.
- **KMP (d)** costs little if the protocol module is written without java.* APIs, using cryptography-kotlin, or Tink/BouncyCastle behind `expect`/`actual`. Its main value would be a desktop or iOS companion, where Apple's own Remote already exists.

### Gaps
- No measured startup time or APK size for a Chaquopy + pyatv build. The Chaquopy docs give only "several MB per ABI".
- Not verified whether pyatv's asyncio event loop, aiohttp server components or `miniaudio` (used for AirPlay audio streaming) run cleanly under Chaquopy. No public report found.
- Not verified whether zeroconf's sdist builds a working pure-Python wheel (no Cython) for local installation into Chaquopy.
- No Go/gomobile Apple TV controller precedent found.

## 2. Crypto building blocks on Android (SRP-6a 3072/SHA-512, Ed25519, X25519, HKDF-SHA512, ChaCha20-Poly1305)

### Takeaway
Platform JCA (Conscrypt) covers SHA-512, HMAC-SHA512 (for HKDF), and ChaCha20-Poly1305 with a 96-bit nonce since API 28. X25519 (`XDH`) and `Ed25519` appear in the JCA tables only from API 33. Importing raw 32-byte Curve25519 keys through JCA is awkward or undocumented, so every Android precedent uses either BouncyCastle's lightweight API (bcprov-jdk18on 1.78–1.80) or a hand-written implementation. Google Tink's `subtle` classes (`X25519`, `Ed25519Sign/Verify`, `Hkdf`) take raw bytes and are a good alternative. **No precedent uses a library SRP:** all four apps hand-roll HomeKit-flavoured SRP-6a with `java.math.BigInteger` (about 100 lines), because HAP's padding and M1 rules differ from Nimbus's and BouncyCastle's defaults.

### Cited Findings
- **Android JCA tables:**
  - `KeyAgreement`: DH 1+, ECDH 11+, **XDH 33+**. — [KeyAgreement reference](https://developer.android.com/reference/javax/crypto/KeyAgreement)
  - `Signature`: **Ed25519 33+**. — [Signature reference](https://developer.android.com/reference/java/security/Signature)
  - `KeyPairGenerator`: XDH 33+. — [KeyPairGenerator reference](https://developer.android.google.cn/reference/java/security/KeyPairGenerator)
  - `Cipher`: "ChaCha20 | NONE Poly1305 | NoPadding | **28+** | ChaCha with 20 rounds, 96-bit nonce, and 32-bit counter as described in RFC 7539." AES/GCM is 10+. — [Cipher reference](https://developer.android.com/reference/javax/crypto/Cipher)
- **Conflicting claim on Ed25519 at API 33:** dvdsgl's Curve25519.kt says "Android 13's providers don't document importing raw 32-byte X25519 or Ed25519 keys (Ed25519 signing isn't listed at all), so these are done here instead." Its X25519/Ed25519 are written with BigInteger, "slow next to native code (a few milliseconds per operation) and not constant time." This conflicts with the current Signature table, which lists Ed25519 at 33+; the table may have been updated after the author checked. The raw-key import gap is consistent with the docs: nothing documents raw import. — [dvdsgl Curve25519.kt](https://github.com/dvdsgl/tv-remote/blob/main/android/app/src/main/java/xyz/dvdsgl/tvremote/companion/Curve25519.kt) vs [Signature reference](https://developer.android.com/reference/java/security/Signature)
- **JCA naming difference:** dvdsgl uses JCA for SHA-512, HMAC-SHA512-based HKDF and ChaCha20-Poly1305. It notes "Android names the cipher 'ChaCha20/Poly1305/NoPadding'; the JDK calls it 'ChaCha20-Poly1305'". It creates "a fresh Cipher per message: the JDK refuses to re-init one with the same key and nonce." Its HKDF is a hand-written extract-and-expand over `Mac("HmacSHA512")`, and its pairing nonces are "four zero bytes, then the label" (e.g. "PS-Msg05"). — [dvdsgl Crypto.kt](https://github.com/dvdsgl/tv-remote/blob/main/android/app/src/main/java/xyz/dvdsgl/tvremote/companion/Crypto.kt)
- **BouncyCastle in the other precedents:** msthind04 and heNch1g use BouncyCastle's lightweight classes `X25519PrivateKeyParameters`, `Ed25519Signer`/`Ed25519PrivateKeyParameters`, `HKDFBytesGenerator`/`HKDFParameters` and `ChaCha20Poly1305` (bcprov-jdk18on 1.80). gavinsan33 uses the same BC HKDF and ChaCha20Poly1305 classes (BC 1.78.1). — source code of [msthind04](https://github.com/msthind04/AppleTV-Remote), [heNch1g](https://github.com/heNch1g/atv_remote_android), [gavinsan33](https://github.com/gavinsan33/tvremote)
- **Tink (Java, main branch 2026-11-13):**
  - `com.google.crypto.tink.subtle` has `X25519` (with `computeSharedSecret(byte[] privateKey, byte[] peersPublicValue)` and `publicFromPrivate(byte[])`), `Ed25519Sign(byte[] privateKey)`, `Ed25519Verify`, `Hkdf.computeHkdf(...)` and `ChaCha20Poly1305`.
  - A caller-supplied-nonce AEAD, `InsecureNonceChaCha20Poly1305` ("@since 1.1.0", RFC 8439), is in the **internal** package `com.google.crypto.tink.aead.internal`.
  - Source: [tink-java subtle](https://github.com/tink-crypto/tink-java/tree/main/src/main/java/com/google/crypto/tink/subtle); [aead/internal](https://github.com/tink-crypto/tink-java/tree/main/src/main/java/com/google/crypto/tink/aead/internal)
- **Tink Ed25519 history:** a 2020 paper described a key-extraction bug in Tink Java's Ed25519 (misuse of `final`) that was fixed before release. — [arXiv 2004.01403](https://arxiv.org/pdf/2004.01403)
- **SRP is hand-rolled everywhere:** all four Android projects implement SRP themselves (`class Srp` in dvdsgl, msthind04 and heNch1g; `SrpClient` in gavinsan33). msthind04 documents the HAP-specific encoding:
  - "`k` and `u` hash operands zero-padded to the full 384-byte modulus length."
  - "The integers in the M1/M2 proofs use *minimal-length* big-endian encoding."
  - "The salt is hashed as the raw bytes the device sent."
  - M1 = H(H(N) XOR H(g) | H(I) | s | A | B | K), and x = H(s | H(I ":" P)).
  - "This mirrors the encoding the reference Python implementation produces, which is what Apple devices actually interoperate with."
  - Source: [msthind04 Srp.kt](https://github.com/msthind04/AppleTV-Remote/blob/main/protocol/src/main/kotlin/dev/atvremote/protocol/crypto/Srp.kt); [dvdsgl Srp.kt](https://github.com/dvdsgl/tv-remote/blob/main/android/app/src/main/java/xyz/dvdsgl/tvremote/companion/Srp.kt)
- **Library SRP options:**
  - HAP uses the RFC 5054 3072-bit group with SHA-512, and "the match, M, hash calculation is not padded". — [Crystal secure-remote-password docs](https://crystaldoc.info/github/spider-gazelle/secure-remote-password)
  - A BouncyCastle issue reports its SRP "is working only when BouncyCastle is being used on both sides". — [bc-csharp issue #506](https://www.github.com/bcgit/bc-csharp/issues/506)
  - The Nimbus SRP tools page shows only SHA-1 examples and no HAP configuration. — [Nimbus SRP tools](https://connect2id.com/products/nimbus-srp/tools)

### Inferences
- **Recommended crypto stack (minSdk below 33):**
  - JCA for SHA-512, HMAC-SHA512/HKDF and ChaCha20-Poly1305 (API 28+). Below 28, use BC's or Tink's ChaCha20-Poly1305.
  - Tink `subtle` or BouncyCastle's lightweight API for X25519 and Ed25519 on raw 32-byte keys.
  - A small, test-vector-verified hand-written SRP-6a client.
  - Tink is smaller and Google-maintained, but its fixed-nonce ChaCha20-Poly1305 is internal API. Use JCA for that piece, or BC.
  - BouncyCastle (bcprov-jdk18on) is large but has everything, and three precedents prove it works. Use its lightweight classes directly rather than registering it as a JCA provider, which avoids clashing with Android's platform BC remnants. This is the conventional practice but was not verified here.
- **At minSdk 33+**, JCA XDH/Ed25519 work if raw keys are wrapped in X.509/PKCS#8 DER with the fixed 12/16-byte prefixes. dvdsgl chose not to rely on this, so test it on several OEMs before depending on it.
- Do not try Nimbus SRP or BC `SRP6Client` without customising the evidence routines and padding. The HAP variant differs from both libraries' defaults. Hand-rolling with pyatv-generated test vectors is lower risk.

### Gaps
- lazysodium-android: maintenance status and 16 KB-page compatibility were not researched.
- No authoritative Android documentation found on importing raw X25519/Ed25519 keys via `XECPublicKeySpec`/`EdECPublicKeySpec`, or on the API level of those spec classes.
- HAP-Java (server side) SRP internals were not checked.

## 3. Serialization: protobuf (MRP), TLV8, OPACK on the JVM

### Takeaway
None of the precedents uses a protobuf code generator. msthind04 parses MRP with a small hand-written protobuf wire codec, reading fields by number, to avoid protoc in the Android build and to tolerate unknown fields. OPACK and TLV8 are hand-written in every Kotlin project (an `object Opack` in all four). There is no standalone JVM OPACK library, but there are four MIT/open Kotlin implementations to learn from.

### Cited Findings
- msthind04's `mrp/Protobuf.kt` is "a generic wire-format codec rather than generated sources. MRP defines dozens of message types and now-playing touches about six fields across four of them, so reading the wire format directly avoids putting protoc into an Android build and tolerates unknown fields from newer tvOS revisions." — [msthind04 README](https://github.com/msthind04/AppleTV-Remote/blob/main/README.md); [Protobuf.kt](https://github.com/msthind04/AppleTV-Remote/blob/main/protocol/src/main/kotlin/dev/atvremote/protocol/mrp/Protobuf.kt)
- msthind04 also lists MRP field-number pitfalls:
  - "`PlayerPath.client` is field 2, not 1".
  - "Elapsed time is `ContentItemMetadata.elapsedTime`, field 35, alongside `duration` at 14".
  - Track metadata is in `SetStateMessage.playbackQueue.contentItems`, not `nowPlayingInfo`.
  - It recommends checking field numbers "against pyatv's `.proto` definitions rather than inferring".
  - The AirPlay data channel "nests three framings: a 32-byte big-endian header, a binary plist, and `params.data` holding varint-length-prefixed protobufs".
  - Source: [msthind04 README](https://github.com/msthind04/AppleTV-Remote/blob/main/README.md)
- `object Opack` implementations exist in dvdsgl (`companion/Opack.kt`), msthind04 (`protocol/opack/Opack.kt`), gavinsan33 (`core/Opack.kt`, with `OpackTest`) and heNch1g. dvdsgl and msthind04 also include TLV8 and binary-plist code. — [dvdsgl Opack.kt](https://github.com/dvdsgl/tv-remote/blob/main/android/app/src/main/java/xyz/dvdsgl/tvremote/companion/Opack.kt); [msthind04 Opack.kt](https://github.com/msthind04/AppleTV-Remote/blob/main/protocol/src/main/kotlin/dev/atvremote/protocol/opack/Opack.kt); [gavinsan33 Opack.kt](https://github.com/gavinsan33/tvremote/blob/main/core/src/main/kotlin/tvremote/Opack.kt)
- A web search for a JVM/Kotlin OPACK library returned nothing relevant. — WebSearch, 2026-11-14 (no result to cite)

### Inferences
- For the full MRP surface (dozens of messages), code generation from pyatv's `.proto` files with **protobuf-javalite** (Google's recommended Android runtime) or **Square Wire** (Kotlin-first, small runtime, good for KMP) is more maintainable than a hand-written codec. For a now-playing-only feature set, the hand-written approach is proven.
- OPACK, TLV8 and binary plist are each a few hundred lines. Write them in the shared protocol module and test them against pyatv-generated vectors, as dvdsgl does.
- Binary plist on Android: there is no platform API, so either hand-roll it (as msthind04 does) or use a JVM library such as dd-plist. This was not verified for current maintenance.

### Gaps
- No current (2026) figures on protobuf-javalite vs Wire method count or size were gathered.
- Not verified whether pyatv's `.proto` files compile cleanly under protobuf-javalite or Wire without changes.

## 4. Discovery on Android: NsdManager, MulticastLock, jmDNS, and the Android 17 local network permission

### Takeaway
Use **NsdManager**.
- Since "T extensions 22" (all Android 14+ devices), `registerServiceInfoCallback` combines discovery and resolution and pushes TXT, address and loss updates.
- Since T extensions 7, foreground apps no longer need a `MulticastLock`.
- On Android 13 and below, resolve one service at a time and hold a MulticastLock while discovering.

**The big 2026 change is Android 17's `ACCESS_LOCAL_NETWORK`.** It is a runtime permission in the NEARBY_DEVICES group, mandatory for apps that target API 37. It gates *all* LAN traffic: TCP, UDP and multicast. NsdManager's new system picker (`DiscoveryRequest.FLAG_SHOW_PICKER`) lets an app reach one user-chosen device without the permission. jmDNS or other raw-multicast libraries are now strictly worse on Android: they need the multicast lock, cost battery, and need the permission under API 37.

### Cited Findings
- **NsdManager evolves through SDK extensions:**
  - The `NsdManager` API "is updated via SDK extensions… Applications should generally use SDK extension version checks instead of SDK version checks."
  - `registerServiceInfoCallback(DiscoveryRequest…)` is "available starting from 'T extensions 22' (which covers all Android 14+ devices)" and "combines discovery and resolution into a single operation. It automatically notifies the app when a service is found, when its properties (like IP addresses or TXT records) change, and when it becomes unavailable."
  - Source: [NsdManager reference](https://developer.android.com/reference/android/net/nsd/NsdManager)
- **Multicast lock:** "Before T extensions 7 (Android 12 and below devices, and Android 13 devices that have not received the T extensions 7 update): Apps must manually acquire a WifiManager.MulticastLock to receive mDNS packets, even when the app is in the foreground. Starting from T extensions 7: The system automatically manages multicast reception for apps in the foreground. Background apps should still avoid taking the lock unless absolutely necessary to minimize battery impact." — [NsdManager reference](https://developer.android.com/reference/android/net/nsd/NsdManager)
- **MulticastLock battery cost:** holding the lock makes the Wi-Fi stack pass multicast packets through, which "can cause a noticeable battery drain", so it should be released when not needed. Community jmDNS guidance is to declare `CHANGE_WIFI_MULTICAST_STATE` and acquire the lock before `JmDNS.create()`. — [WifiManager.MulticastLock reference](https://developer.android.com/reference/android/net/wifi/WifiManager.MulticastLock); [Stack Overflow archive: jmDNS doesn't discover devices](https://kiwix.lema.org/stackoverflow.com_en_all_2023-11/questions/23805893/android-jmdns-doesn-t-discover-devices)
- **NsdServiceInfo (API 34 changes):**
  - `getHostAddresses()` was added in API 34 ("all host addresses are valid for the resolved service. All addresses share the same port"), and `getHost()` was deprecated.
  - TXT attributes are "only valid for a resolved service"; key + value must be under 255 bytes.
  - A ServiceInfo from the deprecated `resolveService` "may go stale and may not contain some information that could be delivered later, like additional host addresses."
  - Source: [NsdServiceInfo reference](https://developer.android.com/reference/android/net/nsd/NsdServiceInfo); [NsdManager.ServiceInfoCallback (API 34)](https://developer.android.com/reference/android/net/nsd/NsdManager.ServiceInfoCallback)
- **Pre-API 34 pattern:** dvdsgl's Discovery.kt discovers `_companion-link._tcp`.
  - On API 34+ it calls `registerServiceInfoCallback` per found service and prefers an IPv4 address from `hostAddresses`.
  - Otherwise it queues services and resolves them serially: "Android 13 resolves one service at a time".
  - It filters out HomePods (which also advertise Companion but cannot pair) by model "AppleTV*".
  - NsdManager defines `FAILURE_ALREADY_ACTIVE` and `FAILURE_MAX_LIMIT` (the latter "maximum outstanding requests from the applications have reached").
  - Source: [dvdsgl Discovery.kt](https://github.com/dvdsgl/tv-remote/blob/main/android/app/src/main/java/xyz/dvdsgl/tvremote/Discovery.kt); [NsdManager reference](https://developer.android.com/reference/android/net/nsd/NsdManager)
- **Android 17 local network permission:** "Starting in Android 17 (API level 37), local network protections are enforced for apps targeting Android 17 or higher."
  - The permission is `ACCESS_LOCAL_NETWORK`, in the NEARBY_DEVICES group ("If a user has already granted another permission in this group (such as Bluetooth permissions), they won't be re-prompted").
  - Outgoing TCP, incoming TCP, and sending or receiving UDP unicast, multicast or broadcast all require it. "These restrictions are implemented deep in the networking stack, and thus they apply to **all networking APIs**… sockets created in the platform or managed code, networking libraries like Cronet and OkHttp."
  - "Trying to resolve services on the local network that have a `.local` suffix requires local network permission."
  - Exceptions: the local DNS server on port 53, and Output Switcher for casting.
  - Source: [Local network permission](https://developer.android.com/privacy-and-security/local-network-permission)
- **Split-permission model:** apps targeting below 37 "with INTERNET permission receive an implicit permission grant… Don't request `ACCESS_LOCAL_NETWORK` at runtime prior to targeting SDK 37." When denied, "TCP Connections will typically result in a **timeout error**" and UDP gives `EPERM`; NDK `android_getnetworkblockedreason()` returns `ANDROID_NETWORK_BLOCKED_REASON_LNP`. A "permission request reset counter strategy" gives apps extra chances to re-request after an earlier NEARBY_DEVICES denial. — [Local network permission](https://developer.android.com/privacy-and-security/local-network-permission)
- **Picker path (no permission):** `DiscoveryRequest.Builder(type).setFlags(DiscoveryRequest.FLAG_SHOW_PICKER)` with `registerServiceInfoCallback` shows a system dialog. "Once the user selects a service, the app is granted permission to communicate with that specific device through `getHostAddresses()` on `getNetwork()`." Related APIs:
  - `FLAG_USER_APPROVED_ONLY` rediscovers previously approved services without the UI.
  - `checkPermissionForService(...)` checks that a grant still exists ("access may be revoked for storage reasons or by the user").
  - `FAILURE_PERMISSION_DENIED` (API 37, also T Extensions 22) is returned when resolving or discovering without the permission.
  - Sources: [NsdManager reference](https://developer.android.com/reference/android/net/nsd/NsdManager); [Local network permission](https://developer.android.com/privacy-and-security/local-network-permission)
- **Android 16 opt-in (deprecated):** the Android 16 test mode used the `RESTRICT_LOCAL_NETWORK` compat flag, with `NEARBY_WIFI_DEVICES` as the temporary grant. During that opt-in, out-of-process APIs such as NsdManager were not affected. This guidance is now deprecated. — [Local network permission](https://developer.android.com/privacy-and-security/local-network-permission)
- **Dates and ecosystem:** the Android 17 security/privacy features were reported in June 2026 — [Help Net Security, 2026-06-17](https://www.helpnetsecurity.com/2026/06/17/android-17-security-and-privacy-features/). A July 2026 Kodi nightly targeting API 37 broke because of a missing permission declaration (reported in search results; primary source not fetched). Flutter documents `Permission.accessLocalNetwork.request()`. — [Flutter: local network permission](https://docs.flutter.dev/platform-integration/android/local-network-permission)
- **Conflicting permission claim:** msthind04's README says it requests `NEARBY_WIFI_DEVICES`, "required for local service discovery on Android 13+". The official NsdManager docs list no such requirement for NsdManager. The only documented role of NEARBY_WIFI_DEVICES here is the deprecated Android 16 opt-in. — [msthind04 README](https://github.com/msthind04/AppleTV-Remote/blob/main/README.md) vs [NsdManager reference](https://developer.android.com/reference/android/net/nsd/NsdManager)
- **Play target API:** since 2026-08-31, Google Play requires new apps and updates to target Android 16 (API 36); an extension to 2026-11-01 was available on request. — [Play Console help: target API level](https://support.google.com/googleplay/android-developer/answer/11926878); [Target SDK requirements](https://developer.android.com/google/play/requirements/target-sdk)

### Inferences
- **What to declare and request today (targetSdk 36):** only `INTERNET` (plus `ACCESS_NETWORK_STATE` for connectivity callbacks). dvdsgl ships with INTERNET only. Do not declare or request `ACCESS_LOCAL_NETWORK` yet.
- **When moving to targetSdk 37:** declare `ACCESS_LOCAL_NETWORK` and request it at runtime before discovery or connection, with a rationale screen. Handle denial (TCP connects will hang until timeout, so use short connect timeouts and check the permission first).
  - Alternative: build onboarding on `FLAG_SHOW_PICKER` and reconnect with `FLAG_USER_APPROVED_ONLY`, so no permission prompt is needed.
  - Uncertainty: the picker grant is described per device (its host addresses on that network). Whether it covers every port on that Apple TV (Companion, AirPlay 7000, the rotating Companion port) needs device testing.
  - Given the Play cycle, API 37 will probably be mandatory around August 2027. This follows the annual pattern; no date has been published.
- Use `SdkExtensions.getExtensionVersion(Build.VERSION_CODES.TIRAMISU)` checks (≥22 for the callback API, ≥7 to skip the multicast lock) rather than `SDK_INT`, as the docs advise. dvdsgl uses `SDK_INT >= 34`, which is a safe approximation.
- Avoid jmDNS and raw zeroconf on Android: they need `CHANGE_WIFI_MULTICAST_STATE` plus a MulticastLock, drain battery, and run into OEM multicast filtering. They are fine for a desktop CLI built from the same protocol module (as msthind04 does).

### Gaps
- No systematic data on OEM-specific NsdManager bugs (e.g. Samsung or Xiaomi) on Android 14–17.
- Not confirmed whether the Android 17 picker grant covers all ports and services on the selected host, or only the selected service's address and port.
- Not verified which Android release fully replaced the mdnsresponder daemon with the in-process Java mDNS stack in the Connectivity module. Source fragments show both paths without release dates.

## 5. Networking and process lifetime: persistent TCP, foreground services, reconnects, Doze

### Takeaway
Keep the Companion/MRP sockets in a coroutine-driven connection manager. Tie them to a **foreground service only while a session is active** (needed for the lock-screen media notification and background volume). Expect the Apple TV to drop connections and **rotate its Companion port**: re-discover over mDNS before each reconnect, key credentials on a stable device ID, and send periodic keep-alives. On Android 14+, every foreground service must declare a type and a matching permission, and Play requires declaring the type in the Console. `connectedDevice` is the semantically correct type for a network remote. Existing apps use `mediaPlayback`.

### Cited Findings
- **Reconnect lessons from msthind04:**
  - "An Apple TV rotates the ephemeral port it advertises for Companion Link, even while it is awake and playing. When that happens the existing connection dies, and reconnecting to the cached port fails outright."
  - Mitigations: "Credentials are keyed on the device's stable `rpMRtID`, never on host:port… Reconnection rediscovers the device over mDNS first… An idle connection is kept warm with periodic `NoOp` frames, and a failed write is treated as a lost connection — the reader blocks in `readFully` and will not notice a peer that vanished without a clean shutdown."
  - "Commands transparently reconnect and retry once."
  - Source: [msthind04 README](https://github.com/msthind04/AppleTV-Remote/blob/main/README.md)
- **Foreground service design in msthind04:** the service "is the point rather than the notification. It is what keeps the process, and therefore the Companion and MRP connections, alive once the app leaves the screen." It runs for the whole connection "instead of only during playback: Android 12 forbids starting a foreground service from the background." Swiping the app from recents closes the connections; "surviving that would mean moving connection ownership into the service." — [msthind04 README](https://github.com/msthind04/AppleTV-Remote/blob/main/README.md)
- **FGS type requirement:** "Beginning with Android 14 (API level 34), you must declare an appropriate service type for each foreground service." — [Foreground service types](https://developer.android.com/develop/background-work/services/fgs/service-types)
  - `connectedDevice` needs `FOREGROUND_SERVICE_CONNECTED_DEVICE`. Runtime prerequisite: declare at least one of `CHANGE_NETWORK_STATE`, `CHANGE_WIFI_STATE`, `CHANGE_WIFI_MULTICAST_STATE`, `NFC` or `TRANSMIT_IR`, or hold a Bluetooth/UWB runtime permission or USB permission. Described use: "Interactions with external devices that require a Bluetooth, NFC, IR, USB, or network connection."
  - `mediaPlayback` needs `FOREGROUND_SERVICE_MEDIA_PLAYBACK`. Described use: "Continue audio or video playback from the background." Apps targeting 15+ cannot launch it from `BOOT_COMPLETED`.
  - `dataSync` cannot be launched from `BOOT_COMPLETED` on 15+, and Android 15 suggests alternatives. `shortService` is limited to about 3 minutes.
  - Play: "If your app targets Android 14 or higher, you'll need to declare your app's foreground service types in the Play Console's app content page."
- **What the precedents use:** msthind04 and heNch1g use `foregroundServiceType="mediaPlayback"` with `FOREGROUND_SERVICE_MEDIA_PLAYBACK` and `POST_NOTIFICATIONS`. dvdsgl and gavinsan33 use no foreground service. — manifests in [msthind04](https://github.com/msthind04/AppleTV-Remote), [heNch1g](https://github.com/heNch1g/atv_remote_android), [dvdsgl](https://github.com/dvdsgl/tv-remote), [gavinsan33](https://github.com/gavinsan33/tvremote)
- **Android 17 background audio hardening:** "the audio framework enforces restrictions on background audio interactions including audio playback, audio focus requests, and volume change APIs." Apps targeting 37 must have a foreground service with while-in-use capabilities to interact with audio from the background. — [Android 17 behavior changes](https://developer.android.com/about/versions/17/behavior-changes-17)
- **Android 17 activity-launch hardening:** Background Activity Launch hardening now extends to `IntentSender`; the legacy `MODE_BACKGROUND_ACTIVITY_START_ALLOWED` must be replaced with granular modes such as `..._ALLOW_IF_VISIBLE`. This is relevant to tiles, notifications and widgets that launch the remote UI. — [Android 17 behavior changes](https://developer.android.com/about/versions/17/behavior-changes-17)

### Inferences
- **Recommended service model:**
  - The connection lives in an app-scoped repository (not a ViewModel).
  - Start a `connectedDevice` foreground service (declare `CHANGE_NETWORK_STATE` or `CHANGE_WIFI_STATE` to satisfy the prerequisite) when the user connects while the app is visible.
  - Show a MediaStyle notification bound to a MediaSession.
  - Stop the service after an idle timeout or when the user disconnects.
  - Using `mediaPlayback` for a remote that plays no local audio carries Play-review risk under the FGS policy. Precedents use it, but whether Play accepts it is unverified.
- **Reconnect:** use `ConnectivityManager.NetworkCallback` for Wi-Fi changes, re-resolve via NsdManager on every reconnect (port rotation), apply exponential backoff with a cap, and only pair-verify again rather than pair-setup.
- **Doze and App Standby:** a foreground-service process normally keeps network access. Without a foreground service, sockets will be cut once the app is cached. The remote should be a "foreground-first" app that tolerates reconnects on resume.

### Gaps
- Official Doze documentation on socket behaviour for foreground-service processes on Android 15–17 was not fetched.
- No Play policy decision found on whether a network remote may use `mediaPlayback` versus `connectedDevice`.

## 6. Secure storage of pairing credentials

### Takeaway
`androidx.security:security-crypto` (EncryptedSharedPreferences, EncryptedFile, MasterKey) is **deprecated**: all APIs were deprecated in 1.1.0-alpha07 (April 2025) and the final stable 1.1.0 shipped July 2025, with the recommendation to use platform APIs and the Android Keystore directly. The current pattern is to wrap the long-term Ed25519 identity seed and the Apple TV's LTPK/pairing ID with an AES-GCM key held in Android Keystore (or via Tink's Android Keystore integration), stored in DataStore or files, with backup disabled for those files.

### Cited Findings
- security-crypto 1.1.0-alpha07 (April 9, 2025) and 1.1.0-beta01 (June 4, 2025): "Deprecated all APIs in favour of existing platform APIs and direct use of Android Keystore." Stable 1.1.0 was released July 30, 2025, and is still the latest; the page was updated September 2026 for other artifacts. — [androidx.security release notes](https://developer.android.com/jetpack/androidx/releases/security)
- msthind04's `SecureStore` uses an `AndroidKeyStore` AES key ("atv-credentials", `AES/GCM/NoPadding`, 12-byte IV, 128-bit tag) to encrypt pairing credentials. Its README says: "Pairing credentials are encrypted with an AES-GCM key held in the Android Keystore, which on most devices is non-extractable. Backups are disabled, since Keystore-wrapped ciphertext cannot be restored onto other hardware." — [SecureStore.kt](https://github.com/msthind04/AppleTV-Remote/blob/main/app/src/main/kotlin/dev/atvremote/app/SecureStore.kt); [README](https://github.com/msthind04/AppleTV-Remote/blob/main/README.md)
- dvdsgl stores keys in plain `SharedPreferences` ("Pair once… Keys stay on the phone"). — [dvdsgl README](https://github.com/dvdsgl/tv-remote/blob/main/README.md); source grep of SharedPreferences use
- **Keystore Ed25519/X25519:** AOSP code registers `KeyPairGenerator.ED25519` and `XDH` (x25519 only) in the AndroidKeyStore provider. This comes from an AOSP mirror, not official docs, and the API level is unconfirmed. — [e/OS AOSP mirror commit](https://gitlab.e.foundation/e/os/android_frameworks_base/-/commit/711efaa0297f845c60a1de0acce680493f90bf02)

### Inferences
- Generating the controller's Ed25519 identity **inside** Keystore is not workable for HAP, because pair-verify needs signatures over protocol data that the app must compute with the same key on all supported API levels. Keep the raw seed in app storage, encrypted with a Keystore-held AES-GCM key (or a Tink `AndroidKeysetManager` keyset wrapped by a Keystore master key), and store the blobs in DataStore.
- Exclude credential files from Auto Backup and device-to-device transfer (`dataExtractionRules`): a restored ciphertext cannot be decrypted, and the user would simply need to re-pair.

### Gaps
- Not verified whether Tink's `AndroidKeysetManager` (`tink-android`) is still the recommended replacement as of 2026. Official migration guidance beyond "use platform APIs/Keystore" was not found.

## 7. Android UX integration opportunities

### Takeaway
The highest-value integrations, all with precedents, are:
- hardware volume keys mapped to the Apple TV (in-app via `onKeyDown`, system-wide via a MediaSession with `setPlaybackToRemote`);
- a MediaSession + MediaStyle notification for Apple TV now-playing on the lock screen and shade;
- a Quick Settings tile (use the PendingIntent overload on API 34+);
- haptics on the touchpad through `HapticFeedbackConstants`;
- IME text relay that opens automatically when the TV asks for text, plus a direct-reply notification.

Device Controls (`ControlsProviderService`) still exists, but AOSP code signals it is being folded into Quick Settings, so it is not a priority. Glance widgets are viable (Glance 1.2.0 stable, Aug 2026), but Android 17 enforces a RemoteViews bitmap-memory cap. Wear OS was not researched.

### Cited Findings

**Volume keys**
- dvdsgl: "The phone's volume buttons control the TV while the app is open," implemented by handling `KEYCODE_VOLUME_UP`/`KEYCODE_VOLUME_DOWN` in `MainActivity.onKeyDown`. — [dvdsgl README](https://github.com/dvdsgl/tv-remote/blob/main/README.md); [MainActivity.kt](https://github.com/dvdsgl/tv-remote/blob/main/android/app/src/main/java/xyz/dvdsgl/tvremote/MainActivity.kt)
- msthind04 creates a platform `VolumeProvider(VOLUME_CONTROL_ABSOLUTE, …)` and calls `session.setPlaybackToRemote(provider)`. Volume becomes "a slider instead of two buttons — but only when the Apple TV reports it can route volume." — [NowPlayingService.kt](https://github.com/msthind04/AppleTV-Remote/blob/main/app/src/main/kotlin/dev/atvremote/app/NowPlayingService.kt); [README](https://github.com/msthind04/AppleTV-Remote/blob/main/README.md)
- Media3 equivalent: `DeviceInfo.PLAYBACK_TYPE_REMOTE` ("Playback happens outside of the device (e.g. a cast device)") with min/max volume, and `SimpleBasePlayer` device-volume methods. — [Media3 DeviceInfo](https://developer.android.com/reference/androidx/media3/common/DeviceInfo); [SimpleBasePlayer](https://developer.android.com/reference/androidx/media3/common/SimpleBasePlayer)

**Now-playing notification**
- msthind04: "The now-playing notification is a platform `MediaSession` rather than a hand-drawn layout, so the system's own media controls drive it: artwork, a scrubber that animates between updates, and transport buttons." The UI extrapolates position from "the last reported playhead and the moment it landed". "The Apple TV sends no artwork" for apps, so app icons are fetched from Apple's public iTunes endpoints, which leaks the installed-app list to Apple. — [msthind04 README](https://github.com/msthind04/AppleTV-Remote/blob/main/README.md)

**Quick Settings tile**
- dvdsgl (`RemoteTile`) and gavinsan33 (`RemoteTileService`) ship tiles. dvdsgl calls `startActivityAndCollapse(PendingIntent.getActivity(..., FLAG_IMMUTABLE))` on API ≥34 and the Intent overload below that. — [dvdsgl RemoteTile.kt](https://github.com/dvdsgl/tv-remote/blob/main/android/app/src/main/java/xyz/dvdsgl/tvremote/RemoteTile.kt)
- For apps targeting 14+, the Intent overload of `startActivityAndCollapse` is deprecated and throws (`UnsupportedOperationException`); use the PendingIntent overload. `TileServiceCompat` picks the right one. — [Android 14 behavior changes](https://developer.android.com/about/versions/14/behavior-changes-14); [TileServiceCompat](https://developer.android.com/reference/kotlin/androidx/core/service/quicksettings/TileServiceCompat)

**Device Controls**
- The developer guide still documents `ControlsProviderService` for the "Device controls" area. An AOSP change deprecates a related setting because "Controls are migrated to Quick Settings… will be removed in a future release". The release was not named, and I found no official deprecation of the API itself. — [Control external devices guide](https://developer.android.com/guide/topics/ui/device-control); [AOSP mirror commit](https://gitlab.e.foundation/e/os/android_frameworks_base/-/commit/a2be0990e29b452dade901bdd9ac5ef74d7acf2f)

**Widgets (Glance)**
- Glance 1.2.0 is listed as stable (Aug 26, 2026), with 1.3.0 in alpha. Localized copies of the page lag and show 1.1.1. — [Glance release notes](https://developer.android.com/jetpack/androidx/releases/glance)
- Android 17 enforces a RemoteViews bitmap-memory ceiling of `6 * width * height` bytes for apps targeting 37 (throws `IllegalArgumentException`). For Glance it recommends downsampling, `RGB_565`, and decoding bitmaps once outside `provideContent`. — [Android 17 behavior changes](https://developer.android.com/about/versions/17/behavior-changes-17)

**Haptics**
- dvdsgl uses `performHapticFeedback` with `KEYBOARD_TAP`, `CONTEXT_CLICK`, `VIRTUAL_KEY`, `LONG_PRESS`, `CONFIRM` and `REJECT`. gavinsan33 adds `CLOCK_TICK` (touchpad detents). heNch1g uses `VibrationEffect`. — source of [dvdsgl](https://github.com/dvdsgl/tv-remote), [gavinsan33](https://github.com/gavinsan33/tvremote), [heNch1g](https://github.com/heNch1g/atv_remote_android)

**Text input relay**
- dvdsgl: "Keyboard. Opens by itself when the TV asks for text, prefilled with what's in the field," using `InputMethodManager`. — [dvdsgl README](https://github.com/dvdsgl/tv-remote/blob/main/README.md)
- msthind04: "The keyboard prompt is a separate notification carrying an Android direct-reply field, so text can be typed and sent from the shade without unlocking. It is raised by the TV's own focus event." — [msthind04 README](https://github.com/msthind04/AppleTV-Remote/blob/main/README.md); [KeyboardNotification.kt](https://github.com/msthind04/AppleTV-Remote/blob/main/app/src/main/kotlin/dev/atvremote/app/KeyboardNotification.kt)

**Large screens**
- Android 17 removes the opt-out for ignoring orientation, resizability and aspect-ratio limits on screens ≥600dp. dvdsgl already has a two-pane foldable layout. — [Android 17 behavior changes](https://developer.android.com/about/versions/17/behavior-changes-17); [dvdsgl README](https://github.com/dvdsgl/tv-remote/blob/main/README.md)

### Inferences
- **Volume keys while backgrounded or locked:** a MediaSession with remote playback (platform `setPlaybackToRemote`, or Media3 `DeviceInfo.PLAYBACK_TYPE_REMOTE` plus `COMMAND_SET/ADJUST_DEVICE_VOLUME`) should make the system route hardware volume keys and the volume panel to the Apple TV while that session is active. This is the established Cast-style mechanism, but key-routing priority versus local media was not verified on Android 17 hardware.
- **Touchpad:** combine `HapticFeedbackConstants.CLOCK_TICK`/`SEGMENT_TICK`-style ticks for swipe detents with `CONFIRM` on select. Check API levels for newer constants before using them.
- **Tiles and widgets** should launch with PendingIntents and respect the Android 17 BAL rules. A Glance widget with D-pad, play/pause and volume needs a live connection, so widget taps should start the connected-device foreground service only when allowed. Background FGS-start exemptions for widget and notification taps were not verified.

### Gaps
- Wear OS companion (Tiles, Ongoing Activity, rotary input for volume) was not researched; no Wear OS Apple TV remote precedent was identified.
- No official confirmation of hardware volume-key routing priority to a remote MediaSession on Android 15–17.
- Whether `ControlsProviderService` is formally deprecated in Android 16 or 17 is unconfirmed.

## 8. Minimum SDK recommendations

### Takeaway
**Recommended: minSdk 26–28, compileSdk 37, targetSdk 36 now and 37 when the local network permission flow is in place.** Use Tink or BouncyCastle for X25519/Ed25519 and runtime-gate the newer APIs:
- ChaCha20-Poly1305 via JCA at 28+;
- `registerServiceInfoCallback` at T-ext 22 / API 34+;
- MulticastLock only below T-ext 7;
- tile PendingIntent at 34+;
- `ACCESS_LOCAL_NETWORK` at 37.

minSdk 33 (dvdsgl's choice) only removes the need for a curve library if raw-key JCA import is acceptable, and dvdsgl wrote its own curve code anyway. So 33 buys little and drops Android 8–12 users.

### Cited Findings
- **API-level facts:**
  - JCA ChaCha20-Poly1305: 28+. — [Cipher](https://developer.android.com/reference/javax/crypto/Cipher)
  - XDH / Ed25519: 33+. — [KeyAgreement](https://developer.android.com/reference/javax/crypto/KeyAgreement); [Signature](https://developer.android.com/reference/java/security/Signature)
  - `registerServiceInfoCallback` and `getHostAddresses`: API 34 / T-ext 22. Multicast lock needed before T-ext 7. — [NsdManager](https://developer.android.com/reference/android/net/nsd/NsdManager); [NsdServiceInfo](https://developer.android.com/reference/android/net/nsd/NsdServiceInfo)
  - Foreground service types mandatory at 34. — [FGS types](https://developer.android.com/develop/background-work/services/fgs/service-types)
  - `ACCESS_LOCAL_NETWORK` enforced for targetSdk 37. — [Local network permission](https://developer.android.com/privacy-and-security/local-network-permission)
  - Chaquopy minSdk 24. — [Chaquopy android.rst](https://github.com/chaquo/chaquopy/blob/master/product/runtime/docs/sphinx/android.rst)
- **Precedent minSdk values:** dvdsgl 33 (Android 13+), gavinsan33 29, msthind04 26 (README: "requires Android 8.0 or newer"), heNch1g 26. — build files of [dvdsgl](https://github.com/dvdsgl/tv-remote/blob/main/android/app/build.gradle.kts), [gavinsan33](https://github.com/gavinsan33/tvremote), [msthind04](https://github.com/msthind04/AppleTV-Remote), [heNch1g](https://github.com/heNch1g/atv_remote_android)
- **Play target requirements:** API 36 for new apps and updates since 2026-08-31. Wear OS apps must target API 35 and Android TV apps API 34. — [Play Console help](https://support.google.com/googleplay/android-developer/answer/11926878)

### Inferences
- **minSdk 26** (Android 8.0) is the floor every crypto path supports via BC or Tink, and matches msthind04. **minSdk 28** lets ChaCha20-Poly1305 come from the platform with no extra code, a reasonable compromise.
- With targetSdk 36 (current Play requirement), no local network permission work is needed yet. Plan the `ACCESS_LOCAL_NETWORK` and picker flows before moving to 37, which will likely be required around August 2027 by the annual Play pattern (not a published date).
- A Chaquopy build would be constrained by Python 3.13+ (16 KB pages) and the arm64-v8a/x86_64 ABIs, which effectively means 64-bit-only devices.

### Gaps
- No current (late-2026) Android version distribution figures were gathered to quantify what share of users minSdk 26, 28 or 33 drops.
