# ADR-010: Email abstraction — `IMailer` with Gmail SMTP and Provider API

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 1

## Context

The platform sends transactional email from Phase 1 onward:

- Email verification (P1).
- Password reset (P1).
- Instructor invite (P1/P2).
- Enrollment status updates — approved/rejected (P2).
- Notifications digest (Phase 6).

Constraints:

- Local dev must not send real email. Mailpit (already in compose)
  catches everything.
- Production should support either a Gmail SMTP account (for solo
  bootstrapping) or a provider HTTP API (e.g. Resend, Postmark, SES via
  HTTP) without code change.
- Templates need both HTML and a text fallback.
- Provider-agnostic discipline (`CLAUDE.md` §3) applies here too.

## Decision

Define `IMailer` in `packages/mailer` and provide two implementations
selected at boot via `MAILER_DRIVER`.

```ts
interface IMailer {
  send(args: {
    to: string;
    subject: string;
    html: string;
    text: string;
    replyTo?: string;
    attachments?: Array<{
      filename: string;
      content: Buffer | string;
      contentType?: string;
    }>;
  }): Promise<{ messageId: string }>;
}
```

Implementations:

- `GmailSmtpMailer`: nodemailer with SMTP transport. Configured via
  `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS` (Gmail App
  Password for the Gmail case).
- `ProviderApiMailer`: generic HTTP client following a Resend-style
  shape (`POST /emails` with `{ from, to, subject, html, text }` and a
  bearer auth header). Endpoint and auth header are configurable, so
  any provider with that contract slots in. Providers with different
  shapes get a small adapter file rather than a parallel interface.

Selection: `MAILER_DRIVER` ∈ `gmail-smtp | provider-api`. Validated at
boot by `packages/config`.

### Templates

- **React Email** for templates. HTML is rendered server-side; the
  text fallback is rendered alongside from the same component using
  React Email's `render`.
- Templates live in `packages/mailer/templates/`.
- Each template exports `subject(data)`, `html(data)`, `text(data)`.

### Local dev

- Mailpit container in compose catches all SMTP regardless of mode.
- Default dev config: `MAILER_DRIVER=gmail-smtp`,
  `SMTP_HOST=mailpit`, `SMTP_PORT=1025`. UI at
  `http://localhost:8025`.
- Engineers wanting to exercise the `ProviderApiMailer` path point it
  at **Mailpit's HTTP send API** (Mailpit exposes a JSON `POST /api/v1/send`
  endpoint at port 8025 in compose). This preserves the "all dev mail
  lands in Mailpit" invariant regardless of which driver is selected.

## Consequences

### Positive

- Same business code paths regardless of backend; mailer choice is a
  config change.
- Mailpit gives a complete dev inbox without sending real mail.
- React Email keeps templates type-safe and previewable.
- HTML + text rendering in lockstep avoids drift between formats.
- Gmail SMTP path lets the project bootstrap without a paid email
  provider.

### Negative / Trade-offs

- Two implementations means two integration paths to maintain.
- Gmail SMTP has well-known sending limits (~500/day for free
  accounts); this is fine for early phases but we will outgrow it,
  hence the `ProviderApiMailer` exists for prod migration.
- The generic Resend-style shape is a happy-path assumption; truly
  divergent providers (e.g. SES SDK signing) need a small adapter, not
  a parallel interface.
- Template authoring requires a small React Email preview workflow
  (run `react-email dev` locally).

### Neutral

- We do not implement bounce/complaint handling in P1; provider-side
  features cover most of it.
- Per-recipient rate limiting and queueing happen at the call-site for
  bulk sends (notification digests in Phase 6) — not in the interface.
- DKIM/SPF/DMARC are deploy-time concerns.

## Alternatives considered

- **Resend SDK directly**: would lock business logic to Resend; the
  generic `ProviderApiMailer` keeps the door open with negligible
  extra code.
- **AWS SES SDK directly**: same lock-in; SES is also possible to use
  via SMTP through the existing `GmailSmtpMailer` shape if needed.
- **No abstraction (call nodemailer everywhere)**: loses the dev
  story (Mailpit-by-default), and would force a refactor when we move
  off Gmail SMTP.
- **mjml** for templates: powerful but heavier; React Email gives the
  same mailbox-tested HTML with a TS-React DX that fits the rest of
  the stack.

## Related ADRs / docs

- `CLAUDE.md` §3, §7.
- ADR-005 (auth) — verification + reset emails go through this.
- ADR-009 (storage) — same abstraction pattern.
- `docs/development/phase_1.md` — initial mail flows.
- `docs/development/phase_6.md` — notification digests.
