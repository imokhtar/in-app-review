# Contributing

Thanks for considering a contribution. This skill is deliberately opinionated and prose-heavy, and it's easier to keep it sharp with a small set of contribution principles.

## Scope of the skill

This skill is **implementation-only**. It covers:

- How to call the native rating API correctly across mobile stacks
- When to call it (behavioral peak-moment triggers via code scan)
- Quota-aware budgeting with a local cooldown
- Provider-neutral logging via the `ReviewEventLogger` abstraction
- Platform-specific code for Swift / SwiftUI / Kotlin / Dart / TypeScript
- Refusing anti-patterns with verbatim Apple/Google quotes

This skill is **not**:

- A review-response strategy guide (see the `review-management` skill)
- A general ASO or rating-growth marketing guide
- A drop-in library or package — it produces adapted code per project, not reusable artifacts

Contributions that drift into those areas will be asked to move to a different skill.

## Opening an issue

Great issues to open:

- **A primary-source citation I got wrong or is outdated.** If Apple updated Guideline 1.1.7, Google changed the Play In-App Review docs, or a package version changed, point me at the source and I'll update verbatim.
- **A platform I didn't cover.** React Native on Fabric, Kotlin Multiplatform, Jetpack Compose, SwiftData integration — concrete code patterns welcome.
- **A new analytics adapter.** If you use a provider I don't list and want to contribute a 5-line `ReviewEventLogger` adapter, those are always welcome.
- **An edge case in the code-scan algorithm.** If you found a peak-moment keyword family I missed (Tier 1-4 in section 4), let me know — real-world examples from your codebase are the best evidence.
- **An anti-pattern I should reject.** If you've seen a bad pattern in the wild that the skill doesn't push back on, share it and cite the Apple/Google guideline it violates.

Less great issues:

- "This should recommend the sentiment gate pattern." The gray-area framing is intentional, fact-checked, and not moving. See section 5 of `SKILL.md`.
- "Add X framework's state management layer." The architecture is deliberately framework-agnostic (no Clean Architecture, Riverpod, MVVM, Bloc, Redux) so it integrates into whatever pattern the project already uses.
- "Include code for Y niche package." Every code template shown in the skill should work for most devs using a mainstream package. Niche-package adapters can live as issue comments for reference.

## Opening a PR

1. **Cite sources.** Every factual claim added to the skill should link to a primary source (Apple docs, Google docs, reputable engineering blog). See the `Appendix: Sources` section in `SKILL.md` for the convention.
2. **Stay prose-heavy, code-light.** The skill is Claude's reading material, not a runnable package. Keep code snippets to the minimum needed to illustrate the pattern — ~15 lines max per example.
3. **Stack-agnostic by default.** Any new guidance should apply across at least two of the five target stacks (iOS UIKit, iOS SwiftUI, Android Kotlin, Flutter, React Native/Expo). Single-stack additions go in the per-platform sections (10-14), not the core sections.
4. **Don't duplicate code into multiple sections.** The `ReviewCoordinator` pseudocode lives in section 9. Platform sections should reference it, not re-inline it.
5. **Update the anti-pattern playbook if you add a rejection.** New rejections should include (a) the bad request pattern, (b) the verbatim Apple/Google quote being cited, (c) the positive alternative.
6. **Keep the frontmatter trigger description accurate.** If you add a new keyword (e.g., a new framework's API name), add it to the `description` field so the skill auto-loads on that phrase.

## Style conventions

- **"The skill"** = this document (`SKILL.md`). **"Claude"** = the agent reading it. Write instructions as *"Run a Grep for X"*, not *"The skill should run Grep for X"*.
- **Verbatim quotes** from Apple/Google use blockquote + italic: `> *"..."*`
- **Tables** for anything comparative (platform matrix, quota matrix, adapter matrix). Prose for principles.
- **No emojis** in the skill body. The `✅ Recommended` in section 4 Phase 4 is the one exception and is a UI affordance for Claude's output to users, not a decorative element.

## License on contributions

By opening a PR, you agree your contribution is licensed under the MIT License (see `LICENSE`).
