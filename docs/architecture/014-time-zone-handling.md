# ADR-014: Time zone handling — UTC storage, user-TZ display, batch default TZ

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 1 (User.timezone), Phase 2 (Batch.default_timezone, Session scheduling)

## Context

The platform is run from India but expects students from multiple
countries and time zones. Cohort scheduling has hard requirements:

- A session scheduled for 8 PM IST must show as 8 PM IST in the batch
  view, regardless of who is looking, *and* show "(your local: 9:30 AM
  PST)" alongside.
- Deadlines are absolute moments in time, not "midnight in some TZ".
- Daylight Saving Time changes for student or instructor must not
  silently shift session times.
- ICS calendar exports must produce the correct local time on a
  student's calendar app.

## Decision

- All timestamp columns are PostgreSQL **`timestamptz`** storing UTC.
  No `timestamp without time zone` anywhere.
- `User.timezone` is an **IANA name** (e.g., `Asia/Kolkata`), default
  `Asia/Kolkata`. Captured at registration; editable in profile.
- `Batch.default_timezone` is the IANA TZ for the cohort. Used as the
  default display TZ for `Session` and deadline rendering inside that
  batch.
- **Display rule**: render in batch TZ when looking at batch/session
  pages; render in the user's TZ when looking at personal views
  (dashboard, my deadlines). Whenever batch TZ ≠ user TZ, show both.
- **Input rule**: instructors enter session times in the batch's TZ.
  The FE sends ISO-8601 with offset; the API recomputes the UTC
  instant using the IANA TZ valid on that date (handles DST
  correctly).
- FE formatting via `date-fns-tz` (or
  `Intl.DateTimeFormat` with `timeZone` for one-off cases). No moment.
- ICS export emits `VTIMEZONE` blocks for the batch's IANA TZ plus
  `DTSTART` in UTC, so calendar clients render in the recipient's
  local TZ correctly.

## Consequences

### Positive

- One canonical instant per event, stored once. No ambiguity in DB.
- DST transitions are handled by the IANA database via `date-fns-tz`;
  no manual offset bookkeeping.
- Showing both batch TZ and user TZ removes a class of "I missed it
  because the time looked weird" complaints from students.
- ICS exports work in Apple Calendar, Google Calendar, and Outlook
  without per-vendor workarounds.

### Negative / Trade-offs

- Every server response that includes a timestamp ships ISO-8601 UTC
  and the FE has to convert. This is the right default but requires
  discipline in code review (no "format date on server" shortcuts).
- IANA TZ data updates ship with Node and the FE bundle; we need to
  keep these reasonably current, especially after countries change DST
  rules. Documented in the upgrade checklist.
- Email rendering needs a recipient TZ — we store the user's TZ on
  every notification at send time so the email body is rendered server
  side in the right zone.

### Neutral

- A small `formatInZone(date, tz, fmt)` helper lives in
  `packages/shared` and is the only sanctioned way to format dates.
  Lint rule forbids `toLocaleString` and `Date.prototype.toString`
  in app code.
- Tests use `vi.setSystemTime` and explicit TZ args; no test depends
  on the runner's local TZ.

## Alternatives considered

- **Single fixed platform TZ (e.g., IST everywhere)**: simplest, but
  user-hostile for international students and a nightmare to undo
  later.
- **UTC-everywhere display**: developer-friendly, end-user-hostile.
  No instructor will say "the session is at 14:30 UTC".
- **Storing fixed offsets (`+05:30`) instead of IANA names**: breaks
  on DST changes and on countries that have changed their offset
  historically. Same trap moment.js used to fall into.
- **Letting the browser silently convert via local TZ only**: works
  for 90% of cases and silently fails for the 10% who travel or
  whose system clock is in a different zone than they think.

## Related ADRs / docs

- `CLAUDE.md` §3 (Tech stack — Time zones row), §5 (Data model — note
  on `timestamptz`).
- ADR-016 (Data model) — `User.timezone`, `Batch.default_timezone`,
  `Session.scheduled_at_utc`, `Task.deadline_utc`.
- `docs/development/phase_2.md` — first feature using batch TZ.
