# 08 — Implementation Plan
**NextShift · Production Implementation Roadmap**
Status: Final — based on the completed Product Audit and Production Architecture. No scope changes.

---

## How to read this plan

Phases are ordered so nothing is built twice. The rule driving the order: **infrastructure before features, data before UI, mocked-data UI before real-data wiring, single feature complete before the next feature starts.** Each phase lists what it depends on and what it unlocks, so the sequence can't be reshuffled without breaking a dependency.

Complexity ratings are relative to this app's scope, not absolute.

---

## PART A — Development Phases

### Phase 1 — Project Setup
**Goal:** A clean, correctly configured Xcode project that the whole team can build from day one.
**Tasks:**
- Create Xcode project (SwiftUI lifecycle, iOS min target decided — recommend iOS 17+ for SwiftData/Observation)
- Configure project structure matching the approved architecture folders (empty groups, no code yet)
- Set up `.gitignore`, SwiftLint/SwiftFormat config, git repo, branch strategy (trunk-based with feature branches)
- Configure build schemes (Debug/Release), bundle ID, provisioning, App Group entitlement (for future Widget/iCloud sharing)
- Set up CI skeleton (build + lint on PR) — even a minimal GitHub Actions/Xcode Cloud config
**Dependencies:** None — this is the starting point.
**Expected result:** Empty app that builds and runs on device/simulator with the correct folder skeleton committed.
**Complexity:** Low

---

### Phase 2 — Design System (`Theme/`)
**Goal:** Every color, font, spacing, and radius token from the Design System spec exists in code, before any screen is built against it.
**Tasks:**
- Implement `ColorTokens`, `Typography`, `Spacing`, `Radius`, `Shadows`
- Add color sets to `Assets.xcassets` (light/dark variants, semantic shift-type colors locked per the audit)
- Build `ThemeManager` skeleton (mode storage only, no persistence yet — that comes with Repositories)
- Create a `PreviewData` theme-preview canvas (a single SwiftUI Preview rendering all tokens) for visual QA of the system itself
**Dependencies:** Phase 1
**Expected result:** A visually verifiable, centralized design system with zero screens using hardcoded colors/fonts.
**Complexity:** Low

---

### Phase 3 — Navigation Skeleton
**Goal:** The full app shell (Splash → Onboarding → Login → 5-tab main app) navigates correctly with placeholder screens, before any real feature exists.
**Tasks:**
- Build `AppRouter`/`NavigationCoordinator`, `Route` enum, `Tab` enum
- Build `RootView` state machine (`.splash / .onboarding / .auth / .main`)
- Build `TabBarWithFAB` shell with 5 placeholder tab screens (empty `Text("Home")` etc.)
- Wire Create Shift as a `.sheet` presentation (empty placeholder sheet)
- Verify independent `NavigationStack` path per tab (push/pop preserved across tab switches)
**Dependencies:** Phase 2 (uses theme tokens for the tab bar/FAB styling)
**Expected result:** Tapping through the entire app shell — onboarding, login, all 5 tabs, opening/closing Create Shift — works end to end with placeholder content. **This is Milestone 2.**
**Complexity:** Medium

---

### Phase 4 — Core Components (`Components/`)
**Goal:** The full reusable UI library exists and is visually finished, built and tested against static preview data — before any feature screen consumes it.
**Tasks:**
- Build in this order (low-dependency first): `SectionHeader`, `PrimaryButton`, `SecondaryButton`, `DestructiveTextButton` → `GlassCard` → `ShiftBadge` → `StatTile`, `ProgressRing` → `ShiftCard` (compact + expanded) → `MiniCalendar` → `ChartContainer` → `SettingsRow`, `FormField` → `PremiumFeatureRow` → `EmptyStateView`, `LoadingView`, `ErrorView`
- Every component gets a SwiftUI Preview with realistic `PreviewData`, including edge cases (long text, zero state, error state)
**Dependencies:** Phase 2 (design tokens), Phase 1
**Expected result:** A component catalog (a debug-only "Component Gallery" screen is worth building here) that any feature can now be assembled from without inventing new UI patterns.
**Complexity:** Medium

---

### Phase 5 — SwiftData Models (`Models/`)
**Goal:** The full data schema exists, matching the Production Architecture exactly, including the reserved v1.1 fields (`recurrenceGroupID`, etc.) to avoid future migrations.
**Tasks:**
- Implement `Shift`, `Workplace`, `SalaryRecord`, `UserProfile`, `AppSettings`, `Reminder`, `StatisticsSnapshot`, `PremiumSubscription` as `@Model` classes
- Implement supporting enums (`ShiftType`, `ShiftStatus`, `EmploymentType`, `TimeFormat`, `Weekday`, `AppearanceMode`, `ReminderTrigger`, `PremiumPlan`)
- Configure `ModelContainer` in `App/`, including CloudKit-backed container setup (schema only — sync behavior comes later)
- Write a schema migration plan document (even if v1 has no migrations yet) so the team has a process ready
**Dependencies:** Phase 1
**Expected result:** Schema compiles, a `ModelContainer` initializes successfully, and seed/preview data can be inserted and fetched in a throwaway test harness. **This is the foundation for Milestone 3.**
**Complexity:** Medium

---

### Phase 6 — Repositories
**Goal:** Every Feature-facing data operation (CRUD + queries) is available through a typed, async repository — no ViewModel will ever touch `ModelContext` directly.
**Tasks:**
- Define repository protocols: `ShiftRepository`, `WorkplaceRepository`, `SalaryRepository`, `SettingsRepository`, `ReminderRepository`
- Implement SwiftData-backed concrete types for each
- Implement in-memory mock implementations of each (for Previews/tests) — do this alongside, not after, the real ones
- Unit test basic CRUD for each repository against an in-memory `ModelContainer`
**Dependencies:** Phase 5
**Expected result:** Local data can be created, fetched, updated, deleted through a clean async API, verified by passing unit tests. **This completes Milestone 3 (Local Data Works).**
**Complexity:** Medium

---

### Phase 7 — Managers
**Goal:** All business-logic managers exist and are independently testable, injected at the app root, before any Feature ViewModel depends on them.
**Tasks:**
- `SalaryCalculator`, `StatisticsCalculator` — pure functions over `[Shift]` + `AppSettings`, unit tested with fixed input/output cases first (these have no framework dependency, build and test them first)
- `ThemeManager` — wire real persistence via `SettingsRepository` (upgrade from Phase 2's skeleton)
- `NotificationManager` + underlying `NotificationService` — permission request flow, schedule/cancel wrapper
- `ReminderScheduler` — depends on `NotificationManager` + `ShiftRepository` + `ReminderRepository`
- `PremiumManager` + `StoreKitService` — entitlement check, purchase/restore flow (can be built against StoreKit's local testing/sandbox config)
- `SyncManager` + `CloudKitService` — sync status tracking, conflict policy (last-write-wins, explicitly implemented)
- Inject all managers via `.environment(...)` at `App/` root
**Dependencies:** Phase 6 (Repositories), Phase 5 (Models)
**Expected result:** Every manager has unit tests where feasible (Calculators fully, others via mocked Services), and is available app-wide via environment injection.
**Complexity:** High

---

### Phase 8 — Home Feature
**Goal:** First real, fully wired feature screen — validates the whole stack (Theme → Components → Models → Repositories → Managers → Feature) works end to end.
**Build first:** Today's Shift hero card (`ShiftCard.expanded` + `ProgressRing` countdown), This Month stat row (`StatTile` × 3, powered by cached `SalaryRecord`/`StatisticsSnapshot`), read-only mini calendar.
**Wait until later:** Any personalization/AI-driven insight content — not in MVP.
**Can be mocked during development:** Use `PreviewData` + mock repositories to build the full UI before wiring `SalaryCalculator`/`StatisticsCalculator` outputs — this lets UI and data-layer work happen in parallel if the team splits.
**Dependencies:** Phases 4, 6, 7
**Expected result:** Home tab shows live data from the local store, reacts correctly to a shift being added/edited elsewhere (via `@Query`/Observation).
**Complexity:** Medium

---

### Phase 9 — Calendar Feature
**Goal:** Full month calendar with real shift data, day selection, and day-detail entry point to Create Shift.
**Build first:** Month grid (`MiniCalendar.fullMode`) bound to real `Shift` data, day-detail view showing that day's shift(s), legend.
**Wait until later:** Multi-shift-per-day conflict UI, recurring-shift visual grouping (v1.1) — build the data model support now (already reserved) but not the UI.
**Can be mocked:** Month navigation logic can be built and tested against `PreviewData` fixtures spanning multiple months before the real repository is wired.
**Dependencies:** Phase 8 (reuses the same calendar component, patterns proven)
**Expected result:** Calendar reflects real persisted shifts, tapping a day opens Create Shift pre-filled with that date (validates the sheet-with-context pattern from the architecture).
**Complexity:** Medium

---

### Phase 10 — Create Shift Feature
**Goal:** Full create/edit shift flow, the only mutation-heavy screen in MVP, so gets first-class validation and error handling.
**Build first:** Shift type picker (`ShiftBadge` grid), date/time pickers, workplace/department selection, save/cancel, validation (end time after start time, etc.)
**Wait until later:** Recurring shift toggle/pattern builder — explicitly v1.1, do not build the UI now even though the model field exists.
**Can be mocked:** Workplace picker can initially hardcode a single default workplace before the Workplace management UI (Phase 12/Profile) exists.
**Dependencies:** Phases 6, 7 (needs `ShiftRepository`, and triggers `ReminderScheduler` on save if Premium reminders are enabled)
**Expected result:** A shift can be created, edited, deleted, and immediately reflected in Home and Calendar. **This is the core functional milestone — Milestone 4 candidate.**
**Complexity:** Medium-High

---

### Phase 11 — Statistics Feature
**Goal:** Charts and aggregate breakdowns powered by `StatisticsCalculator`, using the shared `ChartContainer`.
**Build first:** Top stat tiles (worked hours, salary, overtime, avg shift), weekly hours line chart, monthly earnings bar chart.
**Wait until later:** Advanced/Premium-only breakdowns (shift-type distribution deep dive, forecasting) — MVP ships the free-tier view fully functional; Premium-gated sections come once `PremiumManager` gating is proven on one screen first (Phase 15).
**Can be mocked:** Chart layout and interactions can be built entirely against static `PreviewData` series before `StatisticsCalculator` is finalized.
**Dependencies:** Phase 7 (`StatisticsCalculator`), Phase 4 (`ChartContainer`)
**Expected result:** Statistics tab renders real, correct aggregates for the current and past periods, verified against manually-calculated expected values for a test dataset.
**Complexity:** Medium

---

### Phase 12 — Salary Feature
**Goal:** Full salary breakdown (regular/overtime/bonus/tax/net), earnings history chart, upcoming payday — reachable via push from Home/Statistics, not a tab.
**Build first:** Current earnings ring + breakdown list (`StatTile`/`GlassCard` composition), earnings history bar chart.
**Wait until later:** Payslip export, salary forecasting (both correctly Premium/v1.1 per audit) — build the "Quick Actions" row with disabled/Premium-locked states, not full functionality, in MVP.
**Can be mocked:** Same as Statistics — build against `PreviewData` first, wire `SalaryCalculator` output second.
**Dependencies:** Phase 7 (`SalaryCalculator`), Phase 11 (shared chart patterns already proven)
**Expected result:** Salary screen matches the audited design, correctly computed from real shift data and `AppSettings` (rate, overtime multiplier, tax %).
**Complexity:** Medium

---

### Phase 13 — Profile Feature
**Goal:** Account display, work information, edit flow, achievement badges (lightweight, static logic).
**Build first:** Profile header, work information rows (`SettingsRow` reuse), Edit Profile form (`FormField` reuse).
**Wait until later:** Multi-workplace management UI — v1.1, per audit. MVP profile shows one primary workplace only.
**Can be mocked:** Achievement badge logic (streaks, perfect attendance) can be stubbed with fixed sample values initially, then wired to a real streak-calculation utility.
**Dependencies:** Phase 6 (`WorkplaceRepository`, settings repository), Phase 4
**Expected result:** Editable, persisted user profile, correctly reflecting real shift-derived stats where shown (e.g. career earnings total).
**Complexity:** Low-Medium

---

### Phase 14 — Settings Feature
**Goal:** Full settings screen exactly matching the audited grouping, every toggle/value wired to `AppSettings` via `SettingsRepository`.
**Build first:** Calendar & Time section, Salary Settings section (these directly affect `SalaryCalculator`/`StatisticsCalculator` output, so must be correct early), Notifications toggle (top-level).
**Wait until later:** Granular Weekly Summary / Monthly Report toggles were flagged in the audit as redundant with the general Notifications toggle — implement as **sub-options nested under Notifications**, not separate top-level rows, resolving that finding during build rather than after.
**Can be mocked:** Data & Sync section (iCloud toggle, backup, export/import) can show correct UI state early but defer real CloudKit wiring until `SyncManager` (Phase 7) is fully validated in isolation.
**Dependencies:** Phase 6, Phase 7 (`ThemeManager`, `SyncManager`, `NotificationManager`)
**Expected result:** Every setting persists correctly, changes take effect immediately (theme, time format, currency) without app restart.
**Complexity:** Medium

---

### Phase 15 — Premium Feature (Paywall + Smart Shift Reminders)
**Goal:** Purchase flow, feature gating, and the one confirmed Premium feature (Smart Shift Reminders) fully functional.
**Build first:** Paywall screens exactly matching the audited design (feature list, Free vs Premium table, plan selection, purchase button), `PremiumManager.canAccess(_:)` gating wired into every already-built screen's Premium-flagged UI (Statistics advanced section, Salary export button, Create Shift recurring toggle — shown but locked).
**Wait until later:** Additional Premium features beyond what's approved (multi-workplace, export, recurring shifts) get their *gating* wired now but their *functionality* ships in v1.1 per roadmap — do not build the functionality early.
**Can be mocked:** Use StoreKit's local `.storekit` configuration file for the entire development cycle; real App Store Connect product wiring happens in Phase 20 (App Store Prep).
**Dependencies:** Phase 7 (`PremiumManager`, `StoreKitService`), all prior Feature phases (since gating touches every screen)
**Expected result:** A real sandbox purchase unlocks Premium state app-wide; Smart Shift Reminders can be configured per shift and fire local notifications correctly at the configured offset.
**Complexity:** High

---

### Phase 16 — Onboarding & Auth Polish
**Goal:** Replace Phase 3's placeholder Onboarding/Login with final, real content and real auth.
**Tasks:**
- Build final 3-page onboarding matching approved visuals
- Wire Sign in with Apple / Google, email/password auth (choose real auth provider — Firebase Auth, Supabase, or CloudKit-only identity, decided at this phase, not before)
- Forgot Password flow, Create Account flow
- Face ID app-lock (using `LocalAuthentication`, gated by the Settings toggle built in Phase 14)
**Dependencies:** Phase 3 (shell), Phase 14 (Face ID setting)
**Expected result:** A new user can complete the full first-run flow from Splash to a populated Home screen with zero placeholder content remaining.
**Complexity:** Medium

---

### Phase 17 — Cross-Feature Integration Pass
**Goal:** Everything built in isolation now gets tested and fixed as one coherent app — this phase exists specifically to catch integration bugs before they're mistaken for feature bugs.
**Tasks:**
- Full manual walkthrough of every user flow end to end
- Fix state-sync issues (e.g. editing a shift in Create Shift correctly refreshes Home, Calendar, Statistics, Salary simultaneously via Observation)
- Verify tab-switch navigation state preservation
- Verify Premium gating consistency across every screen
- Verify dark/light mode across every screen (not just primary dark theme)
**Dependencies:** All of Phases 8–16
**Expected result:** The app behaves as one product, not eight independently-built screens. **This is Milestone 4 (Core Features Complete).**
**Complexity:** Medium-High

---

### Phase 18 — Performance, Accessibility, Localization Pass
**Goal:** Non-functional quality bar met before opening to beta testers.
**Tasks:**
- Performance: Instruments pass (Time Profiler, Allocations) on Home/Calendar/Statistics scroll performance; verify no real-time `.ultraThinMaterial` blur behind scrolling lists per the architecture's performance guidance
- Accessibility: VoiceOver pass on every screen, Dynamic Type support up to at least AX3, sufficient color contrast on all text/chart elements, minimum tap target sizes
- Localization: extract all strings to `Localizable.strings`/String Catalog, verify layout doesn't break with longer translated strings (test with a pseudo-localized build), decide launch language set (recommend English + 1–2 more if targeting EU markets given €-based salary data)
**Dependencies:** Phase 17
**Expected result:** App passes an internal accessibility audit and is structurally ready for localization (even if only English ships at launch).
**Complexity:** Medium

---

### Phase 19 — Beta (TestFlight)
**Goal:** Real users validate the product before public release.
**Tasks:**
- Prepare TestFlight build, internal testing group first, then external
- Set up crash reporting/analytics (privacy-respecting, disclosed in Privacy Manifest)
- Collect structured feedback (a simple in-app feedback channel, already partially planned via Settings → Send Feedback)
- Triage and fix beta-reported bugs, prioritizing data-loss/crash bugs first
**Dependencies:** Phase 18
**Expected result:** A stable build with real usage data and no critical/blocking bugs remaining. **This is Milestone 5 (Beta Ready) → exit criteria for Milestone 6.**
**Complexity:** Medium

---

### Phase 20 — App Store Preparation
**Goal:** Everything Apple requires for submission is ready, independent of code changes.
**Tasks:**
- Finalize App Store Connect product setup (real subscription products matching sandbox-tested IDs from Phase 15)
- Write App Store listing copy (title, subtitle, description, keywords) — ASO pass
- Produce screenshots for all required device sizes (see Pre-Release Checklist)
- Produce Privacy Manifest (`PrivacyInfo.xcprivacy`) declaring all data collection/tracking, third-party SDK privacy manifests included
- Complete App Privacy questionnaire in App Store Connect
- Final legal review: Terms of Service, Privacy Policy live at real URLs (already linked from Settings)
**Dependencies:** Phase 19 (stable build to screenshot/reference)
**Expected result:** A complete, submittable App Store Connect listing. **This is Milestone 6 (Release Candidate) once paired with a final signed build.**
**Complexity:** Medium

---

### Phase 21 — Release Candidate & Submission
**Goal:** Final build submitted and approved.
**Tasks:**
- Freeze feature work, RC build cut, full regression pass against the testing checklists below
- Submit for App Review
- Respond to any App Review feedback/rejection promptly (have a fast-turnaround process defined in advance)
- Plan phased release rollout (recommend Apple's phased release over 7 days rather than 100% instant)
**Dependencies:** Phase 20
**Expected result:** App is live on the App Store. **This is Milestone 7 (App Store Release).**
**Complexity:** Low (if all prior phases were done correctly) / High (if rushed)

---

## PART B — Feature Build Order Summary

| Feature | Build First | Wait Until Later | Mock During Development |
|---|---|---|---|
| Home | Hero shift card, month stat tiles | Personalized insights | Static `PreviewData`, then real Repositories |
| Calendar | Month grid + day detail | Recurrence visual grouping, multi-shift conflict UI | Multi-month fixture data |
| Create Shift | Core form + validation | Recurring pattern builder | Hardcoded single workplace |
| Statistics | Core tiles + 2 charts | Premium deep-dive breakdowns | Static chart series |
| Salary | Breakdown + history chart | Export, forecasting (Quick Actions shown but locked) | Static chart series |
| Profile | Header + work info + edit | Multi-workplace management | Fixed sample achievement values |
| Settings | Calendar/Time, Salary settings, Notifications (unified) | — (all settings are MVP) | Sync status UI before real CloudKit wiring |
| Premium | Paywall + gating + Smart Reminders | All other gated features' *functionality* | StoreKit local config file |

---

## PART C — Milestones

### Milestone 1 — Project Compiles
**Exit criteria:** Phase 1 complete. Empty app builds and installs on a physical device.
**Testing checklist:**
- [ ] Clean build succeeds with zero warnings on CI
- [ ] App launches on both a real device and simulator
- [ ] Folder structure matches approved architecture exactly

### Milestone 2 — Navigation Works
**Exit criteria:** Phase 3 complete.
**Testing checklist:**
- [ ] Splash → Onboarding → Login → Home flow completes with placeholder screens
- [ ] All 5 tabs reachable, each with independent back-stack behavior
- [ ] Create Shift sheet opens/dismisses from every tab
- [ ] No navigation dead-ends or crashes on rapid tab-switching

### Milestone 3 — Local Data Works
**Exit criteria:** Phases 5 and 6 complete.
**Testing checklist:**
- [ ] Shift, Workplace, and Settings records can be created/read/updated/deleted via unit tests
- [ ] Data persists across app relaunch
- [ ] In-memory mock repositories work identically for Previews
- [ ] No data corruption after forced-quit during a write

### Milestone 4 — Core Features Complete
**Exit criteria:** Phases 8–17 complete.
**Testing checklist:**
- [ ] Full create → edit → delete shift flow reflected correctly across Home, Calendar, Statistics, Salary
- [ ] All calculated values (salary, overtime, stats) manually verified against a known test dataset
- [ ] Premium gating correctly blocks/unblocks every gated element
- [ ] Dark and light mode both fully correct on every screen
- [ ] No placeholder/lorem ipsum content remains anywhere

### Milestone 5 — Beta Ready
**Exit criteria:** Phase 18 complete, Phase 19 in progress.
**Testing checklist:**
- [ ] VoiceOver can complete every core flow (create shift, view salary, change settings)
- [ ] Dynamic Type at largest accessibility size doesn't break any layout
- [ ] Instruments shows no memory leaks or major frame drops on primary scroll views
- [ ] Crash reporting and feedback channel are live
- [ ] TestFlight build installs and runs for external testers

### Milestone 6 — Release Candidate
**Exit criteria:** Phase 19 bugs triaged/fixed, Phase 20 complete.
**Testing checklist:**
- [ ] Zero known crash bugs, zero known data-loss bugs
- [ ] Full regression pass across all 8 Features on both light/dark mode
- [ ] Real (sandbox-verified) subscription purchase flow works against production App Store Connect config
- [ ] Privacy Manifest accurately reflects all data collection
- [ ] All legal links (Privacy Policy, ToS) resolve to live pages

### Milestone 7 — App Store Release
**Exit criteria:** Phase 21 complete, app approved and live.
**Testing checklist:**
- [ ] Fresh install from the actual App Store (not TestFlight) completes onboarding correctly
- [ ] Purchase flow works with a real payment method in production
- [ ] Push/local notifications (Smart Shift Reminders) fire correctly on a clean-install device
- [ ] App icon, screenshots, and listing copy render correctly on the live App Store page
- [ ] Phased rollout monitored for crash-rate spikes before reaching 100%

---

## PART D — Final Pre-Release Checklist

### Performance
- [ ] No dropped frames on Home/Calendar/Statistics scroll (verified via Instruments, not just visual impression)
- [ ] Cold launch time measured and under target (recommend < 2s to interactive)
- [ ] `SalaryCalculator`/`StatisticsCalculator` results cached, not recomputed on every render
- [ ] No synchronous disk/network work on the main thread anywhere in the Feature layer

### Accessibility
- [ ] VoiceOver labels present and meaningful on every interactive element (not just default "Button")
- [ ] Dynamic Type supported through at least Accessibility Large sizes without truncation/overlap
- [ ] Color contrast meets WCAG AA minimum on all text and chart elements
- [ ] Reduce Motion respected (progress ring/chart animations degrade gracefully)
- [ ] Minimum 44×44pt tap targets throughout

### Localization
- [ ] All user-facing strings externalized (no hardcoded strings in Feature code)
- [ ] Currency, date, and time formatting use `Locale`-aware formatters, not hardcoded `€`/`24h` assumptions
- [ ] Pseudo-localized build tested for layout breakage
- [ ] Launch language set finalized and confirmed with App Store Connect metadata

### App Store Assets
- [ ] App icon (all required sizes) finalized, no placeholder art
- [ ] Screenshots produced for all required device sizes (6.9", 6.5" iPhone; iPad if supporting iPad)
- [ ] App preview video (optional but recommended for a visually strong app like this)
- [ ] Marketing copy (title ≤30 chars, subtitle ≤30 chars, keywords, description) finalized per ASO review
- [ ] Support URL and marketing URL live

### Screenshots
- [ ] Feature the strongest visual screens first: Home hero card, Calendar, Salary breakdown, Statistics chart
- [ ] Consistent device frame/status bar treatment across all screenshots
- [ ] Localized screenshots if shipping non-English locales at launch
- [ ] No real user data visible — all screenshot data is realistic sample data

### Privacy Manifest
- [ ] `PrivacyInfo.xcprivacy` declares all collected data types (account info, usage data if analytics included)
- [ ] Required Reason API usage declared (e.g. if using UserDefaults, file timestamp APIs)
- [ ] Third-party SDKs' own privacy manifests included and aggregated correctly
- [ ] App Privacy "Nutrition Label" in App Store Connect matches the manifest exactly

### App Review Requirements
- [ ] Sign in with Apple offered if any other third-party login (Google) is offered, per Apple guideline 4.8
- [ ] Subscription terms, price, and duration clearly displayed before purchase (already satisfied by the audited Premium screen design)
- [ ] Restore Purchases button present and functional
- [ ] Account deletion available in-app (already present in Settings — verify it fully deletes/disassociates data, not just signs out)
- [ ] Demo account credentials prepared for App Review if login is required to test the app
- [ ] App functions fully offline for core features (shift creation, calendar, salary calc) since this is a local-first app — Review may test with network disabled

---

**This document reflects the final, approved Product Audit and Production Architecture. No new features, screens, or architectural changes are introduced in this plan — it defines sequencing, dependencies, and quality gates only.**
