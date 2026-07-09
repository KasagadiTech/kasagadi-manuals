# Kasagadi AI — System Manual

A working reference for how the Kasagadi AI fact-checking platform is built and how it
behaves. Useful to a new engineer joining the team, a product or operations person who
needs to reason about the lifecycle of a claim and who is accountable for it, or a
designer wanting to know which states each screen represents.

---

## What the platform does

Kasagadi AI is a Ghana-focused fact-checking marketplace. Members of the public submit
claims they want investigated. Admins triage those claims and assign them to accredited
fact checkers. Fact checkers research the claim, write a verdict, and submit it back. The
**admin who assigned the claim owns it** and is the only one who can approve and publish
the verdict, at which point the claim becomes visible on the public marketplace. A single
**Super Admin** oversees the whole platform: admins, fact-checker accounts, global claim
visibility, the audit trail, and platform analytics.

The platform is built so that the journey from "I saw a suspicious headline" to "here is a
verified verdict with evidence and a research trail" lives in one **auditable, accountable**
system — every consequential action is attributed to a person and recorded.

---

## The actors

A `User` record holds the credentials and shared profile (name, email, avatar). A user
becomes a role by being associated with a `Member`, `FactChecker`, or `Admin` profile
record. The **Super Admin** is an `Admin` with the `super_admin` flag set. Role helpers
(`user.member?`, `user.fact_checker?`, `user.admin?`, `user.super_admin?`) drive routing
and authorization.

| Role           | How they get one                                   | What they can do |
|----------------|----------------------------------------------------|------------------|
| **Member**     | Self-signs up at `/registration/new`               | Submit claims, save drafts, edit/destroy own drafts, browse the marketplace, view their own claims in every state |
| **Fact checker** | Invited by an admin (`/admins/invitations`)      | View assignments, accept/reject them, upload evidence & research, recommend a verdict, mark work complete |
| **Admin**      | Invited by the Super Admin (no public sign-up)     | Triage claims, assign/reassign fact checkers, **own** claims they assign, approve/unpublish/republish **their own** claims, transfer ownership, invite fact checkers, manage news posts |
| **Super Admin** | Promoted from an existing admin (exactly one)     | Everything an admin can do, **plus** invite/suspend/activate admins, reset admin passwords, suspend/reactivate fact checkers, see every claim (incl. drafts), view the audit log and per-admin performance |

Public visitors (no account) can browse the marketplace and read published claims.

### Capability matrix

| Action | Member | Fact checker | Admin | Super Admin |
|---|:---:|:---:|:---:|:---:|
| Submit a claim | ✅ | — | — | — |
| Assign / reassign a fact checker | — | — | ✅ | ✅ |
| Submit a verdict | — | ✅ | — | — |
| Publish / unpublish a claim | — | — | owner only | owner only |
| Transfer claim ownership | — | — | owner only | owner only |
| Invite a fact checker | — | — | ✅ | ✅ |
| Invite an admin | — | — | — | ✅ |
| Suspend / reactivate an account | — | — | — | ✅ |
| Reset an account's password | — | — | — | ✅ |
| See drafts / every claim | — | — | — | ✅ |
| View the audit log | — | — | — | ✅ |

> **Key accountability rule:** a claim has exactly **one owner admin** at a time, and only
> that admin can publish, unpublish, republish, or transfer it. The Super Admin has full
> *visibility* but does **not** override ownership — their oversight role grants no publish
> power over claims they don't own.

---

## The lifecycle of a claim

A claim moves through five states, governed by the AASM state machine on the `Claim`
model. The transitions and who triggers them:

```
   ┌────────┐   submit    ┌─────────────────────┐   assign_checker     ┌─────────────────┐
   │ draft  │ ──────────► │ pending_assignment  │ ───────────────────► │   in_progress   │
   └────────┘  (member)   └─────────────────────┘   (admin → owner)    └─────────────────┘
                                ▲                                          │   │
                                │ reject_assignment (fact checker)         │   │ approve
                                └──────────────────────────────────────────┘   ▼  (owner admin)
                                                                        ┌─────────────┐
                                                          republish     │  published  │
                                                       ┌───────────────►└─────────────┘
                                                       │                       │ unpublish
                                              ┌─────────────────┐              ▼  (owner admin)
                                              │  unpublished    │◄─────────────┘
                                              └─────────────────┘
```

State-by-state:

- **draft** — the member is composing the claim. Validations on title/content/source/
  topic/region are skipped while drafting (`Claim#skip_presence_validation?`). Drafts are
  private to the member; only the Super Admin can also see them.
- **pending_assignment** — submitted to the platform, waiting for an admin to assign a fact
  checker. Has **no owner yet**.
- **in_progress** — a `FactChecker` has been assigned. The assigning admin becomes the
  **owner** (`claim.owner_admin`). The `Assignment` row tracks the due date, admin notes,
  and its own status (`pending → in_progress → completed / rejected`).
- **published** — the owner admin approved the verification; the claim is live on the public
  marketplace and `verification.published_at` is stamped with the approval time.
- **unpublished** — the owner admin hid a previously published claim from the public and
  members. `republish` restores it (and keeps the original publish date).

The state machine lives in [`app/models/claim.rb`](https://github.com/KasagadiTech/kasagandi_platform/blob/main/app/models/claim.rb).

> **Note:** earlier versions used an `is_archived` boolean. That no longer exists — hiding a
> claim is the `unpublished` **state** reached via the `unpublish` event, and restoring it is
> `republish`.

### Derived helpers

The Claim model exposes "computed states" that combine the AASM state with the Assignment /
Verification rows. These power the UI badges:

- `awaiting_review?` — `in_progress` with a `pending` assignment (checker hasn't started)
- `under_review?` — `in_progress` with an `in_progress` assignment (actively worked)
- `awaiting_approval?` — `in_progress` with a verification submitted (waiting for the owner
  admin to approve)
- `owned_by?(admin)` / `publishable_by?(admin)` — true only for the claim's owner admin

---

## Claim ownership & accountability

Ownership is the backbone of the accountability model.

- **When ownership attaches:** the moment an admin assigns a fact checker, that admin
  becomes `owner_admin`. Reassigning to a different checker does **not** change ownership.
- **Who can publish:** only the owner admin can run `approve` (publish), `unpublish`, or
  `republish`. The button is hidden for other admins, and the controller rejects the action
  server-side — so it can't be forced via a crafted request. Non-owners see an "Owned by …"
  context instead.
- **Transfer:** the owner admin can hand a claim to another **active** admin. The transfer
  is recorded in the audit log (`from_admin_id` → `to_admin_id`). Transferring to a suspended
  admin is blocked (they could never publish, which would orphan the claim).
- **Awaiting approval:** the owner's dashboard surfaces an "Awaiting your approval" list —
  claims they own whose fact checker has submitted a verdict. It previews the most recent 8
  with a "View all" link to the owner-scoped claims view.

The Super Admin's claims view shows **every** claim (including drafts) read-only, with owner,
fact checker, and the assigned / completed / published dates.

---

## The supporting records

- **Evidence** — many per claim. Description plus optional file attachments (images, videos,
  PDFs via Active Storage). Members add evidence when submitting.
- **Assignment** — one per claim. Tracks the fact checker, the assigning admin
  (`assigned_by`), due date, notes, and status. Fires `ClaimMailer.assigned` on creation and
  on a fact-checker swap (not on same-checker re-saves).
- **Verification** — one per claim. The fact checker's verdict (`true`, `false`,
  `misleading`, `partly_true`, `unverifiable`), summary, research write-up, reference link,
  and `published_at` (stamped when the owner admin approves — *not* at verdict submission).
- **AuditLog** — an immutable, append-only event record. One row per consequential action:
  who did it (`actor`), what (`action`), to which record (polymorphic `auditable`), and
  metadata. See §8.

---

## Walkthroughs by role

### 6.1 Member

1. Signs up at `/registration/new` → a `User` + `Member` are created, signed in, landed on
   `/members/dashboard`.
2. `/members/claims/new` — fills title, content, source, **one or more optional source
   links**, topic, region, and zero or more evidence rows.
3. **Submit** runs the `submit` event → `pending_assignment` (validated). **Save draft**
   keeps it as `draft` with relaxed validation.
4. Drafts can be edited/destroyed; submitted claims cannot.
5. On publication the member receives `ClaimMailer.published`.
6. A **suspended** member cannot sign in at all (see §7).

### 6.2 Fact checker

1. An admin invites them by email at `/admins/invitations/new` (24-hour, single-use token).
2. They accept at `/invitations/:token/accept`, which creates the `User` + `FactChecker`,
   flips the invitation to `accepted`, and signs them in.
3. `/fact_checkers/dashboard` shows pending / in-progress / completed counts, a verdict
   breakdown, and **recent verdicts ordered by submission time**.
4. From `/fact_checkers/assignments` they filter/search the queue and open an assignment.
5. Three actions:
   - **Start review** — assignment `pending → in_progress` (claim stays `in_progress`).
   - **Reject assignment** — claim back to `pending_assignment`, assignment marked
     `rejected`; writes a `rejected` audit entry. Wrapped in a transaction.
   - **Submit verdict** — creates the `Verification`, marks the assignment `completed`, and
     writes a `completed` audit entry. The claim is now *awaiting approval* — the owner admin
     still has to publish.
6. Fact checkers never publish, assign, transfer, or change ownership. A **suspended** fact
   checker can't sign in, and can't be assigned new claims.

### 6.3 Admin

1. Signs in at `/session/new` (no public admin sign-up — admins are invited by the Super
   Admin).
2. Lands on `/admins/dashboard`, which includes an **"Awaiting your approval"** section for
   claims they own.
3. From `/admins/claims` they filter by status/topic/region, search, and open a claim:
   - **Assign / reassign** — pick a fact checker (suspended ones are not offered), set a due
     date and notes. First assignment makes the admin the **owner** and writes an `assigned`
     audit entry.
   - **Approve & publish** (owner only) — runs `approve`, stamps `published_at`, writes a
     `published` audit entry, and emails the member. Only shown once a verdict exists.
   - **Unpublish / Republish** (owner only) — hides/restores the claim; audited.
   - **Transfer ownership** (owner only) — hand the claim to another active admin; audited.
4. From `/admins/invitations` an admin can invite **fact checkers** (the role is locked to
   fact-checker for ordinary admins), view/resend invitations.
5. Admins can post and manage news.

### 6.4 Super Admin

The Super Admin is an admin with the `super_admin` flag. There is **exactly one**, enforced
by a database partial-unique index. They keep full admin abilities and gain an
**Administration** area in the sidebar:

1. **Admins** (`/super_admins/admins`) — list every admin; open an admin's page to see their
   performance (claims **owned / published / awaiting approval / in progress**) and the claims
   they own. From there: **suspend / reactivate** the account, **reset password**, or (from
   the list header) **invite a new admin**. They can't suspend themselves or the super-admin
   account.
2. **Fact checkers** — the existing `/admins/fact_checkers` pages gain **suspend / reactivate**
   controls that only the Super Admin sees.
3. **Members** (`/admins/members`) — list every member; open a member's page to see their
   submitted claims and profile, and (Super Admin only) **suspend / reactivate** the account or
   **reset password** (a reset emails the standard 15-minute link; refused while the member is
   suspended).
4. **Claims** — the standard `/admins/claims` list, but the Super Admin additionally sees a
   **Drafts** tab and drafts in "All" (full platform visibility). Still read-only oversight —
   they can only publish/transfer claims they personally own.
5. **Audit Log** (`/super_admins/audit_logs`) — the full activity feed, filterable by actor
   and date range (see §8).

Provisioning the Super Admin is an operations task — see §11.

### 6.5 Public visitor

- `/marketplace` — paginated, filterable list of **published** claims with verdicts, topics,
  regions, search.
- `/marketplace/:id` — full claim detail: verdict, body, evidence, **source links**, research,
  fact-checker attribution, and **published date + publishing admin**. Related claims by topic at the
  bottom.

---

## AI chat assistant

Kasagadi runs a floating **AI chat assistant** (the widget in the corner) across the
public site and the member / fact-checker / admin portals. Anyone can ask the
fact-checking AI a question and get a streamed, sourced answer.

- **Who can use it:** everyone — signed-in users *and* anonymous visitors. A signed-in
  user gets a `Conversation` owned by their account; an anonymous visitor gets one keyed
  by a per-browser token in the session cookie (`session[:chatbot_token]`). A conversation
  must have a user or a token, never neither.
- **Records:** a `Conversation` has many `Message`s; each has a `role` (`user` /
  `assistant`), `content`, and (for assistant turns) a `sources` JSON list. The last ~20
  turns are sent upstream as history.
- **Backend:** `Chatbot::MessagesController` (under `/chatbot`), open to unauthenticated
  access.

Two endpoints — this is the closest thing the app has to a programmatic **API**:

| Endpoint | Method | Returns | Use |
|---|---|---|---|
| `/chatbot/messages` | POST | JSON `{ response, html }` | One-shot, non-streaming reply (scripts / tests). |
| `/chatbot/messages/stream` | POST | Turbo Stream | Live, token-by-token reply in the widget (default path). |

**How the streaming works** (the interesting part):

1. `#stream` appends an empty assistant bubble to the widget via Turbo Stream and enqueues
   `ChatReplyJob`. The bubble's DOM id doubles as its Action Cable stream name, so they're
   created together and can't desync.
2. `ChatReplyJob` calls `FactCheckAi.client.chat_stream` (the external fact-check service)
   and, as tokens arrive, **coalesces** them into ~7 fps updates
   (`CHUNK_FLUSH_INTERVAL = 0.15s`) broadcast over Action Cable
   (`Turbo::StreamsChannel.broadcast_update_to`).
3. On `done` it persists the user + assistant messages atomically and swaps the bubble for
   the final styled card (rendered Markdown + sources). On error it replaces the bubble with
   a danger notice carrying a **retry** button — the original message is embedded, so a click
   re-sends with no client-side state.

**External dependency:** answers come from the **FactCheckAI** service — a thin Ruby client
(`lib/fact_check_ai/`) over a configurable REST API (`FACTCHECK_AI_API_URL`). The same client
powers claim analysis (`analyse` / `analyse_stream`). If the upstream is down, the user sees
"Sorry, the fact-checking service is temporarily unavailable." with a retry.

**Realtime + suspension:** `ApplicationCable::Connection` allows anonymous connections (so
visitors can chat) and refuses suspended users — see "Account suspension".

## Account suspension

Any account can be set `active` or `suspended` (`users.status`). Suspension is a hard block
on access, enforced in depth:

- **Login** — `SessionsController#create` refuses a suspended user.
- **Existing sessions** — `User#suspend!` destroys all of the user's sessions in a
  transaction, so they're signed out everywhere immediately.
- **Session resume** — `Authentication#find_session_by_cookie` won't resume a session whose
  user isn't `active_for_auth?`.
- **Password reset** — self-service reset is refused for suspended users; the Super Admin's
  "reset password" is also refused until the account is reactivated.
- **Real-time** — `ApplicationCable::Connection` refuses a suspended user, so they get no
  live Turbo Stream updates.
- **Work routing** — suspended fact checkers can't be assigned claims; suspended admins can't
  receive transferred ownership.

Who can suspend whom: the **Super Admin** suspends/reactivates admins, fact checkers, and members. The
controls live on each account's page (and the Admins list), never for the Super Admin's own
account.

---

## The audit log

Every consequential action writes an immutable `AuditLog` row: `actor` (the User who did
it), `action`, a polymorphic `auditable` (usually a Claim or User), `metadata` (JSONB), and
`created_at`. The model blocks updates and deletes, so the trail can't be tampered with.

Actions recorded today:

| Action | Written when |
|---|---|
| `assigned` | An admin assigns a fact checker (becomes owner) |
| `completed` | A fact checker submits a verdict |
| `rejected` | A fact checker rejects an assignment |
| `published` | The owner admin approves/publishes |
| `unpublished` / `republished` | The owner admin hides / restores a claim |
| `transferred` | Ownership moves between admins (`from`/`to` in metadata) |
| `suspended` / `activated` | An account is suspended / reactivated |
| `invited` | An invitation is sent (role in metadata) |
| `password_reset` | The Super Admin triggers a password reset |

The Super Admin reads the feed at `/super_admins/audit_logs`, newest first, filterable by
**actor** and **date range**. It reads like "Admin A assigned Claim #100", "Fact Checker B
completed Claim #100", "Admin A transferred Claim #101 to Admin C".

---

## Authentication & authorization

- **Sessions** — cookie-based; a signed cookie holds a `Session.id` stored in the DB, so
  sessions can be terminated server-side. Every request is gated by
  `Authentication#require_authentication` unless a controller opts out with
  `allow_unauthenticated_access`. Suspended users are rejected at login and on resume (§7).
- **Role gating** — role-namespaced base controllers (`Members::BaseController`,
  `FactCheckers::BaseController`, `Admins::BaseController`) enforce the role via
  `before_action`. `SuperAdmins::BaseController` inherits the admin gate and adds
  `ensure_super_admin` (shared `SuperAdminGuard` concern). Hitting a forbidden URL redirects
  away.
- **Ownership gating** — publish/unpublish/republish/transfer check `claim.owned_by?` /
  `publishable_by?` in both the view (hide controls) and the controller (reject the request).
- **Single super admin** — a partial unique index on `admins.super_admin` guarantees at most
  one.
- **Password reset** — `User.generates_token_for(:password_reset)` mints a 15-minute signed
  token bound to the password salt, so changing the password invalidates outstanding tokens.
  The request always returns the same generic notice to avoid leaking account existence.

---

## Dashboards & analytics

The admin dashboard renders lazy-loaded chart frames built by
`Admins::DashboardChartBuilder`. What each one **counts**:

- **Claim pipeline** — all non-draft claims by stage: Pending, In progress, **Awaiting**
  (in_progress with a verdict), Published, Unpublished. (Awaiting is split out of In progress
  so the two don't double-count.)
- **Verification requests** — `Assignment` lifecycle (pending / in progress / completed /
  rejected) and average completion time.
- **Published claims by category / by verdict** — **published claims only**, by topic or by
  verdict, so the two views reconcile to the same total.
- **Claims submitted** — claims by `created_at` (submission date).
- **Fact-checks published** — published claims by `verification.published_at` (the real
  approval/publish date). Long ranges (90 days / all time) are bucketed by week or month so
  sparse data stays readable; the current period is highlighted.
- **Top fact-checkers** — by **published** verdicts. **Top members** — by **published** claims.

The Super Admin also gets **per-admin performance** (owned / published / awaiting / in
progress) on each admin's page, and **system activity** via the audit log.

---

## Operating the platform

Day-two operations, mostly for the Super Admin / ops.

### Promote the (single) Super Admin

The person must already be an **active admin**. Run the rake task in production (the app
deploys with **Kamal**):

```bash
kamal app exec 'bin/rails "rbac:promote_super_admin[someone@kasagadi.com]"'
```

It clears the flag on everyone else and sets it on the target (the DB index guarantees one).
To bootstrap the very first admin when none exists, open a console
(`kamal app exec -i 'bin/rails console'`) and create the `Admin` record before promoting.

> **Never run `db:seed` in production** — the seed wipes claims (`Claim.destroy_all`). Use the
> rake task or console to set the super admin.

### Suspend / reactivate an account

As the Super Admin, open the admin's page (`/super_admins/admins/:id`), a fact checker's
page (`/admins/fact_checkers/:id`), or a member's page (`/admins/members/:id`) and use
**Suspend account** / **Reactivate** (admin and member pages also offer **Reset password**).
Suspension signs the user out everywhere immediately and is recorded in the audit log.

### Transfer a claim

As the **owner** admin, open the claim and use **Transfer ownership** to hand it to another
active admin. The receiving admin then sees it in their "Awaiting approval" / owned views.

---

## Email & background jobs

- **Mailers**: `ClaimMailer` (assigned, published), `InvitationMailer`, `PasswordsMailer`.
- All dispatches use `deliver_later`, queued through Solid Queue (database-backed Active Job;
  runs in-process when `SOLID_QUEUE_IN_PUMA=1`).
- Triggers:
  - `ClaimMailer.assigned` — `Assignment` `after_create_commit` and on fact-checker change.
  - `ClaimMailer.published` — Claim `approve` event `after_commit`.
  - `InvitationMailer` — explicit dispatch in the invitations controller.
  - `PasswordsMailer.reset` — `PasswordsController#create` and the Super Admin reset action
    (both skip suspended users).

---

## Storage

Active Storage is configured for two services:

- **local** — disk-based, development.
- **cloudflare** — Cloudflare R2 (S3-compatible) via `aws-sdk-s3`. R2 quirks handled in
  `config/storage.yml` (`force_path_style: true`, checksum options `when_required`).

Cover images are auto-promoted from the first attached image among the evidence files
(`Claim#promote_cover_image_from_evidence`). The shared attachment partials render images and
`mp4/webm/ogg` videos inline (with a download overlay) and other types as file cards.

---

## Frontend

- **Tailwind CSS 4** with a Ghana-inspired palette (charcoal primary, gold accent, green
  success, red danger).
- **Stimulus** for interactivity (accordions, file-upload previews, verdict picker,
  multi-step claim form, mobile menu, auto-submit filters).
- **Material Symbols** outline icons; **Plus Jakarta Sans** display font (no serifs).

Layout conventions:

- List pages and **detail/show pages** both use `max-w-7xl`; show pages use a full-bleed
  `border-b` hero with the breadcrumb inside (matching the fact-checker / member / claim
  show pages).
- Portal/detail section headers: left-aligned title, right-aligned meta; homepage sections
  centered.
- "Back" links: omitted on editorial pages (marketplace), included on portal detail pages.
- Destructive actions (suspend) are solid red; restorative actions (reactivate) solid green.

---

## Codebase tour

```
app/
├── controllers/
│   ├── admins/                ← admin portal (claims, invitations, members,
│   │                            fact_checkers, posts, dashboard)
│   ├── super_admins/          ← super-admin area (admins, audit_logs)
│   ├── fact_checkers/         ← fact-checker portal (dashboard, assignments)
│   ├── members/               ← member portal (dashboard, claims)
│   ├── concerns/
│   │   ├── authentication.rb  ← session auth + suspension guard
│   │   └── super_admin_guard.rb ← ensure_super_admin
│   ├── marketplace_controller.rb / invitation_acceptances_controller.rb
│   ├── registrations_controller.rb / sessions_controller.rb
│   └── passwords_controller.rb / pages_controller.rb
├── channels/application_cable/ ← suspension-aware Action Cable connection
├── models/
│   ├── user.rb (status, suspend!/activate!, super_admin?)
│   ├── admin.rb (super_admin, owned_claims) / member.rb / fact_checker.rb (active scope)
│   ├── claim.rb (AASM, owner_admin, publishable_by?, transfer_to!, stamp_published_at)
│   ├── assignment.rb / verification.rb / evidence.rb / invitation.rb
│   ├── audit_log.rb (immutable event log)
│   └── session.rb / current.rb
├── services/admins/dashboard_chart_builder.rb ← analytics
├── helpers/ (claim_actions, audit_logs, navigation, …)
├── mailers/ / views/ / javascript/controllers/ / assets/tailwind/
config/
├── routes.rb / storage.yml / deploy.yml (Kamal) / importmap.rb
lib/tasks/rbac.rake             ← rbac:promote_super_admin
test/                            ← controllers, models, services, helpers, fixtures
```

---

## Running locally

Stack: Ruby 3.3.0, Rails 8.0, PostgreSQL, Tailwind v4, Stimulus / Importmap, Solid Queue.

```bash
bin/setup            # bundle install, db:prepare
bin/dev              # boots web + tailwind watch + Solid Queue
```

Sign in with the seeded accounts (see `db/seeds.rb` — `admin@kasagandi.com` is seeded as the
super admin in dev) or register a fresh member at `/registration/new`.

---

## Testing

```bash
bin/rails test                 # full suite (~470 tests)
bin/rubocop                    # style
bin/brakeman                   # security scan
bin/importmap audit            # JS dep audit
```

CI runs all four jobs on every push to `main` and on pull requests.

Conventions:

- Controller tests under `test/controllers/<namespace>/`; sign-in helper in
  `test/test_helper.rb` (`sign_in(user)`).
- Fixtures cover all roles and lifecycle states (`pending_claim`, `under_review_claim`,
  `verified_claim`, `awaiting_approval_claim`) plus a second admin for ownership/transfer
  tests.

---

## Glossary

- **Claim** — a public statement someone wants verified. The unit of work.
- **Owner admin** — the single admin accountable for a claim; the only one who can publish,
  unpublish, republish, or transfer it. Set when they assign a fact checker.
- **Transfer** — moving claim ownership from one admin to another (audited).
- **Evidence** — supporting material the member supplies (description + files).
- **Assignment** — the relationship between a claim and a fact checker (due date, notes,
  status).
- **Verification** — the fact checker's verdict and research write-up.
- **Verdict** — one of True, False, Misleading, Partly True, Unverifiable.
- **Audit log** — immutable record of who did what; the platform's accountability trail.
- **Suspension** — an account state that blocks all access (login, sessions, reset, realtime).
- **Super Admin** — the single platform owner; oversees admins, fact-checker accounts, global
  claim visibility, the audit log, and analytics.
- **Marketplace** — the public catalogue of published claims.
- **Accredited** — fact checkers with `verified: true`, marked with a check icon.

---

## Public claims API

External partners can list Kasagadi's **published, fact-checked claims** from their own
apps through a read-only JSON API. The full, interactive reference lives at
**[docs.kasagadi.ai/api.html](api.html)** (Swagger UI), backed by the machine-readable
[`openapi.yaml`](openapi.yaml) contract — import that into Postman/Insomnia or a client
generator.

- **Base URL:** `https://kasagadi.ai/api/v1`
- **Auth:** a per-partner API key sent as `Authorization: Bearer kg_live_…`. Keys are
  issued by the Kasagadi team, secret, and revocable. Use them **server-to-server** —
  never embed a key in a browser or mobile app.
- **Scope:** only **published** claims are ever returned. Drafts and in-progress claims
  are structurally impossible to reach through the API.

**Endpoints:**

| Endpoint | Method | Returns |
|---|---|---|
| `/api/v1/claims` | GET | Paginated list of published claims (`{ data, meta }`). |
| `/api/v1/claims/:id` | GET | A single published claim (`{ data }`); `404` if not published. |

**List filters** (all optional): `topic`, `region`, `verdict`
(`verdict_true` / `verdict_false` / `misleading` / `partly_true` / `unverifiable`),
`q` (search over title + content), `since` (ISO-8601 — only claims published on/after it,
for incremental syncing), `page`, and `per_page` (capped at 100).

**Example:**

```bash
curl -H "Authorization: Bearer kg_live_xxxxxxxx" \
  "https://kasagadi.ai/api/v1/claims?topic=Health&per_page=25"
```

Each claim carries `title`, `content`, `source`, `topics[]`, `regions[]`, `language`,
`cover_image_url`, `published_at`, `updated_at`, a public `url`, and a `verdict` block
(verdict, summary, research, reference link, and the fact-checker's name + organization).
Submitter (member) personal data is never exposed.

**Security posture:** published-only scoping · keys stored as SHA-256 digests and revocable ·
Bearer over HTTPS · per-key rate limiting (~300 req/min → `429`) · server-to-server by default
(cross-origin browser access is allowed only from the docs Swagger UI) · an explicit field
allowlist (no PII, no internal columns).

**Issuing keys (ops):** keys are managed from the console for now —
`bin/rails "api_key:issue[Partner Name]"` prints the key **once** (store it immediately),
`bin/rails api_key:list` shows all keys and usage, and `bin/rails "api_key:revoke[ID]"`
disables one.
