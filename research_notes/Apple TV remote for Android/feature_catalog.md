# Apple TV remote-control feature catalog for third-party clients (protocol/message per feature, state as of Nov 2026)

Scope and method note: the main primary source is the pyatv source tree and docs, read from a shallow clone of `postlund/pyatv` at master commit `b277a4c`, dated 2026-06-19. That commit is the v0.18.0 "Willie" release, the latest one. The tree includes `docs/documentation/protocols.md`, `supported_features.md`, the Companion and MRP implementation code, and `CHANGES.md`. These are combined with pyatv GitHub issues and PRs up to 2026-11-14 and with the Home Assistant Apple TV integration doc, read from its GitHub source because home-assistant.io was not reachable from this session. macrumors.com was also unreachable, so the tvOS 27 facts below come from search-result snippets only. File links use `blob/master` and point to the content as of v0.18.0.

**Important version context:** the brief says "tvOS 26 era", but tvOS 27 shipped on 14 Sep 2026. The current release in Nov 2026 is therefore tvOS 27.x, and pyatv issue reports already mention tvOS 27.2 beta. Most published protocol documentation was written between tvOS 13 and tvOS 18 (2019–2024). Breakage reports from tvOS 18.4 through 26.5 come from 2025–2026. **I found no published reverse-engineering of protocol changes specific to tvOS 27.**

---

## 1. Basic navigation and hardware buttons (D-pad, select, menu/back, home, double/long-press home, TV button, play/pause, skip, Siri button)

### Takeaway
Every Siri Remote button except voice input can be emulated. The Companion protocol's `_hidC` request (`_hBtS` 1 = down, 2 = up, with `_hidC` = button code) is the modern path. Press, double-tap and hold are synthesized by timing the down/up events. The legacy MRP path is `SendHIDEventMessage` with USB HID usage pages, tunneled over AirPlay 2 since tvOS 15. Playback keys can also go through Companion `_mcc` (media control) or MRP `SendCommandMessage`. Pressing the Siri button is defined as `_hidC` 10, but **Siri voice input does not work for third parties as of Nov 2026**.

### Cited Findings
- **Companion `_hidC` button codes (pyatv protocol docs):** 1=Up, 2=Down, 3=Left, 4=Right, 5=Menu, 6=Select, 7=Home, 8=Volume up, 9=Volume down, 10=Siri, 11=Screensaver, 12=Sleep, 13=Wake, 14=PlayPause, 15=Channel Increment, 16=Channel Decrement, 17=Guide, 18=Page Up, 19=Page Down. The button state goes in `_hBtS` (1=down/pressed, 2=up/released). Example message: `{'_i': '_hidC', '_t': 2, '_c': {'_hBtS': 2, '_hidC': 12}}` puts the device to sleep. — [pyatv protocols doc, "Buttons/Commands (_hidC)"](https://pyatv.dev/documentation/protocols/) ([source](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md))
- **Same enum in code.** `HidCommand` in the Companion API has the same values. pyatv maps **Control Center to `HidCommand.PageDown` (19)** and Guide to 17. — [pyatv companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py); [pyatv companion/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/__init__.py)
- **Conflicting meaning of codes 18 and 19.** A Jan 2024 capture of the iOS 17.3 Remote lists `_hidC` codes 6 (touch/select), 14 (play/pause), **18 (mute)** and **19 (power)**. pyatv instead calls 18/19 "Page Up/Page Down" and uses 19 for Control Center. — [pyatv issue #2325 "[Companion] Documentation Dumps"](https://github.com/postlund/pyatv/issues/2325) vs [pyatv api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py)
- **Guide and Control Center buttons.** Dedicated Guide and Control Center commands were added in pyatv 0.17.0 "Velma" (2026-01-21), PR #2699 by @henrikwidlund ("discrete commands to open control center and toggle EPG (Guide)"). — [pyatv CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md); [PR #2699](https://github.com/postlund/pyatv/pull/2699)
- **How pyatv times presses (`_press_button`).** SingleTap is one down+up. DoubleTap is two down+up pairs sent back-to-back. Hold is down, a 1 s sleep, then up. — [pyatv companion/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/__init__.py)
- **Companion buttons exposed as pyatv features:** Up, Down, Left, Right, Select, Menu, Home, VolumeUp, VolumeDown, PlayPause, ChannelUp, ChannelDown, Screensaver, Guide, ControlCenter. **Siri (10) is in the enum but not exposed as a feature.** — [pyatv companion/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/__init__.py)
- **MRP HID mapping, as `[usage_page, usage]`:**

  | Key | Usage |
  |---|---|
  | up | (1, 0x8C) |
  | down | (1, 0x8D) |
  | left | (1, 0x8B) |
  | right | (1, 0x8A) |
  | select | (1, 0x89) |
  | menu | (1, 0x86) |
  | topmenu | (12, 0x60) |
  | home | (12, 0x40) |
  | suspend | (1, 0x82) |
  | wakeup | (1, 0x83) |
  | volume_up | (12, 0xE9) |
  | volume_down | (12, 0xEA) |
  | stop | (12, 0xB7) |
  | next | (12, 0xB5) |
  | previous | (12, 0xB6) |

  The Siri "mic" key `(12, 0x04)` is **commented out**. Each MRP key press is sent as `SEND_HID_EVENT_MESSAGE` down/up and followed by a `GENERIC_MESSAGE` that acts as a flush. — [pyatv mrp/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/__init__.py)
- **Companion media-control commands (`_mcc`):** 1=Play, 2=Pause, 3=NextTrack, 4=PreviousTrack, 5=GetVolume, 6=SetVolume, 7=SkipBy, 8/9=FastForward Begin/End, 10/11=Rewind Begin/End, 12/13=Get/SetCaptionSettings. — [pyatv companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py)
- **Skip by a custom interval.** Companion skip uses `_mcc` 7 with `_skpS`, a float number of seconds (e.g., `10.0` or `-10.0`). Custom intervals were added in pyatv 0.15.0 (2024-08-10). — [issue #2325](https://github.com/postlund/pyatv/issues/2325); [CHANGES.md 0.15.0](https://github.com/postlund/pyatv/blob/master/CHANGES.md)
- **Which skip/seek actions are currently available** is signalled by the `_iMC` event's `_mcF` bitmask:

  | Bit | Meaning |
  |---|---|
  | 0x0001 | Play |
  | 0x0002 | Pause |
  | 0x0004 | Previous |
  | 0x0008 | Next |
  | 0x0010 | Fast forward |
  | 0x0020 | Rewind |
  | 0x0100 | Volume |
  | 0x0200 | Skip forward |
  | 0x0400 | Skip backward |

  — [pyatv protocols doc, "Events"](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md)
- **MRP `SendCommand` (CommandInfo.proto) command IDs include:**

  | Command | ID |
  |---|---|
  | Play | 1 |
  | Pause | 2 |
  | TogglePlayPause | 3 |
  | Stop | 4 |
  | NextTrack | 5 |
  | PreviousTrack | 6 |
  | AdvanceShuffleMode | 7 |
  | AdvanceRepeatMode | 8 |
  | Begin/EndFastForward | 9/10 |
  | Begin/EndRewind | 11/12 |
  | SkipForward | 18 |
  | SkipBackward | 19 |
  | ChangePlaybackRate | 20 |
  | Like/DislikeTrack | 22/23 |
  | NextChapter | 25 |
  | SeekToPlaybackPosition | 45 |
  | ChangeRepeatMode | 46 |
  | ChangeShuffleMode | 47 |
  | SetPlaybackQueue | 48 |
  | InsertIntoPlaybackQueue | 52 |
  | Enable/DisableLanguageOption | 53/54 |
  | Reorder/Remove/PlayItemInPlaybackQueue | 55/56/57 |

  — [pyatv CommandInfo.proto](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/protobuf/CommandInfo.proto)
- **Long-press Home (`home_hold`) is unreliable.** An open issue from Nov 2023 (20 comments) reports `home_hold` behaving like a plain Home press on tvOS 17.1 in Home Assistant instead of opening Control Center. — [pyatv issue #2286](https://github.com/postlund/pyatv/issues/2286)
- **Closing apps in the app switcher.** A tester reported in Mar 2024 that "swipe-up-to-close" in the app switcher is hard to mimic, and that double-clicking Up closes apps there. — [issue #2325 (cherpake/thiccaxe comments, Mar 2024)](https://github.com/postlund/pyatv/issues/2325)
- **Commands Home Assistant exposes:** wakeup, suspend, home, top_menu, menu, control_center, turn_on, turn_off, select, play, pause, up/down/left/right, volume_up/down, previous/next, skip_backward/forward. HA notes "Not all commands are supported by all Apple TV versions". Its FAQ says apps decide which playback commands they honor, so check against Apple's iOS Remote app first. — [HA apple_tv integration doc (GitHub source)](https://github.com/home-assistant/home-assistant.io/blob/current/source/_integrations/apple_tv.markdown)
- **Siri voice is not achievable as of the last activity (2026-07-28).** In the open "Voice Support / Siri" investigation (opened 2026-06-21 by thiccaxe), voice is carried by Companion `_siriStart` → `_siA` events (concatenated Opus/SILK narrowband frames, 8 kHz mono, 20 ms each, plus frame-offset descriptors) → `_siriStop`. The Apple TV also sends a `_siriEndpoint` request. Replaying a captured session opened the Siri UI, but speech was never transcribed. Malformed voice data can "soft lock" the Apple TV. Theories under test are that processing is gated by Siri capability data in `_systemInfo` (`_siriInfo`) and/or an Apple ID/iCloud check. — [pyatv issue #2875](https://github.com/postlund/pyatv/issues/2875); [issue #2325 (Mar/May 2024 comments on `_siriStart`/`_siA`/`_siriStop`/`_siriEndpoint`)](https://github.com/postlund/pyatv/issues/2325)
- **Voice-related MRP messages exist but are unused.** pyatv ships MRP protobufs `RegisterVoiceInputDeviceMessage`, `SendVoiceInputMessage`, `VoiceInputDeviceDescriptorMessage` and `AudioFormatSettingsMessage`, but nothing in pyatv uses them. — [pyatv mrp/protobuf directory](https://github.com/postlund/pyatv/tree/master/pyatv/protocols/mrp/protobuf)

### Inferences
- **Basic remote tier.** D-pad, Select, Menu, Home, Play/Pause, Volume ±, Channel ±, Guide, Control Center and Screensaver can all be built on Companion `_hidC` alone. That is one request per edge, two per tap.
- **Home double-press and long-press.** The app switcher (Home double-press) has no dedicated code. It relies on the timing of two quick Home taps, which is untested in public reports. Long-press Home is better replaced by the dedicated Control Center code (19 in pyatv) than by a timed hold.
- **"TV/Home button" vs Top Menu.** On Companion both are effectively `_hidC` 7. MRP has separate Home (12,0x40) and TopMenu (12,0x60) usages.
- **Codes 18 and 19 are uncertain.** They may mean Page Up/Down or mute/power depending on context, so a client should test on real hardware before labeling them.
- **Siri.** Treat the Siri button as "press only": it may open the Siri UI. Voice/dictation is out of scope for a third-party Android app as of Nov 2026.

### Gaps
- No public, verified mapping confirms what `_hidC` 18/19 do on tvOS 26/27. Mute vs Page Up/Down vs Control Center is unclear.
- No user report confirms that two rapid `_hidC` 7 taps reliably open the app switcher on tvOS 26/27.
- No source says whether `_hidC` 10 (Siri) alone opens the Siri UI on tvOS 26/27 without the `_siriStart` handshake.

---

## 2. Touchpad emulation (swipes, continuous drag, click-and-hold, scrubbing, latency)

### Takeaway
Touchpad emulation works through Companion. A client registers a virtual 1000×1000 surface with `_touchStart` and then streams `_hidT` events (phase `_tPh` 1=press, 3=move/hold, 4=release, 5=click-end) every ~16–20 ms. These are fire-and-forget events with no acknowledgement, which keeps latency low. A click is synthesized as `_hidC` 6 down/up plus a `_hidT` phase-5 event at (1000,1000). pyatv has supported this since 0.15.0 (Aug 2024).

### Cited Findings
- **Registering the touch surface.** `_touchStart` is sent with `{'_height': 1000.0, '_tFl': 0, '_width': 1000.0}`. Coordinates `_cx`/`_cy` must lie in [0,1000]. `_ns` is a timestamp in nanoseconds. — [pyatv protocols doc, "Touch gestures"](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md)
- **Gesture sequence.** One `_hidT` event with `_tPh`=1, then N events with `_tPh`=3 sent ~20 ms apart with changing coordinates, then `_tPh`=4 on release. "When reaching the edge of the touch area, a release event should be sent with a new press event otherwise the cursor will move in the opposite way". — [pyatv protocols doc](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md)
- **Single tap / click.** Send `_hidC` 6 with `_hBtS` 1, then 2, then a `_hidT` event with `_tPh`=5 at `_cx`=`_cy`=1000. — [pyatv protocols doc, "Single tap"](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md)
- **pyatv implementation details.** pyatv uses `TOUCHPAD_DELAY_MS = 16` between events. `swipe(start_x, start_y, end_x, end_y, duration_ms)` interpolates points. `action(x, y, mode)` sends raw touch phases. `click(InputAction)` supports single, double and hold; hold is select-down, 1 s, select-up. — [pyatv companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py)
- **When touch arrived.** "Added touch gesture to companion protocol (#2402)" shipped in pyatv 0.15.0 (2024-08-10). The pyatv features are Swipe, Action and Click. — [pyatv CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md); [pyatv const.py FeatureName](https://github.com/postlund/pyatv/blob/master/pyatv/const.py)
- **Observed behavior from iOS Remote captures.** `_hidT` events get no OPACK acknowledgement. Phase 5 is click/tap/press-hold end. Tap coordinates are always 1000,1000. `_tPh` 4 is also used for two-finger taps, which tvOS ignores. — [pyatv issue #2325](https://github.com/postlund/pyatv/issues/2325)
- **Swipe left/right.** A pyatv-based remote (cleverdevil/pnut) could not originally emulate the "swipe left" one streaming app needs for navigation. That issue predates touch support; it opened 2023-09-21 and is still open. — [pyatv issue #2202](https://github.com/postlund/pyatv/issues/2202)
- **MRP touch messages.** MRP also has `SendPackedVirtualTouchEventMessage` and `VirtualTouchDeviceDescriptorMessage`, but pyatv implements touch over Companion only. — [pyatv mrp/protobuf directory](https://github.com/postlund/pyatv/tree/master/pyatv/protocols/mrp/protobuf)

### Inferences
- **Smoothness depends on the client.** Events are unacknowledged one-way frames, so smoothness depends mainly on the Android client sending at a steady ~60 Hz (16 ms) and on Wi-Fi jitter, not on round-trips.
- **Pointer bookkeeping.** An Android trackpad view should map its surface to 0–1000 and split long drags at the edges (release, then press again). The same applies when a finger moves past the virtual bounds.
- **Scrubbing.** Scrubbing in the tvOS player can be done with horizontal `_hidT` drags while paused. A more deterministic route is `_mcc` 7 (SkipBy with `_skpS`) or MRP `SeekToPlaybackPosition`.

### Gaps
- No measured latency figures (ms) were found for `_hidT` on tvOS 26/27.
- No report confirms whether the tvOS 26/27 player's touch-scrubbing behaves the same with emulated touch as with a real Siri Remote.

---

## 3. Keyboard / text input (focus detection, get/set/append/clear, secure fields, app coverage, tvOS requirements)

### Takeaway
Text input goes through the Companion RTI ("Remote Text Input") session.
- **Session:** `_tiStart` and `_tiStop` requests.
- **Focus notifications:** the `_tiStarted` and `_tiStopped` events act as the iPhone "keyboard appeared" notification.
- **Editing:** a `_tiC` event carries an NSKeyedArchiver `_tiD` payload with the session UUID.

pyatv implements focus state, get, set, append and clear (0.12.0, May 2023; focus listener 0.13.0) and Home Assistant exposes them. A spring 2026 report says focus-change events stopped firing with pyatv 0.17.0 on tvOS 26.5.

### Cited Findings
- **pyatv keyboard API (Companion required):** `text_focus_state`, `text_get()`, `text_set()`, `text_append()`, `text_clear()`, plus push updates through a keyboard listener. — [pyatv Keyboard doc](https://pyatv.dev/development/keyboard/) ([source](https://github.com/postlund/pyatv/blob/master/docs/development/keyboard.md))
- **Protocol flow in pyatv.** On connect pyatv sends `_tiStart` and listens for `_tiStarted`/`_tiStopped`. Focus is "Focused" if the event or response contains `_tiD`; the comment notes "_tiStarted will not be sent if session is started while already focused".
  - To read or modify text, pyatv restarts the session (`_tiStop` then `_tiStart`) to get fresh data. It extracts `sessionUUID` and the current text (`documentState.docSt.contextBeforeInput`) from the keyed archive.
  - It then sends a `_tiC` event `{'_tiV': 1, '_tiD': <RTI payload>}`, either a clear payload or an insert-text payload.

  — [pyatv companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py); [pyatv companion/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/__init__.py)
- **Release history.** "Text/keyboard input API (get and alter current text)" shipped in pyatv 0.12.0 (2023-05-24, @michalmo, PR #1905). The keyboard focus listener came in 0.13.0 (2023-06-15). — [pyatv CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md)
- **Home Assistant.** HA creates a `binary_sensor` for "on-screen keyboard is active" and the actions `apple_tv.set_keyboard_text`, `append_keyboard_text` and `clear_keyboard_text`. These "require that the keyboard is currently focused on the device". HA's example types a search query ("Severance") when the keyboard appears. — [HA apple_tv doc (GitHub source)](https://github.com/home-assistant/home-assistant.io/blob/current/source/_integrations/apple_tv.markdown)
- **Regression report.** Keyboard focus-change events do not fire with pyatv 0.17.0 on an Apple TV 4K with tvOS 26.5 (e.g., moving in and out of the Search text box), but did fire with 0.16.1. Opened 2026-05-18, open, no replies. — [pyatv issue #2853](https://github.com/postlund/pyatv/issues/2853)
- **Other open keyboard issues:** "Keyboard Listener" (#2841, Apr 2026) and "PyATV - Send text" (#2578, Dec 2024). — [pyatv issue search](https://github.com/postlund/pyatv/issues?q=keyboard)
- **Legacy MRP keyboard.** MRP had keyboard messages (`GetKeyboardSessionMessage`, `KeyboardMessage`, `TextInputMessage`, `GetRemoteTextInputSessionMessage`), and an older request asked for "keyboard entry over MediaRemoteTV protocol" (closed). Since tvOS 15 the supported path is Companion. — [pyatv mrp/messages.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/messages.py); [issue #879](https://github.com/postlund/pyatv/issues/879)
- **User motivation.** A 2022 user asked for keyboard input specifically to avoid letter-by-letter arrow navigation in Netflix search and sign-in. — [pyatv issue #1705](https://github.com/postlund/pyatv/issues/1705)

### Inferences
- **Focus notification.** The `_tiStarted`/`_tiStopped` push events are the closest thing to the iOS Remote's "keyboard appeared" notification. An Android app can pop up its keyboard on `_tiStarted` and use the `_tiStart` response's `_tiD` to catch a field that was already focused when it connected.
- **Which apps work.** RTI is a system text-input service, so it should cover any app using the standard tvOS text field / system keyboard (Search, App Store, most streaming-app search boxes). Apps that draw their own on-screen letter grid without a system text field would not raise an RTI session. This is unverified per app.
- **Get is best-effort.** It returns `contextBeforeInput`, the text before the cursor, so it may not equal the full field contents.
- **Tier placement.** Text input should be a core feature, but the client needs a fallback (D-pad typing) and its own diagnostics, given the tvOS 26.5 focus-event regression.

### Gaps
- **Secure fields.** No source found on whether `_tiStart` returns `_tiD` and accepts `_tiC` for password fields. The iOS Remote can autofill passwords, but no third-party report confirms this works or is blocked.
- No per-app compatibility list (Netflix, YouTube, Disney+, App Store, login screens) for Companion text input on tvOS 26/27.
- No documented minimum tvOS version for Companion RTI. pyatv docs only say Companion must be available. 2022–2023 reports are on tvOS 15–16.
- Root cause and fix status of the tvOS 26.5 / pyatv 0.17.0 focus-event regression (#2853) are unknown.

---

## 4. Apps (list installed apps, icons, launch by bundle ID, deep links/URLs, current foreground app)

### Takeaway
- **List:** `FetchLaunchableApplicationsEvent` returns a dictionary of bundle ID → localized name. It returns **no icons**.
- **Launch:** `_launchApp` takes `_bundleID` or a URL (`_urlS`) for deep links.
- **Icons:** must come from outside, e.g. the iTunes Search/Lookup API, which does not cover built-in apps.
- **Foreground app:** not directly exposed. Only the now-playing app is available, via MRP.
- **Reliability:** launch and app-list failures on tvOS 26.5 and 27.2 beta are reported in 2026.

### Cited Findings
- **App list.** The `FetchLaunchableApplicationsEvent` request with an empty `_c` returns `_c` as `{bundleID: localizedName}`, e.g. `'com.netflix.Netflix': 'Netflix'`, `'com.google.ios.youtube': 'YouTube'`, `'com.apple.TVSettings': 'Inställningar'`. Names come back localized. — [pyatv protocols doc](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md)
- **Launch.** Request: `{'_i': '_launchApp', '_c': {'_bundleID': 'com.netflix.Netflix'}}`. pyatv sends `_urlS` instead of `_bundleID` when the argument is a URL or scheme. — [pyatv protocols doc](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md); [pyatv companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py)
- **Deep links.** Deep-link support in `launch_app` came in pyatv 0.13.0 (@michalmo). Known working links include tv.apple.com, disneyplus.com, play.hbomax.com and netflix.com/title URLs. Ways to find more: share sheets, or `apple-app-site-association` / `CFBundleURLSchemes` inspection. — [pyatv Apps doc](https://pyatv.dev/development/apps/) ([source](https://github.com/postlund/pyatv/blob/master/docs/development/apps.md)); [CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md)
- **Session prerequisite.** `_launchApp` "won't work after the Apple TV has been restarted until the app list has been requested". The theory is that `rapportd` proxies to `com.apple.tvremoteservices` and needs `_sessionStart` (`_srvT` = `com.apple.tvremoteservices`). — [pyatv protocols doc, "Sessions"](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md)
- **No icons.** "Companion only provides localized app names and bundle identifiers". The proposed workaround is querying the iTunes API, which does not include system apps like `com.apple.TVMusic` and needs a supported language/storefront. Issue open since 2021-12-15. — [pyatv issue #1536](https://github.com/postlund/pyatv/issues/1536)
- **tvOS 26.5 launch failure.** On an Apple TV 4K gen 2, tvOS 26.5 (23L471, June 2026), `_launchApp` returned an error (`_ec` 4, `FBSOpenApplicationErrorDomain`, "TransactionalFailure", `_rT` 8) that pyatv silently ignored. `FetchLaunchableApplicationsEvent` and `FetchAttentionState` timed out in the same session. The issue was closed 2026-10-13; the resolution is not stated in the text retrieved. — [pyatv issue #2868](https://github.com/postlund/pyatv/issues/2868)
- **tvOS 26.5 app-list failure on gen 3.** `FetchLaunchableApplicationsEvent` gets no response on an Apple TV 4K gen 3 (AppleTV14,1) on tvOS 26.5, while gen 2 on the same tvOS works. — [pyatv issue #2845](https://github.com/postlund/pyatv/issues/2845)
- **tvOS 27.2 beta.** A report from 2026-10-15 shows intermittent `BlockedStateError: apps is blocked` when launching an app URL after opening another app, alongside repeated Companion `connection_closed`. — [pyatv issue #2928](https://github.com/postlund/pyatv/issues/2928)
- **Now-playing app (MRP only).** `Metadata.app` returns the app "currently playing something" (name + identifier). It is fed by MRP `SET_NOW_PLAYING_CLIENT_MESSAGE` / `SetNowPlayingPlayer`. — [pyatv metadata doc](https://pyatv.dev/development/metadata/) ([source](https://github.com/postlund/pyatv/blob/master/docs/development/metadata.md)); [pyatv mrp/player_state.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/player_state.py)
- **`SupportedActions` event.** The iOS Remote subscribes (`_interest`) to `SupportedActions`, which fires "on major window changes" with `GuideSupportedKey`. — [issue #2325](https://github.com/postlund/pyatv/issues/2325)
- **`PlayMediaEvent`.** This event (`TVRCURLKey`) is seen from the iOS Remote. In testing it only opened the title's page and did not start playback. — [issue #2461](https://github.com/postlund/pyatv/issues/2461); [issue #2325](https://github.com/postlund/pyatv/issues/2325)
- **Home Assistant.** HA launches apps via `media_player.select_source` or the media browser's "Apps" folder, and deep links via `media_player.play_media`. — [HA apple_tv doc (GitHub source)](https://github.com/home-assistant/home-assistant.io/blob/current/source/_integrations/apple_tv.markdown)

### Inferences
- **Launcher tier.** App list + launch by bundle ID + deep-link URLs is feasible and high-value. The client must send `_sessionStart` first, check `_launchApp` responses for `_ec`/`_eu` errors (pyatv itself did not until at least June 2026), and retry or fall back to the app list on tvOS 26.5+.
- **Icons.** Bundle a small set of icons for Apple system apps (licensing permitting) and look up third-party apps via the iTunes Lookup API by bundle ID and country.
- **"Current foreground app".** Approximate it with MRP's now-playing client. It will be empty when nothing is playing (e.g., browsing a menu).

### Gaps
- No Companion command was found that returns app icons, app versions or the true foreground app.
- No root cause or fix has been published for `_launchApp` `FBSOpenApplicationErrorDomain` code 4 on tvOS 26.5. No data on tvOS 27 behavior.

---

## 5. Now playing (metadata, artwork, position, state, shuffle/repeat, seek, queue; third-party vs Apple apps)

### Takeaway
Full now-playing data comes from **MRP**, which since tvOS 15 is tunneled inside AirPlay 2 and needs AirPlay (HAP) pairing:
- **State and metadata:** `SetStateMessage` (playback state, supported commands, playback queue with content-item metadata), plus `SetNowPlayingClient`/`SetNowPlayingPlayer`.
- **Artwork:** `PlaybackQueueRequestMessage`.
- **Controls:** `SendCommand` for seek, shuffle and repeat.

Companion offers only the `_iMC` capability flags in pyatv. tvOS 18+ also has a richer, NSKeyedArchiver-based Companion `NowPlayingInfo`/`FetchUpNextInfoEvent` channel used by Apple's Remote, which is not implemented in pyatv. Data quality depends on what each app publishes.

### Cited Findings
- **Field support by protocol** "(assuming app sets metadata correctly)": MRP: all fields; DMAP: all except series/season/episode; **AirPlay and Companion: none**. — [pyatv metadata doc](https://github.com/postlund/pyatv/blob/master/docs/development/metadata.md)
- **Feature names pyatv exposes for now playing:** Title, Artist, Album, Genre, TotalTime, Position, Shuffle, Repeat, SeriesName, SeasonNumber, EpisodeNumber, ContentIdentifier, Artwork, App, SetPosition, SetShuffle, SetRepeat, PushUpdates. — [pyatv const.py](https://github.com/postlund/pyatv/blob/master/pyatv/const.py)
- **MRP message handling.** pyatv handles `SET_STATE_MESSAGE` (playbackState, supportedCommands, playbackQueue), `SET_NOW_PLAYING_CLIENT_MESSAGE` and `SET_NOW_PLAYING_PLAYER_MESSAGE`.
  - **Artwork:** fetched with `playback_queue_request(location, width, height)`.
  - **Seek:** `SendCommand` `SeekToPlaybackPosition` (45).
  - **Shuffle/repeat:** `ChangeShuffleMode` (47) / `ChangeRepeatMode` (46).

  — [pyatv mrp/player_state.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/player_state.py); [pyatv mrp/messages.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/messages.py); [CommandInfo.proto](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/protobuf/CommandInfo.proto)
- **Queue-editing commands are defined but unused.** MRP defines `InsertIntoPlaybackQueue` (52), `ReorderPlaybackQueue` (55), `RemoveFromPlaybackQueue` (56), `PlayItemInPlaybackQueue` (57) and `SetPlaybackQueue` (48), but pyatv exposes no queue-editing API. — [CommandInfo.proto](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/protobuf/CommandInfo.proto); [pyatv const.py](https://github.com/postlund/pyatv/blob/master/pyatv/const.py)
- **MRP requires AirPlay since tvOS 15.** MRP "was demoted from a separate protocol … and moved to a special stream type in AirPlay 2". "Devices running tvOS 15 (beta or later) require AirPlay to be set up to function properly." — [pyatv Supported Features](https://pyatv.dev/documentation/supported_features/) ([source](https://github.com/postlund/pyatv/blob/master/docs/documentation/supported_features.md))
- **Newer Companion now-playing channel (tvOS 18 / iOS 18 era Remote):**
  - Subscribe with `_interest` to `NowPlayingInfo`, then send `FetchCurrentNowPlayingInfoEvent`. The `NowPlayingInfo` event arrives a few seconds later (`NowPlayingInfoKey`, a bplist/NSKeyedArchiver `TVRCNowPlayingInfo` with title, episode, artwork template, timing, `imageData`, and "timed metadata" for InSight cast/music).
  - `FetchUpNextInfoEvent` returns a paginated (`PaginationTokenKey`, `UpNextInfosKey`) Up Next list of ~22 KB in ~1.2 s.
  - Netflix also populates this channel.
  - The maintainer wants it mapped into the stable API; nothing was merged as of the last activity.

  — [pyatv issue #2461](https://github.com/postlund/pyatv/issues/2461); [issue #2325](https://github.com/postlund/pyatv/issues/2325)
- **`NowPlayingInfo` caveat.** `playbackRate` 1.0 vs 0.0 indicates play/pause but "does not guarantee media is playing (e.g., Netflix previews)". — [issue #2325](https://github.com/postlund/pyatv/issues/2325)
- **Apps decide which commands work.** "The tvOS apps themselves decide what commands they support and when they support them". If a command fails, first check it against Apple's iOS Remote app. — [HA apple_tv doc (GitHub source)](https://github.com/home-assistant/home-assistant.io/blob/current/source/_integrations/apple_tv.markdown)
- **Long-standing metadata bugs:** artwork/title not updating on app change (#1900, 2023), push updates breaking when seeking (#1934), position errors when polling (#2321). — [pyatv issue search](https://github.com/postlund/pyatv/issues?q=metadata)
- **Reverse-engineered MRP definitions.** An open PR (2026-03-17) adds "Reverse-engineered MRP protobuf definitions from Apple shared libraries". — [pyatv PR #2834](https://github.com/postlund/pyatv/pull/2834)

### Inferences
- **Two pairings needed for full now playing.** A full now-playing screen (artwork, progress bar, seek, shuffle/repeat) requires both Companion pairing and AirPlay pairing (for the MRP tunnel). A Companion-only client gets buttons, capability flags (`_iMC`) and volume, but no titles or artwork unless it implements the newer `NowPlayingInfo` channel itself.
- **Apple apps vs third-party apps.** Apple apps (TV, Music) publish complete MRP metadata. Third-party apps vary in title, artwork and position completeness and in which commands they accept. Netflix is confirmed only for the Companion NowPlayingInfo channel.
- **Treat as advanced tier:** queue browsing/editing, Up Next, and InSight.

### Gaps
- No systematic, current (tvOS 26/27) per-app matrix of MRP metadata completeness (Netflix, YouTube, Disney+, Prime Video, etc.).
- No public decoder or spec of the `TVRCNowPlayingInfo` / Up Next archive schema beyond the issue examples.

---

## 6. Power (on/off, power-state detection, waking from deep sleep, HDMI-CEC)

### Takeaway
- **Off and on:** Companion `_hidC` 12 (Sleep) turns the box off and `_hidC` 13 (Wake) turns it on; MRP's alternative is `WakeDeviceMessage`.
- **Power state:** `FetchAttentionState` plus pushed `SystemStatus`/`TVSystemStatus` events. The states are 1=asleep, 2=screensaver, 3=awake, 4=idle.
- **Breakage and fixes:** state detection broke repeatedly in 2025–2026 (tvOS 18.4 "No request handler"; Apple TV 4K gen 3 on 26.5 silently dropping requests). Fixes were the `TVRCSessionStart` registration, a non-null `_i` in `_systemInfo`, and subscribing even when the initial fetch fails (pyatv 0.18.0, 2026-06-19).
- **Deep sleep:** waking a box that has dropped off the network is unreliable. It depends on the Bonjour Sleep Proxy, and plain Wake-on-LAN did not work in one user report.

### Cited Findings
- **pyatv Companion power code.** `turn_on` sends `_hidC` Wake (13) and `turn_off` sends `_hidC` Sleep (12). Each is sent as a single "up" event, and `await_new_state` is "not supported by Companion yet". — [pyatv companion/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/__init__.py)
- **pyatv MRP power code.** `turn_on` sends `WAKE_DEVICE_MESSAGE`. `turn_off` does Home-hold, then Select (navigating the Control Center power button). Power state comes from `DeviceInfo.logicalDeviceCount` (≥1 on, 0 off). pyatv docs call MRP power management "not very robust (relies on navigating and pressing buttons)" and recommend Companion. — [pyatv mrp/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/__init__.py); [Supported Features](https://github.com/postlund/pyatv/blob/master/docs/documentation/supported_features.md)
- **MRP `turn_off` broke on tvOS 18.** It focuses the power-off button but does not click it (opened 2024-09-17, open). — [pyatv issue #2502](https://github.com/postlund/pyatv/issues/2502)
- **System status protocol.** `FetchAttentionState` → `{'state': n}`, and the `SystemStatus` event registered via `_interest`. The Apple TV only pushes `SystemStatus`/`TVSystemStatus` to a client that sent a non-null `_i` in `_systemInfo`. A client should send `TVRCSessionStart {'ProtocolVersionKey': '1.2'}` after `_sessionStart`, because "Some Apple TVs will not answer commands such as `FetchAttentionState` until this session has been registered". — [pyatv protocols doc](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md); [companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py)
- **When it arrived.** "proper standby detection via Companion … SystemStatus" was added in pyatv 0.13.0 (2023-06-15). The iOS Remote uses `TVSystemStatus` instead of `SystemStatus` (Mar 2024 observation). — [CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md); [issue #2325](https://github.com/postlund/pyatv/issues/2325)
- **tvOS 18.4 breakage.** `FetchAttentionState` → "Command failed: No request handler" (Mar 2025). — [pyatv issue #2648](https://github.com/postlund/pyatv/issues/2648)
- **tvOS 26.5 on Apple TV 4K gen 3.** `FetchAttentionState` (and the app list) silently get no response and the connection drops, while gen 2 on the same build works. A community monkey-patch adding `TVRCSessionStart` worked for testers. Multiple simultaneous clients still caused random disconnects. — [pyatv issue #2845](https://github.com/postlund/pyatv/issues/2845)
- **Fix PRs.** "Fix Companion Protocol Power State Handling" (PR #2855) merged 2026-05-25. "companion: keep power-state subscription when initial fetch fails (tvOS 26.x)" (PR #2870) merged 2026-06-19 and shipped in 0.18.0. pyatv 0.16.1 (2025-07-12) fixed "Connection issues with tvOS 18.4+" by using a static `_i` in `_systemInfo`. — [PR #2855](https://github.com/postlund/pyatv/pull/2855); [PR #2870](https://github.com/postlund/pyatv/pull/2870); [CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md)
- **tvOS 27.2 beta.** Companion `connection_closed` repeats every ~0–10 s even with pyatv 0.18.0 (2026-10-15, open). — [pyatv issue #2928](https://github.com/postlund/pyatv/issues/2928)
- **Deep sleep detection.** "When a device is in deep sleep mode, another node on the network called a sleep proxy will announce its presence. It will also wake up the device using Wake-On-LAN in case a particular service is requested." pyatv flags `deep_sleep` from scan results and calls this experimental. — [pyatv concepts doc](https://github.com/postlund/pyatv/blob/master/docs/documentation/concepts.md)
- **Standby dropout.** Apple TVs on Ethernet drop off the network in standby (ping fails after a few minutes). HA `turn_on` worked early in standby but later failed. A `wakeonlan` magic packet to the MAC did not wake it. The Bluetooth remote and AirPlay from an iPhone did wake it.
  - Workaround: reload the HA integration when an HDMI-CEC-linked receiver or a smart plug detects power-on.
  - The reporter concluded the cause was the device (heavy DHCP renewal), not pyatv, and closed the issue 2026-05-28.

  — [pyatv issue #2369](https://github.com/postlund/pyatv/issues/2369)
- **Home Assistant behavior.** HA's on/off button "only toggles the power state in Home Assistant". HA suggests a script using the quick-action menu (tvOS 14.0+) and `suspend`. On "Is it possible to see if a device is on without interacting with it" the FAQ says "No". This FAQ text may predate the Companion SystemStatus support. — [HA apple_tv doc (GitHub source)](https://github.com/home-assistant/home-assistant.io/blob/current/source/_integrations/apple_tv.markdown)
- **Wake service in `_stA`.** `com.apple.devicediscoveryui.rapportwake` appears in the `_stA` service list returned by a tvOS 26.5 Apple TV. — [pyatv issue #2868](https://github.com/postlund/pyatv/issues/2868)

### Inferences
- **Sleep and wake while reachable.** `_hidC` 12/13 is the right path. Since tvOS 26 the client must also send `TVRCSessionStart`, a stable non-null `_i`, and subscribe to both `SystemStatus` and `TVSystemStatus`.
- **Waking from deep sleep.** The Android app must first re-resolve the device via mDNS, which lets the sleep proxy wake it. Then it reconnects Companion and sends Wake. A WoL magic packet can be tried as a best-effort fallback but should not be relied on.
- **HDMI-CEC.** Whether the TV turns on with the Apple TV depends on the Apple TV's CEC setting, not the protocol.
- **Power is a core feature with caveats.** Show power state as "unknown" when the snapshot fails, and rely on pushed events.

### Gaps
- No documented Companion/Rapport-level "wake" handshake (e.g., what `rapportwake` does) for waking a deep-sleeping Apple TV from a non-Apple device.
- No primary source on HDMI-CEC "Control TVs and Receivers" interaction with remote wake/sleep commands.
- No confirmed root cause for the Apple TV 4K gen 3-specific failure, and no data on tvOS 27 behavior after the 0.18.0 fixes.

---

## 7. Volume (up/down/mute, absolute volume, CEC/IR vs AirPlay/HomePod output, readback)

### Takeaway
Relative volume (`_hidC` 8/9, or the MRP HID consumer page 0xE9/0xEA) works whenever the Apple TV controls volume. With HDMI-CEC volume control, only up/down works. With IR-learned volume, nothing works remotely.

Absolute set and readback (Companion `_mcc` 5/6 with `_vol` 0–1, or MRP `SetVolumeMessage` per output device with `VolumeDidChange` push) work only when the output is a HomePod/AirPlay device. There is no explicit mute command in pyatv.

### Cited Findings
- **HA FAQ by audio output:**
  - Connected to a HomePod or HomePod stereo pair: "all volume controls should work".
  - TV speakers with volume control over HDMI-CEC: "only volume up/down controls will work".
  - Volume over IR: "volume cannot be controlled remotely through the Apple TV".

  — [HA apple_tv doc (GitHub source)](https://github.com/home-assistant/home-assistant.io/blob/current/source/_integrations/apple_tv.markdown)
- **Companion volume.** Volume up/down are `_hidC` 8/9. Absolute volume uses `_mcc` 6 SetVolume `{'_vol': level/100}`, and readback uses `_mcc` 5 GetVolume → `_c._vol`. pyatv re-reads volume when the `_iMC` event's `_mcF` has the Volume bit (0x100); when the bit is absent, volume is reported as 0 ("we know nothing about the volume"). — [pyatv companion/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/__init__.py); [protocols doc](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md)
- **MRP volume.** `set_volume` sends `SET_VOLUME_MESSAGE` with an output-device UID and level 0–1. `volume_up`/`down` use HID keys when relative control is available, or step ±5 when only absolute is. MRP protobufs include `VolumeControlAvailabilityMessage`, `VolumeControlCapabilitiesDidChangeMessage`, `VolumeDidChangeMessage` and `GetVolumeMessage`. — [pyatv mrp/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/__init__.py); [mrp/protobuf](https://github.com/postlund/pyatv/tree/master/pyatv/protocols/mrp/protobuf)
- **Per-device volume.** "Added ability to set volume per (connected) device (#2673)" shipped in pyatv 0.17.0 (2026-01-21). — [CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md)
- **Earlier volume fix.** "Absolute/relative volume management is greatly improved" in 0.13.0 (2023-06-15). — [CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md)
- **Mute.** A user emulates mute with `set_volume(0)` but cannot restore the previous level (open, Nov 2024). The iOS 17.3 Remote capture lists `_hidC` 18 as "mute", which conflicts with pyatv's "Page Up". — [pyatv issue #2555](https://github.com/postlund/pyatv/issues/2555); [issue #2325](https://github.com/postlund/pyatv/issues/2325)

### Inferences
- **Show the right controls per setup.** Use the `_iMC` Volume flag and MRP volume-capability messages. Show a slider only when absolute volume is available (HomePod/AirPlay output). Otherwise show ± buttons only, and with IR-only setups expect no effect.
- **Mute.** Implement mute client-side (remember the level, then set 0) for HomePod outputs. Test `_hidC` 18 experimentally for CEC setups.

### Gaps
- No confirmation of an Apple-native mute `_hidC` code on tvOS 26/27.
- No data on volume readback accuracy for HDMI-eARC/CEC setups on tvOS 26/27.

---

## 8. User accounts (list profiles, switch user)

### Takeaway
Companion `FetchUserAccountsEvent` lists tvOS user profiles as account ID → name, and `SwitchUserAccountEvent {SwitchAccountID}` switches users. pyatv has supported this since 0.11.0 (May 2023). No 2025–2026 breakage reports were found.

### Cited Findings
- **Request formats.** `account_list()` sends `FetchUserAccountsEvent` with an empty `_c`. The response `_c` is a dict `{account_id: name}`. `switch_account(id)` sends `SwitchUserAccountEvent` with `{"SwitchAccountID": account_id}`. — [pyatv companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py); [companion/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/__init__.py)
- **When it arrived.** "account switching (using Companion protocol) is now also possible thanks to @michalmo" landed in pyatv 0.11.0 (2023-05-15). The features are AccountList and SwitchAccount. — [CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md); [const.py](https://github.com/postlund/pyatv/blob/master/pyatv/const.py)
- **tvOS 27 parental controls.** Reportedly lets parents block individual apps for specific profiles (from a search-result summary; the MacRumors pages could not be fetched). — [MacRumors, "Apple Releases tvOS 27" (2026-09-14)](https://www.macrumors.com/2026/09/14/apple-releases-tvos-27/)

### Inferences
- **Tier.** A profile switcher is a mid-tier feature that is cheap to implement (two request/response messages).

### Gaps
- No source on whether account avatars are retrievable, or whether switching works on tvOS 27 with the new per-profile app restrictions.

---

## 9. Other extras (screensaver, audio output selection, AirPlay to TV, Find My Remote, Control Center, system status, restart, notifications, new in tvOS 18/26/27)

### Takeaway
The confirmed extras are:
- **Screensaver** (`_hidC` 11).
- **Control Center / Guide** (`_hidC` 19/17).
- **Audio output selection** (MRP output-device add/remove/set).
- **AirPlay streaming** of URLs and audio from the client (AirPlay/RAOP).
- **Captions** get/set (`_mcc` 12/13).
- **Find My Siri Remote** (`FetchSiriRemoteInfo` / `ToggleFindingMode`): reverse-engineered but not in pyatv.
- **Screensaver/awake status** (SystemStatus).
- **Up Next / InSight** (tvOS 18 Companion).

I found **no** known command for restart or notifications.

### Cited Findings
- **Screensaver.** "Support for activating screensaver" (Companion) shipped in pyatv 0.13.0. It is `_hidC` 11, and FeatureName.Screensaver exists. — [CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md); [protocols doc](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md)
- **Screensaver status is useful in practice.** Screensaver state (SystemStatus 2) matters because "when screensaver is active a first (any) command has to be sent in order to exit screensaver, and then the command can be processed". — [pyatv issue #2648](https://github.com/postlund/pyatv/issues/2648)
- **Audio output devices.** "Added support for changing output devices in tvOS (@michalmo)" shipped in 0.13.0 (PR #2044). The API offers `output_devices`, `add_output_devices`, `remove_output_devices` and `set_output_devices`, via MRP `ModifyOutputContextRequestMessage`, with updates from `DeviceInfo`/`UpdateOutputDevice`. Changing the *default* output (e.g., one HomePod pair to another for eARC) remained unanswered (open, Mar 2025). — [CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md); [mrp/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/__init__.py); [issue #2638](https://github.com/postlund/pyatv/issues/2638); [issue #778](https://github.com/postlund/pyatv/issues/778)
- **AirPlay play URL / streaming.** pyatv supports "AirPlay play URL (including local files)" and audio streaming via RAOP/AirPlay 2. Open tvOS 26 issue: `play_url` fails with HTTP 500 on `/playback-info` (2026-10-22). Remote-control commands do not work over RAOP alone (#1068). — [Supported Features](https://github.com/postlund/pyatv/blob/master/docs/documentation/supported_features.md); [issue #2906](https://github.com/postlund/pyatv/issues/2906); [issue #1068](https://github.com/postlund/pyatv/issues/1068)
- **Find My Siri Remote.**
  - `FetchSiriRemoteInfo` returns `SiriRemoteInfoKey` (a bplist `TVRCSiriRemoteInfo` with name, model, serial, Bluetooth MAC, IRK, firmware and connection state).
  - `_interest`/`PushSiriRemoteInfo` gives updates.
  - `ToggleFindingMode` (`FindingModeEnabledKey` `'YES'`/`'NO'`) turns finding mode on or off.
  - Captured from the iOS 17.3 Remote (Jan 2024); not implemented in pyatv.

  — [pyatv issue #2325](https://github.com/postlund/pyatv/issues/2325)
- **Captions.** `_mcc` 12 GetCaptionSettings returns `_mcs` 2 (on) or 3 (off). `_mcc` 13 SetCaptionSettings toggles them but "had no visible effect in Netflix, YouTube, or Zee". — [issue #2325](https://github.com/postlund/pyatv/issues/2325)
- **Fast-forward/rewind.** Supported by `_mcc` 8–11 (Begin/End FF/Rewind) and MRP commands 9–12. — [companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py); [CommandInfo.proto](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/protobuf/CommandInfo.proto)
- **Companion services on a tvOS 26.5 Apple TV.** `_systemInfo` `_stA` lists `com.apple.tvremoteservices`, `com.apple.bluetooth.remote`, `com.apple.devicediscoveryui.rapportwake`, `com.apple.siri.wakeup`, `com.apple.workflow.remotewidgets`, `com.apple.announce`, `com.apple.Seymour`/`SeymourSession` (Fitness), `com.apple.home.messaging` and others. — [pyatv issue #2868](https://github.com/postlund/pyatv/issues/2868)
- **Unused MRP messages.** MRP protobufs include `NotificationMessage`, `SetHiliteModeMessage`, `RegisterForGameControllerEventsMessage`, `RegisterHIDDeviceMessage` and `AudioFadeMessage`; pyatv exposes no API for them. — [mrp/protobuf directory](https://github.com/postlund/pyatv/tree/master/pyatv/protocols/mrp/protobuf)
- **tvOS 27 (released 2026-09-14), from search-result summaries only.** It brings a "more responsive Control Center" and a new Apple TV Remote app for iOS 27 where Siri can be used to search, control playback or ask about a show. The AI-powered Siri is not in tvOS 27. — [MacRumors, "Apple Releases tvOS 27"](https://www.macrumors.com/2026/09/14/apple-releases-tvos-27/); [MacRumors tvOS 27 release notes (2026-09-25)](https://www.macrumors.com/2026/09/25/tvos-27-release-notes/)
- **New hardware.** Issues from Nov 2026 reference an "Apple HomeView (HomeAccessory17,1, software 27.2)" whose AirPlay pairing never shows a PIN. This indicates new tvOS/HomeOS-27 based hardware that third-party clients are starting to encounter. — [pyatv issue #2951](https://github.com/postlund/pyatv/issues/2951); [issue #2950](https://github.com/postlund/pyatv/issues/2950)

### Inferences
- **Cheap extras worth adding:** screensaver, Control Center, Guide, captions toggle, FF/rewind, skip-by-N-seconds. Each is a single `_hidC`/`_mcc` message.
- **Find My Remote** is feasible but undocumented outside one issue thread, so it is experimental.
- **Audio output switching** requires the MRP-over-AirPlay tunnel.
- **No restart or notifications.** No restart command or notification feed is known, so treat both as not possible.

### Gaps
- No source describing a remote "restart" command, a notifications feed, or game-controller emulation used by third-party clients.
- No reverse-engineering yet of what changed in the tvOS 27 / iOS 27 Remote app at the protocol level. MacRumors pages could not be fetched in this session (network allowlist), so tvOS 27 details are from search snippets only.

---

## 10. What pyatv supports today (v0.18.0, 2026-06-19), by protocol, and what is unsupported and why

### Takeaway
- **Companion:** apps, accounts, power, navigation/HID buttons, keyboard and touch.
- **MRP (via AirPlay tunnel on tvOS 15+):** metadata, artwork, playback commands, shuffle/repeat/seek, absolute volume and output devices.
- **AirPlay/RAOP:** streaming.

Not supported: Siri/voice, app icons, the tvOS 18 rich now-playing/Up Next channel, Find My Remote, captions, and queue editing. The reasons are protocol knowledge gaps, the maintainer's insistence on a stable abstracted API, and limited maintainer time.

### Cited Findings
- **Feature list.** "Supported Features" lists: discovery, device metadata, push updates, pairing, feature listing, AirPlay play URL/streaming, playback controls, navigation controls, tap/double-tap/hold input actions, artwork, now-playing metadata, playing app, media type/state, set position, shuffle/repeat, volume, power management, deep-sleep detection, launch app, list installed apps, storage. — [pyatv Supported Features](https://pyatv.dev/documentation/supported_features/)
- **Companion section** of that page: pairing (only if the target can show a PIN, so Apple TV but not HomePod/Mac), the app interface, turn on/off, remote control, "Touch gestures and click". Its note still reads "Early stage of development - not many features supported", which is outdated relative to the code. — [Supported Features (source)](https://github.com/postlund/pyatv/blob/master/docs/documentation/supported_features.md)
- **Feature constants Companion declares in code:** AppList, LaunchApp, AccountList, SwitchAccount, PowerState, TurnOn, TurnOff, Up/Down/Left/Right, Select, Menu, Home, VolumeUp/Down, PlayPause, ChannelUp/Down, Screensaver, Guide, ControlCenter, TextFocusState/TextGet/TextClear/TextAppend/TextSet, Swipe, Action, Click. It also maps Play/Pause/Next/Previous/Volume/SetVolume/SkipForward/SkipBackward to `_iMC` flags. — [companion/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/__init__.py)
- **MRP section:** pairing, metadata, push updates, remote control with input actions, artwork, playing metadata, device/playback state, shuffle/repeat, volume, current playing app, power management (not robust). On tvOS 15+ MRP requires AirPlay. — [Supported Features (source)](https://github.com/postlund/pyatv/blob/master/docs/documentation/supported_features.md)
- **Full FeatureName list in v0.18.0** (67 entries, with values up to 68). It includes HomeHold, TopMenu, Suspend, WakeUp, OutputDevices, Add/Remove/SetOutputDevices, PlayUrl and StreamFile. There is no Siri, Mute, icon, Up Next or FindRemote feature. — [pyatv const.py](https://github.com/postlund/pyatv/blob/master/pyatv/const.py)
- **Why some features are missing:**
  - **Rich now playing:** the maintainer said "Raw protocol values should never be exposed" and wants a stable mapped API for the tvOS 18 now-playing data (issue open since Jul 2024).
  - **Icons:** need an external iTunes lookup and raise copyright concerns (#1536).
  - **Siri:** the packet format and gating are still under investigation (#2875).

  — [issue #2461](https://github.com/postlund/pyatv/issues/2461); [issue #1536](https://github.com/postlund/pyatv/issues/1536); [issue #2875](https://github.com/postlund/pyatv/issues/2875)
- **Release cadence.** The 0.18.0 notes say: "overdue changes needed for compatibility with newer versions of tvOS. More things will come… Due to my lack of free time, please feel free to send me PRs". — [CHANGES.md](https://github.com/postlund/pyatv/blob/master/CHANGES.md)
- **Pairing failures on tvOS 26.x/27.x.** Recent reports: Companion pairing silently fails on tvOS 26.3 with no PIN displayed (#2837, open, Nov 2026), and AirPlay `pair-pin-start` times out on tvOS 26.6 (#2908). Pairing is covered by another researcher, but these block every feature above. — [issue #2837](https://github.com/postlund/pyatv/issues/2837); [issue #2908](https://github.com/postlund/pyatv/issues/2908)

### Inferences
- **Reference implementation.** pyatv 0.18.0 (or its master) is still the best reference for message formats. An Android client should port the Companion flow exactly in this order:
  1. `_systemInfo` with a stable `_i`
  2. `_touchStart`
  3. `_sessionStart`
  4. `TVRCSessionStart`
  5. `_tiStart`
  6. `_interest` `_iMC` (+ SystemStatus/TVSystemStatus)

  The client should also handle error responses (`_em`/`_ec`/`_eu`), which pyatv historically swallowed.
- **Concurrent clients.** Multiple simultaneous Companion clients to one Apple TV are associated with disconnects (#2845, #2928). The Android app should hold one connection and reconnect gracefully.

### Gaps
- No pyatv release after 0.18.0 had shipped as of 2026-11-14, so tvOS 27 compatibility fixes, if any, are unreleased or not yet made.

---

## 11. Consolidated feature matrix and proposed tiers for an Android Apple TV remote

### Takeaway
A Companion-only client (one pairing) delivers a complete "remote + keyboard + launcher + power" experience. Adding AirPlay pairing (for the MRP tunnel) unlocks the now-playing screen, absolute volume and output switching. Siri voice, app icons from the device, a true foreground app, restart and notifications are not achievable as of Nov 2026.

### Cited Findings
Every row is backed by the sources cited in sections 1–10. The key per-row source is given in the last column.

| Feature | Possible? | Protocol / message | Known limitations / reliability (dated) | Key source |
|---|---|---|---|---|
| D-pad, Select, Menu, Home | Yes | Companion `_hidC` 1–7 (`_hBtS` 1/2); MRP HID (1,0x8C..0x89/0x86, 12,0x40) | Stable; tap/double/hold by timing | [protocols.md](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md) |
| Top Menu / TV button | Yes | MRP HID (12,0x60); Companion Home `_hidC` 7 | — | [mrp/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/__init__.py) |
| App switcher (double Home) | Likely | `_hidC` 7 ×2 | No public confirmation for tvOS 26/27 | [companion/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/__init__.py) |
| Control Center | Yes | `_hidC` 19 (pyatv "PageDown") | Added pyatv 0.17.0 (Jan 2026); `home_hold` unreliable (#2286) | [PR #2699](https://github.com/postlund/pyatv/pull/2699) |
| Guide / Channel ± | Yes | `_hidC` 17 / 15, 16 | Guide only meaningful in live-TV apps (`SupportedActions` `GuideSupportedKey`) | [issue #2325](https://github.com/postlund/pyatv/issues/2325) |
| Play/Pause, Next/Prev | Yes | `_hidC` 14; `_mcc` 1–4; MRP SendCommand 1–6 | App-dependent support | [HA doc](https://github.com/home-assistant/home-assistant.io/blob/current/source/_integrations/apple_tv.markdown) |
| Skip ±N s, FF/RW | Yes | `_mcc` 7 `_skpS`; `_mcc` 8–11; MRP 18/19, 9–12 | `_iMC` flags show availability | [companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py) |
| Siri button press | Partial | `_hidC` 10 | Not exposed by pyatv | [api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py) |
| Siri voice / dictation | No (Nov 2026) | `_siriStart`/`_siA`/`_siriStop` | Replay opens the UI but speech is not processed; can soft-lock | [issue #2875](https://github.com/postlund/pyatv/issues/2875) |
| Touchpad swipe/drag/click | Yes | `_touchStart` + `_hidT` (`_tPh` 1/3/4/5) + `_hidC` 6 | ~16–20 ms cadence; edge re-press; pyatv 0.15.0 | [protocols.md](https://github.com/postlund/pyatv/blob/master/docs/documentation/protocols.md) |
| Keyboard focus notification | Yes | `_tiStart` response / `_tiStarted` / `_tiStopped` | Focus events not firing on pyatv 0.17.0 + tvOS 26.5 (#2853) | [issue #2853](https://github.com/postlund/pyatv/issues/2853) |
| Get/set/append/clear text | Yes | `_tiC` with RTI `_tiD` (NSKeyedArchiver) | Secure fields unknown | [companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py) |
| App list | Yes | `FetchLaunchableApplicationsEvent` | No icons; timeouts on 4K gen 3 + tvOS 26.5 | [issue #2845](https://github.com/postlund/pyatv/issues/2845) |
| Launch app / deep link | Yes | `_launchApp` `_bundleID` / `_urlS` | Needs `_sessionStart`; FBSOpenApplication error 4 on tvOS 26.5 (#2868); "apps is blocked" on 27.2 beta (#2928) | [issue #2868](https://github.com/postlund/pyatv/issues/2868) |
| App icons | Not from device | iTunes Lookup API (external) | No system-app coverage | [issue #1536](https://github.com/postlund/pyatv/issues/1536) |
| Now-playing app | Yes (MRP) | `SetNowPlayingClient` | Only the playing app, not the foreground app | [metadata.md](https://github.com/postlund/pyatv/blob/master/docs/development/metadata.md) |
| Title/artist/album/artwork/position/state | Yes (MRP) | `SetStateMessage`, `PlaybackQueueRequest` | Needs AirPlay pairing (tvOS 15+); app-dependent | [Supported Features](https://github.com/postlund/pyatv/blob/master/docs/documentation/supported_features.md) |
| Seek, shuffle, repeat | Yes (MRP) | SendCommand 45, 47, 46 | App-dependent | [CommandInfo.proto](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/mrp/protobuf/CommandInfo.proto) |
| Queue edit / Up Next / InSight | Experimental | MRP 48/52/55–57; Companion `FetchUpNextInfoEvent`, `NowPlayingInfo` | Not in pyatv | [issue #2461](https://github.com/postlund/pyatv/issues/2461) |
| Sleep / wake | Yes | `_hidC` 12 / 13; MRP `WakeDevice` | Deep-sleep wake unreliable (#2369) | [companion/__init__.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/__init__.py) |
| Power/screensaver state | Yes, fragile | `FetchAttentionState`; `SystemStatus`/`TVSystemStatus` events; needs `TVRCSessionStart` | Broke on 18.4 and 26.5; fixed in pyatv 0.18.0 (Jun 2026) | [PR #2870](https://github.com/postlund/pyatv/pull/2870) |
| Volume ± | Yes (if ATV controls volume) | `_hidC` 8/9; MRP HID 12,0xE9/0xEA | IR setups: none | [HA doc](https://github.com/home-assistant/home-assistant.io/blob/current/source/_integrations/apple_tv.markdown) |
| Absolute volume + readback | HomePod/AirPlay output only | `_mcc` 5/6 `_vol`; MRP `SetVolume` | CEC: no absolute | [HA doc](https://github.com/home-assistant/home-assistant.io/blob/current/source/_integrations/apple_tv.markdown) |
| Mute | No native (emulate) | (maybe `_hidC` 18, unverified) | — | [issue #2555](https://github.com/postlund/pyatv/issues/2555) |
| Audio output selection | Yes (MRP) | `ModifyOutputContextRequest` add/remove/set | Default output change unknown | [CHANGES 0.13.0](https://github.com/postlund/pyatv/blob/master/CHANGES.md) |
| User profiles list/switch | Yes | `FetchUserAccountsEvent` / `SwitchUserAccountEvent` | — | [companion/api.py](https://github.com/postlund/pyatv/blob/master/pyatv/protocols/companion/api.py) |
| Screensaver trigger | Yes | `_hidC` 11 | — | [CHANGES 0.13.0](https://github.com/postlund/pyatv/blob/master/CHANGES.md) |
| Captions toggle | Experimental | `_mcc` 12/13 | No visible effect in Netflix/YouTube | [issue #2325](https://github.com/postlund/pyatv/issues/2325) |
| Find My Siri Remote | Experimental | `FetchSiriRemoteInfo`, `ToggleFindingMode` | Not in pyatv | [issue #2325](https://github.com/postlund/pyatv/issues/2325) |
| AirPlay URL / audio from phone | Yes | AirPlay `play_url`; RAOP/AirPlay 2 | `play_url` HTTP 500 on tvOS 26 (#2906) | [issue #2906](https://github.com/postlund/pyatv/issues/2906) |
| Restart, notifications, foreground app | Not found | — | — | (gap) |

### Inferences
Proposed tiers (synthesis):
- **Tier 1, "Basic remote" (Companion pairing only).** D-pad, Select, Menu, Home, Play/Pause, Volume ±, Control Center, Screensaver, Sleep/Wake, power state.
- **Tier 2, "Smart remote" (still Companion only).** Touchpad, keyboard with auto-popup on `_tiStarted`, app list + launch + deep links (with externally sourced icons), profile switching, skip ±N s, Guide/Channel ±.
- **Tier 3, "Media hub" (add AirPlay pairing for MRP).** Now playing with artwork, progress/seek, shuffle/repeat, absolute volume slider with readback (HomePod outputs), audio output picker, AirPlay "send URL to TV".
- **Tier 4, "Experimental / labs".** Up Next, InSight and the rich `NowPlayingInfo` channel, Find My Remote, captions toggle, queue editing, mute via `_hidC` 18, deep-sleep wake via mDNS/WoL.
- **Not feasible as of Nov 2026.** Siri voice, device-provided app icons, true foreground app, restart, notifications.

Each tier should degrade gracefully, given the tvOS 26.5/27.2-beta regressions. Use `_iMC` flags and SystemStatus to decide what to show, and expect per-model differences (Apple TV 4K gen 3 vs gen 2).

### Gaps
- No tvOS 27-specific protocol analysis exists yet. All tier assignments should be re-verified on tvOS 27.x hardware, especially Apple TV 4K gen 3 and the new HomeAccessory17,1 devices.
