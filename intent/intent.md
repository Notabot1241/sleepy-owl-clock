# Intent: Sleepy Owl Alarm Clock

| | |
|---|---|
| **Author** | Originator: _[add your name]_ (drafted with Claude) |
| **Date** | 2026-11-11 |
| **Status** | Accepted 2026-11-11; amended 2026-11-11 (decisions 12 to 17) |
| **Next artifact** | `spec.md` (written only after this intent is accepted) |

> Items marked **(assumed)** are my best reading of the idea, not something you said.
> Everything under "Decisions made during review" was answered by you directly.
> Correct or delete anything that is wrong.

## Problem

**(assumed)** There is no alarm clock I enjoy using on both of my devices. Stock alarm and
timer apps work but have no personality, look different on each platform, and give me no sense
that the clock is "mine". I want one small, charming clock that looks and behaves the same way
on my Windows laptop and my iPhone, and that is calm and friendly to wake up to.

_You described the product, not the motivation, so the paragraph above is my inference.
Replace it with your real reason if it differs._

Who is affected today: me, as the sole user and owner. I set alarms and timers on both devices
and sometimes need the time in a different time zone.

## Proposed outcome

A software alarm clock whose whole character is a **sleepy owl**, shipped as two apps from one
idea and one shared codebase: a **Windows laptop app** and an **iPhone app**.

What I can do with it:

1. **Read the time at a glance.** An analog clock face with a moving **hour hand, minute
   hand, and a thin second hand**, and a **digital readout below it** showing **hours and
   minutes only (no seconds)** with **AM / PM**. The second hand is analog only.
   The owl's round body is the clock face. Drowsy, half-closed eyes sit above it, and the
   eyes open wide when an alarm or timer goes off.
2. **Set alarms.** Create **multiple alarms**, each with an **on/off switch**, an optional
   **label** (for example "Gym"), **repeat days** (for example Mon-Fri), and a **snooze**
   option. When one goes off, the owl wakes me with a **hooting sound**.
3. **Set timers.** Start **several countdown timers at once**. They keep counting when the app
   is closed or in the background, and each alerts me with the hoot when it finishes.
4. **Control the volume.** On Windows, one **in-app volume slider** sets how loud the hoot is,
   with a test button to preview it. On iPhone the hoot plays at the iPhone's own alarm volume
   (iOS gives apps no control over it), so there is no slider there. On both, the hoot starts
   gentle and grows more insistent the longer an alarm or timer goes unanswered.
5. **Pick a time zone.** A selector for time zones that **defaults automatically to the zone
   my device is in**, so it is correct the first time I open it without any setup. Choosing a
   different zone changes the clock face and the digital time. Each alarm has a **Follow time
   zone** switch, **on by default**: when on, the alarm is interpreted in whatever zone is
   selected and moves when I change the selection; when off, it stays pinned to the zone that
   was selected when I created it. The default follows the device when I travel, until I choose a zone myself.
6. **Recognise the Windows app.** The laptop app has its own **special icon**: a bold,
   simplified sleepy owl face that stays readable at small sizes in the taskbar, Start menu,
   and window title bar. The iPhone app has its own standard app icon.
7. **Feel like a night-sky owl.** A deep blue/purple night palette with soft stars that
   follows the device's light or dark setting.

Success looks like:

- I open either app for the first time and the hands and digital time already show the correct
  local time, with AM/PM, without my touching a setting.
- I set an alarm for a few minutes from now, leave the app, and hear the hoot at the right
  moment on both Windows and iPhone.
- I set a 1-minute timer and it hoots when it ends, even if I closed the app; I can run two
  timers at the same time.
- On Windows I turn the volume slider down and the next hoot is audibly quieter; I turn it up
  and it is louder.
- I leave an alarm unanswered and the hoot becomes more insistent over time; I tap snooze and
  it re-rings after a few minutes.
- I switch the time zone and the clock hands and digital time change to match it, along with
  every alarm whose Follow time zone switch is on; alarms with it off stay put.
- The second hand makes one full turn per minute and lines up with the top of the dial as the
  minute changes; the digital readout never shows seconds.
- Someone looking at the screen would say "that's a sleepy owl" without being told.

## Affected users and systems

- **User:** one person (me). No accounts, sharing, or other users.
- **Windows laptop:** Windows 11 (the machine this is being built on).
- **iPhone:** my own iPhone, running **iOS 26 or newer**.
- **Platform services touched:** the device clock and time-zone setting, local notifications
  and audio playback on each platform, and the Windows app icon / packaging.
- **No backend (assumed):** everything runs on-device. Nothing is sent to a server.

## Constraints

- **Two platforms, one identity.** Same owl, same layout concept, same behaviour on Windows
  and iPhone, built from **one shared codebase**. Platform-appropriate conventions are fine
  where they differ.
- **Alarms must actually ring.** An alarm that only works while the app is open and in the
  foreground does not meet this intent. Both platforms restrict background behaviour (iOS
  especially; Windows when asleep or when the app is closed), so the spec must establish what
  is achievable and state any limits honestly. On Windows this includes ringing when the app
  has been closed and waking the laptop from sleep where the hardware and power settings allow it.
- **Building for iPhone from a Windows laptop.** iOS apps are normally built with Apple's
  tools on a Mac. The approach must be workable with the hardware I have (a Windows laptop and
  an iPhone), for example by building on a cloud Mac service. I will install on my own iPhone
  with a **free personal Apple ID**, which means the install **expires and must be re-signed
  about every 7 days**; the spec should make that re-signing step as painless as possible.
- **Accuracy.** The analog hands and the digital readout must always agree with each other and
  with the device clock, including across midnight, noon, and daylight-saving changes.
- **Audio rights.** The hoot is **synthesised from scratch** (original), so there are no
  licensing concerns. No ripped audio.
- **Accessibility (assumed).** Digital numbers readable at a glance (large, high contrast);
  nothing important conveyed by animation or colour alone.
- **Privacy (assumed).** No analytics, no tracking, no network use required for core features.
- **Personal-scale project.** Favour a small, finishable scope over completeness. Prefer the
  simplest approach that meets the outcome.

## Non-goals (assumed, for this first version)

- Syncing alarms between laptop and iPhone.
- Android, macOS, or web versions.
- Accounts, cloud storage, or sharing.
- Smart-home, calendar, or sleep-tracking integrations.
- App Store or TestFlight publication (decided: free personal install only).

## Decisions made during review

| # | Question | Decision |
|---|---|---|
| 1 | What does "shaped like a sleepy owl" mean? | The owl's body is the clock face; sleepy eyes above it; eyes open on alarm. |
| 2 | Time-zone behaviour | Selected zone changes display **and** alarms. Default follows the device until I choose one. Refined by decision 14. |
| 3 | iPhone / iOS version | iOS 26 or newer. |
| 4 | iPhone install method | Free personal install (re-sign about every 7 days). |
| 5 | Alarm features in v1 | Multiple alarms with on/off, snooze, repeat days, and labels. |
| 6 | Timers | Several at once; they keep running when the app is closed. |
| 7 | Volume control | One in-app slider with a rising hoot; no separate alarm and timer volumes. Windows only; see decision 13. |
| 8 | Hoot audio | Synthesised, original. |
| 9 | Windows icon | A distinct, simplified owl face for Windows. |
| 10 | Look and feel | Night-sky palette that follows the system light/dark theme. |
| 11 | Technology approach | One shared codebase for both apps. |
| 12 | Second hand (added after acceptance) | Add an analog second hand. The digital display stays hours and minutes with AM/PM only, no seconds. |
| 13 | iPhone volume (spec concern C2) | No in-app slider on iPhone; the hoot follows the iPhone's alarm volume. The slider stays on Windows. Amends decision 7. |
| 14 | Alarm time zones (spec concern C5) | Each alarm has a "Follow time zone" switch, on by default. On: uses the selected zone and moves with it. Off: pinned to the zone selected at creation. Refines decision 2. |
| 15 | Windows alarms when closed (spec concern C4) | Also relaunch the app and wake the PC for alarms and timers through Windows Task Scheduler, where the hardware allows it. |
| 16 | Hoot on iPhone (spec concern C3) | The hoot is required on iPhone alarms. If the system alarm API cannot play it, use a notification fallback that may be weaker on Silent mode, and decide again after the first iPhone test. |
| 17 | No-spend constraint | No money is spent on this project: no paid Apple Developer account and no paid build minutes (the GitHub repo is public, so standard runners are free). If free signing blocks a feature, the iPhone app gets a reduced feature set instead. |

## Remaining questions for the spec to resolve

These are design and research items, not product decisions, so they do not block accepting
this intent.

- **iOS alarm mechanism.** iOS 26 may offer a true system alarm API that sounds on silent or
  during Focus modes. Confirm it exists, what it allows (custom sound, snooze, repeat,
  rising volume), and what the fallback is.
- **Windows alarms when the app is closed or the laptop sleeps.** Establish what is reliably
  possible (scheduled notifications, background task, wake from sleep) and document the limits.
- **Shared-codebase framework choice.** Pick the framework and the cloud build path that work
  from a Windows machine, and confirm they support custom alarm sounds and background alerts.
- **Defaults.** Snooze length, maximum number of alarms and timers, and how quickly the hoot
  escalates.
- **Time-zone edge cases.** How alarms behave across daylight-saving changes and when the
  device zone changes while an alarm is already set.

## Review and sign-off

- [x] Intent reviewed and corrected by the product owner (answered open questions in review, 2026-11-11)
- [x] Open questions answered or consciously deferred to `spec.md` (answered 2026-11-11)
- [x] Accepted: proceed to `spec.md` (2026-11-11)
