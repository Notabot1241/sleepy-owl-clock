# Spec: Sleepy Owl Alarm Clock

| | |
|---|---|
| **Derived from** | [`intent.md`](intent.md) (accepted 2026-11-11; amended by decisions 12 to 16) |
| **Date** | 2026-11-11 |
| **Status** | Accepted 2026-11-11 (after review of concerns C1 to C8); ALM-7 clarified 2026-11-11 (plan gap G1) |
| **Next artifact** | `plan.md` (implementation plan; written only after this spec is accepted) |

This spec turns the intent into testable requirements and a design. It also records what
research established, because three of the intent's promises (alarms that ring when the app is
closed, the hoot, and a free iPhone install from a Windows laptop) depend on platform behaviour
that is only partly documented. Where something is unverified it is marked **[unverified]**, and
a spike in milestone M0 settles it before the rest is built.

## 1. Scope

**In:** a Windows 11 app and an iPhone (iOS 26+) app, built from one Flutter codebase, with an
owl-shaped analog clock (hour, minute and second hands) over a digital hours-and-minutes
readout with AM/PM, multiple alarms, multiple timers, a synthesised rising hoot, a 5-step volume
slider on Windows (the iPhone uses its own alarm volume), a time-zone selector that defaults to
the device's zone with a per-alarm "Follow time zone" switch, alarms that relaunch the app and
wake the PC on Windows, and a distinct Windows icon.

**Out (from the intent):** sync between devices, Android/macOS/web, accounts or cloud,
smart-home or calendar integrations, App Store / TestFlight publication.

## 2. What the research established

| # | Finding | Confidence | Effect on the design |
|---|---|---|---|
| R1 | iOS 26 **AlarmKit** supports one-time alarms, weekly repeating alarms, and countdown timers; alarms **override Silent mode and Focus**; they can have a label and a stop button, plus a secondary button that either **repeats** (snooze, after a set interval) or opens the app. Needs `NSAlarmKitUsageDescription` and user authorisation. ([Apple sample](https://developer.apple.com/documentation/alarmkit/scheduling-an-alarm-with-alarmkit)) | High (Apple docs) | AlarmKit is the iOS alarm engine. It is the only way to get a real alarm on a locked or silenced iPhone. |
| R2 | Apple says AlarmKit "expects a widget extension if an app supports a countdown presentation"; otherwise "the system may unexpectedly dismiss alarms and fail to alert". An alert-only alarm needs no countdown presentation. ([Apple sample](https://developer.apple.com/documentation/alarmkit/scheduling-an-alarm-with-alarmkit)) | High | Prefer **extension-free** alarms and timers (timers as fixed-time alerts). Add the extension only if needed for snooze or lock-screen countdowns. |
| R3 | An alarm rings **until the user stops it**; there is no public maximum duration and no API to auto-stop. ([Apple DTS](https://developer.apple.com/forums/thread/852856)) | High (Apple engineer) | Acceptable for an alarm. The hoot's rising pattern is baked into the audio file, not driven by app code. |
| R4 | AlarmKit alarms play at the **system alarm/ringer volume**; there is **no per-alarm volume API**. ([forum reports](https://developer.apple.com/forums/thread/813519)) | Medium (forum) | The volume slider is **Windows-only** (decision C2). iPhone alarm loudness follows the iPhone's volume. |
| R5 | **Custom alarm sounds are unreliable**: developers report mixed results by file format and location, and inconsistent looping. Apple has not documented format, length, or looping for AlarmKit. ([forum](https://developer.apple.com/forums/thread/814732)) | Low | Hoot-on-iPhone is the **top spike** (M0-A). The hoot is **required** (decision C3); the fallback is in 5.2. |
| R6 | The `flutter_alarmkit` plugin exists (v0.5.0, young: about 1.6k downloads) but **requires a Live Activity widget extension**, Flutter 3.38+, and Xcode 26+. ([pub.dev](https://pub.dev/packages/flutter_alarmkit)) | High | Use it only as a reference or fallback; the primary path is a thin own plugin (about 150 lines of Swift) behind an interface, so either can be swapped in. |
| R7 | A free Apple ID lets Sideloadly install an `.ipa` from **Windows**, but the install **expires after 7 days**, is limited to **3 apps** and 10 App IDs per week, and one guide says **extensions such as widgets may not work with free signing**. ([Sideloadly FAQ](https://sideloadly.io/faq)) | Medium | Build an unsigned `.ipa` on a cloud Mac runner and re-sign on Windows with Sideloadly. The extension question is spike M0-B. |
| R8 | `flutter build ios --no-codesign` on a macOS CI runner yields `Runner.app`; zipping it as `Payload/Runner.app` gives an `.ipa` that can be signed afterwards. | Medium (several guides) | CI workflow produces the `.ipa` artifact (section 5.8). |
| R9 | `flutter_timezone` 5.x supports iOS and Windows and returns the device's time-zone identifier. ([pub.dev](https://pub.dev/packages/flutter_timezone)) | Medium (Windows detail not documented) | Use it plus the `timezone` database package. Verify Windows output in M1. |
| R10 | Windows wake-from-sleep timers are documented for S3/S4 sleep, but **Modern Standby** laptops follow different rules, and a wake timer set by a process is tied to that process. ([Microsoft](https://learn.microsoft.com/en-us/windows/win32/power/system-wake-up-events)) | Low | Use Task Scheduler (R11) rather than an in-process wake timer; verify on this laptop in M0-C. |
| R11 | A Task Scheduler task can be set to **wake the computer** (`WakeToRun`) and to **run after a missed start** (`StartWhenAvailable`); both default to false. `DisallowStartIfOnBatteries` defaults to **true**, so it must be turned off. ([Microsoft schema](https://learn.microsoft.com/en-us/windows/win32/taskschd/taskschedulerschema-settingstype-complextype)) | High (Microsoft docs) | Windows alarms and timers are mirrored as Task Scheduler tasks (decision C4, section 5.3). Whether this wakes **this** laptop, especially in Modern Standby, is **[unverified]**. |

## 3. Requirements

Priority: **M** = must for v1, **S** = should (drop first if scope tightens).

### 3.1 Clock face and digital readout

| ID | Req | Pri |
|---|---|---|
| CLK-1 | An analog dial shows an **hour hand, a minute hand and a thin second hand** on the owl's round body. | M |
| CLK-2 | Angles are computed from the current time in the selected zone: second = `(s + ms/1000) × 6°`; minute = `(m + s/60) × 6°`; hour = `((h mod 12) + (m + s/60)/60) × 30°`. Hands are **recomputed from the clock every frame, never accumulated**, so they are correct after sleep, backgrounding, or a zone change. | M |
| CLK-3 | A digital readout **below** the dial shows `h:mm` and `AM`/`PM` (for example `7:05 AM`). **No seconds.** Always 12-hour, whatever the system locale setting. No leading zero on the hour. | M |
| CLK-4 | The digital minute changes in the same frame in which the second hand passes 12 (within one frame, about 17 ms at 60 Hz). | M |
| CLK-5 | The second hand sweeps smoothly while the app is visible, and stops drawing when the app is hidden or minimised. | M |
| CLK-6 | Owl behaviour: eyes half-closed by default, a slow blink every 6 to 10 seconds, eyes **open wide** when an alarm or timer rings. Decorative motion (blink, wake-up) is turned off when the system's reduce-motion setting is on; the hands are not. | M |

### 3.2 Time zone

| ID | Req | Pri |
|---|---|---|
| TZ-1 | A searchable selector lists IANA time zones with their current UTC offset, plus an **Auto (device)** entry at the top. | M |
| TZ-2 | The default is **Auto**, resolved from the device (for example "Auto: America/New_York") with no setup. | M |
| TZ-3 | Choosing an explicit zone persists across launches until changed back to Auto. | M |
| TZ-4 | The selected zone drives the dial and the digital readout. | M |
| TZ-5 | Each alarm has a **Follow time zone** switch, **on by default**. **On:** the alarm's time is interpreted in the app's currently selected zone (Auto means the device's zone) and is re-interpreted whenever the selection changes. **Off:** the alarm is pinned to the zone that was selected when it was created, and its row shows that zone. _(decision C5)_ | M |
| TZ-6 | While Auto is selected, a change of the device's zone (travel) updates the clock and every alarm whose Follow time zone switch is on, immediately. | M |
| TZ-7 | Daylight-saving rule: a local time that does not exist (spring-forward gap) fires at the first valid instant after the gap; a time that occurs twice (fall-back) fires at its first occurrence. | M |

### 3.3 Alarms

| ID | Req | Pri |
|---|---|---|
| ALM-1 | Create, edit, delete **multiple alarms** (limit 20). | M |
| ALM-2 | Each alarm has an **on/off switch**. | M |
| ALM-3 | Optional **label**, up to 40 characters. | M |
| ALM-4 | **Repeat days**: none (one-time), or any set of weekdays. A one-time alarm switches itself off after it rings. | M |
| ALM-5 | **Snooze** from the ring screen; length is a setting, default **9 minutes** (1 to 30). | M |
| ALM-6 | iPhone: rings with the app closed, the phone locked, on Silent, and in Focus (but see 5.2 for the weaker notification fallback). Windows: rings with the app running (including minimised to the **system tray**) **and with the app closed or the PC asleep** (WIN-1 to WIN-3). | M |
| ALM-7 | When the app is open, the ring screen shows the owl waking up, the label, the time, and **Stop** and **Snooze** buttons. On iPhone with the app closed or the phone locked, Apple's system alarm screen is shown instead (Stop, plus Repeat if available). _(plan gap G1)_ | M |
| ALM-8 | The hoot **rises in insistence** the longer the alarm goes unanswered, until Stopped (section 5.5). | M |
| ALM-9 | Windows, after sleep or restart: an alarm that came due within the last **10 minutes** rings immediately and is marked "late"; older ones are listed as "missed" and not rung. | M |
| ALM-10 | On every launch and on resume, the app reconciles its stored alarms with what is actually scheduled on the device (AlarmKit on iPhone, Task Scheduler on Windows) and repairs any difference. | M |

### 3.4 Timers

| ID | Req | Pri |
|---|---|---|
| TMR-1 | Run **several timers at once** (limit 10), each with an optional label. | M |
| TMR-2 | Duration is set in hours, minutes and seconds, up to 23:59:59. | M |
| TMR-3 | Timers **keep running when the app is closed** and ring at the right moment (same engines as alarms). | M |
| TMR-4 | Each running timer shows its remaining time live in the app; it can be cancelled. | M |
| TMR-5 | A finished timer rings with the same hoot and the same Stop control as an alarm (no snooze). | M |

### 3.5 Sound and volume

| ID | Req | Pri |
|---|---|---|
| SND-1 | The hoot is **synthesised by a script in this repo**; no third-party audio. | M |
| SND-2 | **Windows:** one volume slider with **5 steps** (step 1 = -16 dB, then -12, -8, -4, and step 5 = 0 dB, the loudest), default step 5, and a **test** button that plays a 3-second sample at the chosen step. **iPhone:** no in-app slider; the hoot plays at the iPhone's alarm volume. _(decision C2)_ | M |
| SND-3 | Settings on iPhone explains that the hoot's loudness follows the iPhone's volume. | M |
| SND-4 | The same hoot design is used on both platforms. | M |
| SND-5 | Windows playback applies the chosen step's gain exactly once. | M |

### 3.6 Look, icons, accessibility, platform

| ID | Req | Pri |
|---|---|---|
| UI-1 | Night-sky palette (deep blue/purple, soft stars) with a light variant; **follows the system light/dark setting**. | M |
| UI-2 | **Windows icon**: a bold, simplified sleepy owl face, supplied as a multi-size `.ico` (16, 24, 32, 48, 64, 128, 256 px) with hand-tuned small sizes; used for the exe, taskbar, Start menu shortcut, window title bar and tray. | M |
| UI-3 | **iPhone icon**: the full owl on a night-sky background (1024 px master). | M |
| UI-4 | Text and the digital readout meet **4.5:1** contrast in both themes. The digital readout is at least 48 pt on iPhone and 56 px on Windows. | M |
| UI-5 | Every control has an accessible name, and the clock exposes the current time as text to screen readers (VoiceOver, Narrator). | M |
| PLAT-1 | Windows 11 and iOS 26+ only. One shared Flutter codebase; platform code limited to the alarm engines, tray, Task Scheduler, icon, and signing. | M |
| PLAT-2 | No network access is required or used. No analytics. | M |
| PLAT-3 | iPhone shows a **"Re-sign in N days" banner** when the free-signing profile has 2 days or less left (read from the embedded provisioning profile; hide if unreadable). | S |

### 3.7 Windows background alarms (decision C4)

| ID | Req | Pri |
|---|---|---|
| WIN-1 | For every armed alarm and running timer, the app registers a **Windows Task Scheduler task** for its next fire instant that **launches the app** (`--ring <id>`), **wakes the computer** (`WakeToRun`) and **runs after a missed start** (`StartWhenAvailable`). It works with the app closed or quit. | M |
| WIN-2 | The tasks mirror the store: created, updated and removed whenever alarms, timers or the zone selection change, and reconciled on launch (ALM-10). For a repeating alarm only the next occurrence is registered; the launched app registers the following one. | M |
| WIN-3 | Tasks are allowed to start and keep running **on battery** and run while the user is signed in. If the app is already running, the launch hands over to it (single instance) and the alarm rings **once**. | M |
| WIN-4 | Settings shows a **"Wake from sleep" status** (Ready, or Needs attention with the "Allow wake timers" power setting explained). The limits are stated plainly: a fully shut-down PC cannot ring. | S |

## 4. Architecture

```
lib/
  core/            pure Dart, no Flutter or platform imports, fully unit-tested
    clock_math.dart       angles and 12-hour formatting (CLK-2, CLK-3)
    next_fire.dart        next fire instant for an alarm and selection, DST rules (TZ-5, TZ-7)
    zones.dart            zone list, offsets, Auto resolution
    task_xml.dart         builds Task Scheduler XML for an alarm instant (WIN-1)
  data/
    models.dart           Alarm, TimerItem, Settings
    store.dart            one JSON file in the app's data folder
  engine/
    alarm_engine.dart     interface: schedule / cancel / list / onRing
    ios_alarmkit_engine.dart       talks to the Swift plugin
    windows_engine.dart            in-process scheduler plus Task Scheduler mirror (5.3)
    fake_engine.dart               used by tests and by desktop development
  ui/
    clock_page.dart  owl_painter.dart  alarms_page.dart  timers_page.dart
    settings_page.dart  ring_page.dart  zone_picker.dart
ios/          Swift plugin (AlarmKit bridge) and Info.plist keys
windows/      runner changes: icon, tray, single instance, `--ring` argument
tools/        synth_hoot.py, make_icons.py
assets/       generated hoot.wav / hoot.caf
.github/workflows/ios.yml    cloud build of the unsigned .ipa
```

**Why Flutter.** It is the one mainstream toolkit with stable Windows desktop and iOS targets
from a single Dart codebase, and the owl, hands and eye animation are a natural fit for its
`CustomPainter`. The Windows app can be developed and tested entirely on this laptop; only the
small AlarmKit bridge needs the device loop in 5.8. _Alternatives considered:_ .NET MAUI
(iOS needs a paired Mac, AlarmKit would need a C# binding), React Native for Windows (two
separate stacks), Tauri (iOS support still young).

**Single source of truth.** The app's own store holds alarms, timers and settings. Each engine
is a mirror that is reconciled against the store (ALM-10); the store never reads state back
from the engine except to detect drift.

## 5. Design

### 5.1 Clock rendering

`OwlPainter` draws the body, dial ticks, eyes and hands from a single `DateTime` in the selected
zone, passed in by a ticker that runs only while the app is in the foreground. Hands use the
formulas in CLK-2. The digital readout is a separate widget fed by the same instant, rebuilt
only when the minute or AM/PM changes. Reduce-motion turns off the blink and wake-up animations.

### 5.2 iPhone alarm engine (AlarmKit), with a fallback

The Swift plugin exposes: `requestAuthorization`, `schedule(id, label, schedule, soundFile)`,
`cancel(id)`, `list()`, and a stream of alarm updates.

| Feature | Primary mechanism | Needs widget extension? | If it fails |
|---|---|---|---|
| One-time alarm | AlarmKit alert, relative or fixed schedule | No | Rung 2 |
| Repeating alarm that follows the device zone (Follow on, selector on Auto) | AlarmKit weekly relative schedule | No | Rung 2 |
| Repeating alarm in any other case (pinned, or following an explicitly selected zone) | Rolling window of the next 14 occurrences as fixed-time alerts, refilled on launch (C6) | No | Rung 2 |
| Timer | **Fixed-time alert at now + duration**; the app draws the countdown itself | No | Rung 2 |
| Snooze | AlarmKit secondary "Repeat" button (interval = snooze length) | **Possibly** (R2); checked in M0-B | Secondary "Open" button, then the app schedules a new alert at now + snooze |
| Lock-screen countdown for timers | AlarmKit countdown presentation | Yes | Dropped (not required) |

**The hoot is required (decision C3), so the sound fallback has two rungs:**

1. AlarmKit with the custom hoot file (target).
2. If AlarmKit cannot play the hoot reliably: **local notifications carrying the hoot** (each
   under 30 s), repeated every 30 seconds for several minutes. This is a **weaker alarm**: it
   may stay silent on Silent mode or in some Focus modes, and a time-sensitive notification may
   not be available under free signing **[unverified]**.

If M0-A lands on rung 2, **you decide again** at the M0 gate between "hoot with weaker
reliability" and "default alarm sound with full reliability". On rung 2, Settings shows a plain
warning that alarms may not sound on Silent.

### 5.3 Windows alarm engine

Two layers work together (decision C4).

1. **In-app scheduler** (app running, including in the tray). Not one long timer: **every second
   it compares the current instant with each armed alarm's next fire instant**, so sleep,
   resume, and clock changes are handled naturally (ALM-9). On fire it plays the hoot, shows the
   ring screen, restores and focuses the window, and briefly sets it always-on-top. Closing the
   window **hides it to the system tray** (the owl icon); **Quit** is in the tray menu. The
   scheduler recomputes when Windows reports a time or zone change.
2. **Task Scheduler mirror** (app closed, quit, or PC asleep). For each armed alarm and timer
   the app writes a task from an XML definition and registers it with `schtasks`. The task has a
   one-time trigger at the next fire instant, `WakeToRun` on, `StartWhenAvailable` on,
   `DisallowStartIfOnBatteries` and `StopIfGoingOnBatteries` off (R11), and the action
   `owl.exe --ring <id> --at <utc instant>`. When the launched app finishes ringing a repeating
   alarm it registers the next occurrence. A single-instance guard hands a launch over to an
   already running app, and a fired-set keyed by `(id, instant)` guarantees one ring. Tasks are
   named `SleepyOwl\<id>` so reconciliation (ALM-10) can find and clean them. Creating tasks for
   the signed-in user should not need administrator rights **[unverified]**.

**Honest limits.** A fully shut-down PC cannot ring. Whether the laptop actually wakes depends
on its power settings (**Allow wake timers**) and on Modern Standby behaviour (R10, R11)
**[unverified]**. With the screen locked, the ring window may appear behind the lock screen,
though the hoot plays. M0-C measures all of this on this laptop (sleep, hibernate, lid closed, on
battery) and records the results. If waking turns out not to work, the task still relaunches the
app at the next wake or sign-in and ALM-9 applies.

### 5.4 Time and zones

All instants are stored and compared in UTC. An alarm stores its wall-clock time, weekdays, a
`followZone` flag (default true) and, only when the flag is off, a pinned IANA zone. The app's
selection is an IANA identifier or `auto` (the device's zone). An alarm's **effective zone** is
the selection when `followZone` is on, otherwise its pinned zone. `next_fire(alarm, selection,
now)` converts the wall-clock time and weekdays into the next UTC instant using the `timezone`
database, applying TZ-7.

On iPhone, an alarm whose effective zone is the device's own (follow on, selection `auto`) uses
AlarmKit's relative (device-local) schedule, so the OS itself follows travel (TZ-6); every other
alarm uses fixed instants. **Changing the selection reschedules every alarm that follows it**,
on both platforms. Windows recomputes `next_fire` on every selection or device-zone change and
rewrites the affected Task Scheduler entries.

### 5.5 Sound design

`tools/synth_hoot.py` (Python standard library only) renders a two-note "hoo-hoo": a soft
sine near 350 Hz gliding to about 300 Hz with a few quiet harmonics and a slow attack and
decay. It then renders a **28-second rising sequence**: a single gentle hoot every 3 seconds,
tightening to a double hoot every 1.5 seconds with a gradual rise in level, so one file carries
the whole escalation (ALM-8). It writes one full-level `.wav` for Windows and one full-level
`.caf` for iPhone (converted on the CI Mac runner).

- **Windows** plays the `.wav` with the audio plugin and scales playback by the volume step's
  gain, so the gain is applied exactly once (SND-5).
- **iPhone** hands the `.caf` to AlarmKit, which plays it at the system alarm volume (R4, C2).
  Whether the 28-second file loops until Stopped is **[unverified]** (R3, R5) and is checked in
  M0-A.

### 5.6 Persistence

A single JSON file (alarms, timers, settings, schema version) in the platform's app-data folder.
Writes are atomic (write a temp file, then rename). A timer stores its **end instant in UTC**,
not a countdown, so it survives restarts.

### 5.7 Icons

A master vector owl is drawn once, then rasterised by `tools/make_icons.py`. The Windows set
uses a simplified face (large eyes, no hands) with hand-checked 16, 24 and 32 px versions.
The iPhone icon is the full owl on the night-sky background.

### 5.8 Build and install

- **Windows:** `flutter build windows`; a script creates a Start-menu shortcut carrying the icon.
  An installer is deferred.
- **iPhone:** a GitHub Actions macOS job (Xcode 26+; confirm the runner image) builds the unsigned
  app, zips it as an `.ipa` and uploads it as an artifact. On Windows, **Sideloadly** signs it with
  the free Apple ID and installs it over USB. Re-signing is needed about every 7 days (R7);
  PLAT-3 shows a countdown.
- **Development loop:** the owl, clock, alarms UI, zones, and Windows engine are built and tested
  on this laptop with the fake engine. Only the AlarmKit bridge needs the slower loop
  (push, cloud build, download, sideload, test on the phone).

## 6. Verification plan

**Automated (run on Windows, no phone needed)**

| Area | Tests |
|---|---|
| `clock_math` | Known times give known angles (for example 10:09:36 gives hour 304.8°, minute 57.6°, second 216°); noon, midnight, 12 AM / 12 PM formatting; no seconds in the digital string. |
| `next_fire` | Weekday sets; one-time; two zones; **Follow on versus pinned** when the selection changes; **DST gap and overlap** (for example America/New_York, 2027-03-14 02:30 and 2027-11-07 01:30); end-of-month and year rollover. |
| Windows scheduler | With a fake clock: fires once; does not fire twice; after a simulated sleep, rings as "late" if within 10 minutes of the due time and is listed "missed" beyond that. |
| `task_xml` | Generated XML for a given alarm contains the right trigger instant (with offset), `WakeToRun`, `StartWhenAvailable`, battery restrictions off, and the `--ring` arguments. |
| Widget goldens | Dial at fixed times in light and dark themes; eyes half-closed and wide. |
| Store | Round trip; schema upgrade; atomic write survives a killed write. |

**Manual checklist (each milestone records pass/fail with date and device)**

| ID | Check | Covers |
|---|---|---|
| A1 | First launch shows the correct local time, hands and digital agree, AM/PM correct | CLK, TZ-2 |
| A2 | Second hand reaches 12 as the digital minute changes | CLK-4 |
| A3 | Change zone to Tokyo then back to Auto; dial and digital follow | TZ-3, 4, 6 |
| A4 | Alarm for +2 minutes rings with the app closed (iPhone, locked, on Silent, in a Focus) | ALM-6 |
| A5 | Same on Windows with the window closed to the tray | ALM-6 |
| A6 | Rising hoot: audibly more insistent over 30 seconds and continues until Stopped | ALM-8 |
| A7 | Windows: volume step 1 versus step 5 audibly different | SND-2 |
| A8 | Two timers (1 and 2 minutes) finish in order; closing the app does not stop them | TMR |
| A9 | Snooze re-rings after the set interval; Stop ends it | ALM-5 |
| A10 | Windows: sleep the laptop across an alarm time, then wake: "late" or "missed" behaves per ALM-9 | ALM-9 |
| A11 | iPhone: after 7 days, check whether already scheduled alarms still ring after the signature expires | concern C1 |
| A12 | Light and dark themes, reduce motion on, screen reader reads the time | UI |
| A13 | Windows: quit the app completely; an alarm 2 minutes ahead relaunches it and rings once | WIN-1 to 3 |
| A14 | Windows: sleep the laptop (lid open, then lid closed, then on battery) across an alarm time; it wakes and rings, or the result is recorded as a limit | WIN-1, C4 |
| A15 | Alarm with Follow time zone on versus off; change the selected zone: the "on" alarm moves, the "off" alarm stays | TZ-5 |

## 7. Milestones

| # | Milestone | Exit criteria |
|---|---|---|
| **M0** | **Risk spikes** (no UI beyond a button) | **A** A tiny app schedules an AlarmKit alert with the custom hoot; it rings with the app closed, on Silent; the rising pattern is audible; looping is known. **B** The same app with a widget extension installs and runs under free signing, and Repeat works. **C** On this laptop: tray keeps the app alive; an alarm rings from the tray; a Task Scheduler task relaunches a quit app and wakes the PC from sleep and hibernate (lid open and closed, on battery), with results recorded. The cloud build to Sideloadly path works end to end. **Decision gate** for C1 (what a reduced iPhone app looks like if free signing blocks something; paying is ruled out), C3 (hoot versus reliability) and C4 (wake results). |
| M1 | Clock, zones | A1 to A3; unit tests for `clock_math`, `zones`. |
| M2 | Alarms | Windows A5, A9, A10, A13, A14; iPhone A4; A15; `next_fire`, scheduler and `task_xml` tests. |
| M3 | Timers | A8. |
| M4 | Sound and volume | A6, A7; hoot tuned by ear. |
| M5 | Icons, polish, packaging | UI-2, UI-3, UI-4, UI-5; A12; release checklist done. |

## 8. Traceability to the intent

| Intent item | Requirements |
|---|---|
| Analog hour and minute hands, digital below with AM/PM | CLK-1 to CLK-4 |
| Second hand; digital stays hours and minutes (decision 12) | CLK-1, CLK-3, CLK-4, CLK-5 |
| Sleepy owl shape | CLK-6, UI-1 |
| Alarms (multiple, on/off, labels, repeat, snooze) | ALM-1 to ALM-10 |
| Alarms ring when closed; Windows relaunch and wake (decision 15) | ALM-6, WIN-1 to WIN-4 |
| Timers (several, keep running) | TMR-1 to TMR-5 |
| Hooting sound, rising hoot; volume (Windows, decision 13); hoot required (decision 16) | SND-1 to SND-5, ALM-8, section 5.2 |
| Time-zone selector, auto-default; Follow time zone switch (decision 14) | TZ-1 to TZ-7 |
| Special Windows icon | UI-2 |
| Windows and iPhone, one codebase, no backend | PLAT-1, PLAT-2 |
| Free personal iPhone install, painless re-signing | PLAT-3, section 5.8 |
| Night-sky look following system theme | UI-1 |
| Accessibility and privacy constraints | UI-4, UI-5, PLAT-2 |

## 9. Concerns and decisions

All eight concerns were answered on 2026-11-11. The right-hand column is what is still open.

| # | Concern | Your decision | What remains |
|---|---|---|---|
| **C1** | Free Apple ID signing expires weekly; unknown whether scheduled alarms survive expiry, or whether a widget extension installs under free signing (R7). | **Free only, no paid Apple Developer account** (confirmed 2026-11-11: no money is to be spent on this project; intent decision 17). | Checks A11 and M0-B. Until A11 passes, **do not rely on the iPhone app as your only wake-up alarm.** If either fails, the iPhone app gets a reduced feature set; we do not pay. |
| **C2** | The system sets AlarmKit's loudness; apps cannot (R4). | **Hide the volume slider on iPhone**; keep it on Windows. | Intent amended (decision 13). SND-3 tells the user the iPhone volume applies. |
| **C3** | AlarmKit may not play the custom hoot (R5). | **The hoot is required.** | If AlarmKit fails, the iPhone uses the notification fallback (5.2), which may not sound on Silent and may not be time-sensitive under free signing. You re-decide at the M0 gate, because this trades reliability for the hoot. |
| **C4** | Windows alarms ring only while the app runs (R10). | **Also relaunch the app and wake the PC** through Task Scheduler. | New requirements WIN-1 to WIN-4 and checks A13, A14. A fully shut-down PC still cannot ring, and waking this laptop is unproven until M0-C. |
| **C5** | What happens to existing alarms when the zone selection changes. | **Per-alarm "Follow time zone" switch, default on**; on = follows the app's selected zone, off = pinned to the zone chosen at creation. | Intent refined (decision 14). Interpretation recorded in TZ-5 and 5.4; see check A15. |
| **C6** | Rolling window of 14 occurrences for repeating alarms that don't follow the device zone; alarm and timer limits. | **Accepted** (14 occurrences, 20 alarms, 10 timers). | For a daily repeating alarm in this case, the app must be opened at least every two weeks. The app shows a reminder. |
| **C7** | Native iOS code cannot be tested on this laptop, so iPhone iterations are slow. | **No objection; treated as accepted.** | The design keeps that code minimal (5.8). |
| **C8** | Two extras I added. | **Keep the re-sign banner only.** | PLAT-3 stays (should). Launch at sign-in and keep-awake are dropped; the Task Scheduler design covers the Windows need. |

## 10. Sources

- [Anthropic, The AI-Native SDLC Playbook](https://claude.com/resources/articles/the-ai-native-sdlc-playbook)
- [Apple, Scheduling an alarm with AlarmKit](https://developer.apple.com/documentation/alarmkit/scheduling-an-alarm-with-alarmkit)
- [Apple Developer Forums: AlarmKit alert timeout (Apple DTS)](https://developer.apple.com/forums/thread/852856)
- [Apple Developer Forums: AlarmKit custom sounds](https://developer.apple.com/forums/thread/814732)
- [Apple Developer Forums: AlarmKit volume](https://developer.apple.com/forums/thread/813519)
- [pub.dev: flutter_alarmkit](https://pub.dev/packages/flutter_alarmkit)
- [pub.dev: flutter_timezone](https://pub.dev/packages/flutter_timezone)
- [Sideloadly FAQ](https://sideloadly.io/faq)
- [Microsoft Learn: System wake-up events](https://learn.microsoft.com/en-us/windows/win32/power/system-wake-up-events)
- [Microsoft Learn: Task Scheduler settingsType](https://learn.microsoft.com/en-us/windows/win32/taskschd/taskschedulerschema-settingstype-complextype)

## 11. Review and sign-off

- [x] Concerns C1 to C8 answered (2026-11-11)
- [x] Spec reviewed and corrected by the product owner (concerns reviewed and answered, 2026-11-11)
- [x] Accepted: proceed to `plan.md` (2026-11-11)
