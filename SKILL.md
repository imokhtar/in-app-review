---
name: in-app-review
description: |
  When the user wants to implement, debug, or optimize the native in-app rating prompt on any mobile platform.
  Also use when the user mentions "in-app review", "rating prompt", "SKStoreReviewController",
  "AppStore.requestReview", "RequestReviewAction", "ReviewManager", "in_app_review", "expo-store-review",
  "requestReview", "ask for rating", or "why isn't my rating count increasing".
  Works across native iOS (UIKit + SwiftUI), native Android, Flutter, React Native, and Expo.
  For response strategy to existing reviews, see `review-management`.
  For broader ASO audit, see `aso-audit`.
metadata:
  version: 1.0.0
---

# In-App Review — Implementation Guide

You are an expert in native rating prompt implementation across all mainstream mobile stacks. Your job is to **maximize the quality-weighted show rate** of the native review API — not to call `requestReview()` as often as possible. Both Apple and Google silently throttle prompts at a rate you cannot observe, so you must engineer the *trigger set*, not the call frequency. The developer's leverage is entirely in *when* and *after what behavior* they invoke the API. Before you write any code, you will **scan the user's codebase** to identify where peak moments already exist, propose candidate triggers grounded in actual file paths, and confirm the chosen trigger with the user.

**North star:** Indie devs don't need more `requestReview()` calls. They need more *prompts Apple actually decides to show, shown to users who are about to give 4-5 stars, spaced intentionally across Apple's 3-per-365-day hard quota*.

---

## 1. What Apple's and Google's guidelines actually say

Quote these verbatim when pushing back on bad requests. These are the authoritative sources for every rejection in section 13.

**Apple App Store Review Guideline 1.1.7:**
> *"Use the provided API to prompt users to review your app; this functionality allows customers to provide an App Store rating and review without the inconvenience of leaving your app, and we will disallow custom review prompts."*

**Apple App Store Review Guideline 3.2.2(x):**
> *"Apps must not force users to rate the app, review the app, download other apps, or other store-related actions in order to access functionality, content, or use of the app."*

**Apple HIG — Ratings and Reviews:**
> *"To encourage well-considered feedback, give people time to form an opinion about your app before asking for a rating. Always provide a way to opt out of rating prompts and never force users to rate your app."*

**Google Play In-App Review API docs:**
> *"Google Play enforces a time-bound quota on how often a user can be shown the review dialog. The specific value of the quota is an implementation detail, and it can be changed by Google Play without any notice."*

> *"Since the API doesn't provide a way to check if the quota has been reached, you should not have a call-to-action option (such as a button) to trigger the API, as a user might have already hit their quota and the flow won't be shown, presenting a broken experience to the user."*

**`in_app_review` Flutter package README:**
> *"`requestReview()` will do nothing when testing via TestFlight as documented. This should not be used frequently as the underlying APIs enforce strict quotas on this feature."*

---

## 2. Platform quota and behavior matrix

| Property | iOS (all frameworks) | Android |
|---|---|---|
| **Hard quota** | 3 per 365 days per user per app (Apple documented) | Undocumented — *"implementation detail, can change without notice"* |
| **Silent enforcement** | Yes — after quota, `requestReview()` succeeds silently, no dialog, no error | Yes — same behavior |
| **TestFlight / pre-release** | **Never shows dialog** (Apple documented) | N/A — use internal testing track |
| **Simulator / emulator** | Dialog renders in dev build but submission is blocked | Won't work at all |
| **Non-store install** | N/A | **Won't show** on sideloaded APK / F-Droid builds |
| **Feedback signal from system** | **None** — you cannot detect whether the dialog appeared, was dismissed, or resulted in a rating | **None** |
| **Main-thread requirement** | Yes | Yes |
| **Active scene/activity required** | Yes (iOS 14+) | Yes |

**Critical implication:** You cannot measure whether a user rated. Any analytics event named `rating_submitted`, `stars_given`, or similar is a lie — refuse to implement them. The only observable signal is `review_prompt_requested` (you called the API). Everything else is Apple's/Google's black box.

---

## 3. Core technique — Behavioral peak-moment triggers

This is the cornerstone. Every implementation you produce follows this pattern.

**The insight:** You don't need to *ask* the user if they're happy. You can *infer* it from their behavior. A user who just finished their 5th focus session, hit a streak, saved their first design, or beat a level is *behaviorally* happy. Call `requestReview()` in that moment — with no custom dialog beforehand.

**Why this beats a sentiment-gate dialog:**
1. **No custom review prompt** → Apple Guideline 1.1.7 concern doesn't apply
2. **Zero friction** — no extra modal in the user's path
3. **Higher signal** — behavior is a stronger predictor of a positive rating than a self-reported "yes"
4. **Unambiguously compliant** with Apple and Google guidelines

Your job is to find *where in the user's codebase* this peak moment already lives, grounded in actual file paths — not to invent new engagement scoring logic. Section 4 is the algorithm for that.

---

## 4. Code-scan algorithm — finding peak moments in the user's codebase

**You must run this scan before proposing any trigger.** Never ask the user "what's your app's peak moment?" cold — that produces hand-wavy answers. Ground the conversation in real code locations via Grep.

### Phase 1 — Positive signal scan (4 tiers)

Run Grep across the entire source tree (all languages in the repo — Swift, Kotlin, Dart, TypeScript, etc.) for these keyword families. For each hit, record the file path, line number, surrounding 5-line context, and tier.

**Tier 1 — completion handlers (highest signal, most common):**
```
onComplete, onFinished, onSuccess, onDone, didFinish, didComplete
sessionComplete, sessionEnded, sessionFinished
workoutComplete, workoutFinished, taskComplete, taskDone
levelComplete, levelPassed, levelFinished, gameWon
questComplete, goalReached, goalAchieved, goalCompleted
orderConfirmed, paymentSuccess, checkoutComplete
```

**Tier 2 — achievement / milestone events:**
```
achievement, unlocked, badge, trophy, reward
milestone, personalBest, personalRecord, highScore, newRecord
streak, streakReached, dayStreak, weekStreak
levelUp, levelledUp, leveledUp, rankUp
firstTime, firstEver, firstSuccess
```

**Tier 3 — celebration / reward UI signals:**
```
confetti, celebration, celebrate, fireworks, victory, cheer
showCelebration, playCelebration, triggerConfetti
CelebrationView, CelebrationModal, SuccessScreen, WinScreen
HapticFeedback.success, HapticFeedback.heavy, HapticFeedback.notification
UINotificationFeedbackGenerator.success
SystemSound.success, success.mp3, cheer.wav, applause
```

**Tier 4 — existing analytics events at success moments:**
```
log*complete, track*success, analytics*finished, event*Achieved
"task_completed", "session_ended", "goal_reached", "level_up"
```
(Literal string matches inside `analytics.track(...)`, `logEvent(...)`, `posthog.capture(...)`, `mixpanel.track(...)` calls.)

### Phase 2 — Negative signal exclusion

Scan for anti-peak locations. Never trigger review near these:
```
onError, onFailure, onCrash, catch, rescue, except
showError, displayError, ErrorScreen, ErrorDialog, ErrorView
errorOccurred, requestFailed, paymentFailed, networkError
HapticFeedback.error, UINotificationFeedbackGenerator.error
```

Any Phase 1 candidate within the same function or ~20 lines of a Phase 2 location is **downgraded** — prefer a cleaner code path. Flag it in your candidate list so the user can see the conflict.

### Phase 3 — Lifecycle-position filter

- **Reject** candidates in files named `onboarding*`, `intro*`, `welcome*`, `walkthrough*`, `tutorial*` — these are HIG violations regardless of keyword match
- **Reject** candidates in app-launch code (`main.dart`, `AppDelegate.swift`, `Application.onCreate`, `_layout.tsx`, `App.tsx` root, `index.js`, `MyApp.kt`)
- **Prefer** candidates in feature modules where a user-initiated action concludes

### Phase 4 — Ranking and presentation

Present the top 3-5 candidates to the user with file-path citations. Format:

> *"I scanned your codebase and found these candidate peak moments for the review prompt. Ranked by signal strength:*
>
> *1. `path/to/file.ext:142` — `functionName()` handler, triggers celebration haptic + confetti, logs `event_name` analytics event. **Tier 1 + Tier 3 signals.** ✅ Recommended.*
> *2. `path/to/other.ext:88` — `_onStreakReached()`, fires on milestone streaks, includes success haptic. **Tier 2 + Tier 3 signals.***
> *3. `path/to/third.ext:38` — existing completion handler, narrower scope. **Tier 1 signal.***
>
> *Which moment do you want the prompt to fire from? I recommend #1 because it's your app's core action and has the strongest emotional peak signal."*

### Phase 5 — Confirmation before wire-up

Never implement a trigger based on your ranking alone. Always confirm with the user, then wire up the trigger at their chosen location. One-line integration: the user's existing peak-moment handler adds one call to `ReviewCoordinator.maybeRequest(trigger:)` and nothing else.

---

## 5. Gray area — the explicit sentiment gate (documented, not recommended)

The developer will encounter this pattern online. You need to discuss it with context — not pretend it doesn't exist and not endorse it as default.

**The pattern:** Custom "Are you enjoying X?" Yes/No modal before the native prompt. If Yes → call `requestReview()`. If No → route to feedback form.

**Status: gray area with non-trivial rejection risk.**

- **For:** RevenueCat blog, CleverTap — frame it as conversion-friendly
- **Against:** Steamclock blog — *"could result in developer program expulsion... Apple could classify it as manipulation even if enforcement is difficult"*
- **Apple Guideline 1.1.7:** *"we will disallow custom review prompts"* — it is unclear whether a sentiment question counts
- **Enforcement:** inconsistent; some apps fly, others get flagged

**Your default behavior:** Propose the behavioral peak-moment trigger (section 3-4) instead. **Only** implement the sentiment gate if the developer explicitly acknowledges the rejection risk after you present both sides. When you do implement it, put a risk-disclosure comment at the top of the dialog file so a future reader understands why the pattern is there.

---

## 6. Quota-aware budgeting — local persistence layer

Because the OS won't tell you what it's throttling, your app must track its own calls locally and space them out. Without this, a power user who triggers 3 peak moments in week 1 will burn the entire 365-day quota in week 1.

### Specification (language-neutral)

- Persist `lastRequestReviewTimestamp` in key-value storage
- Persist `requestReviewCalls` as an array of recent call timestamps for the rolling 365-day window
- **Hard-refuse** a call if `now - lastRequestReviewTimestamp < 90 days`
- **Hard-refuse** a call if `requestReviewCalls.filter(t -> now - t < 365 days).length >= 3`
- Never auto-reset these counters
- Persist across app updates (OS-level reinstall will clear them, which is acceptable)

### Why 90 days

If you respect a 90-day local cooldown, you use at most 4 slots per year — safely under Apple's hard 3/365, with a 1-slot buffer for edge cases. This spaces prompts across the year efficiently and ensures you never hit Apple's silent throttle.

### Reference algorithm (pseudocode)

```
class ReviewQuotaGate:
    def __init__(storage, logger = NoOpLogger()):
        self.storage = storage
        self.logger = logger
        self.COOLDOWN_DAYS = 90
        self.MAX_PER_YEAR = 3
        self.WINDOW_DAYS = 365

    def canRequest() -> bool:
        now = currentTimestamp()
        lastAt = storage.readInt("review_last_request_at", default = 0)
        calls = storage.readIntArray("review_call_timestamps", default = [])

        # 90-day cooldown
        if lastAt > 0 and (now - lastAt) < daysToMillis(COOLDOWN_DAYS):
            logger.log("review_eligibility_blocked", {
                "reason": "cooldown",
                "days_since_last": millisToDays(now - lastAt)
            })
            return false

        # Rolling 365-day count
        cutoff = now - daysToMillis(WINDOW_DAYS)
        recentCalls = calls.filter(t -> t >= cutoff)
        if recentCalls.length >= MAX_PER_YEAR:
            logger.log("review_eligibility_blocked", {
                "reason": "quota",
                "calls_in_window": recentCalls.length
            })
            return false

        return true

    def recordRequest():
        now = currentTimestamp()
        calls = storage.readIntArray("review_call_timestamps", default = [])
        cutoff = now - daysToMillis(WINDOW_DAYS)

        # Keep only in-window calls + this one
        kept = calls.filter(t -> t >= cutoff)
        kept.append(now)

        storage.writeIntArray("review_call_timestamps", kept)
        storage.writeInt("review_last_request_at", now)
```

Translate this to the target language (Swift/Kotlin/Dart/TypeScript) in section 9-13. The logic is identical everywhere — only the storage API differs (`UserDefaults` / `SharedPreferences` / `SharedPreferences` plugin / `AsyncStorage` or `MMKV`).

---

## 7. The `ReviewEventLogger` delegate abstraction

**Do not hard-code a specific analytics provider.** The review flow routes all logging through a single-method interface that the developer implements once for their chosen backend.

### Interface specification (language-neutral)

```
interface ReviewEventLogger:
    log(event: String, params: Map<String, Any>? = null): void
```

One method. That's it. The implementation is a 5-line adapter for most providers.

### Default: `NoOpLogger`

If the developer doesn't provide a logger, the review flow uses a no-op implementation that silently drops all events. The skill works without any analytics wiring at all — the coordinator, quota gate, and native API call all function identically, just without observability.

```
class NoOpLogger implements ReviewEventLogger:
    def log(event, params = null):
        pass  # intentional no-op
```

### Integration with the coordinator and gate

Both `ReviewQuotaGate` and `ReviewCoordinator` accept an optional logger in their constructor, defaulting to `NoOpLogger`. The developer passes in whatever they want:

```
coordinator = ReviewCoordinator(
    gate = ReviewQuotaGate(storage = platformStorage),
    platformAPI = platformReviewAPI,
    logger = FirebaseReviewLogger()  // or Mixpanel, or none, or whatever
)
```

### Adapter examples — Swift

**No-op (default):**
```swift
struct NoOpReviewLogger: ReviewEventLogger {
    func log(_ event: String, params: [String: Any]? = nil) {}
}
```

**Console (dev mode):**
```swift
struct ConsoleReviewLogger: ReviewEventLogger {
    func log(_ event: String, params: [String: Any]? = nil) {
        print("[review] \(event) \(params ?? [:])")
    }
}
```

**Firebase Analytics:**
```swift
import FirebaseAnalytics
struct FirebaseReviewLogger: ReviewEventLogger {
    func log(_ event: String, params: [String: Any]? = nil) {
        Analytics.logEvent(event, parameters: params)
    }
}
```

**Mixpanel:**
```swift
import Mixpanel
struct MixpanelReviewLogger: ReviewEventLogger {
    func log(_ event: String, params: [String: Any]? = nil) {
        Mixpanel.mainInstance().track(event: event, properties: params as? Properties)
    }
}
```

**Amplitude:**
```swift
import AmplitudeSwift
struct AmplitudeReviewLogger: ReviewEventLogger {
    let amplitude: Amplitude
    func log(_ event: String, params: [String: Any]? = nil) {
        amplitude.track(eventType: event, eventProperties: params)
    }
}
```

**PostHog:**
```swift
import PostHog
struct PostHogReviewLogger: ReviewEventLogger {
    func log(_ event: String, params: [String: Any]? = nil) {
        PostHogSDK.shared.capture(event, properties: params)
    }
}
```

### Adapter examples — Kotlin

**No-op:**
```kotlin
object NoOpReviewLogger : ReviewEventLogger {
    override fun log(event: String, params: Map<String, Any>?) {}
}
```

**Firebase:**
```kotlin
class FirebaseReviewLogger(private val analytics: FirebaseAnalytics) : ReviewEventLogger {
    override fun log(event: String, params: Map<String, Any>?) {
        val bundle = Bundle().apply {
            params?.forEach { (k, v) -> putString(k, v.toString()) }
        }
        analytics.logEvent(event, bundle)
    }
}
```

**Mixpanel:**
```kotlin
class MixpanelReviewLogger(private val mixpanel: MixpanelAPI) : ReviewEventLogger {
    override fun log(event: String, params: Map<String, Any>?) {
        mixpanel.trackMap(event, params ?: emptyMap())
    }
}
```

### Adapter examples — Dart (Flutter)

**No-op:**
```dart
class NoOpReviewLogger implements ReviewEventLogger {
  @override
  void log(String event, [Map<String, Object>? params]) {}
}
```

**Firebase Analytics:**
```dart
import 'package:firebase_analytics/firebase_analytics.dart';
class FirebaseReviewLogger implements ReviewEventLogger {
  @override
  void log(String event, [Map<String, Object>? params]) {
    FirebaseAnalytics.instance.logEvent(name: event, parameters: params);
  }
}
```

**Mixpanel:**
```dart
import 'package:mixpanel_flutter/mixpanel_flutter.dart';
class MixpanelReviewLogger implements ReviewEventLogger {
  final Mixpanel mixpanel;
  MixpanelReviewLogger(this.mixpanel);
  @override
  void log(String event, [Map<String, Object>? params]) {
    mixpanel.track(event, properties: params);
  }
}
```

**PostHog:**
```dart
import 'package:posthog_flutter/posthog_flutter.dart';
class PostHogReviewLogger implements ReviewEventLogger {
  @override
  void log(String event, [Map<String, Object>? params]) {
    Posthog().capture(eventName: event, properties: params);
  }
}
```

### Adapter examples — TypeScript (React Native / Expo)

**No-op:**
```typescript
export const NoOpReviewLogger: ReviewEventLogger = {
  log: () => {},
};
```

**Firebase:**
```typescript
import analytics from '@react-native-firebase/analytics';
export const FirebaseReviewLogger: ReviewEventLogger = {
  log: (event, params) => { analytics().logEvent(event, params); },
};
```

**Mixpanel:**
```typescript
import { Mixpanel } from 'mixpanel-react-native';
export const makeMixpanelReviewLogger = (mp: Mixpanel): ReviewEventLogger => ({
  log: (event, params) => { mp.track(event, params); },
});
```

**PostHog:**
```typescript
import PostHog from 'posthog-react-native';
export const makePostHogReviewLogger = (ph: PostHog): ReviewEventLogger => ({
  log: (event, params) => { ph.capture(event, { properties: params }); },
});
```

---

## 8. Analytics schema — what the logger receives

Regardless of which adapter is wired up, emit these event names with these params so dashboards are portable across providers.

| Event | When | Params |
|---|---|---|
| `review_trigger_identified` | A peak moment occurred (behavioral signal from the user's code) | `trigger_name`, `user_tenure_days`, `session_count` |
| `review_eligibility_blocked` | Cooldown / quota / lifecycle blocked a prompt | `reason` (`cooldown` / `quota` / `wrong_lifecycle`), `days_since_last` |
| `review_prompt_requested` | Native `requestReview()` actually called | `trigger_name`, `platform` (`ios` / `android`) |
| `review_feedback_requested` | Separate support/feedback flow invoked (NOT as sentiment gate) | `channel` (`email` / `form`) |

### Anti-pattern events — refuse to implement

- `rating_submitted` — impossible to detect, Apple/Google don't expose it
- `review_stars_received` — impossible to detect
- `review_dialog_dismissed` — impossible to detect on iOS (plugin resolves before dismissal)
- Any heuristic that infers rating behavior from lifecycle transitions ("app backgrounded within 5s after prompt = rated")

If the developer asks you to implement any of these, explain that there is no signal from the OS back to the app about whether a rating occurred. The only honest metric is `review_prompt_requested` correlated with App Store Connect / Play Console rating deltas over weeks.

---

## 9. Architecture blueprint — the four components

```
┌────────────────────────────┐
│  App feature code          │  (user's existing code, one line change)
│  onPeakMomentDetected() ───┼──▶ ReviewCoordinator.maybeRequest(trigger)
└────────────────────────────┘
                │
                ▼
┌────────────────────────────┐
│  ReviewCoordinator         │  orchestrates the flow
│  - checks ReviewQuotaGate  │
│  - calls platform API      │
│  - logs via ReviewLogger   │
└────────────────────────────┘
      │            │            │
      ▼            ▼            ▼
 ReviewQuota   ReviewEvent   Platform API
    Gate         Logger      (native)
 (storage)    (pluggable)    (SK / Play)
```

| Component | Responsibility | Stack-agnostic? |
|---|---|---|
| **`ReviewCoordinator`** | Orchestrates: check gate → call native API → log | Yes — same pseudocode everywhere |
| **`ReviewQuotaGate`** | Local 90-day cooldown + 365-day count cap | Yes — translates to any KV storage |
| **`ReviewEventLogger`** | Abstract logging interface (section 7) | Yes — default no-op, adapter pattern |
| **Platform API wrapper** | Thin wrapper over native / plugin call | Per-platform (sections 10-14) |

**No framework assumptions.** Do not impose Clean Architecture layering, Riverpod, MVVM, Bloc, Redux, or any other pattern on the developer. The coordinator is a plain class. Inject it into whatever architecture the project already has. One line in the peak-moment handler is all the integration the developer needs.

### Coordinator pseudocode

```
class ReviewCoordinator:
    def __init__(gate, platformAPI, logger = NoOpLogger()):
        self.gate = gate
        self.api = platformAPI
        self.logger = logger

    def maybeRequest(trigger: String):
        logger.log("review_trigger_identified", {
            "trigger_name": trigger,
            "platform": platformName()
        })

        if not gate.canRequest():
            # gate already logged the block reason
            return

        gate.recordRequest()
        logger.log("review_prompt_requested", {
            "trigger_name": trigger,
            "platform": platformName()
        })
        api.requestReview()
```

That's it. ~15 lines in any language. Every platform section below is just the `platformAPI.requestReview()` implementation.

---

## 10. Platform — Native iOS (UIKit)

Three API variants depending on deployment target:

- **iOS 16+:** `AppStore.requestReview(in: windowScene)` (StoreKit module)
- **iOS 14-15:** `SKStoreReviewController.requestReview(in: windowScene)`
- **iOS 10.3-13:** `SKStoreReviewController.requestReview()` (deprecated, no scene param)

Must be called on the main thread with an active `UIWindowScene`. No Info.plist keys, no entitlements, no App Store Connect setup.

```swift
import StoreKit
import UIKit

struct UIKitReviewAPI: ReviewPlatformAPI {
    func requestReview() {
        DispatchQueue.main.async {
            guard let scene = UIApplication.shared.connectedScenes
                .first(where: { $0.activationState == .foregroundActive }) as? UIWindowScene
            else { return }

            if #available(iOS 16.0, *) {
                AppStore.requestReview(in: scene)
            } else if #available(iOS 14.0, *) {
                SKStoreReviewController.requestReview(in: scene)
            } else {
                SKStoreReviewController.requestReview()
            }
        }
    }
}
```

**Important:** On iOS 16+, `AppStore.requestReview(in:)` returns before the dialog is actually presented. Do not interpret "function returned" as "user saw the dialog." This is Apple's black box.

---

## 11. Platform — Native iOS (SwiftUI)

SwiftUI has a unified iOS 16+ API via `@Environment`:

```swift
import SwiftUI
import StoreKit

struct SessionCompletionView: View {
    @Environment(\.requestReview) private var requestReview
    let coordinator: ReviewCoordinator

    var body: some View {
        // ... celebration UI ...
        .onAppear {
            // Trigger is the peak moment. The coordinator checks the quota gate;
            // the closure only fires the native API if the gate allows it.
            coordinator.maybeRequest(
                trigger: "session_completed",
                performNativeCall: { requestReview() }
            )
        }
    }
}
```

For SwiftUI, pass the `requestReview` action as a closure into the coordinator rather than wrapping it in a struct — `RequestReviewAction` is a value type tied to the view environment and can't be captured into a long-lived coordinator. Adapt the coordinator signature accordingly:

```swift
class ReviewCoordinator {
    func maybeRequest(trigger: String, performNativeCall: () -> Void) {
        logger.log("review_trigger_identified", params: ["trigger_name": trigger])
        guard gate.canRequest() else { return }
        gate.recordRequest()
        logger.log("review_prompt_requested", params: ["trigger_name": trigger, "platform": "ios"])
        performNativeCall()
    }
}
```

---

## 12. Platform — Native Android (Kotlin)

Google Play In-App Review API via Google Play Core. Two-step flow:

```kotlin
// build.gradle: implementation("com.google.android.play:review:2.0.1")

import com.google.android.play.core.review.ReviewManagerFactory
import com.google.android.play.core.review.ReviewManager
import android.app.Activity
import kotlinx.coroutines.suspendCancellableCoroutine
import kotlin.coroutines.resume

class PlayReviewAPI(context: Context) : ReviewPlatformAPI {
    private val manager: ReviewManager = ReviewManagerFactory.create(context)

    suspend fun requestReview(activity: Activity) {
        val reviewInfo = suspendCancellableCoroutine<ReviewInfo?> { cont ->
            manager.requestReviewFlow().addOnCompleteListener { task ->
                cont.resume(if (task.isSuccessful) task.result else null)
            }
        } ?: return

        suspendCancellableCoroutine<Unit> { cont ->
            manager.launchReviewFlow(activity, reviewInfo).addOnCompleteListener {
                // Completion fires whether or not the dialog was actually shown.
                // This is Google's black box — do not infer "user rated" from this.
                cont.resume(Unit)
            }
        }
    }
}
```

**Important:** Won't show on installs from outside the Play Store (sideloaded APK, F-Droid, direct install). Use a real Play Store build from the internal testing track for QA.

---

## 13. Platform — Flutter

Package: **`in_app_review` ^2.0.11** (current). No Info.plist keys or AndroidManifest keys required.

```dart
// pubspec.yaml: in_app_review: ^2.0.11

import 'package:in_app_review/in_app_review.dart';

class FlutterReviewAPI implements ReviewPlatformAPI {
  final InAppReview _plugin = InAppReview.instance;

  @override
  Future<void> requestReview() async {
    if (await _plugin.isAvailable()) {
      await _plugin.requestReview();
    }
  }
}
```

**Important iOS 16+ behavior:** The plugin dispatches `AppStore.requestReview(in: scene)` to `DispatchQueue.main.async` and immediately returns `result(nil)`. The Dart `Future` resolves *before* the native dialog is presented. Never treat `await _plugin.requestReview()` as "the dialog was shown."

---

## 14. Platform — React Native / Expo

- **React Native (bare):** `react-native-in-app-review`
- **Expo (managed):** `expo-store-review`

```typescript
// Expo example
import * as StoreReview from 'expo-store-review';

export const ExpoReviewAPI: ReviewPlatformAPI = {
  requestReview: async () => {
    if (await StoreReview.hasAction()) {
      await StoreReview.requestReview();
    }
  },
};
```

```typescript
// React Native (bare) example
import InAppReview from 'react-native-in-app-review';

export const RNReviewAPI: ReviewPlatformAPI = {
  requestReview: async () => {
    if (InAppReview.isAvailable()) {
      await InAppReview.RequestInAppReview();
    }
  },
};
```

Same principles: behavioral peak-moment triggers, quota gate, no button-triggered calls.

---

## 15. Testing strategy

- Always add a **debug-only force-show path** that bypasses eligibility and the quota gate (manual QA)
- Never auto-trigger in debug builds — keeps production metrics clean
- **TestFlight: dialog will never appear.** Don't debug there.
- **Simulator/emulator:** may render a dev-build dialog on iOS but submission is blocked; Android won't work at all
- Real QA requires a **production App Store / Play Store build** on a physical device with a test user whose 3/365 quota hasn't already been burned
- Manual QA checklist:
  1. Fresh install → reach the peak moment via normal user flow
  2. Verify dialog appears (or confirm it's quota-throttled — check local quota gate state)
  3. Verify events flow through the wired logger (check Firebase DebugView / Mixpanel Live / PostHog event feed / console logs — whatever adapter is in use)
  4. Verify rejecting the dialog doesn't log any fake "rated" signal
- Debug override example (Dart):
  ```dart
  if (kDebugMode && forceShow) {
    await _plugin.requestReview();  // bypass gate entirely
    return;
  }
  ```

---

## 16. Anti-pattern rejection playbook

When the user asks for one of these, **do not implement it**. Cite Apple's/Google's verbatim wording from section 1 and offer the alternative.

### 1. "Show it on first launch" / "Show it during onboarding"
> **Refuse.** Apple HIG: *"give people time to form an opinion about your app before asking for a rating."* Users who've never used the app have no basis to rate it — you'll get low-quality ratings and burn a quota slot. Run the code scan in section 4 to find a behavioral peak moment instead.

### 2. "Show it every session"
> **Refuse.** You'll burn all 3 quota slots in week 1 and be unable to prompt for the rest of the year. The quota gate in section 6 enforces a 90-day local cooldown plus a rolling 365-day counter.

### 3. "Show it after an error or crash"
> **Refuse.** The user is actively frustrated — worst possible emotional state to ask for a rating. HIG guidance is to ask during natural and happy moments. Section 4 Phase 2 explicitly excludes error-adjacent code paths.

### 4. "Gate a feature behind rating the app"
> **Refuse.** Direct violation of App Store Review Guideline 3.2.2(x): *"Apps must not force users to rate the app... in order to access functionality, content, or use of the app."* This is an immediate rejection condition.

### 5. "Redirect to the App Store review URL as the primary trigger"
> **Refuse as primary.** Acceptable only as a secondary path from an explicit "View in App Store" button in Settings. Never as the primary rating trigger — creates friction and violates UX guidance. Google explicitly warns: *"you should not have a call-to-action option (such as a button) to trigger the API."*

### 6. "Add a 'Rate us' button in Settings that calls `requestReview()`"
> **Refuse.** Both Apple's plugin maintainers and Google's docs explicitly warn against this. The quota may already be burned, so the button will silently do nothing — *"presenting a broken experience to the user"*. For a user-initiated rate-us button, open the App Store / Play Store listing URL instead. Reserve `requestReview()` for behavior-driven triggers.

### 7. "Track whether the user actually submitted a rating" / "Log a `rating_submitted` event"
> **Refuse.** Neither Apple nor Google exposes this. Any event you log for "rated" is a lie. Track `review_prompt_requested` and correlate with App Store Connect / Play Console rating deltas over weeks, not days. Section 8 lists the honest event schema.

### 8. "Add an 'Are you enjoying the app?' dialog before the native prompt"
> **Present both sides (section 5) and offer the alternative.** This is a gray area. Apple Guideline 1.1.7 says *"we will disallow custom review prompts,"* and it's unclear whether a sentiment question counts. Some apps use it without issue; others get flagged. The safer alternative is behavioral inference via section 4 — detect happy users via their *actions* rather than asking them. Only implement the sentiment gate if the developer explicitly accepts the rejection risk.

### 9. "Make this Flutter-specific" / "I only care about iOS"
> **Push back on unnecessary scope limitation.** The architecture is identical across stacks. I'll generate the coordinator + quota gate + logger abstraction in the language you need, but the pattern stays the same. If you later add another platform, you only implement the platform API wrapper — the rest is reused. Which stack are you currently writing code in?

### 10. "Just log directly to Firebase Analytics" / "Hard-code Mixpanel calls"
> **Refuse hard-coding.** The review flow uses a `ReviewEventLogger` abstraction (section 7) so it works for devs who use Mixpanel, Amplitude, PostHog, or no analytics at all. You provide a 5-line adapter for your chosen provider. Here's the adapter for your provider — wired through the interface, not hard-coded into the coordinator.

---

## 17. Debugging checklist — "I implemented this but ratings aren't increasing"

Run through these in order. Stop at the first one that applies.

1. **Is the app on the App Store / Play Store, not TestFlight / internal testing?** TestFlight never shows the iOS dialog. Internal testing Play tracks work but with throttling quirks.
2. **Is the tester on a real device?** Simulator renders but can't submit; emulator won't work at all on Android.
3. **Has the tester already seen 3 dialogs in the last 365 days from any app?** Apple's throttle is per-user cross-app. If they burned their quota on another app, yours won't show either.
4. **Is the prompt in onboarding or within the first 30 seconds of first launch?** Move it to a behavioral peak moment via the code scan in section 4.
5. **Volume math:** `attempts × ~15% dialog-shown × ~30% star-tap × ~50% submit` ≈ **~1 rating per 40 attempts**. Check whether you actually have enough volume to expect a visible lift. At <100 attempts/week, most of what you're measuring is noise.
6. **How long since release?** App Store Connect rating counts settle over 24-72h per rating and can **decrease** due to silent fraud detection — this is normal. Trend over weeks, not days.
7. **Is the quota gate respecting the 90-day cooldown?** If you set it too low (e.g., 7 days), power users burn all 3 slots in the first month. Check the quota gate's stored state via the debug override path.
8. **Is the `ReviewEventLogger` wired?** Check the events flowing through whatever adapter the dev is using — Firebase DebugView, Mixpanel Live View, PostHog event feed, or console output. If `review_trigger_identified` fires but `review_prompt_requested` doesn't, the gate is blocking (check params for reason). If `review_prompt_requested` fires but no ratings arrive on the store, that's the normal Apple/Google black box — see items 1-6.

---

## 18. Integration with `review-management`

This skill is **implementation only**. For the following, defer to the `review-management` skill:

- Responding to reviews that come in (response templates, tone, the HEAR framework)
- Sentiment analysis of review *text* (not pre-prompt sentiment gating)
- Portfolio-level rating strategy across multiple apps
- Dealing with a rating average drop or review velocity collapse

Invoke both skills in sequence for a full rating strategy review: `in-app-review` for the code, `review-management` for the strategy.

---

## Appendix: Sources

- [Apple App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) — Guidelines 1.1.7 and 3.2.2(x)
- [Apple HIG — Ratings and Reviews](https://developer.apple.com/design/human-interface-guidelines/ratings-and-reviews)
- [Apple `SKStoreReviewController` docs](https://developer.apple.com/documentation/storekit/skstorereviewcontroller)
- [Apple `AppStore.requestReview(in:)` docs](https://developer.apple.com/documentation/storekit/appstore/3954432-requestreview)
- [Apple `RequestReviewAction` (SwiftUI) docs](https://developer.apple.com/documentation/storekit/requestreviewaction)
- [Google Play In-App Review API docs](https://developer.android.com/guide/playcore/in-app-review)
- [pub.dev — `in_app_review` package](https://pub.dev/packages/in_app_review)
- [Steamclock — Rate This App (against sentiment gates)](https://steamclock.com/blog/2019/09/app-reviews/)
- [RevenueCat — How to hack your app store ratings (for sentiment gates)](https://www.revenuecat.com/blog/engineering/how-to-hack-your-app-store-ratings/)
- [Appbot — Prompting for reviews ultimate guide](https://appbot.co/blog/prompting-for-app-reviews-ratings-ios-android-ultimate-guide/)
- [SwiftLee — SKStoreReviewController guide](https://www.avanderlee.com/swift/skstorereviewcontroller-app-ratings/)
- [sarunw — Review in UIKit](https://sarunw.com/posts/how-to-request-users-to-review-app-in-uikit/)
- [nilcoalescing — Review in SwiftUI](https://nilcoalescing.com/blog/RequestingAppStoreReviewsInSwiftUI/)
