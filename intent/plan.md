# Plan: Sleepy Owl Alarm Clock

| | |
|---|---|
| **Derived from** | [`spec.md`](spec.md) (accepted 2026-11-11), which derives from [`intent.md`](intent.md) |
| **Date** | 2026-11-11 |
| **Status** | Accepted 2026-11-11 (including the ALM-7 change, gap G1) |
| **Next stage** | Implementation, one milestone at a time, starting with P0 and M0 |

This plan says what to build, in what order, in which files, and how each step is checked.
It is meant to be detailed enough that someone else could pick up any task. If implementation
departs from it, **this file is updated in the same commit** (section 9), and a change to a
*requirement* goes back to `spec.md` and you.

**Task markers:** **[C]** Claude writes the code or document. **[Y]** you do it (device,
account, or system setting I should not change for you). **[CI]** runs in GitHub Actions.

## 1. Starting point on this laptop

I checked this machine on 2026-11-11 (read-only commands). Facts that shape the plan:

| Check | Result | Consequence |
|---|---|---|
| Git | installed | Used as is. |
| Flutter / Dart | **not installed** | Install in P0 (Flutter 3.38 or newer, per spec R6). |
| Visual Studio (C++ desktop workload) | **not installed** | Flutter's Windows target needs it. Install in P0. |
| Developer Mode | **not enabled** | Flutter plugins need symlinks on Windows; enable in P0. |
| Python | **not installed** (only the Microsoft Store shortcut exists) | The sound and icon scripts need it. Install in P0. |
| GitHub CLI (`gh`) | not installed | Optional; the cloud iPhone build needs a GitHub repo. |
| `winget` | available (v1.29) | Used for the installs in P0. |
| Machine type | physical laptop with a battery; a hypervisor is running (virtualisation features on) | Real hardware, so M0-C results will be meaningful. |
| Sleep states (`powercfg /a`) | **Standby (S3) and Hibernate available; Modern Standby (S0) not supported by the firmware**; Fast Startup available | Good news for spec risk R10: classic S3 wake timers are the documented case. |
| "Allow wake timers" | **Enabled on AC power, Disabled on battery** | A Task Scheduler wake **will not happen on battery** until you enable it. WIN-4's status will say "Needs attention" on battery. M0-C tests AC first, then battery after you change the setting. |

## 2. Working agreements

- **One repo, one branch per milestone** (`m0-spikes`, `m1-clock`, and so on), merged when its
  exit criteria in the spec pass. The three documents live in `intent/` in the repo.
- **Tests first for pure logic** (`core/`): write the failing test, then the code.
- **Small commits**; each commit message says what changed and why. No secrets, Apple IDs,
  certificates, or signing files in the repo, ever.
- **CI** runs `flutter analyze` and `flutter test` on a **Windows** runner (golden images are
  platform-specific, so goldens are generated and compared on Windows only) and the iPhone
  build on a macOS runner. GitHub Actions are pinned to a commit SHA, not a moving tag.
- **Unverified items stay marked** until a spike result is written down in `docs/spikes/`.
- Native iOS code stays under about 200 lines of Swift and compiles on every push, because it
  cannot be compiled on this laptop.

## 3. Technology choices

Versions are pinned when the packages are added (`flutter pub add`), not guessed here.

| Need | Choice | Notes |
|---|---|---|
| UI, drawing | Flutter (stable), Dart | `CustomPainter` for the owl and hands. |
| Time zones | `timezone` + `flutter_timezone` | Spec R9; verify Windows output in T1.2. |
| Storage | `path_provider` + one JSON file | Atomic write (temp file, then rename). |
| Windows window and tray | `window_manager`, `tray_manager` | Hide to tray on close; `--ring` launch argument arrives in `main(args)`. |
| Single instance (Windows) | a maintained single-instance package; else about 60 lines of C++ in the runner (named mutex plus a message to the first window) | Decide in M0-C. |
| Audio playback (Windows, in-app) | `audioplayers` (alternative if it misbehaves: `just_audio` or `flutter_soloud`) | Gain applied once (SND-5). |
| iPhone alarms | Own Swift `MethodChannel` bridge to AlarmKit (`ios/Runner/AlarmBridge.swift`) | `flutter_alarmkit` is used only in spike M0-B to generate and test the widget extension. |
| iPhone notification fallback | `flutter_local_notifications`, only if rung 2 is needed | Spec 5.2. |
| Sound and icon scripts | Python 3.12 (`wave`, `struct`, `math`; Pillow for icons) | Scripts in `tools/`. |
| iPhone cloud build | GitHub Actions macOS runner, Xcode 26 or newer | Confirm the runner image name when writing the workflow. |
| iPhone install | Sideloadly on Windows, free Apple ID | Spec R7. |

## 4. P0: Set up (before any code)

| ID | Task | Who | Done when |
|---|---|---|---|
| P0.1 | Install Python and the GitHub CLI: `winget install --id Python.Python.3.12`, `winget install --id GitHub.cli` | Y | `python --version` and `gh --version` work in a new terminal. |
| P0.2 | Install Visual Studio 2022 with the **Desktop development with C++** workload (`winget install --id Microsoft.VisualStudio.2022.Community`, then add the workload in the Visual Studio Installer) | Y | The installer lists the workload as installed. |
| P0.3 | Install the Flutter SDK (stable, 3.38 or newer) following the official Windows guide, and add it to PATH | Y | `flutter --version` works. |
| P0.4 | Turn on **Developer Mode** (Settings, System, For developers) | Y | `flutter doctor` no longer warns about symlinks. |
| P0.5 | Run `flutter doctor` and fix the Windows items (Android is not needed) | Y, C helps | The "Windows Version" and "Visual Studio" lines are green. |
| P0.6 | Decide the **reverse-domain prefix** for the iPhone bundle ID (for example `com.<something unique>.sleepyowl`); free Apple IDs need a unique ID | Y | You give me the string. |
| P0.7 | Pick the project folder (a normal folder you choose; I will move this session there) and run `git init` | Y, C | Folder exists, empty repo. |
| P0.8 | Create the GitHub repo. **Decision:** a public repo makes the macOS builds free but shows the code (no secrets are stored in it); a private repo may use paid macOS minutes, so check current GitHub pricing | Y | Repo URL exists; I only push when you say so. |
| P0.9 | Scaffold: `flutter create --platforms=windows,ios --org <prefix> --project-name sleepy_owl .`; add `intent/` with the three documents, `.gitignore`, `flutter_lints`; set the iOS deployment target to 26.0 in `project.pbxproj` and `AppFrameworkInfo.plist`; add `NSAlarmKitUsageDescription` to `Info.plist`; commit | C | `flutter run -d windows` shows the starter app. |
| P0.10 | CI part 1: `.github/workflows/ci.yml` runs analyze and test on Windows | C | Green check on the first push. |

## 5. M0: Risk spikes

Goal: turn every **[unverified]** in the spec into a recorded result. Each spike writes
`docs/spikes/M0-<letter>.md` with date, device, steps, and pass or fail.

### M0-A: iPhone AlarmKit with the hoot (spec R3, R4, R5; concerns C1, C3)

| ID | Task | Who |
|---|---|---|
| A1 | `tools/synth_hoot.py --quick`: write a 10 s and a 28 s placeholder hoot as `.wav` (final design comes in M4) | C |
| A2 | `ios/Runner/AlarmBridge.swift` and `AppDelegate` registration: methods `requestAuthorization`, `schedule(id, label, fixedEpochMs or hour/minute/weekdays, soundName)`, `cancel(id)`, `list` | C |
| A3 | `tools/ios_add_resource.rb`: a Ruby script (uses the `xcodeproj` gem that ships with CocoaPods; confirm it is on the runner) that adds `hoot.caf` to the Runner target's resources, because Windows cannot edit the Xcode project by tool | C |
| A4 | `.github/workflows/ios.yml`: select Xcode 26+, `flutter build ios --release --no-codesign`, convert `.wav` to `.caf` with `afconvert -f caff -d LEI16`, run the Ruby script, zip `Runner.app` into `Payload/` and upload `Runner.ipa` as an artifact | C, CI |
| A5 | Debug screen with buttons: "Alarm in 1 min (hoot)", "Alarm in 1 min (default sound)", "Cancel all" | C |
| A6 | Install Sideloadly on Windows; on the iPhone turn on **Developer Mode**, install the `.ipa` with your free Apple ID, and trust the developer profile | Y |
| A7 | Run the test matrix: app closed, locked, Silent switch on, a Focus on; file formats (`.caf` first, then `.wav`, `.aiff`); file in the app bundle versus `Library/Sounds`; 10 s versus 28 s; does it **loop until Stopped**; is the **rising pattern** audible; app force-quit; phone restarted | Y, C records |

**Pass:** the hoot rings from the lock screen on Silent and loops until Stopped. **Fail:** fall
to rung 2 (notification fallback) and **you decide again** (spec C3).

### M0-B: widget extension and free signing (R2, R7; concern C1)

| ID | Task | Who |
|---|---|---|
| B1 | On a throwaway branch, add `flutter_alarmkit` and run its `dart run flutter_alarmkit:setup` to generate the widget extension, then `--doctor` | C |
| B2 | Build in CI, sideload with the free Apple ID, and check: does it install, does the extension run, does an alarm with the **Repeat (snooze)** button work, and does the Apple ID's App ID limit get hit | Y, C records |
| B3 | Decide snooze design: Repeat button with the extension, or the "Open app" fallback (spec 5.2) | C proposes, you decide |

### M0-C: Windows tray, Task Scheduler, wake (R10, R11; concern C4)

| ID | Task | Who |
|---|---|---|
| C1 | Tray icon, hide-on-close, restore, and quit using `window_manager` and `tray_manager` | C |
| C2 | Hand-written sample task XML (below) registered with `schtasks /Create /XML`; confirm it can be created **without administrator rights** and launches the app with `--ring` | C, Y |
| C3 | In-app scheduler tick (every second) that rings the hoot from a placeholder `.wav` | C |
| C4 | Single-instance handover: a second launch with `--ring` signals the first and exits | C |
| C5 | Run the wake matrix, **AC first**: sleep (S3) with lid open; sleep with lid closed; hibernate; Fast-Startup shutdown (expected: no wake). Then enable "Allow wake timers" on battery **yourself** (Power Options, Advanced settings, Sleep) and repeat on battery | Y, C records |

Sample task (the final generator is `core/task_xml.dart`; write the file as **UTF-16 with a
byte-order mark** and confirm this in C2):

```xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.4" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <Triggers><TimeTrigger>
    <StartBoundary>2026-11-12T07:00:00-05:00</StartBoundary><Enabled>true</Enabled>
  </TimeTrigger></Triggers>
  <Principals><Principal id="Author">
    <LogonType>InteractiveToken</LogonType><RunLevel>LeastPrivilege</RunLevel>
  </Principal></Principals>
  <Settings>
    <MultipleInstancesPolicy>Parallel</MultipleInstancesPolicy>
    <DisallowStartIfOnBatteries>false</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>false</StopIfGoingOnBatteries>
    <StartWhenAvailable>true</StartWhenAvailable>
    <WakeToRun>true</WakeToRun>
    <ExecutionTimeLimit>PT0S</ExecutionTimeLimit>
  </Settings>
  <Actions Context="Author"><Exec>
    <Command>C:\path\to\sleepy_owl.exe</Command>
    <Arguments>--ring ALARM_ID --at 2026-11-12T12:00:00Z</Arguments>
  </Exec></Actions>
</Task>
```

### M0 gate (decisions)

Write `docs/spikes/M0-summary.md` and decide, with you: **C1** (stay on free signing or pay);
**C3** (hoot on rung 1, or choose between the weaker notification hoot and the default alarm
sound); **C4** (what the wake matrix proved, and the exact wording of WIN-4). If a result
changes a requirement, `spec.md` is amended and re-accepted before M1 starts.

## 6. M1 to M5: Build

Every task lists the spec requirements it satisfies. Test files sit beside the code in `test/`.

### M1: Clock and zones (branch `m1-clock`)

| ID | Task | Files | Reqs |
|---|---|---|---|
| T1.1 | Test first, then `clock_math`: angles from a time (hour uses fractional minutes), `h:mm` and AM/PM formatting; cases 12 AM, 12 PM, 10:09:36 (304.8°, 57.6°, 216°), no seconds in the string | `core/clock_math.dart` | CLK-2, 3 |
| T1.2 | `zones`: initialise the tz database, resolve the device zone with `flutter_timezone`, list zones with current offsets; confirm what Windows returns and add a mapping if it returns a Windows-style name | `core/zones.dart` | TZ-1, 2, 6 |
| T1.3 | `OwlPainter`: body, dial ticks, half-closed eyes, three hands (second hand thin, accent colour), blink and wake-up animation controllers that respect reduce-motion | `ui/owl_painter.dart` | CLK-1, 5, 6 |
| T1.4 | Clock page: dial above, digital readout below (`7:05 AM`, no seconds), a ticker that runs only when the app is visible, time-zone chip | `ui/clock_page.dart` | CLK-3, 4, 5 |
| T1.5 | Zone picker: search, "Auto (device)" first, persists the selection | `ui/zone_picker.dart`, `data/store.dart` | TZ-1 to 4 |
| T1.6 | Theme: night-sky palette, light variant, follows the system setting; contrast checked to 4.5:1 | `ui/theme.dart` | UI-1, 4 |
| T1.7 | Semantics: accessible names; the clock exposes the time as text | `ui/*` | UI-5 |
| T1.8 | Golden tests of the dial at fixed times, light and dark, eyes half and wide | `test/golden/` | CLK |
| T1.9 | Run checks A1, A2, A3 on Windows | | |

### M2: Alarms (branch `m2-alarms`)

| ID | Task | Files | Reqs |
|---|---|---|---|
| T2.1 | Models and store: `Alarm` (time, weekdays, label, enabled, `followZone`, pinned zone), `Settings`, schema version | `data/models.dart`, `data/store.dart` | ALM-1 to 4, TZ-5 |
| T2.2 | Test first, then `next_fire`: weekday sets; one-time; Follow versus pinned when the selection changes; **DST gap and overlap** (America/New_York 2027-03-14 02:30 and 2027-11-07 01:30); month and year rollover | `core/next_fire.dart` | TZ-5, 7 |
| T2.3 | Engine interface and `FakeEngine` (injectable clock) | `engine/alarm_engine.dart`, `fake_engine.dart` | ALM-10 |
| T2.4 | Alarms list and editor: time picker, label (40 characters), weekday chips, on/off, **Follow time zone** switch (default on), zone shown when pinned | `ui/alarms_page.dart` | ALM-1 to 4, TZ-5 |
| T2.5 | Windows in-app scheduler (1 s tick), fired-set `(id, instant)`, late (10 minutes) and missed rules, ring page, snooze as a derived one-off instant | `engine/windows_engine.dart`, `ui/ring_page.dart` | ALM-5, 7, 9 |
| T2.6 | `task_xml` (test first), `schtasks` wrapper, create, update, delete, and reconcile at launch and resume; repeating alarms register only the next occurrence, the launched app registers the next | `core/task_xml.dart`, `engine/windows_engine.dart` | WIN-1 to 3, ALM-10 |
| T2.7 | Windows runner: `--ring` handling, single instance, tray, hide on close | `windows/runner/`, `lib/main.dart` | ALM-6 |
| T2.8 | Settings: snooze length, Windows wake-status line (reads "Allow wake timers" through `powercfg`) with the plain explanation | `ui/settings_page.dart` | ALM-5, WIN-4 |
| T2.9 | iPhone engine: real `AlarmBridge` calls; relative schedule for follow-device alarms, fixed instants and the **14-occurrence rolling window** otherwise; refill on launch; reschedule every following alarm when the selection changes; snooze per the M0-B decision; show the ring page when the app is open and an alarm is alerting | `ios/Runner/AlarmBridge.swift`, `engine/ios_alarmkit_engine.dart` | ALM-5, 6, 8, 10, C6 |
| T2.10 | If M0-A chose rung 2: notification fallback and the Settings warning | `engine/ios_notification_engine.dart` | ALM-6 |
| T2.11 | iPhone "Re-sign in N days" banner from `embedded.mobileprovision` (hide if unreadable) | `ios/Runner/AlarmBridge.swift`, `ui/` | PLAT-3 |
| T2.12 | Checks A4, A5, A9, A10, A13, A14, A15; record results | | |

### M3: Timers (branch `m3-timers`)

| ID | Task | Files | Reqs |
|---|---|---|---|
| T3.1 | `TimerItem` with the end instant in UTC; engine calls reuse the alarm path (fixed-time alert on iPhone, Task Scheduler on Windows) | `data/models.dart`, `engine/*` | TMR-1 to 3 |
| T3.2 | Timers page: h/m/s picker (to 23:59:59), optional label, live remaining time, cancel; limit 10 | `ui/timers_page.dart` | TMR-1, 2, 4 |
| T3.3 | Finished timer uses the ring page without snooze | `ui/ring_page.dart` | TMR-5 |
| T3.4 | Check A8 | | |

### M4: Sound (branch `m4-sound`)

| ID | Task | Files | Reqs |
|---|---|---|---|
| T4.1 | Final `synth_hoot.py`: two-note hoot, the 28 s rising sequence (gentle every 3 s, tightening to double every 1.5 s, rising level); writes `assets/hoot.wav` (and the CI makes the `.caf`) | `tools/synth_hoot.py` | SND-1, 4, ALM-8 |
| T4.2 | **You listen** to 2 or 3 variants I generate and pick one or ask for changes | | |
| T4.3 | Windows volume: 5 steps (-16, -12, -8, -4, 0 dB), test button, gain applied once; a unit test of the step to gain mapping | `core/volume.dart`, `ui/settings_page.dart` | SND-2, 5 |
| T4.4 | iPhone Settings text: loudness follows the phone's volume | `ui/settings_page.dart` | SND-3 |
| T4.5 | Checks A6, A7 | | |

### M5: Icons, polish, packaging (branch `m5-release`)

| ID | Task | Files | Reqs |
|---|---|---|---|
| T5.1 | `make_icons.py`: master owl artwork; simplified-face set; `.ico` with 16, 24, 32, 48, 64, 128, 256 px and hand-checked small sizes; set as the Windows exe, tray and window icon | `tools/make_icons.py`, `windows/runner/resources/app_icon.ico` | UI-2 |
| T5.2 | iPhone 1024 px icon set (full owl on night sky) | `ios/Runner/Assets.xcassets/` | UI-3 |
| T5.3 | Accessibility pass: contrast, text sizes, Narrator and VoiceOver walk-through, reduce-motion | | UI-4, 5, A12 |
| T5.4 | Windows packaging script: `flutter build windows`, then a Start-menu shortcut with the icon | `tools/package_windows.ps1` | PLAT-1 |
| T5.5 | `README.md` with: build steps, the weekly re-sign routine, the "Allow wake timers" setting, known limits | `README.md` | |
| T5.6 | Release checklist: all A-checks green, spikes archived, spec and plan up to date, and **no networking or analytics packages** in `pubspec.yaml` and no network permission in either app | | PLAT-2 |

## 7. What I need from you, and when

| When | You |
|---|---|
| P0 | Install the tools, enable Developer Mode, choose the bundle prefix, project folder and repo visibility. |
| M0 | Sideload to your iPhone, run the sound, signing, and wake tests, and enable battery wake timers yourself for the battery run. |
| M0 gate | Decide C1, C3, C4 from the results. |
| M4 | Listen to the hoot variants and choose. |
| Each milestone | Run the manual checks that need a device and tell me pass or fail. |

## 8. Risks and fallbacks

| Risk | Plan |
|---|---|
| iPhone iteration loop is slow (cloud build, download, sideload, test) | Develop and test everything else on Windows first; keep Swift under about 200 lines; batch iPhone tests per milestone. |
| `flutter_alarmkit` or Sideloadly behaves differently from the guides | Spike M0-B isolates it on a throwaway branch; the own-bridge path does not depend on it. |
| The Xcode project cannot be edited from Windows | Resource and extension changes go through the Ruby script and the plugin's setup script, run on the macOS runner or by Dart. |
| macOS runner minutes cost money on a private repo | Build the iPhone app only on demand (manual trigger and tags), not on every push. |
| Weekly re-sign is forgotten | PLAT-3 banner, README routine; C1 gate decides on a paid account. |
| Wake does not work on battery or lid closed | Documented limit; WIN-4 status explains the setting; ALM-9 handles late rings. |

## 9. Gaps found while planning (for your review)

These are things the spec did not settle. None changes a requirement without your say-so.

| # | Gap | Proposed handling |
|---|---|---|
| G1 | **ALM-7 on iPhone.** The ring screen with the waking owl can only appear when the app is open. When the phone is locked, Apple's own alarm screen is shown. | Treat ALM-7 as "owl ring screen when the app is open; the system alert otherwise" and amend the spec. **Resolved 2026-11-11:** you confirmed, and `spec.md` ALM-7 is amended. |
| G2 | **Battery wake timers are off on this laptop** (section 1). | Not a code change. WIN-4 reports it and the README explains how to enable it. |
| G3 | **Snooze on Windows** is not described in the spec. | A snooze is a derived one-off instant (now plus the snooze length) with its own Task Scheduler entry, removed when stopped. |
| G4 | **Python was assumed** by the spec but is not installed. | Install in P0.1. (Alternative if you prefer no Python: write the scripts in Dart; say so before M4.) |
| G5 | **Adding the hoot file to the iPhone project** cannot be done on Windows. | `tools/ios_add_resource.rb` runs in CI (A3). |

## 10. Plan change log

| Date | Change | Why |
|---|---|---|
| 2026-11-11 | First draft | Spec accepted. |
| 2026-11-11 | Plan accepted; ALM-7 clarified in the spec (gap G1) | Product-owner review. |
| 2026-11-11 | P0.1 to P0.3, P0.5 to P0.7, P0.9, P0.10 done. Flutter is 3.50.0 (plan said 3.38 or newer). Bundle ID is `app.owlnestd8f1e60e.sleepyOwl`. Flutter's iOS template here has a `SceneDelegate` and no Podfile, so task A2 registers the AlarmKit bridge where the template's engine is created (confirm when writing A2), not necessarily in `AppDelegate`. | Observed while scaffolding. P0.4 (Developer Mode) and P0.8 (GitHub repo) are still open. |
