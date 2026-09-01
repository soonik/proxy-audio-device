# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this project is

Proxy Audio Device is a macOS **HAL virtual audio driver** (an AudioServerPlugIn) that publishes a
fake output device. Everything written to it is forwarded to a real ("proxied") output device. Its
purpose is to let the macOS system volume controls (menu bar slider, volume keys) attenuate audio
going to external interfaces that don't expose their own volume control — the driver applies the
volume/mute in software before handing samples to the real device.

Two products are built from one Xcode project:

| Target | Product | Language | Runs in |
| --- | --- | --- | --- |
| `ProxyAudioDevice` | `ProxyAudioDevice.driver` (bundle, `WRAPPER_EXTENSION = driver`) | C++ | `coreaudiod`, as root |
| `Proxy Audio Device Settings` | `Proxy Audio Device Settings.app` | Objective-C / Objective-C++ | Normal user session |

The driver is installed to `/Library/Audio/Plug-Ins/HAL/` owned `root:wheel`; `coreaudiod` must be
restarted to pick up changes. The settings app depends on the driver target, so building the app
builds both.

## Repository layout

```
proxyAudioDevice/            The driver target
  ProxyAudioDevice.cpp/.h    ~5.8k lines. The entire AudioServerPlugIn: COM entry points,
                             property tables for plug-in/box/device/stream/control objects,
                             the IO path, and the settings-storage layer.
  AudioRingBuffer.cpp/.h     Frame-addressed ring buffer bridging the two IO threads.
  utilities.cpp/.h           getUserIdleTimeInterval() via IOKit HID idle time.
  debugHelpers.h             DebugMsg / FailIf / FailWithAction macros (no-ops unless DEBUG).
  PublicUtility/             Apple's CoreAudio Utility Classes, vendored verbatim.
                             CAMutex, CAHostTimeBase, CADebugMacros, CADebugPrintf.
  Info.plist                 CFPlugInFactories -> ProxyAudio_Create.
  English.lproj/             Default device/box/manufacturer names (UTF-16 .strings).
  _clang-format              Formatting rules for the whole codebase.

ProxyAudioDeviceSettings/    The settings app target
  WindowDelegate.mm/.h       All the real logic: reads/writes driver settings, keeps the
                             output-device list fresh.
  AppDelegate.m/.h, main.m   Boilerplate.
  Base.lproj/MainMenu.xib    The single window; all IBOutlets/IBActions live on WindowDelegate.

shared/                      Compiled into BOTH targets
  AudioDevice.cpp/.h         Thin RAII-ish wrapper over an AudioObjectID plus static helpers
                             (device enumeration, UID<->ID lookup, name/identify properties).
  CFTypeHelpers.h            CFTypeSmartRef<T> + CFStringSmartRef / CFArraySmartRef / etc.

proxyAudioDevice.xcodeproj/  Only shared scheme: "Proxy Audio Device Settings".
```

`ProxyAudioDevice.h` is also compiled into the settings app (for `kBox_UID`, `ConfigType`,
`ActiveCondition`), even though `ProxyAudioDevice.cpp` is not. See "The configuration channel".

## Build and run

There is no CI, no test suite, no package manifest, and no command-line build script in the repo.
Everything is built from the Xcode project on macOS:

```sh
xcodebuild -project proxyAudioDevice.xcodeproj -target ProxyAudioDevice -configuration Release
xcodebuild -project proxyAudioDevice.xcodeproj -target "Proxy Audio Device Settings" -configuration Release
```

Build configurations are `Debug`, `Release`, and `Debug-Opt` — the last is an optimized
(`GCC_OPTIMIZATION_LEVEL = s`) build that still defines `DEBUG=1`, which matters because this is
real-time audio code and unoptimized builds miss deadlines. `Release` defines `DEBUG=0`, and `DEBUG`
gates every `DebugMsg` call, so a Release driver is silent.

Install/reload loop after changing driver code:

```sh
sudo rm -rf /Library/Audio/Plug-Ins/HAL/ProxyAudioDevice.driver
sudo cp -R build/Release/ProxyAudioDevice.driver /Library/Audio/Plug-Ins/HAL/
sudo chown -R root:wheel /Library/Audio/Plug-Ins/HAL/ProxyAudioDevice.driver
sudo killall coreaudiod                                  # macOS >= 14.4
sudo launchctl kickstart -k system/com.apple.audio.coreaudiod   # macOS <= 13
```

Debugging is via syslog, not a debugger — the driver runs inside `coreaudiod`. `DebugMsg` writes at
`LOG_NOTICE` and real problems at `LOG_WARNING`, all prefixed `ProxyAudio`. Watch with
`log stream --predicate 'eventMessage CONTAINS "ProxyAudio"'`.

Key build settings: `MACOSX_DEPLOYMENT_TARGET = 10.11`, `CLANG_CXX_LANGUAGE_STANDARD = gnu++14`
(driver) / `c++0x` (project default), `GCC_C_LANGUAGE_STANDARD = gnu11`. Version lives in
`MARKETING_VERSION` and `CURRENT_PROJECT_VERSION` (both `1.0.7`) in `project.pbxproj` — bump both.
The driver is code-signed manually with a Developer ID Application identity; the settings app uses
automatic signing. Bundle IDs: `net.briankendall.ProxyAudioDevice` (must match `kPlugIn_BundleID`)
and `net.briankendall.Proxy-Audio-Device-Settings`.

## How the driver works

### Object model

The plug-in publishes a fixed set of `AudioObjectID`s, hardcoded in an enum in `ProxyAudioDevice.h`:
plug-in, box (2), device (3), output stream (4), left/right volume controls (5, 6), master mute (7),
master data source (8). Adding an object means adding it to that enum *and* to the owned-object
lists returned by `GetPlugInPropertyData` / `GetDevicePropertyData`.

Audio format is fixed: 32-bit float, 2 channels, interleaved (`gDevice_BytesPerFrameInChannel = 4`,
`gDevice_ChannelsPerFrame = 2`). Supported sample rates are
`{44100, 48000, 96000, 176400, 192000, 384000, 768000}` (`gDevice_SampleRates`). Volume range is
-25 dB … 0 dB.

### Audio path

Two independent IO threads meet at `AudioRingBuffer inputBuffer` (88200 frames), guarded by
`IOMutex`:

1. `DoIOOperation(kAudioServerPlugInIOOperationWriteMix)` — the HAL hands over mixed client audio.
   It is `Store`d into the ring buffer at `inIOCycleInfo->mOutputTime.mSampleTime`, and
   `lastInputFrameTime` / `lastInputBufferFrameSize` are recorded.
2. `outputDeviceIOProc` — the real device's IO proc. It `Fetch`es from the ring buffer at
   `inOutputTime->mSampleTime + inputOutputSampleDelta`, applies the volume/mute factors, and
   **adds** (`*out += ...`) into the output buffer rather than overwriting it.

`inputOutputSampleDelta` is the offset between the two timelines. It is recomputed (from the last
input frame time minus the output buffer size and safety offset) whenever it is `-1`, which is what
`resetInputData()` sets it to. Reset input data whenever the timeline is invalidated: start/stop IO,
device switch, sample-rate change.

Clock drift is corrected in `GetZeroTimeStamp`: `outputDeviceIOProc` accumulates the real device's
`mRateScalar`, and the zero timestamp advances the ring buffer's host tick count by that running
average, then clears the accumulator. This keeps the virtual device's clock locked to the real one.

Sample rates must match exactly or nothing plays. `matchOutputDeviceSampleRateNoLock()` reads the
target's nominal rate and, if it differs, clears `outputDeviceReady`, stops output, and calls
`gPlugIn_Host->RequestDeviceConfigurationChange(...)`; the HAL stops IO and calls back into
`PerformDeviceConfigurationChange`, which is the only place `gDevice_SampleRate` changes.

### Active condition

`ActiveCondition` decides when the real device is actually started, trading idle power/hiss against
first-sound latency: `proxiedDeviceActive` (only while clients are playing), `userActive` (default —
also when HID idle time < 30 s), `always`. `monitorUserActivity()` runs off a 500 ms dispatch timer
on `audioOutputQueue` and calls `updateOutputDeviceStartedState()`.

### Threading rules

These are the constraints most likely to bite you:

- **Never call CoreAudio APIs from the HAL's calling thread.** Doing so deadlocks `coreaudiod`.
  Anything touching `AudioObject*` or `gPlugIn_Host` must go through
  `ExecuteInAudioOutputThread(^{ ... })`, which dispatches onto the serial `audioOutputQueue`.
  `initializeOutputDevice()` even delays one second before its first CoreAudio call.
- Four `CAMutex`es, and the order they're taken in matters: `outputDeviceMutex` (output device
  lifecycle) → `IOMutex` (ring buffer + input timeline) → `stateMutex` (device state, volume, config)
  and `getZeroTimestampMutex` (rate-ratio accumulator). Functions come in `x()` / `xNoLock()` pairs;
  the `NoLock` variant assumes `outputDeviceMutex` is already held.
- `outputDevice`'s fields (`sampleRate`, `bufferFrameSize`, `safetyOffset`) are read unlocked from
  `outputDeviceIOProc`. The code relies on never mutating them while the device is started —
  `deinitializeOutputDeviceNoLock()` first, then reassign. Preserve that invariant.
- `AudioRingBuffer::Store`/`Fetch` are not internally synchronized (see the `$$$` comment in
  `AudioRingBuffer.cpp`); `IOMutex` is what makes them safe.

## The configuration channel

macOS gives an unprivileged app no sanctioned way to configure a HAL driver, so this project abuses
two writable properties on the **box** object. Read the long comment at
`proxyAudioDevice/ProxyAudioDevice.cpp` in `SetBoxPropertyData`'s `kAudioObjectPropertyIdentify`
case before touching any of this.

- The settings app writes its own pid to the box's `kAudioObjectPropertyIdentify`. That pid becomes
  `configuratorPid`, and the driver then treats that process's reads/writes specially.
- **Read**: the app writes `-(SInt32)ConfigType` to `identify`, then reads the box's
  `kAudioObjectPropertyName`. The driver returns the current value of that setting (ints stringified)
  instead of the box name.
- **Write**: the app sets the box's name to `"settingName=value"`. `parseConfigurationString()`
  turns it into a `ConfigType` + value and `setConfigurationValue()` applies it.
- Setting names on the wire are `deviceName`, `outputDevice` (a device UID),
  `outputDeviceBufferFrameSize`, `outputDeviceActiveCondition` (the integer `ActiveCondition`).
- `identify` values `0` and `1` are ignored so genuine identify requests still behave.

Consequences to keep in mind:

- `ConfigType` and `ActiveCondition` in `ProxyAudioDevice.h` are a wire protocol shared by both
  targets. Reordering the enumerators silently breaks an installed driver talking to a new app (or
  vice versa) — append, don't reorder.
- Adding a setting means touching all of: the enum, `parseConfigurationString`,
  `setConfigurationValue`, `copyConfigurationValue`, a `retrieve…FromStorage` / `set…` pair, and the
  app side in `WindowDelegate.mm`.

Persistence is `gPlugIn_Host->WriteToStorage` / `CopyFromStorage` under the keys `deviceName`,
`outputDeviceUID`, `outputDeviceBufferFrameSize`, `outputDeviceActiveCondition`, `box acquired`.
All settings are loaded in `Initialize()`. Buffer frame size is clamped to at least
`kOutputDeviceMinBufferFrameSize` (4); the default is 512 — too small produces pops and crackles,
which is the single most common user-reported symptom.

If no output device is stored, `copyDefaultProxyOutputDeviceUID()` picks the system default output
device, or the first non-proxy device with output capabilities.

## Settings app notes

- All state is pulled from the driver on demand; the app keeps no preferences of its own.
- `keepTryingToInitializeUntilSuccess` retries with a growing interval (3 s, +2 s each attempt).
  Querying CoreAudio too soon after boot returns junk and can crash the audio server — do not
  shorten this or poll aggressively.
- A missing driver is not a failure: `proxyAudioDeviceAvailable` returning false leaves the UI
  disabled and stops retrying.
- The app registers a `kAudioHardwarePropertyDevices` listener and refreshes the combo box when
  devices come and go.
- Objective-C++ (`.mm`) is required in `WindowDelegate` because it includes the C++ headers. Keep
  bridging casts explicit (`__bridge_transfer` for CF→NS ownership transfer, `__bridge_retained`
  going the other way) — the existing code is deliberate about which is which.
- User-visible strings go through `NSLocalizedString`; `.strings` files are UTF-16.

## Conventions

- **Formatting**: `proxyAudioDevice/_clang-format` is authoritative for all C/C++/ObjC in the repo
  (4 spaces, no tabs, 120 columns, attached braces, `Type *name` pointer alignment, no short
  one-line if/loop bodies, sorted+regrouped includes). Note the leading underscore — copy or
  symlink it to `.clang-format` if your tooling needs that name; don't add a second config.
- **Error handling in the driver** follows Apple's AudioServerPlugIn sample style: declare
  `OSStatus theAnswer = 0`, validate with `FailWithAction(...)` / `FailIf(...)`, `goto Done`, return.
  Keep that idiom in `ProxyAudioDevice.cpp` rather than introducing exceptions or early returns in
  the property handlers.
- **`#pragma unused(...)`** for every unused parameter; the project builds warning-clean.
- **CF memory**: use `CFStringSmartRef` and friends from `CFTypeHelpers.h` for scoped ownership.
  Raw `CFStringRef` members (`deviceName`, `boxName`, `outputDeviceUID`) are manually released
  before reassignment under `stateMutex`.
- **Naming**: globals mirroring Apple's sample keep the `g` prefix (`gDevice_SampleRate`); constants
  use `k` (`kObjectID_Device`, `kDevice_UID`); everything else is lowerCamelCase.
- **`#pragma mark`** sections organize `ProxyAudioDevice.cpp`; put new code in the matching section.
- **Don't edit `PublicUtility/`** — it's vendored Apple code under Apple's license.
- The project's own code is public domain (Unlicense, see `LICENSE`).

## Working on this repo

- There is no way to build, run, or test this on a non-macOS machine, and no automated tests exist.
  Changes to the driver are verified by installing it and listening. Be explicit in your summary
  about what you could not verify.
- Property-handler code is highly repetitive by design (five nearly identical
  `Has/IsSettable/GetDataSize/GetData/SetData` families). Resist "simplifying" it into shared
  helpers unless asked; the parallel structure is what makes it auditable against Apple's headers.
- The README's install/uninstall commands are user-facing documentation; update them if the bundle
  name, install path, or macOS-version-specific restart command changes.
- Commit messages are short imperative summaries ("Fix compiler warning", "Update version to
  1.0.7…"); some use Conventional Commits prefixes (`fix(readme): …`). Either is acceptable.
