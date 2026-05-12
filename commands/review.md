---
name: review
description: Load Firefox in-product messaging review guidance for ASRouter, about:welcome, feature callouts, infobars, spotlights, menu messages, and toast notifications. Covers message authoring, targeting, telemetry, localization, accessibility, and cross-profile concerns. Use when reviewing patches that touch messaging system code.
---

## Standing Conventions

### Message authoring & schemas
- When adding a new field to a message template, update the corresponding JSON schema *and* regenerate `MessagingExperiment.schema.json` via `make-schemas.py`; missing schema regeneration silently breaks experimenter validation tests.
- Document new special message actions and trigger params in `toolkit/components/messaging-system/schemas/SpecialMessageActionSchemas/index.md` and targeting attributes in `browser/components/asrouter/docs/targeting-attributes.md` in the same patch that adds them. Each new entry should call out platform restrictions (e.g. Windows-only), required vs. optional params, and a brief description.
- New `SET_PREF` and impression-action prefs must be added to an explicit allowlist; never widen the allowlist to "any pref" — keep the surface narrow and justified in a comment.
- When adding a `PanelTestProvider` test message, bump the expected count in `test_PanelTestProvider.js`. Prefer adding a discoverable test message over editing `FeatureCalloutMessages.sys.mjs` for one-off demos.

### Targeting
- Targeting attribute identifiers are lowercase camelCase (`isPrivateWindow`, `currentProfileId`, `hasSelectableProfiles`); JEXL is case-sensitive, so a typo in a targeting expression silently evaluates to undefined/falsy and the message appears to "work" but never fires its intended branch.
- Guard targeting getters that depend on services which may not be initialized early in startup (e.g. `SelectableProfileService`, `ExperimentAPI`). Return a safe default and check both `isEnabled` and `initialized` rather than assuming the service is ready.
- Prefer a thin wrapper around an existing service getter to inlining service calls in `ASRouterTargeting`; cache only when the underlying call is measurably expensive.
- Targeting attributes that gate visibility on the current page (third-party detection, URL-pattern triggers) must treat unknown / `about:blank` / error pages conservatively — assume third-party unless proven otherwise.

### State, storage, and multi-profile
- Reads/writes to the shared profiles SQLite DB must call `ProfilesDatastoreService.getConnection()` each time (it returns the cached singleton and may return `null` during shutdown); never cache the connection in a module-level variable. Wrap each query in try/catch and bail out gracefully on `null`.
- Multi-profile ("shared") code paths must be gated on both `canCreateSelectableProfiles` and the relevant scope on the message; do not write to shared tables for messages that have no profile scope set.
- Be deliberate about when to delete vs. retain impression rows in the shared DB: deleting on unenrollment / message disappearance breaks `profileScope: "single"` semantics across profiles. Default to retaining; only clean up via an explicit age-based policy (the module currently uses a 6-month grace period).
- Prefer `conn.executeTransaction` over hand-rolled `executeBeforeShutdown` + manual commit for multi-statement writes; use `executeCached` for hot-path queries.

### Special message actions & prefs
- When an SMA writes to a pref, route it through the existing allowlist mechanism; document why the pref is safe (created-on-the-fly, scoped to messaging-system namespace, etc.) in a comment next to the allowlist entry.
- For prefs that need to propagate across selectable profiles, add them to `SelectableProfileService.permanentSharedPrefs` *and* migrate old pref names explicitly when renaming — leaving stale prefs in the shared DB causes new profiles to inherit pre-migration state. File a follow-up to delete the old pref entries once the migration has shipped for a few releases.
- Avoid `console.log` in production code paths in `ASRouter*.sys.mjs`; use `lazy.ASRouterPreferences.console.debug` so logging respects the `debugLogLevel` pref.

### Localization (Fluent)
- New Fluent IDs go in lowercase-with-hyphens. When changing the meaning of a string, change the ID — never edit a translated string in place. When renaming, use the l10n migration tooling so existing translations carry over.
- Patches that touch `.ftl` files are auto-flagged for fluent-reviewers; expect a separate review. Add a `# Variables:` comment block above any message that takes variables and a `# Context:` comment for strings whose surface or audience isn't obvious from the ID.
- Treat localization output as opaque: never concatenate, split, or post-process a translated string in JS. Pass arguments via `data-l10n-args` / `setAttributes`, and use `<a data-l10n-name="...">` for embedded links rather than string-splitting on tags.
- Use `-brand-product-name` when the string refers to Firefox generically across channels; use `-brand-short-name` only when the running channel name (Nightly, Developer Edition) is intentional. For preview-only/non-shipping strings, keep them out of the shipped FTL files.

### Telemetry & data classification
- Any patch that adds, removes, or changes data collection requires a `#data-classification-*` tag and a comment explaining the choice. Mechanical no-op changes get `#data-classification-unnecessary`.
- Prefer Glean over Legacy Telemetry for new instrumentation; if both are required, instrument Glean primarily and document why Legacy is also needed. Match Glean `data_sensitivity` to the chosen classification.
- When events differentiate user choices (e.g. a "Select" button per tile), include the choice identifier in the event context so downstream analysis can disambiguate; do not rely on screen-level events alone.
- File new data collection events in `metrics.yaml` with descriptions; reviewers will block on missing or vague metric documentation.

### CSS, RTL, and HCM
- Use logical properties (`margin-inline-start`, `padding-block`, `inset-inline-end`, `border-{start/end}-{start/end}-radius`) by default. Reach for `left`/`right` only inside `:dir(rtl)` / `:dir(ltr)` blocks or when forcing LTR for code/URLs/paths.
- For LTR-forced contexts (URLs, paths, code), set `direction: ltr` and `text-align: match-parent`, and handle padding asymmetry via `:dir()` rules on a parent — not via logical properties on the forced-LTR element.
- Mirror icons that imply direction or sequence; do not mirror text, numbers, code, product logos, checkmarks, or size dimensions. Test under `intl.l10n.pseudo = bidi`.
- In High Contrast Mode, `box-shadow` and `background-image` are stripped. Use `border` or `outline` for focus rings, and `<img>` / `list-style-image` for meaningful images. Use the existing `--fc-*` and design-system theme variables rather than literal colors so HCM and theme fallbacks work automatically.
- Prefer expanded CSS syntax over shorthands; omit units on `0`; put each selector and each list-assigned attribute on its own line. Avoid magic pixel values and `!important` — when `!important` is unavoidable, add a comment explaining why.

### Accessibility
- New buttons and interactive elements need meaningful accessible names (aria-label, fluent label, or visible text). Spot-check with screen reader (NVDA / VoiceOver / Narrator) when adding new surfaces; common regressions are unannounced dialogs and focus-trap escapes.
- Do not steal focus from editable elements (urlbar, search) on callout/spotlight render. Make autofocus opt-in and configurable. When tab order matters across regions (tiles → action buttons), allow legitimate Tab navigation to escape the region without being snapped back by focus-restoration logic.
- Provide non-animated fallbacks (or respect `prefers-reduced-motion`) for any animated splash, progress, or loading content.

### JavaScript & code style
- New modules use `.sys.mjs` with `ChromeUtils.defineESModuleGetters` for lazy imports; never import `SelectableProfileService` or other `/browser`-only services unconditionally from `/toolkit`.
- Wrap async I/O and SMA dispatch in try/catch at the top level of message handlers; a single failing `_recordReachEvent` or storage write must not block the message from being shown.
- Prefer JSDoc on new public methods, especially on `InfoBar`, `Spotlight`, `FeatureCallout`, `ToastNotification`, and `ASRouterStorage` — these are the surfaces other teams call into.
- Use `console.error` (not `console.log` or naked `Cu.reportError`) for caught exceptions; include enough context (message id, selector, query name) for an engineer triaging a Sentry/Glean error to locate the call site.

### Testing
- For new message templates and SMAs, add at minimum: a unit test exercising the handler, and a browser test exercising one happy-path render. Tests run under `--verify`; flush observers, close dialogs, and avoid setting prefs without `SpecialPowers.pushPrefEnv` to keep the suite verify-clean.
- Stub services that are unavailable in xpcshell (`SelectableProfileService`, `BackupService`) using the patterns already in `browser/components/profiles/tests/unit/head.js`; link to that head rather than copying it.
- When asserting on telemetry, prefer asserting on the `data-l10n-id` / `setAttributes` payload over asserting on the resolved string — l10n changes shouldn't break tests.
- Use `testing-exception-other` (with a one-line justification) for patches that are strictly content/string/asset changes; everything else needs `testing-approved` or a more specific exception. Filing a follow-up bug is acceptable but the reviewer expects the tag and the bug link.

## Active Campaigns (transient)

- **Multi-profile messaging mitigation.** Until shared impression handling fully replaces it, OMC restricts certain message classes to a single "messaging profile" per group via `messaging-system.profile.messagingProfileId` and `ASRouter.shouldShowMessagesToProfile`. Context: likely to fade once shared impressions are reliable across all message templates and the mitigation pref can be removed (currently default-off as of 143; old prefs will eventually be deleted from `permanentSharedPrefs`).
- **Terms-of-Use pref migration.** New `termsofuse.*` prefs are being introduced alongside legacy `datareporting.policy.*` prefs, with both shared across profiles during the transition. Context: likely to fade once all releases that read the old prefs are out of support and the legacy entries can be deleted from the shared DB.
- **Onboarding splash gating on Nimbus readiness.** A loading-spotlight screen blocks `about:welcome` until Nimbus has fetched experiments on platforms where it loads late (macOS, MSIX, Linux). Context: likely to fade once Nimbus initializes early enough on first run across all platforms that the splash is redundant.

## Common Pitfalls

- Targeting attributes accidentally written in camelCase but referenced with a different case in JEXL (e.g. `nimbusExperimentsLoaded` vs `experimentsLoaded`) — silently evaluates undefined, so the gating never fires.
- Adding a new field to a message schema but forgetting `make-schemas.py`, causing experimenter validation tests to fail with no obvious connection to the patch.
- Hardcoding `console.log` debug statements in shipped code paths instead of `lazy.ASRouterPreferences.console.debug`.
- Caching the `ProfilesDatastoreService` connection in a module variable; this keeps the connection alive past the shutdown blocker and causes shutdown hangs.
- Forgetting to update `permanentSharedPrefs` when introducing prefs that need to propagate across selectable profiles in a group, leading to new profiles failing to inherit ToU/onboarding state.
- Cleaning up shared multi-profile impression rows when a message's recipe disappears (e.g. on unenrollment), which breaks `profileScope: "single"` and lets the message re-show in other profiles.
- Reading from `_documentURI` or other pseudo-private browser internals instead of `gBrowser.currentURI` / public getters; reviewers will ask for the public API.
- Forgetting `text-align: match-parent` when forcing `direction: ltr` on URL / path / username inputs, causing text to left-align inside an RTL dialog.
- Setting `box-shadow` as the focus ring on a callout or button — invisible in HCM; use `outline` or `border`.
- Bundling missing/outdated: changes to `aboutwelcome` JSX require running the `aboutwelcome` bundle script and committing the bundle; reviewers will see a stale bundle in CI.
- Tests that pass in isolation but fail in `--verify` because of leaked prefs, unflushed observers, or stale `_universalInfobars` / `_activeInfobar` state.

## File-Glob Guidance

- `browser/components/asrouter/modules/*.sys.mjs` — Watch for: targeting getter behavior under uninitialized services; SMA allowlist additions; storage code paths that need shared-DB awareness and per-call `getConnection()`.
- `browser/components/asrouter/modules/OnboardingMessageProvider.sys.mjs`, `FeatureCalloutMessages.sys.mjs`, `PanelTestProvider.sys.mjs` — Watch for: targeting strings using existing constants (e.g. `localeLanguageCode`, `!isFxASignedIn`) over hand-rolled regexes; impression-action prefs cleared on dismissal so the message doesn't re-show every startup; PanelTestProvider count bump in xpcshell tests.
- `browser/components/asrouter/content/components/**` (Lit components) — Watch for: optional / nullable content fields rendered with `nothing` rather than empty `<img src="">`; private `#handleX` methods used consistently for primary/secondary/dismiss; story coverage for new variants.
- `browser/components/asrouter/content-src/schemas/*.schema.json`, `templates/**/*.schema.json` — Watch for: regenerated top-level `MessagingExperiment.schema.json`; new fields documented with `description`; platform-specific or template-specific fields scoped properly.
- `browser/components/asrouter/content-src/styles/**.scss` — Watch for: logical properties; theme variables not literal colors; HCM-compatible focus rings; minimal use of `!important`.
- `browser/components/aboutwelcome/content-src/**` — Watch for: bundle regeneration; configurable styles validated against an allowlist; multi-screen flows that respect screen-targeting and don't break under narrow widths or RTL.
- `browser/components/aboutwelcome/content/onboarding.ftl`, `browser/locales/en-US/browser/newtab/asrouter.ftl` — Watch for: comments for context and variables; use of `-brand-product-name` vs `-brand-short-name`; preview-only strings kept out of shipped files; migration entries when renaming.
- `toolkit/components/messaging-system/lib/SpecialMessageActions.sys.mjs` and its schema — Watch for: SMA-level platform guards (Windows-only actions); allowlist scoping; docs page regenerated.
- `**/tests/browser/**`, `**/tests/unit/**`, `**/tests/xpcshell/**` — Watch for: `--verify`-clean cleanup; stubbing of profile/backup services rather than copying head.js; assertions on `data-l10n-id` not resolved strings.
- `browser/components/asrouter/docs/**.md`, `toolkit/components/messaging-system/schemas/**/index.md` — Watch for: every new targeting attribute, SMA, or template field documented in the same patch.

## Review Checklist

- [ ] Schema changes regenerated (`make-schemas.py`) and any new field is documented in the schema's `description`.
- [ ] New targeting attributes / SMAs / template fields documented in the corresponding `docs/*.md` file.
- [ ] Data-collection changes have a `#data-classification-*` tag with a one-line justification; Glean `data_sensitivity` matches.
- [ ] Fluent IDs are new (not edited in place) for any meaning change; brand term (`-brand-product-name` vs `-brand-short-name`) is appropriate; variables and non-obvious context have comments.
- [ ] CSS uses logical properties; focus rings use `outline`/`border` so they survive HCM; no literal colors where a design-system / `--fc-*` variable exists.
- [ ] No hardcoded `console.log` in shipped paths; caught exceptions use `console.error` with enough identifying context.
- [ ] Storage code calls `ProfilesDatastoreService.getConnection()` per use, wraps in try/catch, and tolerates `null`; transactions used for multi-statement writes.
- [ ] Multi-profile code paths are gated on profiles-enabled targeting and a relevant `profileScope`; impression cleanup respects the retention policy.
- [ ] Tests cover the new template/SMA/targeting at unit and browser level; `--verify` cleanup is in place; PanelTestProvider counts updated if needed.
- [ ] Testing-policy tag is present (`testing-approved` or a specific exception with reasoning).
- [ ] For UI changes: tested at narrow widths, in RTL, in dark mode, and with HCM enabled; keyboard focus order verified.
- [ ] Bundle regenerated and committed if any JSX / Lit / SCSS source under `aboutwelcome/content-src` or `asrouter/content` changed.

### Campaign-specific checks
- [ ] If touching multi-profile impression code: behavior verified across two simultaneous profiles, and old-pref cleanup is filed or executed.
- [ ] If touching ToU/onboarding prefs: legacy `datareporting.policy.*` prefs continue to propagate via `permanentSharedPrefs` until the migration window closes.
- [ ] If touching the onboarding splash: targeting uses `experimentsLoaded` (and the gate pref), and the screen is dismissed promptly when Nimbus is already ready.

## House style references

- [CSS Guidelines](https://firefox-source-docs.mozilla.org/code-quality/coding-style/css_guidelines.html)
- [SVG Guidelines](https://firefox-source-docs.mozilla.org/code-quality/coding-style/svg_guidelines.html)
- [RTL Guidelines](https://firefox-source-docs.mozilla.org/code-quality/coding-style/rtl_guidelines.html)
- [JavaScript Coding Style](https://firefox-source-docs.mozilla.org/code-quality/coding-style/coding_style_js.html)
- [Python Coding Style](https://firefox-source-docs.mozilla.org/code-quality/coding-style/coding_style_python.html)
- [Fluent for Firefox Developers](https://firefox-source-docs.mozilla.org/l10n/fluent/tutorial.html)
