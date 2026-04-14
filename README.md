# `in-app-review` — Claude Code Skill

A [Claude Code](https://claude.com/claude-code) skill that teaches Claude how to implement native in-app rating prompts the right way — across **native iOS (UIKit + SwiftUI), native Android, Flutter, React Native, and Expo**.

## Why this exists

Most indie devs reach for `SKStoreReviewController.requestReview()` / Google Play's `ReviewManager`, call it in onboarding or on first launch, and then wonder why their App Store rating count isn't moving. What's actually happening:

- **Apple limits display frequency** — StoreKit shows the prompt at most three times within 365 days for a person who hasn't rated or reviewed your app on that device, and the API may return without presenting anything.
- **TestFlight never shows the dialog, and simulators are only for UI testing** — development builds can show the prompt, but local testing doesn't produce a real review.
- **The system never tells your app** whether the dialog was shown, dismissed, or resulted in a rating.
- **Onboarding prompts run against Apple's HIG guidance** (*"give people time to form an opinion about your app before asking for a rating"*) and usually produce lower-signal reviews.

This skill codifies the patterns that actually work: **behavioral peak-moment triggers** (detect happy users from their actions, not a custom "Are you enjoying?" dialog), a **local 90-day cooldown + 365-day quota gate**, and a **provider-neutral logger abstraction** so the same architecture works with Firebase, Mixpanel, Amplitude, PostHog, Segment, or no analytics at all.

## What it does

When you ask Claude to add or debug an in-app rating prompt in any project, the skill:

1. **Scans your codebase** via a 5-phase Grep algorithm — completion handlers, achievement events, celebration UI, success haptics, existing analytics events at peak moments. Explicitly excludes error handlers and onboarding/launch code paths.
2. **Proposes the top 3-5 candidate triggers** with file paths and line numbers so you can click straight to the code.
3. **Generates a stack-appropriate implementation** — `ReviewCoordinator` + `ReviewQuotaGate` + `ReviewEventLogger` + platform API wrapper, in Swift / SwiftUI / Kotlin / Dart / TypeScript, whichever matches your project.
4. **Wires a 5-line logger adapter** matching your analytics provider (or defaults to no-op if you don't track).
5. **Refuses anti-patterns** with verbatim Apple and Google quotes — onboarding prompts, post-error triggers, "Rate us" buttons, feature-gating behind ratings, `rating_submitted` event tracking (impossible to detect), Flutter-only scoping, hard-coded analytics calls.

## Install

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/imokhtar/in-app-review.git ~/.claude/skills/in-app-review
```

That's it. The skill is now auto-loaded in every Claude Code session across all your projects.

To update later:

```bash
cd ~/.claude/skills/in-app-review && git pull
```

## Usage

Just ask Claude for help with a rating prompt in any project:

> *"Add an in-app review prompt to my Flutter app"*

> *"Why isn't my iOS rating count increasing?"*

> *"Help me implement SKStoreReviewController in my SwiftUI app"*

The skill auto-loads via its description match and walks Claude through the code scan, architecture proposal, and stack-specific implementation.

## What's inside

| File | Purpose |
|---|---|
| [`SKILL.md`](SKILL.md) | The skill file Claude reads — ~800 lines of grounded guidance, verbatim Apple/Google quotes, 5-phase code-scan algorithm, 4-language adapter examples, 10-entry anti-pattern rejection playbook |
| `README.md` | You're reading it |
| `LICENSE` | MIT |
| `CONTRIBUTING.md` | How to propose improvements |

## Core principles the skill enforces

1. **Engineer the trigger set, not the call frequency** — you can't outrun Apple's silent throttle, so every call must land on a genuine peak moment
2. **Behavioral inference over sentiment dialogs** — infer happy users from their actions (completion events, streaks, achievements) rather than asking with a custom "Are you enjoying?" modal (which is a gray area under Guideline 5.6.1)
3. **Local quota gate with 90-day cooldown** — never burn Apple's 3/365 quota in week one of a power user's lifecycle
4. **Provider-neutral logging** — 1-method `ReviewEventLogger` interface with default no-op; 5-line adapters for every major analytics provider
5. **Stack-agnostic architecture** — identical 4-component design (`Coordinator` + `QuotaGate` + `Logger` + `PlatformAPI`) across Swift / SwiftUI / Kotlin / Dart / TypeScript

## Sources

Every factual claim in the skill cites a primary source. See the `Appendix: Sources` section in [`SKILL.md`](SKILL.md) for the full list, including:

- Apple App Store Review Guidelines (5.6.1, 3.2.2(x))
- Apple HIG — Ratings and Reviews
- Apple `SKStoreReviewController` / `AppStore.requestReview(in:)` / `RequestReviewAction` docs
- Google Play In-App Review API docs
- `in_app_review` Flutter package (v2.0.11)
- Community engineering posts from RevenueCat, Steamclock, Appbot, SwiftLee, sarunw, and nilcoalescing

## Related skills

- [`review-management`](https://github.com/anthropics/skills) — for response strategy to reviews that come in (this skill is implementation-only)

## License

MIT — see [LICENSE](LICENSE). Use, modify, and redistribute freely with attribution.

## Credits

Authored with Claude Code. The skill's core pattern (behavioral peak-moment triggers over sentiment gates) was refined after a multi-round fact-check against Apple's and Google's primary sources — v1 of the design endorsed sentiment gates as the core technique and was revised after discovering the gray-area Guideline 5.6.1 risk. See the plan file in the companion [authoring notes](https://github.com/imokhtar/in-app-review/issues) if you want to understand the design rationale.
