# Apple TV (tvOS) third-party control: protocols, discovery, pairing and wire formats (state as of Nov 2026)

Source basis and dating:
- Primary technical source is pyatv (Python, MIT), the de-facto reference implementation. I read its source and docs from a clone at release **0.18.0 "Willie" (2026-06-19)**. That was still the latest tagged release and the latest commit on `master` when I checked on 2026-11-14.
- The pyatv "Protocols" page has not been updated in step with the code. Its MRP and AirPlay 2 sections are mostly from 2021 (tvOS 14/15 era), with packet captures from iOS 14.7.1 / AirPlay 550.10. The Companion section is older still, but some parts were updated in 2026. Treat byte-level examples as illustrative.
- For 2025–2026 status I used GitHub issues on postlund/pyatv and home-assistant/core, up to 2026-11-14. These are mostly user reports, not Apple statements.
- Apple renamed its OS versions by year in 2025: tvOS 18 was followed by tvOS 26 (Sept 2025). As of Nov 2026, tvOS 27 is the current major line: betas started June 2026, and 27.2 was in beta in Oct 2026 per issue reports.

## 1. Which protocols exist, what each is used for, and status on tvOS 17 / 18 / 26 / 27

### Takeaway
A modern third-party Apple TV remote needs two protocols:
- **Companion Link** (`_companion-link._tcp`) for HID buttons, touchpad, app launch/list, power/attention state and keyboard.
- **MRP tunneled inside an AirPlay 2 "remote control" stream** (`_airplay._tcp`) for now-playing metadata, playback commands and volume. The standalone `_mediaremotetv._tcp` service has been gone since tvOS 15 (2021).

DMAP/DACP (`_touch-able._tcp`) only applies to Apple TV 3 and earlier (tvOS ≤ 12 / pre-tvOS). RAOP is for audio streaming, not control. Both Companion and the AirPlay pairing path broke several times in 2025–2026 (tvOS 18.4, 26.4/26.5, 27 betas). Plan for ongoing maintenance.

### Cited Findings
**Protocol roles (pyatv's concepts table):**
- DMAP: "Control device and get metadata", Apple TV ≤ 3 and tvOS ≤ 12.
- MRP: "Control device and get metadata", any tvOS version (Apple TV 4 and later).
- AirPlay: "Stream video to a device", all devices.
- RAOP: "Stream audio to a device", all devices.
- Companion: "Currently used for app management", Apple TV 4+.
- (The Companion row understates its current role; see below.) — [pyatv concepts](https://pyatv.dev/documentation/concepts/) ([source](https://github.com/postlund/pyatv/blob/master/docs/documentation/concepts.md))

**MRP:**
- MRP was introduced with Apple TV 4/tvOS. It is the protocol behind the iOS Remote app and the Control Center remote widget before iOS 13.
- It is Protocol Buffers, with each message prefixed by a varint length.
- It was discovered via `_mediaremotetv._tcp.local.`. The port "can change at any time" and usually starts at 49152. — [pyatv Protocols: MRP](https://pyatv.dev/documentation/protocols/)

**Companion Link:**
- Discovered via `_companion-link._tcp.local.`.
- The main reason it was reverse engineered was to launch apps the way the Shortcuts app does.
- "In iOS 13 Apple also decided to switch from MRP to Companion Link in the remote widget found in action center." — [pyatv Protocols: Companion Link](https://pyatv.dev/documentation/protocols/)
- Today Companion carries HID buttons (`_hidC` command codes 1–19: Up, Down, Left, Right, Menu, Select, Home, Vol±, Siri, Screensaver, Sleep, Wake, PlayPause, Channel±, Guide, PageUp/Down), touch gestures (`_hidT`), `_launchApp`, `FetchLaunchableApplicationsEvent`, `FetchAttentionState` / `SystemStatus` power state, and `_iMC` media-control flags. — [pyatv Protocols: Companion](https://pyatv.dev/documentation/protocols/)

**tvOS 15 (Sept 2021): standalone MRP removed, MRP tunneled over AirPlay 2.**
- pyatv 0.9.0 (2021-09-22): "Apple decided to drop support for the MRP protocol… In practice they didn't get rid of it, they just allocated a special stream type in AirPlay (2) and decided to tunnel MRP over it."
- AirPlay credentials are required, and users had to re-pair. Older (legacy) AirPlay credentials "will only work with `play_url`, not for tunneling MRP over AirPlay."
- pyatv 0.9.3 ignores the MRP service during scan for tvOS ≥ 15. — [pyatv CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md)

**When pyatv sets up the MRP-over-AirPlay tunnel:**
- Only if the AirPlay `model` starts with `AppleTV`, `osvers` ≥ 13, and the credentials are full HAP credentials.
- HomePods (`AudioAccessory*`) allow it only with transient credentials.
- Since 0.16.1 (2025-07-12) there is a user setting `mrp_tunnel = auto|force|disable`. — [pyatv airplay/utils.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/airplay/utils.py), [pyatv airplay/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/airplay/__init__.py), [pyatv settings.py](https://github.com/postlund/pyatv/blob/master/pyatv/settings.py)

**DMAP/DACP:** HTTP server on port 3689 with 4-byte-tag TLV binary responses. Most commands need a `session-id` obtained via `login` (`mlid`). — [pyatv Protocols: DMAP](https://pyatv.dev/documentation/protocols/)

**tvOS 18.4 breakage (Mar 2025):**
- Companion connections "close instantly" on 18.4 while AirPlay still works. — [pyatv #2656](https://github.com/postlund/pyatv/issues/2656), opened 2025-03-18, still open.
- `FetchAttentionState` returned "No request handler" on 18.4. — [pyatv #2648](https://github.com/postlund/pyatv/issues/2648)
- Fixed in pyatv **0.16.1 (2025-07-12)**: "Connection issues with tvOS 18.4+". The relevant commits are `companion: Use static id for _i in _systemInfo`, `airplay: Add setting for MRP tunnel` and "favor companion protocol for power state". — [pyatv CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md)
- Current pyatv code comment: "A null `_i` stops the device from pushing TVSystemStatus (power state) events". pyatv now sends a stable `_i` (`rp_id`, or the lowercase device id without colons). — [pyatv companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py)

**tvOS 26.4 / 26.5 breakage (Mar–May 2026):**
- Home Assistant (HA) users lost power on/off and power-state reporting on tvOS 26.5 (23L471). It had worked on 26.4 (23L243). Play/pause kept working because that goes over MRP.
- A pyatv contributor's diagnosis: tvOS 26.x "now requires a `TVRCSessionStart` step during the Companion-protocol handshake". — [HA core #170441](https://github.com/home-assistant/core/issues/170441), closed as a duplicate of [HA core #166545](https://github.com/home-assistant/core/issues/166545)
- pyatv commit `379d3a4 companion: send TVRCSessionStart during connect` (2026-05-25) is included in **0.18.0 (2026-06-19)**. HA shipped it in **2026.7** ("Bump pyatv to 0.18.0", #174354), and users confirmed the fix. — [pyatv git history / CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md); [HA core #166545](https://github.com/home-assistant/core/issues/166545)
- The pyatv docs now say a client should send `TVRCSessionStart` `{ProtocolVersionKey: "1.2"}` after `_sessionStart`, because "Some Apple TVs will not answer commands such as `FetchAttentionState` until this session has been registered". Older devices return an error, so it is safe to always send it. — [pyatv Protocols: Sessions](https://pyatv.dev/documentation/protocols/)

**`_launchApp` failures on tvOS 26.5 (23L471):** the device answered with an error, not success: `_ec: 4`, `_ed: FBSOpenApplicationServiceErrorDomain`, `_eu: {_ec: 4, _ed: FBSOpenApplicationErrorDomain}`, `_rT: 8`. pyatv was silently ignoring it. — [pyatv #2868](https://github.com/postlund/pyatv/issues/2868), opened 2026-06-16, closed 2026-10-13; I did not see the resolution.

**tvOS 26.5 / 26.6 AirPlay problems:**
- The AirPlay RTSP session loses synchronization during the 2-second `/feedback` heartbeat ("Got response without having a request", then a timeout and connection loss after about 35–40 s). — [pyatv #2893](https://github.com/postlund/pyatv/issues/2893), open
- HAP-transient `POST /pair-pin-start` times out against AirTunes/960.13.1 (tvOS/audioOS 26.6), so the remote-control channel fails. — [pyatv #2908](https://github.com/postlund/pyatv/issues/2908), open

**tvOS 27 betas (June 2026 onward):**
- AirPlay HAP pairing shows no PIN on the TV, while Companion pairing still shows a PIN and succeeds. — [pyatv #2866](https://github.com/postlund/pyatv/issues/2866)
- Root cause found by a contributor: the tvOS log shows "passcodeString is empty or clientName is nil. Bailing." Sending an **`X-Apple-Client-Name`** header on `/pair-pin-start` and every `/pair-setup` request makes the PIN appear. pyatv already sends the name, but only inside the encrypted M5 message, which arrives too late. — [pyatv PR #2894](https://github.com/postlund/pyatv/pull/2894), still open as of 2026-11-14
- Some users also got HTTP **470 "Connection Authorization Required"** after entering the PIN, including on stock tvOS 26.5 on gen-3 hardware. Enabling an AirPlay password was reported as a workaround. — [pyatv #2866](https://github.com/postlund/pyatv/issues/2866)
- The same no-PIN symptom was reported on 2026-11-14 for "Apple HomeView (HomeAccessory17,1, software 27.2)". — [pyatv #2951](https://github.com/postlund/pyatv/issues/2951), [#2950](https://github.com/postlund/pyatv/issues/2950)
- On tvOS 27 beta, Apple's own Remote-app pairing code also failed to appear. — [HA core #173352](https://github.com/home-assistant/core/issues/173352), closed 2026-06-28

**tvOS 27.2 beta:** Companion connection repeatedly receives `connection_closed` 0–10 s after connecting with pyatv 0.18.0. — [pyatv #2928](https://github.com/postlund/pyatv/issues/2928), open, 2026-10-15

**play_url:** `/playback-info` returns HTTP 500 on tvOS 26.2+. — [pyatv #2821](https://github.com/postlund/pyatv/issues/2821), [#2906](https://github.com/postlund/pyatv/issues/2906)

**Maintenance pace:** the 0.18.0 release notes say: "This is an awaited release with some overdue changes needed for compatibility with newer versions of tvOS… Due to my lack of free time, please feel free to send me PRs." — [pyatv CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md)

### Inferences
- **tvOS 17 and 18.0–18.3:** Companion plus MRP-over-AirPlay worked with pyatv 0.14–0.16. I found no specific tvOS 17 breakage.
- **tvOS 18.4+:** a client must send a non-null, stable `_i` in `_systemInfo`.
- **tvOS 26.4/26.5+:** a client must also send `TVRCSessionStart`.
- **tvOS 27:** a client must send `X-Apple-Client-Name` on AirPlay pair-setup requests, or the PIN may not appear. It may also need to handle HTTP 470.
- A from-scratch Android implementation should copy pyatv's current master behavior, not the older doc examples.
- Apple appears to keep Companion and the AirPlay remote-control stream functional because its own iPhone Control Center remote uses them. Small handshake requirements (`_i`, TVRCSessionStart, client-name header) change silently, though. Budget for regressions after each x.4/x.5 and major tvOS release.
- A remote limited to buttons, touchpad, apps and power can work with Companion alone. Now-playing metadata, artwork and absolute volume or position need the AirPlay-tunneled MRP (or Companion `_iMC` flags for coarse capability info).

### Gaps
- I found no Apple statement on any of these protocols. All status comes from community reports.
- I could not confirm the final resolution of #2868 (`_launchApp` error) or #2928 (tvOS 27.2 Companion disconnects).
- I could not confirm whether HTTP 470 has a known fix.
- A user in #2866 wrote that "27.5" fixed the PIN issue on Aug 10, 2026. That is chronologically implausible, since tvOS 27.0 had not shipped yet. It was probably 26.5.x or a 27 beta, so it is unreliable.
- I did not find the exact tvOS 27.0/27.1 release dates or build numbers. pyatv's build-number table stops at 18.1.

## 2. Discovery: mDNS/Bonjour service types, TXT fields, sleep-proxy behavior

### Takeaway
Browse for `_companion-link._tcp`, `_airplay._tcp`, `_raop._tcp` and `_sleep-proxy._udp` (plus `_mediaremotetv._tcp` for tvOS ≤ 14 and `_touch-able._tcp` for Apple TV ≤ 3).
- The AirPlay TXT gives a stable `deviceid`, `model`, `osvers`, the `features` bitmask, `flags` and `pi`/`psi`.
- The Companion TXT gives `rpMd`, `rpFl`, `rpVr` and rotating `rpHA`/`rpHN`/`rpAD`/`rpHI`/`rpBA`.
- Sleeping Apple TVs may be advertised by a Bonjour sleep proxy with port 0. Connecting to a service port wakes them.

### Cited Findings
**Companion TXT keys:**

| Key | Meaning | Example |
| --- | --- | --- |
| `rpHA` | HomeKit AuthTag | |
| `rpHN` | Discovery Nonce | |
| `rpVr` | likely protocol version | `195.2` |
| `rpMd` | model | `AppleTV6,2` |
| `rpFl` | status/feature flags | `0x36782` |
| `rpAD` | Bonjour Auth Tag | |
| `rpHI` | HomeKit rotating ID | |
| `rpBA` | Bluetooth address, can rotate | |

"Most values (except for rpVr, rpMd and rpFl) change every now and then (rotating encryption scheme)". — [pyatv Protocols: Companion Service Discovery](https://pyatv.dev/documentation/protocols/)

**Companion pairing support from `rpFl`:**
- `rpfl & 0x04` → pairing Disabled.
- `rpfl & 0x4000` → PIN pairing supported (Mandatory).
- Observed values: Apple TV 4K `0x367A2` / `0x36782` are pairable. HomePod `0x627B2` / `0x62792`, Mac `0x20000` and iPad `0x30000` are not. — [pyatv companion/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/__init__.py)

**AirPlay TXT keys:**

| Key | Meaning / example |
| --- | --- |
| `features` | e.g. `0x4A7FDFD5,0x3C155FDE`, a 64-bit bitmask written as two 32-bit halves |
| `flags` | status flags, e.g. `0x244` |
| `model` | |
| `osvers` | |
| `srcvers` | e.g. `540.31.41` |
| `pi` | group/pairing UUID |
| `psi` | "Public AirPlay Pairing Identifier" |
| `gid`, `igl`, `gcgl` | group fields |
| `pk` | public key |
| `acl` | Access Control Level |
| `deviceid` | MAC-like id |
| `protovers` | |
| `fex` | |
| `btaddr` | |
| `vv` | |

— [pyatv Protocols: AirPlay Service Discovery](https://pyatv.dev/documentation/protocols/)

**RAOP TXT keys:**

| Key | Meaning |
| --- | --- |
| `et` | encryption types: 0 none, 1 RSA, 3 FairPlay, 4 MFiSAP, 5 FairPlay SAPv2.5 |
| `cn` | codecs |
| `am` | model |
| `pw` | password |
| `ft` | features |
| `sf` | system flags |
| `vs` | version |
| `pk` | public key |

— [pyatv Protocols: RAOP](https://pyatv.dev/documentation/protocols/)

**AirPlay feature bits that matter for pairing/control (pyatv `AirPlayFlags`):**

| Bit | Flag |
| --- | --- |
| 27 | SupportsLegacyPairing |
| 38 | SupportsUnifiedMediaControl |
| 43 | SupportsSystemPairing |
| 46 | SupportsHKPairingAndAccessControl |
| 48 | SupportsCoreUtilsPairingAndEncryption |
| 51 | SupportsUnifiedPairSetupandMFi |
| 55/56 | SupportsWoL / SupportsWoL2 |
| 58 | SupportsHangdogRemoteControl |

- If bit 43 or 48 is set and no stored credentials exist, pyatv defaults to **transient** credentials. — [pyatv airplay/utils.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/airplay/utils.py), [pyatv airplay/auth/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/airplay/auth/__init__.py)
- The full bit list links to the openairplay features page. — [pyatv Protocols](https://pyatv.dev/documentation/protocols/)

**AirPlay pairing requirement from TXT:**
- Mandatory if bit `0x200` (legacy pairing) or `0x8` (PIN required) is set in `sf`/`flags`.
- `act=2` ("Current User" access control) is reported as Unsupported by pyatv. — [pyatv airplay/utils.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/airplay/utils.py)

**Ports:**
- AirPlay is typically 7000 and DMAP 3689.
- MRP starts around 49152 and changes. Always use the SRV record's port.
- pyatv's unicast-scan "knock ports" are `[3689, 7000, 49152, 32498]`. — [pyatv concepts](https://pyatv.dev/documentation/concepts/), [pyatv core/scan.py](https://github.com/postlund/pyatv/blob/master/pyatv/core/scan.py)

**Sleep proxy:**
- pyatv also browses `_sleep-proxy._udp.local`.
- It marks a device as `deep_sleep` when all its services have `port == 0` (proxied by a Bonjour Sleep Proxy).
- Its `knock` module "will open a TCP-connection to one or more ports on a given host and immediately close it… to wake a device sleeping via a Bonjour sleep proxy. Such a device will automatically be woken up in case any of its services are accessed". — [pyatv core/scan.py](https://github.com/postlund/pyatv/blob/master/pyatv/core/scan.py), [pyatv support/knock.py](https://github.com/postlund/pyatv/blob/master/pyatv/support/knock.py)

**Unicast mDNS:** pyatv has a `UnicastMdnsScanner` (`--scan-hosts`) that sends mDNS queries directly to a host IP. Several 2026 issues show multicast discovery is fragile:
- Multicast scan misses replies relayed by an mDNS reflector because the 5353 socket never joins 224.0.0.251. — [#2944](https://github.com/postlund/pyatv/issues/2944)
- Identifier-based multicast discovery fails while unicast works. — [#2904](https://github.com/postlund/pyatv/issues/2904)
- Cross-subnet setup fails. — [HA core #184270](https://github.com/home-assistant/core/issues/184270)

**Stable identity:** Companion has "no unique identifier easily available", so pyatv correlates devices by identifiers from other services. — [pyatv concepts](https://pyatv.dev/documentation/concepts/)

**MAC changes:** after tvOS 18.0, the MAC address pyatv detects can differ from the real network MAC. — [pyatv #2503](https://github.com/postlund/pyatv/issues/2503)

**Wake side effects:** HA's reconnect loop woke a sleeping Apple TV about once a minute, so reconnect and knock logic has visible side effects. — [HA core #184074](https://github.com/home-assistant/core/issues/184074)

### Inferences
- On Android, NsdManager or a raw mDNS implementation must resolve SRV, TXT and A/AAAA records for at least `_companion-link._tcp` and `_airplay._tcp`.
- Key the device on AirPlay `deviceid`/`pi` rather than Companion TXT, because most Companion TXT values rotate.
- Provide a manual-IP plus unicast-query path for networks with broken multicast (mesh routers, VLANs).
- To wake a sleeping Apple TV, a TCP connect to its Companion or AirPlay port via the sleep proxy should work. After that, Companion `_hidC` Wake (13) can be sent once connected.

### Gaps
- The exact semantics of most `rpFl` bits, `rpVr` versions on tvOS 26/27, and `fex` are not documented anywhere I found.
- I did not verify whether tvOS 26/27 changed any TXT keys.
- `pi` vs `psi` semantics for pairing identity are only partially understood.

## 3. Pairing and authentication (HAP SRP pair-setup, pair-verify, key derivation, transient, credential reuse, settings)

### Takeaway
All three modern protocols (Companion, MRP, AirPlay 2) use HomeKit Accessory Protocol (HAP) style pairing:
- **Pair-setup:** SRP-6a over the RFC 5054 3072-bit group with SHA-512, username `Pair-Setup`, password = the 4-digit PIN shown on the TV. Long-term keys are Ed25519.
- **Pair-verify:** ephemeral X25519 plus Ed25519 signatures.
- **Key derivation:** HKDF-SHA512 (32-byte keys) with protocol-specific salt/info strings.
- **Session encryption:** ChaCha20-Poly1305.

Only the carrier differs:
- Companion: OPACK `_pd` inside PS_/PV_ frames.
- MRP: TLV8 inside a protobuf CRYPTO_PAIRING_MESSAGE.
- AirPlay: HTTP POST `/pair-pin-start`, `/pair-setup`, `/pair-verify` on port 7000.

pyatv pairs Companion and AirPlay separately (two PINs) and stores one credential string per protocol. AirPlay "Allow Access" settings decide whether PIN pairing is even offered.

### Cited Findings
**SRP parameters:** `SRPContext("Pair-Setup", str(pin), prime=PRIME_3072, generator=PRIME_3072_GEN, hash_func=sha512)`.
- The client's SRP private value is seeded from its Ed25519 private key bytes.
- The Apple TV's M2 carries the salt (TLV 0x02) and public key (0x03).
- The client sends its public key and proof (0x04) in M3 and verifies the server proof in M4. — [pyatv auth/hap_srp.py](https://github.com/postlund/pyatv/blob/master/pyatv/auth/hap_srp.py), [pyatv Protocols: Companion Pairing](https://pyatv.dev/documentation/protocols/)

**Pair-setup M5/M6 derivation:**
- `iOSDeviceX = HKDF-SHA512(salt="Pair-Setup-Controller-Sign-Salt", info="Pair-Setup-Controller-Sign-Info", SRP_K)`.
- `sessionKey = HKDF(salt="Pair-Setup-Encrypt-Salt", info="Pair-Setup-Encrypt-Info", SRP_K)`.
- The client signs `iOSDeviceX ‖ pairingID ‖ Ed25519_LTPK` with its Ed25519 key.
- TLV {Identifier 0x01, PublicKey 0x03, Signature 0x0A} is encrypted with ChaCha20-Poly1305 under nonce `"PS-Msg05"`. M6 is decrypted with nonce `"PS-Msg06"` and yields the Apple TV's identifier, Ed25519 public key and signature.
- pyatv has a `# TODO: verify signature here`, so it does not verify the M6 signature. — [pyatv auth/hap_srp.py](https://github.com/postlund/pyatv/blob/master/pyatv/auth/hap_srp.py)

**Pair-verify:**
- The client sends an ephemeral X25519 public key.
- The device replies with its X25519 key and encrypted TLV {Identifier, Signature}.
- `key = HKDF(salt="Pair-Verify-Encrypt-Salt", info="Pair-Verify-Encrypt-Info", ECDH_shared)`. Decrypt with nonce `"PV-Msg02"`.
- Check that the identifier equals the stored `atv_id` and verify the Ed25519 signature over `devEphPub ‖ devId ‖ clientEphPub` with the stored device LTPK.
- Then sign `clientEphPub ‖ clientId ‖ devEphPub` with the client LTSK and send it encrypted with nonce `"PV-Msg03"`.
- Session keys are `HKDF(salt, info, ECDH_shared)` with protocol-specific strings. — [pyatv auth/hap_srp.py](https://github.com/postlund/pyatv/blob/master/pyatv/auth/hap_srp.py)

**Per-protocol session key strings:**

| Protocol / channel | Salt | Outgoing info | Incoming info |
| --- | --- | --- | --- |
| Companion | empty | `ClientEncrypt-main` | `ServerEncrypt-main` |
| MRP (standalone, tvOS ≤ 14) | `MediaRemote-Salt` | `MediaRemote-Write-Encryption-Key` | `MediaRemote-Read-Encryption-Key` |
| AirPlay control channel (RTSP on 7000) | `Control-Salt` | `Control-Write-Encryption-Key` | `Control-Read-Encryption-Key` |
| AirPlay event channel | `Events-Salt` | `Events-Write-Encryption-Key` | `Events-Read-Encryption-Key` |
| AirPlay data (MRP tunnel) channel | `DataStream-Salt<seed as unsigned 64-bit decimal>` | `DataStream-Output-Encryption-Key` | `DataStream-Input-Encryption-Key` |

For the AirPlay event channel the read and write keys are swapped on the sender side, because the channel is treated as originating from the receiver. — [pyatv companion/protocol.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/protocol.py), [pyatv mrp/protocol.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/protocol.py), [pyatv airplay/auth/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/airplay/auth/__init__.py), [pyatv airplay/ap2_session.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/airplay/ap2_session.py), [pyatv Protocols: AirPlay 2](https://pyatv.dev/documentation/protocols/)

**Companion pairing specifics:**
- M1 is frame `PS_Start` (0x03) with OPACK `{'_pd': TLV{Method:0x00, State:1}, '_pwTy': 1}`, where `_pwTy` 1 means PIN code. The PIN appears on screen after M1.
- M3 and M5 also carry `_pwTy: 1`.
- M5's encrypted TLV adds tag 17 (0x11) holding OPACK device info, for example `{altIRK, accountID, model, wifiMAC, name, mac}`. pyatv sends just `{name}`.
- Pair-verify M1 is `PV_Start` (0x05) with `{'_pd': TLV{State:1, PublicKey}, '_auTy': 4}`. The following messages are `PV_Next` (0x06). — [pyatv Protocols: Companion Authentication](https://pyatv.dev/documentation/protocols/), [pyatv companion/auth.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/auth.py)
- The tvOS UI path for the Remote-app style PIN is Settings > Remotes and Devices > Remote App and Devices. — [HA core #173352](https://github.com/home-assistant/core/issues/173352)

**MRP (standalone) pairing:** the HAP TLV8 travels in the `pairingData` field of a protobuf crypto-pairing message (`protobuf.extract_inner(resp).pairingData`). — [pyatv mrp/auth.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/auth.py)

**AirPlay HAP pairing:**
- HTTP `POST /pair-pin-start` (shows the PIN), then `POST /pair-setup` (M1–M6), then `POST /pair-verify`.
- Headers: `User-Agent: AirPlay/320.20`, `Connection: keep-alive`, `X-Apple-HKP: 3`, `Content-Type: application/octet-stream`.
- After verify, the HTTP/RTSP connection switches to HAP-framed encryption with the `Control-*` keys.
- On tvOS 27, `X-Apple-Client-Name: <name>` must be added to `/pair-pin-start` and `/pair-setup`, or the PIN UI is suppressed. — [pyatv airplay/auth/hap.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/airplay/auth/hap.py), [pyatv PR #2894](https://github.com/postlund/pyatv/pull/2894)

**AirPlay transient pairing (no user PIN):**
- `X-Apple-HKP: 4`, `POST /pair-pin-start`, then `/pair-setup` with TLV {Method 0, State 1, Flags = TransientPairing}.
- SRP runs with a fixed PIN **3939**, and only M1–M4 are done. The session keys come directly from the SRP shared secret K via HKDF.
- pyatv: "Technically, transient pairing only covers the first four states of regular pairing (M1-M4)". — [pyatv airplay/auth/hap_transient.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/airplay/auth/hap_transient.py)
- Transient credentials are enough for HomePod remote control but not for the Apple TV MRP tunnel, which requires full HAP credentials. — [pyatv airplay/utils.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/airplay/utils.py)

**AirPlay legacy pairing** (pre-HAP, used for `play_url` on old devices) also exists in pyatv as `AuthenticationType.Legacy`, stored as `client_id:ltsk` (2 fields). — [pyatv auth/hap_pairing.py](https://github.com/postlund/pyatv/blob/master/pyatv/auth/hap_pairing.py)

**Credential format:**
- pyatv stores HAP credentials as `hex(ltpk):hex(ltsk):hex(atv_id):hex(client_id)`.
- ltpk is the device Ed25519 public key. ltsk is the client Ed25519 private seed. atv_id is the device pairing identifier. client_id is the client pairing UUID.
- `ltpk == "transient"` denotes transient credentials. — [pyatv auth/hap_pairing.py](https://github.com/postlund/pyatv/blob/master/pyatv/auth/hap_pairing.py)

**Separate pairings per protocol:**
- atvremote's wizard reports "Credentials already exists for AirPlay / Companion / RAOP" separately. — [pyatv #2885](https://github.com/postlund/pyatv/issues/2885)
- On tvOS 27 beta, Companion pairing showed a PIN and succeeded, then "the wizard then continues to AirPlay pairing and waits for another PIN". — [pyatv #2866](https://github.com/postlund/pyatv/issues/2866)

**Access settings:**
- pyatv troubleshooting: "Make sure you have *Allow Access* set to *Anyone on the Same Network* for AirPlay on your Apple TV". A "pairing disabled" state means "the service is turned off or an access restriction is in place". — [pyatv troubleshooting](https://pyatv.dev/support/troubleshooting/)
- 2026 reports describe the equivalent control as Home app > Home Settings > Speakers & TV > Allow = Everyone (no password). — [pyatv #2908](https://github.com/postlund/pyatv/issues/2908), [#2951](https://github.com/postlund/pyatv/issues/2951)
- Enabling an AirPlay password was a reported workaround for tvOS 27 no-PIN pairing. — [pyatv #2866](https://github.com/postlund/pyatv/issues/2866)

**Dependency fragility:** srptools 2.0.0 (Oct 2026) made session arguments keyword-only and broke pyatv pairing. — [pyatv #2934](https://github.com/postlund/pyatv/issues/2934), [#2940](https://github.com/postlund/pyatv/issues/2940)

### Inferences
**Credential reuse across protocols.** The cryptographic material is the same shape in every protocol (Ed25519 LTPK/LTSK plus pairing IDs). Apple appears to scope pairings per service, though, because pyatv needs two PIN rounds.
- An Android client should plan for two pair-setup flows: Companion first, then AirPlay.
- A good UX is "enter the first code", then "enter the second code".
- The client can reuse one client Ed25519 identity and client_id for both, but must store separate device LTPK/atv_id per protocol.

**Crypto stack on Android:** SRP-6a/3072/SHA-512 (no Android platform API), X25519/Ed25519 (Conscrypt on newer API levels, or BouncyCastle/Tink), HKDF-SHA512 and ChaCha20-Poly1305. Pure-Java libraries (BouncyCastle) cover all of these.

**HAP TLV8 rule.** Values longer than 255 bytes are fragmented into consecutive items with the same tag. This is background knowledge of the HAP specification, not verified in this session's sources. The SRP 3072-bit public keys (384 bytes) in M2/M3 need it.

### Gaps
- I did not find authoritative confirmation that Companion credentials are rejected by AirPlay pair-verify, or the reverse. "Separate pairings are required" is inferred from pyatv behavior.
- I did not verify RAOP's credential handling. pyatv lists RAOP credentials separately, but their relation to AirPlay credentials is not confirmed.
- I did not verify the "Allow Access" options (Everyone / Anyone on the Same Network / Only People Sharing This Home / Require Password) on tvOS 26/27 against Apple documentation in this session.
- HTTP 470 "Connection Authorization Required" after PIN entry has no documented root cause.
- I did not determine whether `X-Apple-Client-Name` is also required for Companion on any tvOS version (reports say Companion PIN still works).

## 4. Wire formats: Companion frames, OPACK, TLV8, MRP protobuf framing, AirPlay channels for the MRP tunnel

### Takeaway
- **Companion:** `[frame type 1B][length 3B big-endian][payload]` over TCP. After pair-verify the payload is ChaCha20-Poly1305 encrypted with a 12-byte little-endian counter nonce and the 4-byte header as AAD. Payloads are OPACK dictionaries with `_i`/`_t`/`_x`/`_c`.
- **MRP:** varint length plus a `ProtocolMessage` protobuf.
- **AirPlay remote-control tunnel:**
  1. RTSP `SETUP {isRemoteControlOnly:true}` returns `eventPort`.
  2. `RECORD`.
  3. `SETUP` with stream type 130 returns `dataPort`.
  4. Open encrypted TCP connections to both ports.
  5. On the data channel, 32-byte `sync`/`rply` headers wrap a binary plist `{params:{data: varint-prefixed MRP protobufs}}`.
  6. Send `POST /feedback` every 2 s.

### Cited Findings
**Companion frame types:**

| Id | Name |
| --- | --- |
| 0x00 | Unknown |
| 0x01 | NoOp |
| 0x03 | PS_Start |
| 0x04 | PS_Next |
| 0x05 | PV_Start |
| 0x06 | PV_Next |
| 0x07 | U_OPACK |
| 0x08 | E_OPACK |
| 0x09 | P_OPACK |
| 0x0A | PA_Req |
| 0x0B | PA_Rsp |
| 0x10 | SessionStartRequest |
| 0x11 | SessionStartResponse |
| 0x12 | SessionData |
| 0x20 | FamilyIdentityRequest |
| 0x21 | FamilyIdentityResponse |
| 0x22 | FamilyIdentityUpdate |

The header is 1-byte type plus a 3-byte big-endian length. Only `E_OPACK` responses have been seen for app traffic. — [pyatv Protocols: Companion Binary Format](https://pyatv.dev/documentation/protocols/)

**Companion encryption:**
- ChaCha20-Poly1305 with the `ClientEncrypt-main` / `ServerEncrypt-main` keys.
- The nonce is a per-direction sequence number starting at 0, encoded as 12 bytes little-endian.
- The AAD is the frame header, and the length includes the 16-byte tag. Example: 3 bytes of E_OPACK plaintext gives AAD `0x08000013`. — [pyatv Protocols: Companion Encryption](https://pyatv.dev/documentation/protocols/)

**E_OPACK message envelope:**
- `_i`: message or event identifier, e.g. `_systemInfo`, `_launchApp`.
- `_t`: 1 = event, 2 = request, 3 = response.
- `_x`: transaction ID, echoed in the response.
- `_c`: content.
- `_sid`: session.
- Errors come back as `_em`/`_ec`/`_ed`, e.g. `No request handler`, `RPErrorDomain`. — [pyatv Protocols: E_OPACK](https://pyatv.dev/documentation/protocols/)
- tvOS 26.5 error responses also carry `_eu` and `_rT`. — [pyatv #2868](https://github.com/postlund/pyatv/issues/2868)

**Companion connect sequence in pyatv 0.18.0:**
1. Pair-verify.
2. `_systemInfo`, with a non-null `_i`, `_idsID = client_id`, `model`, `name`, `_sv: "170.18"`, `_bf: 0`, `_cf: 512`, `_clFl: 128`, `_sf: 256`.
3. `_sessionStart {_srvT: "com.apple.tvremoteservices", _sid: random32}`. The 64-bit session id is `(remote_sid << 32) | local_sid`.
4. `TVRCSessionStart {ProtocolVersionKey: "1.2"}`.
5. `_interest {_regEvents: [...]}` (`_iMC`, `SystemStatus`/`TVSystemStatus`, ...).
6. Touch input uses `_touchStart` with a 1000×1000 surface, then `_hidT` events with `_tPh` (1 press, 3 hold, 4 release, 5 tap), `_cx`/`_cy` and `_ns`. — [pyatv companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py), [pyatv Protocols: Sessions/Events/Touch](https://pyatv.dev/documentation/protocols/)

**OPACK** is Apple's CoreUtils serialization, mostly little-endian:

| Bytes | Type |
| --- | --- |
| `0x01` | true |
| `0x02` | false |
| `0x03` | terminator |
| `0x04` | null |
| `0x05` | UUID (16 B) |
| `0x06` | absolute mach time |
| `0x07` | −1 |
| `0x08–0x2F` | small ints 0–39 |
| `0x30–0x33` | ints with 1/2/4/8-byte payload |
| `0x35` | float32 |
| `0x36` | float64 |
| `0x40–0x60` | short string, 0–32 bytes |
| `0x61–0x64` | string with 1–4-byte length |
| `0x6F` | null-terminated string |
| `0x70–0x90` | short data |
| `0x91–0x94` | data with 1–4-byte length |
| `0xA0–0xC0` | back-reference pointer |
| `0xC1–0xC4` | pointer with 1–4-byte index |
| `0xD0+n` | array of n |
| `0xE0+n` | dict of n |
| `0xDF` / `0xEF` / `0x9F` | "endless" collection terminated by 0x03 |

- Pointers index previously decoded non-trivial objects (e.g. `E3416102416244746573744163A2` = `{a:false, b:"test", c:"test"}`).
- The doc warns it is "not complete or verified to be correct". — [pyatv Protocols: OPACK](https://pyatv.dev/documentation/protocols/)
- A macOS reference decoder exists via the private `OPACKDecodeBytes` (from fabianfreyer/opack-tools). — [pyatv Protocols: OPACK Reference Decoding](https://pyatv.dev/documentation/protocols/)

**TLV8 tags seen:**

| Tag | Meaning |
| --- | --- |
| 0x00 | Method |
| 0x01 | Identifier |
| 0x02 | Salt |
| 0x03 | PublicKey |
| 0x04 | Proof |
| 0x05 | EncryptedData |
| 0x06 | State/SeqNo |
| 0x0A | Signature |
| 0x11 (17) | Companion OPACK device info in M5 |
| 0x1B (27) | unknown, present in Companion M2 |

— [pyatv Protocols: Companion Pairing](https://pyatv.dev/documentation/protocols/), [pyatv auth/hap_tlv8.py](https://github.com/postlund/pyatv/blob/master/pyatv/auth/hap_tlv8.py)

**MRP framing:**
- Each `ProtocolMessage` is prefixed with its length as a protobuf varint (`write_variant(len) + serialized`).
- The receive side buffers until a full varint-delimited message is available. — [pyatv mrp/connection.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/connection.py), [pyatv support/variant.py](https://github.com/postlund/pyatv/blob/master/pyatv/support/variant.py)
- Message definitions come from the community `mediaremotetv-protocol` work and pyatv's `.proto` files. — [pyatv Protocols: MRP References](https://pyatv.dev/documentation/protocols/)
- `uniqueIdentifier` (a random UUID4) must be set in the `ProtocolMessage` envelope or "the device doesn't seem to respond". The first message is `DEVICE_INFO_MESSAGE`, then e.g. `SET_CONNECTION_STATE_MESSAGE`. — [pyatv Protocols: AirPlay 2 Data Channel](https://pyatv.dev/documentation/protocols/)

**AirPlay 2 HAP-encrypted framing:** blocks of `[2-byte LE length][ciphertext][16-byte tag]`. HAP caps blocks at 1024 bytes, but AirPlay does not enforce that. — [pyatv Protocols: AirPlay 2 Encryption](https://pyatv.dev/documentation/protocols/)

**AirPlay remote-control stream setup:**
1. `SETUP rtsp://<ip>/<random>` with a binary plist `{isRemoteControlOnly: True, osName, sourceVersion, timingProtocol: "None", model, deviceID, osVersion, osBuildVersion, macAddress, sessionUUID, name}`. The response is `{eventPort}`.
2. Connect to the event port.
3. `RECORD`.
4. The receiver sends `POST /command` with an `updateInfo` plist on the event channel. **The client must reply `RTSP/1.0 200 OK`, or the receiver closes the connection after 30 s.**
5. `SETUP` with `{streams:[{type:130, controlType:2, channelID:UUID, clientUUID:UUID, clientTypeUUID:"1910A70F-DBC0-4242-AF95-115DB30604E1" (Media Remote), seed:int64, wantsDedicatedSocket:true}]}`. The response is `{streams:[{type:130, streamID:1, dataPort}]}`.
6. Connect to the data port.
7. Send `POST /feedback` every ~2 s as a keep-alive. — [pyatv Protocols: AirPlay 2 Remote Control](https://pyatv.dev/documentation/protocols/)

**Data channel message header (32 bytes):**

| Field | Size | Contents |
| --- | --- | --- |
| total size | 4 B | includes the header |
| message type | 12 B | `sync` or `rply`, zero-padded |
| command | 4 B | `comm` from sender, `cmnd` from receiver, zero in `rply` |
| sequence | 8 B | |
| padding | 4 B | |

- The payload is a binary plist `{"params": {"data": <one or more varint-length-prefixed MRP protobufs>}}`.
- Every `sync` must be answered with a `rply` carrying the same sequence number.
- The doc calls this format "mostly educated guesses". — [pyatv Protocols: AirPlay 2 Data Channel](https://pyatv.dev/documentation/protocols/), [pyatv airplay/channels.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/airplay/channels.py)

**Pre-AirPlay-2 step:** `POST /auth-setup` (1-byte type plus a 32-byte Curve25519 key) is needed on MFi devices (`et=4`). It is "always the case for AirPlay 2", and the response can be ignored. — [pyatv Protocols: AirPlay /auth-setup](https://pyatv.dev/documentation/protocols/)

### Inferences
- The minimum code path for a full-featured Android remote is:
  - TLV8 codec.
  - OPACK codec (encode needs only bool/int/float/string/data/array/dict; decode must also handle pointers, endless collections and UUIDs).
  - Companion framer and cipher.
  - Binary-plist encoder/decoder (e.g. dd-plist).
  - Minimal RTSP/HTTP client that can switch to HAP-framed encryption mid-connection.
  - Protobuf (Wire or protobuf-lite) generated from pyatv's MRP `.proto` files.
- The AirPlay tunnel needs four concurrent TCP sockets: control (7000), event, data, and the Companion socket. It also needs careful sequencing of RTSP CSeq and replies, because the 2026 heartbeat desync bug (#2893) shows this is easy to get wrong.
- The pyatv doc has internal inconsistencies. The Companion pairing intro says "PA_Start" where it means PS_Start. The verify sequence diagram labels frames 0x04/0x05 where the text says 0x05/0x06. Some OPACK table examples are mislabeled (0x63/0x64 rows, and the 0xC2 row repeated where 0xC3 is meant). Implementers should follow pyatv source code over prose.

### Gaps
- Full ProtocolMessage `.proto` definitions were not reviewed here. They are in pyatv's `pyatv/protocols/mrp/protobuf/`.
- I did not check whether tvOS 26/27 bumped `lastSupportedMessageType` or added required MRP messages.
- The exact meaning of the `sync`/`rply` sequence fields and of `comm` vs `cmnd` remains unknown.
- I did not research whether Apple added new Companion frame types (e.g. for tvOS 27).

## 5. What Apple officially documents vs. reverse-engineered; legal/ToS considerations

### Takeaway
Apple publicly documents only the HAP specification. A non-commercial edition is freely downloadable; the commercial edition needs an Apple ID or the MFi program. Companion Link, OPACK, MRP, and the AirPlay 2 sender and remote-control protocol are undocumented and known only through reverse engineering, chiefly pyatv, openairplay, owntone and pair_ap.

Building an interoperable client from observed network behavior is generally covered by interoperability exceptions:
- US: DMCA §1201(f).
- EU: Software Directive Arts. 5(3) and 6, which contracts cannot override.

Distribution must avoid Apple trademarks and logos, copied Apple code or keys, and implying endorsement. This is not legal advice.

### Cited Findings
- pyatv describes its protocol knowledge as reverse engineered: "to the extent we know, since they are reverse engineered". It calls the Companion section "mostly speculation and guesses". — [pyatv Protocols](https://pyatv.dev/documentation/protocols/)
- Pairing and encryption "are according to HAP (HomeKit)". The HAP spec is available from Apple and "requires an Apple ID, except for the Non-Commercial version, free to download". — [pyatv Protocols: Companion Authentication](https://pyatv.dev/documentation/protocols/), linking [Apple HomeKit spec page](https://developer.apple.com/homekit/specification/)
- Community references for AirPlay:
  - [openairplay "Unofficial AirPlay Specification"](https://openairplay.github.io/airplay-spec/introduction.html)
  - [nto "Unofficial AirPlay Protocol Specification"](https://nto.github.io/AirPlay.html)
  - [AirPlay 2 Internals (emanuelecozzi.net)](https://emanuelecozzi.net/docs/airplay2)
  - [owntone-server](https://github.com/owntone/owntone-server)
  - [pair_ap](https://github.com/ejurgensen/pair_ap) (C implementations of HAP and transient pairing)
  - [mediaremotetv-protocol](https://github.com/jeanregisser/mediaremotetv-protocol) (MRP)
  - [fabianfreyer/opack-tools](https://github.com/fabianfreyer/opack-tools) (OPACK)
  - — all as listed in [pyatv Protocols references](https://pyatv.dev/documentation/protocols/)
- US law: §1201(f)(1) permits circumventing a technical measure to identify and analyze "those elements of the program that are necessary to achieve interoperability of an independently created computer program with other programs". Its scope for end users has been debated in Copyright Office rulemakings, and a later Office report corrected its narrower earlier reading. — [Perzanowski, UC Davis L. Rev. (interoperability)](https://www.perzanow.ski/s/UC-DavisLR-Interop.pdf); [Project DISCO on the Copyright Office §1201 report](https://project-disco.org/intellectual-property/070617-interoperability-and-the-copyright-offices-section-1201-report/)
- EU law: Software Directive Art. 6 allows decompilation limited to what is necessary for interoperability. Art. 5(3) allows "black box" observation. Art. 9(1) voids contract terms that restrict these. — as summarized in [Perzanowski](https://www.perzanow.ski/s/UC-DavisLR-Interop.pdf) and [project-disco.org](https://project-disco.org/?p=8314) search results
- Prior art for tolerance: Home Assistant (bundling pyatv), Homebridge plugins and others have shipped this reverse-engineered control publicly for years, e.g. the [HA Apple TV integration](https://www.home-assistant.io/integrations/apple_tv/) referenced in the HA issues above. — [HA core #173646](https://github.com/home-assistant/core/issues/173646)

### Inferences
- The protocols need no Apple-signed keys for remote control. HAP pairing is PIN-based, and `/auth-setup` responses can be ignored. So a clean-room Android implementation does not need to circumvent DRM. FairPlay is only relevant to video and mirroring streaming, which is out of scope.
- The main legal risk is trademark (naming the app "Apple TV Remote") and app-store policy, not copyright.
- Because Apple does not commit to these interfaces, it can break them at any time without notice, as tvOS 15, 18.4, 26.5 and 27 show. A product should track pyatv issues and HA release notes as early-warning signals.

### Gaps
- I did not review Apple's tvOS software license agreement or Google Play policy text for specific clauses on unofficial remotes.
- I did not find whether Apple's MFi AirPlay 2 program documents the remote-control (type 130) stream. MFi specs are under NDA.
- I found no case law specific to Apple TV remote apps.
