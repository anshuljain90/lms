# ADR-018: Mobile and PWA strategy — PWA-first, Capacitor in Phase 8

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 0 (responsive baseline), Phase 6 (PWA polish), Phase 8 (Capacitor)

## Context

Students attend cohorts from phones at least as often as from laptops:
notifications about session starts, quick deadline checks,
discussion replies. Instructors are mostly on desktop but expect to
glance at metrics on mobile.

Constraints:

- One-developer build phase; cannot afford a parallel native track.
- App store review cycles (1–7 days) would slow weekly iteration.
- Web Push on iOS requires HTTPS and "added to home screen" — both
  available, but HTTPS only lands in Phase 7 (Caddy).
- The product needs to feel native enough that students install it
  on the home screen.

## Decision

### Phase 1 through Phase 7: PWA-first

- Next.js 15 app with:
  - `manifest.json` (icons, theme color, display: `standalone`, scope,
    start_url).
  - Service worker via `next-pwa` (or equivalent) for offline shell +
    asset caching. **No** offline data sync in P1–P7 — too complex
    for the value at this stage.
  - Installable on Android (TWA-style prompt) and on iOS (Add to Home
    Screen, with the constraints documented below).
- **Mobile-first responsive design from P0**:
  - Tailwind breakpoints used in mobile-up order.
  - Touch-friendly tap targets (`min-h-11 min-w-11`).
  - No hover-only interactions (every hover state has an equivalent
    tap/focus state).
  - Forms tested on real iOS Safari + Android Chrome before phase
    exit, not just desktop responsive view.
- **Web Push deferred to Phase 7+** when HTTPS lands. Until then,
  notifications are in-app + email only.

### Phase 8: Capacitor wrapper

- Wrap the existing Next.js PWA in Capacitor for distribution on the
  Apple App Store and Google Play.
- Minimal native plugins:
  - Push notifications (FCM on Android, APNs on iOS).
  - File picker (for any artifact uploads, e.g., enrollment receipts
    or assignment files).
- The web app and the Capacitor app share the same codebase and
  runtime; Capacitor adds a thin native shell, not a fork.

### Out of scope explicitly

- React Native or Flutter rewrite.
- Native iOS or Android app from scratch.
- Offline-first data sync (CRDTs, conflict resolution). The platform
  needs the network to be useful (LLM assessment, live cohorts).

## Consequences

### Positive

- Single codebase, single deployment, single bug fix path.
- Iteration speed unblocked by app review during the entire build
  phase.
- Users can install today on their home screen and get an
  app-shaped experience without us shipping a binary.
- When Capacitor lands in P8, the engineering cost is wrapping an
  app we've already shipped, not building one.

### Negative / Trade-offs

- **iOS PWA constraints**:
  - Web Push works only on iOS 16.4+ and only when added to home
    screen.
  - Background sync APIs are limited or unavailable.
  - 50 MB cache cap for service worker storage; sufficient for
    shell + UI assets.
- Some native APIs (camera with custom UI, biometric auth,
  contacts) are out of reach until P8. None of these are needed for
  P1–P7 features.
- App-store-only distribution audiences will only see us in P8.

### Neutral

- We avoid `localStorage` for anything sensitive; tokens live in
  HTTP-only cookies (refresh) + memory (access).
- Service worker version is tied to the build hash so cache
  invalidation is automatic on deploy.
- Lighthouse PWA score is tracked in `docs/development/phase_6.md`
  exit criteria.

## Alternatives considered

- **Capacitor from day 1**: would force every iteration through a
  native build pipeline (and eventually app review) before we even
  have a stable feature surface. Slows everything.
- **React Native WebView shell**: same wrapping benefit as Capacitor,
  but Apple has historically rejected pure-WebView wrappers; not a
  reliable path.
- **Native iOS + Android apps**: out of scope (CLAUDE.md §12). Would
  triple maintenance.
- **PWA only, never native**: would limit distribution and notification
  reach on iOS. Capacitor in P8 is the lowest-cost way to clear those
  ceilings.

## Related ADRs / docs

- `CLAUDE.md` §3 (Tech stack — Mobile row), §12 (Out of scope).
- ADR-002 (Frontend framework) — Next.js 15 PWA support.
- ADR-011 (Realtime transport) — SSE works inside webviews.
- `docs/development/phase_6.md` (Notifications & PWA polish),
  `docs/development/phase_8.md` (Capacitor wrapper, store
  distribution).
