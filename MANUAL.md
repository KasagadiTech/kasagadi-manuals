# Kasagadi AI System Manual

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
system: every consequential action is attributed to a person and recorded.

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
| Submit a claim | ✅ | ❌ | ❌ | ❌ |
| Assign / reassign a fact checker | ❌ | ❌ | ✅ | ✅ |
| Submit a verdict | ❌ | ✅ | ❌ | ❌ |
| Publish / unpublish a claim | ❌ | ❌ | owner only | owner only |
| Transfer claim ownership | ❌ | ❌ | owner only | owner only |
| Invite a fact checker | ❌ | ❌ | ✅ | ✅ |
| Invite an admin | ❌ | ❌ | ❌ | ✅ |
| Suspend / reactivate an account | ❌ | ❌ | ❌ | ✅ |
| Reset an account's password | ❌ | ❌ | ❌ | ✅ |
| See drafts / every claim | ❌ | ❌ | ❌ | ✅ |
| View the audit log | ❌ | ❌ | ❌ | ✅ |

> **Key accountability rule:** a claim has exactly **one owner admin** at a time, and only
> that admin can publish, unpublish, republish, or transfer it. The Super Admin has full
> *visibility* but does **not** override ownership. Their oversight role grants no publish
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

- **draft**: the member is composing the claim. Validations on title/content/source/
  topic/region are skipped while drafting (`Claim#skip_presence_validation?`). Drafts are
  private to the member; only the Super Admin can also see them.
- **pending_assignment**: submitted to the platform, waiting for an admin to assign a fact
  checker. Has **no owner yet**.
- **in_progress**: a `FactChecker` has been assigned. The assigning admin becomes the
  **owner** (`claim.owner_admin`). The `Assignment` row tracks the due date, admin notes,
  and its own status (`pending → in_progress → completed / rejected`).
- **published**: the owner admin approved the verification; the claim is live on the public
  marketplace and `verification.published_at` is stamped with the approval time.
- **unpublished**: the owner admin hid a previously published claim from the public and
  members. `republish` restores it (and keeps the original publish date).

The state machine lives in [`app/models/claim.rb`](https://github.com/KasagadiTech/kasagandi_platform/blob/main/app/models/claim.rb).

> **Note:** earlier versions used an `is_archived` boolean. That no longer exists. Hiding a
> claim is the `unpublished` **state** reached via the `unpublish` event, and restoring it is
> `republish`.

### Derived helpers

The Claim model exposes "computed states" that combine the AASM state with the Assignment /
Verification rows. These power the UI badges:

- `awaiting_review?`: `in_progress` with a `pending` assignment (checker hasn't started)
- `under_review?`: `in_progress` with an `in_progress` assignment (actively worked)
- `awaiting_approval?`: `in_progress` with a verification submitted (waiting for the owner
  admin to approve)
- `owned_by?(admin)` / `publishable_by?(admin)`: true only for the claim's owner admin

---

## Claim ownership & accountability

Ownership is the backbone of the accountability model.

- **When ownership attaches:** the moment an admin assigns a fact checker, that admin
  becomes `owner_admin`. Reassigning to a different checker does **not** change ownership.
- **Who can publish:** only the owner admin can run `approve` (publish), `unpublish`, or
  `republish`. The button is hidden for other admins, and the controller rejects the action
  server-side, so it can't be forced via a crafted request. Non-owners see an "Owned by …"
  context instead.
- **Transfer:** the owner admin can hand a claim to another **active** admin. The transfer
  is recorded in the audit log (`from_admin_id` → `to_admin_id`). Transferring to a suspended
  admin is blocked (they could never publish, which would orphan the claim).
- **Awaiting approval:** the owner's dashboard surfaces an "Awaiting your approval" list of
  claims they own whose fact checker has submitted a verdict. It previews the most recent 8
  with a "View all" link to the owner-scoped claims view.

The Super Admin's claims view shows **every** claim (including drafts) read-only, with owner,
fact checker, and the assigned / completed / published dates.

---

## The supporting records

- **Evidence**: many per claim. Description plus optional file attachments (images, videos,
  PDFs via Active Storage). Members add evidence when submitting.
- **Assignment**: one per claim. Tracks the fact checker, the assigning admin
  (`assigned_by`), due date, notes, and status. Notifies the fact checker on creation and on
  a fact-checker swap, not on same-checker re-saves, and never for an imported claim (§14).
- **Verification**: one per claim. The fact checker's verdict (`true`, `false`,
  `misleading`, `partly_true`, `unverifiable`), summary, research write-up, reference link,
  and `published_at` (stamped when the owner admin approves, *not* at verdict submission).
- **AuditLog**: an immutable, append-only event record. One row per consequential action:
  who did it (`actor`), what (`action`), to which record (polymorphic `auditable`), and
  metadata. See §9.

---

## Walkthroughs by role

### 6.1 Member

1. Signs up at `/registration/new` → a `User` + `Member` are created, signed in, landed on
   `/members/dashboard`.
2. `/members/claims/new`: fills title, content, source, **one or more optional source
   links**, topic, region, and zero or more evidence rows.
3. **Submit** runs the `submit` event → `pending_assignment` (validated). **Save draft**
   keeps it as `draft` with relaxed validation.
4. Drafts can be edited/destroyed; submitted claims cannot.
5. On publication the member is told, in the app and by email and SMS (§14).
6. A **suspended** member cannot sign in at all (see §8).

### 6.2 Fact checker

1. An admin invites them by email at `/admins/invitations/new` (24-hour, single-use token).
2. They accept at `/invitations/:token/accept`, which creates the `User` + `FactChecker`,
   flips the invitation to `accepted`, and signs them in.
3. `/fact_checkers/dashboard` shows pending / in-progress / completed counts, a verdict
   breakdown, and **recent verdicts ordered by submission time**.
4. From `/fact_checkers/assignments` they filter/search the queue and open an assignment.
5. Three actions:
   - **Start review**: assignment `pending → in_progress` (claim stays `in_progress`).
   - **Reject assignment**: claim back to `pending_assignment`, assignment marked
     `rejected`; writes a `rejected` audit entry. Wrapped in a transaction.
   - **Submit verdict**: creates the `Verification`, marks the assignment `completed`, and
     writes a `completed` audit entry. The claim is now *awaiting approval*. The owner admin
     still has to publish.
6. Fact checkers never publish, assign, transfer, or change ownership. A **suspended** fact
   checker can't sign in, and can't be assigned new claims.

### 6.3 Admin

1. Signs in at `/session/new` (no public admin sign-up: admins are invited by the Super
   Admin).
2. Lands on `/admins/dashboard`, which includes an **"Awaiting your approval"** section for
   claims they own.
3. From `/admins/claims` they filter by status/topic/region, search, and open a claim:
   - **Assign / reassign**: pick a fact checker (suspended ones are not offered), set a due
     date and notes. First assignment makes the admin the **owner** and writes an `assigned`
     audit entry.
   - **Approve & publish** (owner only): runs `approve`, stamps `published_at`, writes a
     `published` audit entry, and emails the member. Only shown once a verdict exists.
   - **Unpublish / Republish** (owner only): hides/restores the claim; audited.
   - **Transfer ownership** (owner only): hand the claim to another active admin; audited.
4. From `/admins/invitations` an admin can invite **fact checkers** (the role is locked to
   fact-checker for ordinary admins), view/resend invitations.
5. Admins can post and manage news.

### 6.4 Super Admin

The Super Admin is an admin with the `super_admin` flag. There is **exactly one**, enforced
by a database partial-unique index. They keep full admin abilities and gain an
**Administration** area in the sidebar:

1. **Admins** (`/super_admins/admins`): list every admin; open an admin's page to see their
   performance (claims **owned / published / awaiting approval / in progress**) and the claims
   they own. From there: **suspend / reactivate** the account, **reset password**, or (from
   the list header) **invite a new admin**. They can't suspend themselves or the super-admin
   account.
2. **Fact checkers**: the existing `/admins/fact_checkers` pages gain **suspend / reactivate**
   controls that only the Super Admin sees.
3. **Members** (`/admins/members`): list every member; open a member's page to see their
   submitted claims and profile, and (Super Admin only) **suspend / reactivate** the account or
   **reset password** (a reset emails the standard 15-minute link; refused while the member is
   suspended).
4. **Claims**: the standard `/admins/claims` list, but the Super Admin additionally sees a
   **Drafts** tab and drafts in "All" (full platform visibility). Still read-only oversight:
   they can only publish/transfer claims they personally own.
5. **Audit Log** (`/super_admins/audit_logs`): the full activity feed, filterable by actor
   and date range (see §9).

Provisioning the Super Admin is an operations task, see §12.

### 6.5 Public visitor

- `/marketplace` shows a paginated, filterable list of **published** claims with verdicts,
  topics, regions, search.
- `/marketplace/:id` shows the full claim detail: verdict, body, evidence, **source links**,
  research, fact-checker attribution, and **published date + publishing admin**. Related
  claims by topic at the bottom.

---

## AI chat assistant

Kasagadi runs a floating **AI chat assistant** (the widget in the corner) across the
public site and the member / fact-checker / admin portals. Anyone can ask the
fact-checking AI a question and get a streamed, sourced answer.

- **Who can use it:** everyone, both signed-in users *and* anonymous visitors. A signed-in
  user gets a `Conversation` owned by their account. An anonymous visitor gets nothing
  stored at all: no conversation row, no session token. Their thread lives only in the
  open widget, which reads its own bubbles back and posts them as context on each
  message, and it ends when the panel or the tab closes.
- **Records:** a `Conversation` has many `Message`s; each has a `role` (`user` /
  `assistant`), `content`, and (for assistant turns) a `sources` JSON list. The last ~20
  turns are sent upstream as history.
- **Backend:** `Chatbot::MessagesController` (under `/chatbot`), open to unauthenticated
  access.

Two endpoints, the closest thing the app has to a programmatic **API**:

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
3. Until the reply settles, the bubble keeps a pulsing "still writing" indicator beneath
   whatever text has arrived, so a pause between chunks reads as work in progress rather
   than a reply that ended mid-sentence. After 15s with no update the widget backs the dots
   up with "Still working on it..."; after 210s (just past the client's own read timeout) it
   gives up on the bubble and offers the question back.
4. On `done` it persists the user + assistant messages atomically and swaps the bubble for
   the final styled card (rendered Markdown + sources). On error it replaces the bubble with
   a danger notice carrying a **retry** button. The original message is embedded, so a click
   re-sends with no client-side state.
5. The job always settles the bubble exactly once. If the upstream stops without a terminal
   frame (dropped connection, read timeout), whatever text arrived is shown labelled as cut
   off, with a retry. A cut-off reply is never stored and never counted as a turn, so it
   stays out of the history either side sends back.

**External dependency:** answers come from the **FactCheckAI** service, a thin Ruby client
(`lib/fact_check_ai/`) over a configurable REST API (`FACTCHECK_AI_API_URL`). The same client
powers claim analysis (`analyse` / `analyse_stream`). If the upstream is down, the user sees
"Sorry, the fact-checking service is temporarily unavailable." with a retry.

**Realtime + suspension:** `ApplicationCable::Connection` allows anonymous connections (so
visitors can chat) and refuses suspended users (see "Account suspension").

## Account suspension

Any account can be set `active` or `suspended` (`users.status`). Suspension is a hard block
on access, enforced in depth:

- **Login**: `SessionsController#create` refuses a suspended user.
- **Existing sessions**: `User#suspend!` destroys all of the user's sessions in a
  transaction, so they're signed out everywhere immediately.
- **Session resume**: `Authentication#find_session_by_cookie` won't resume a session whose
  user isn't `active_for_auth?`.
- **Password reset**: self-service reset is refused for suspended users; the Super Admin's
  "reset password" is also refused until the account is reactivated.
- **Real-time**: `ApplicationCable::Connection` refuses a suspended user, so they get no
  live Turbo Stream updates.
- **Work routing**: suspended fact checkers can't be assigned claims; suspended admins can't
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

- **Sessions**: cookie-based; a signed cookie holds a `Session.id` stored in the DB, so
  sessions can be terminated server-side. Every request is gated by
  `Authentication#require_authentication` unless a controller opts out with
  `allow_unauthenticated_access`. Suspended users are rejected at login and on resume (§8).
- **Role gating**: role-namespaced base controllers (`Members::BaseController`,
  `FactCheckers::BaseController`, `Admins::BaseController`) enforce the role via
  `before_action`. `SuperAdmins::BaseController` inherits the admin gate and adds
  `ensure_super_admin` (shared `SuperAdminGuard` concern). Hitting a forbidden URL redirects
  away.
- **Ownership gating**: publish/unpublish/republish/transfer check `claim.owned_by?` /
  `publishable_by?` in both the view (hide controls) and the controller (reject the request).
- **Single super admin**: a partial unique index on `admins.super_admin` guarantees at most
  one.
- **Password reset**: `User.generates_token_for(:password_reset)` mints a 15-minute signed
  token bound to the password salt, so changing the password invalidates outstanding tokens.
  The request always returns the same generic notice to avoid leaking account existence.

---

## Dashboards & analytics

The admin dashboard renders lazy-loaded chart frames built by
`Admins::DashboardChartBuilder`. What each one **counts**:

- **Claim pipeline**: all non-draft claims, grouped by stage into Pending, In progress,
  **Awaiting** (in_progress with a verdict), Published, and Unpublished. (Awaiting is split
  out of In progress so the two don't double-count.)
- **Verification requests**: `Assignment` lifecycle (pending / in progress / completed /
  rejected) and average completion time.
- **Published claims by category / by verdict**: **published claims only**, by topic or by
  verdict, so the two views reconcile to the same total.
- **Claims submitted**: claims by `created_at` (submission date).
- **Fact-checks published**: published claims by `verification.published_at` (the real
  approval/publish date). Long ranges (90 days / all time) are bucketed by week or month so
  sparse data stays readable; the current period is highlighted.
- **Top fact-checkers**: by **published** verdicts. **Top members**: by **published** claims.

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

> **Never run `db:seed` in production**: the seed wipes claims (`Claim.destroy_all`). Use the
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

- **Mailers**: `ClaimMailer` (`assigned`, `reassigned`, `published`, `verdict_published`),
  `InvitationMailer`, `PasswordsMailer` (`reset`, `reset_code`).
- **Queue**: Solid Queue, database-backed Active Job, running in-process when
  `SOLID_QUEUE_IN_PUMA=1`.
- **Claim mail is no longer dispatched from the model.** All four `ClaimMailer` methods are
  reached through notifiers, so the email and the in-app row are one event rather than two
  code paths that can come to disagree about who was told what. See §14.
- The notifier's email method calls `deliver_now` from inside its own delivery job, on
  purpose: `deliver_later` would return as soon as the mail was queued, and the delivery log
  would stamp the row `sent` for a message that had not yet been handed to a mail server.
- The remaining direct dispatches are not notifications about a claim, and do use
  `deliver_later`:
  - `InvitationMailer.invite`, from the invitations controller, on create and on resend.
  - `PasswordsMailer.reset`, from `PasswordsController#create`, the Super Admin's admin
    reset, and an admin's member reset. All three skip suspended users.
  - `PasswordsMailer.reset_code`, from the mobile API's password-reset endpoint.

---

## Notifications

Ten things can happen to a claim that somebody needs to hear about. Each one is a
**notifier** under `app/notifiers/`, and each decides three things: who it reaches, what it
says, and which channels carry it.

Every one of the ten writes to the recipient's in-app feed. Only **four** also send an
email, and **seven** also send an SMS. The split follows the audience rather than the
urgency: members and fact checkers are reached on a phone, because that is the device they
submitted from and the one they carry, while the admin desk is reached in the app alone,
because an admin is already sitting in front of the ledger and every desk notifier fans out
to every active admin, so one event would buy several messages.

### 14.1 The ten events

| Notifier | Fires when | Who hears it | In app | Email | SMS |
|---|---|---|:---:|:---:|:---:|
| `ClaimSubmittedNotifier` | A member submits a claim | That member | ✅ | ✗ | ✅ |
| `NewSubmissionNotifier` | The same submission | The admin desk | ✅ | ✗ | ✗ |
| `ClaimAssignedNotifier` | An admin assigns a fact checker, and again when a different one is put on it | The fact checker | ✅ | ✅ | ✅ |
| `ClaimInReviewNotifier` | The `assign_checker` transition | The member | ✅ | ✗ | ✅ |
| `ClaimReassignedNotifier` | An admin hands the claim to a different checker | The **previous** checker | ✅ | ✅ | ✅ |
| `AssignmentRejectedNotifier` | A checker hands the claim back | The admin desk | ✅ | ✗ | ✗ |
| `VerdictAwaitingApprovalNotifier` | A verdict is created | The admin desk | ✅ | ✗ | ✗ |
| `ClaimPublishedNotifier` | The owner admin approves, and again on republish | The member | ✅ | ✅ | ✅ |
| `VerdictPublishedNotifier` | That same approval | The checker who wrote the verdict | ✅ | ✅ | ✅ |
| `ClaimUnpublishedNotifier` | The owner admin unpublishes | The member | ✅ | ✗ | ✅ |

The four email notifiers reach `ClaimMailer` through `DeliveryMethods::Email`, which is why
`ClaimMailer` has exactly four methods: `assigned`, `reassigned`, `published`,
`verdict_published`.

What the table cannot show:

- **"The admin desk" is not always everyone.** `ClaimNotifier#admin_users` sends to the
  claim's owner admin when it has one and to every active admin otherwise, because a fresh
  submission belongs to nobody yet. A suspended owner falls back to the whole desk as well:
  a suspended account cannot open its feed, and a staffing change must not be the reason a
  claim goes unnoticed.
- **Suspended users are never in an audience.** They cannot sign in, so a notification
  addressed to them would be written into a feed nobody can open.
- **An empty audience writes nothing at all.** The `noticed` gem treats an empty relation as
  truthy and would save the event anyway, leaving a row addressed to nobody for the admin
  ledger to render and a delivery job chasing it. Where a notifier worked its own audience
  out and found nobody, it reports a `ClaimNotifier::UnreachableAudience` to `Rails.error`:
  a claim needing attention was announced to an empty room, and every admin being suspended
  is the case that motivated it.
- **Reassignment is dispatched from the controller, not from a callback.** The recipient is
  the *previous* checker, and by the time the `Assignment` row is saved it already points at
  the new one.
- **`ClaimInReviewNotifier` says "assigned", not "started".** It fires on `assign_checker`,
  while the assignment is still `pending` and the checker can still hand it back. The class
  name predates the distinction and is kept because the persisted `type` column carries it.
- **The assignment email and text are guarded, the in-app row is not.** Both outbound
  channels address whoever holds the assignment when the job runs, so a second reassignment
  inside job latency would reach a checker the claim has already left; both are skipped in
  that case, on the one `ClaimAssignedNotifier::STILL_ASSIGNED` condition. The text is the
  worse of the two to get wrong: it costs money and it tells someone to go and work on a
  claim that is not theirs. The in-app row stands either way, because it is history and
  correct as written.
- **A verdict announces itself once.** `Verification` notifies on create only. It stays
  revisable while it waits on an admin, and re-announcing every edit would turn the approval
  queue into noise.
- **Republishing reuses the publication notifier** rather than adding an eleventh. Without
  that, a member would hear their claim was pulled and never hear it came back.
- **A notification failure never takes down what caused it.** `ClaimNotifier.notify` rescues,
  reports to `Rails.error` and returns nil. These run from `after_commit` hooks and from
  controller actions, so the claim is already saved: raising would show a 500 for something
  that actually worked, and would strand every recipient queued behind the one that failed.

### 14.2 Imported claims notify nobody

Dubawa imports carry an `external_source`, which is what `Claim#imported?` reads.
`ClaimNotifier#deliver` returns before anything is written for such a claim, on every
channel, for every recipient.

Two reasons. Nobody asked for the work, so nobody is waiting on the answer. And the member
on an imported claim is a service account that never signs in, so anything addressed to it
goes into a feed with no reader. Without the rule, a single import would put one "verdict
awaiting approval" in front of every admin for every row, because Dubawa imports create a
verification per claim.

The rule lives in the base class rather than at each call site, so a new notifier cannot
forget it. `Assignment` checks `imported?` a second time before it builds the event, which
buys nothing in behaviour and only avoids constructing something nobody would read.

### 14.3 The delivery log

`noticed` records that a recipient was notified. It records nothing about whether the copy
left, which provider carried it, or what it said. `NotificationDelivery` is that record: one
row per outbound copy, per channel.

`ApplicationDeliveryMethod` wraps every channel in an `around_deliver`, so the row is
created **before** the send and stamped after it. A crash mid-flight still leaves a trace,
and no channel has to remember to write to the log.

| Status | Means |
|---|---|
| `queued` | The row exists; the send has not come back |
| `sent` | The channel handed the message off |
| `delivered` | The telco confirmed it reached the handset (SMS only) |
| `failed` | The send raised, or the telco reported a failure |
| `skipped` | The channel declined to send, and said why |

`skipped` is the one that needs stating: it is a decision, not a failure. It is what an
unusable phone number produces. A delivery method declines by calling `skip!` from inside
`deliver`; halting a `before_deliver` chain instead would return normally through
`around_deliver` and stamp the row `sent` for a send that never happened.

Each row also carries `destination`, `subject`, `body`, `provider_message_id`,
`provider_status`, `error`, `sent_at` and `delivered_at`. The email method archives the
rendered subject and body **before** delivering, so the ledger holds the message as the
recipient saw it rather than a reconstruction of it.

### 14.4 What an admin sees

`/admins/notifications` is the ledger. Five numbers sit above the rows and answer the
question the page exists to ask: **sent** (every notification), **reached** (`sent` plus
`delivered`), **failed**, **unreachable** (`skipped`), and **in app only** (notifications
with no delivery rows at all, which is the three admin desk notifiers).

Each row is one notification: the recipient and their role, the title, and two fixed channel
slots, email then SMS, always in that order. A channel that was never used reads as an
absence in a place the eye already knows to look, so the column can be scanned rather than
read. Colour carries the state, and nothing else on the page carries colour.

Opening a row shows the recipient, the notifier, whether they have read it, and every
outbound attempt with its own status. An archived email body renders inside a sandboxed
frame, so its styles stay out of the page and nothing in it can run. An SMS body renders at
the width it was written for with its character count beneath it, so its length reads
honestly against the 160-character segment. A folded "delivery trail" carries the provider
id, the telco status and the timestamps.

The log keeps one row per **attempt**, so a send that failed and then succeeded on retry has
two rows on the same channel. The index shows the newest attempt per channel, which is where
that channel ended up.

Every signed-in user reads their own feed at `/notifications`, whatever their role. Opening
a notification is what marks it read, so the click goes through the app rather than straight
to the subject; a notification whose record has since been deleted lands back on the feed
instead of raising.

### 14.5 SMS through Hubtel, and the switch that keeps it quiet

SMS goes out through **Hubtel** (`lib/hubtel/`), using Regular Send, a POST with Basic auth,
rather than Quick Send, whose GET variant carries the client secret in the query string
where it lands in server logs, proxy logs and error-tracker breadcrumbs.

**The switch is `config.x.sms.enabled`, set per environment:**

| Environment | `config.x.sms.enabled` |
|---|---|
| development | `true` |
| test | `false` |
| production | `true` |

Credentials are deliberately **not** the switch. There is a single credentials file, so the
production Hubtel account resolves in development too, and "has credentials" would mean the
channel silently switched itself on wherever the app ran.

Read the development row again, because it means what it says: **submitting, assigning,
reassigning, publishing or unpublishing a claim on your own machine sends a real, charged
SMS to whatever number is on that member's or fact checker's profile.** Seven of the ten
notifiers text, so most of the claim lifecycle is now live locally. If you are working
through any of it, either blank the phone numbers on your local records or set
`config.x.sms.enabled = false` in `config/environments/development.rb` while you do it. A switched-off channel records a
**skipped** row reading "SMS channel is not configured", so an SMS column full of skips on a
machine with SMS off is correct, not a bug. It used to record nothing at all, on the
reasoning that a channel which does not exist cannot skip a delivery; that reads in the
ledger exactly like a notifier that never fired, which is how an environment deployed
without Hubtel credentials can send no SMS for weeks with nothing to say so.

**The ceiling is five requests a minute, across all of Hubtel's endpoints.** Sends and
delivery-receipt polls draw on the same budget (`Hubtel::REQUESTS_PER_MINUTE = 5`).
`Hubtel::RateLimiter` keeps a sliding window over the last minute in the cache and is checked
*before* the delivery row is created, because a throttle bounce is not a failed delivery and
recording it as one would fill the ledger with alarms for messages that go out fine a moment
later. A throttled job is re-enqueued, waiting exactly as long as the limiter says a slot
needs, for up to **eight attempts**; on exhaustion the job writes the one row that keeps a
message we gave up on from looking like a message we never had to send, reading
"Throttled, gave up after 8 attempts".

The limiter is best effort, not a guarantee. Read, filter and write are three steps rather
than an atomic increment, and Solid Cache offers no atomic list primitive, so concurrent
workers can each see the same free slot. It also fails open: a cache outage reads as "no
recent requests" and lets everything through. Both are the right way round, because Hubtel
itself is the real enforcement. It answers `429`, the client raises `Hubtel::RateLimited`,
and the job retries. The test environment uses `:null_store`, which makes the limiter inert
there unless a real store is injected.

Three more behaviours that are easy to miss:

- **A number that cannot be placed is a `skipped` row, not a failure.** `PhoneNumber.e164`
  returns nil rather than guessing, because a wrong number sends someone else's claim details
  to a stranger. The row reads "No usable phone number on file". Phone lives on the role
  rather than on the user, so a recipient's fact-checker number is preferred and their member
  number is the fallback.
- **Curly punctuation is traded for ASCII before sending.** A segment is 160 characters in
  GSM-7 but only 70 in UCS-2, and one curly quote or dash in a claim title switches the whole
  message over and triples the bill. Claim titles are quoted from the wild, so that is the
  ordinary case rather than a rare one. The copy truncates the title to 70 characters for the
  same reason.
- **Only a connection failure is retried.** A read timeout means the POST went out and the
  reply was lost, so the message has most likely been sent and charged for; retrying would
  bill up to three times and put three copies on the handset. One `failed` row an admin can
  act on is the lesser harm.

**Delivery receipts.** Hubtel's send response only confirms Hubtel took the message, and a
2xx is not enough on its own: a rejection arrives as a 2xx carrying a non-zero status and a
description, so acceptance is the payload status, not the HTTP code. `HubtelStatusJob` asks
for the telco's verdict a minute after the send, and re-checks up to three times, ten minutes
apart. `delivered` and `failed` are final; "Pending" and "Sent" are not. A row that never
settles keeps its `sent` status and its raw provider status, which is the honest description
of what we know.

### 14.6 Where the copy lives

`config/locales/notifiers.en.yml`, scoped by the generated notification class:
`ClaimAssignedNotifier::Notification` reads `notifiers.claim_assigned_notifier.notification`.
Every entry has a `title` and a `body`, and the seven SMS notifiers add an `sms` line. One
segment is 160 characters, and each notifier passes its claim title truncated to 70, so the
fixed part of an `sms` string has to fit in the remaining 90. It must be plain ASCII too: a
single curly quote or dash drops the whole message from 160 characters per segment to 70,
which is why `DeliveryMethods::Sms` folds the text before it goes to the wire.

Claim titles are often whole sentences, so they are quoted inside body copy to keep the
surrounding sentence readable: "New claim assigned to you" over
`"Free bus fares for students in September" is now in your queue.`

### 14.7 The app reads the same feed

The mobile app reads the same `Noticed::Notification` rows the web bell reads, so a
notification opened on a phone is read on the web too. Four endpoints, both roles:

| Endpoint | What it does |
|---|---|
| `GET /api/v1/me/notifications` | the feed, `?filter=unread`, paged |
| `GET /api/v1/me/notifications/unread_count` | the badge on its own |
| `POST /api/v1/me/notifications/:id/read` | mark one read |
| `POST /api/v1/me/notifications/read_all` | clear the badge |

There is no `show`. The web's `show` marks a notification read and then redirects, because
a browser needs somewhere to land. An app already holds the row it tapped, so the only
thing left to ask the server is to record the read.

**Every notifier answers "where does this point" twice.** `url` is a web path, which names
a route in a Rails app rather than a screen in an iOS one. Handing that to a native client
leaves it opening a browser, which throws the user out of the app, or parsing the path with
a regex and hoping the routes never move. So each notifier also declares a `mobile_target`,
naming the **subject** instead, directly under the `url` it already had:

```ruby
def url
  fact_checkers_assignment_path(params[:assignment])   # "/fact_checkers/assignments/294"
end

def mobile_target
  { type: "assignment", id: params[:assignment].id }   # { "type": "assignment", "id": 294 }
end
```

Both answers live in one place, so adding a notifier means writing both or noticing you did
not. That matters where the two differ: `ClaimReassignedNotifier` points a former checker at
the queue rather than at the assignment, because by then it belongs to somebody else, and
`mobile_target` makes the same call one line below for the same reason.

Five target types, each naming exactly one endpoint the app fetches from:

| `mobile_target` | The app fetches | Which notifiers |
|---|---|---|
| `{"type": "claim", "id": 259}` | `GET /api/v1/claims/259` | `ClaimPublished`, `VerdictPublished` |
| `{"type": "my_claim", "id": 345}` | `GET /api/v1/me/claims/345` | `ClaimSubmitted`, `ClaimInReview` |
| `{"type": "my_claims"}` | `GET /api/v1/me/claims` | `ClaimUnpublished` |
| `{"type": "assignment", "id": 294}` | `GET /api/v1/fact_checker/assignments/294` | `ClaimAssigned` |
| `{"type": "assignments"}` | `GET /api/v1/fact_checker/assignments` | `ClaimReassigned` |

`ClaimPublishedNotifier` points at the **public** fact check rather than at the member's own
copy, because the published article is what the member is being told about, which is the
same reason its `url` is `marketplace_path`. `ClaimUnpublishedNotifier` points at the list
rather than at the claim, because an unpublished claim's own screen is unreachable, again
matching the web.

**A null target means there is nowhere to go**, and the app should render the row as text
rather than as something tappable. Two cases produce one: the three admin-only notifiers,
whose audience has no app, and a notification whose claim has since been deleted. The second
is why the serializer rescues `title`, `body` and `target` independently. A notification
renders itself from whatever the notifier captured when it fired, those records can be gone
by the time anyone opens the feed, and one dangling row must not take a whole screen down,
which on a phone is a blank page with no way past it.

The feed orders by `created_at` and then by `id`. The gem's `newest_first` orders on
`created_at` alone, and two notifications from a single event share it to the microsecond;
on a paged feed an arbitrary tiebreak lets a row appear on two pages or on neither.

Push notifications are not built. This is the feed the app reads; APNs and FCM come later.

---

## Storage

Active Storage is configured for two services:

- **local**: disk-based, development.
- **cloudflare**: Cloudflare R2 (S3-compatible) via `aws-sdk-s3`. R2 quirks handled in
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

Sign in with the seeded accounts (`db/seeds.rb` seeds `admin@kasagandi.com` as the super
admin in dev) or register a fresh member at `/registration/new`.

---

## Testing

```bash
bin/rails test                 # full suite (~1,070 tests)
bin/rails test:system          # browser tests, drives Chrome
bin/rubocop                    # style
bin/brakeman                   # security scan
bin/importmap audit            # JS dep audit
```

CI runs all four jobs on every push to `main` and on pull requests. Its test job
runs `test test:system`, so a browser test failing fails the build.

Conventions:

- Controller tests under `test/controllers/<namespace>/`; sign-in helper in
  `test/test_helper.rb` (`sign_in(user)`).
- Fixtures cover all roles and lifecycle states (`pending_claim`, `under_review_claim`,
  `verified_claim`, `awaiting_approval_claim`) plus a second admin for ownership/transfer
  tests.
- System tests live in `test/system/` and drive a real Chrome.
  `HEADED=1` opens a window to watch a run, which is the quickest way to see why
  one failed; failures also leave a screenshot in `tmp/screenshots`.
  `test/system/sms_notifications_test.rb` walks one claim through its whole life
  and checks the text sent at each step, and `LIVE_SMS=1 SMS_TO=+233…` makes it
  send real messages through Hubtel rather than stubbed ones.
- Two things a system test depends on and neither is obvious. The runner needs
  **libvips**, because a claim's evidence image is rendered through an Active
  Storage variant. And **selenium-webdriver is pinned below 4.40** for Ruby
  3.3.0, whose parser rejects the anonymous keyword argument forwarding that
  version introduced; without the pin the driver cannot be loaded and no system
  test reaches a browser at all.
- Wait on the state a step leaves behind, never on its flash notice. Notices are
  toasts and are removed a few seconds later, so a test that waits on one passes
  on a fast machine and fails on CI.

---

## Glossary

- **Claim**: a public statement someone wants verified. The unit of work.
- **Owner admin**: the single admin accountable for a claim; the only one who can publish,
  unpublish, republish, or transfer it. Set when they assign a fact checker.
- **Transfer**: moving claim ownership from one admin to another (audited).
- **Evidence**: supporting material the member supplies (description + files).
- **Assignment**: the relationship between a claim and a fact checker (due date, notes,
  status).
- **Verification**: the fact checker's verdict and research write-up.
- **Verdict**: one of True, False, Misleading, Partly True, Unverifiable.
- **Audit log**: immutable record of who did what; the platform's accountability trail.
- **Suspension**: an account state that blocks all access (login, sessions, reset, realtime).
- **Super Admin**: the single platform owner; oversees admins, fact-checker accounts, global
  claim visibility, the audit log, and analytics.
- **Marketplace**: the public catalogue of published claims.
- **Accredited**: fact checkers with `verified: true`, marked with a check icon.

---

## Public claims API

External partners can list Kasagadi's **published, fact-checked claims** from their own
apps through a read-only JSON API. The full, interactive reference lives at
**[docs.kasagadi.ai/api.html](api.html)** (Swagger UI), backed by the machine-readable
[`openapi.yaml`](openapi.yaml) contract. Import that into Postman/Insomnia or a client
generator.

- **Base URL:** `https://kasagadi.ai/api/v1`
- **Auth:** a per-partner API key sent as `Authorization: Bearer kg_live_…`. Keys are
  issued by the Kasagadi team, secret, and revocable. Use them **server-to-server**.
  Never embed a key in a browser or mobile app.
- **Scope:** only **published** claims are ever returned. Drafts and in-progress claims
  are structurally impossible to reach through the API.

**Endpoints:**

| Endpoint | Method | Returns |
|---|---|---|
| `/api/v1/claims` | GET | Paginated list of published claims (`{ data, meta }`). |
| `/api/v1/claims/:id` | GET | A single published claim (`{ data }`); `404` if not published. |
| `/api/v1/claims/:id/related` | GET | Up to three other published claims sharing the claim's primary topic (`{ data }`, no `meta`). |
| `/api/v1/marketplace/facets` | GET | Published-claim counts per topic, region and verdict, keyed by the value you would filter on. A value with no published claims is absent rather than zero. |

**List filters** (all optional): `topic`, `region`, `verdict`
(`verdict_true` / `verdict_false` / `misleading` / `partly_true` / `unverifiable`),
`q` (search over title + content), `since` (ISO-8601: only claims published on/after it,
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

**Issuing keys (ops):** keys are managed from the console for now:
`bin/rails "api_key:issue[Partner Name]"` prints the key **once** (store it immediately),
`bin/rails api_key:list` shows all keys and usage, and `bin/rails "api_key:revoke[ID]"`
disables one.

## Mobile API journeys

Written for the mobile team. The [Swagger reference](api.html) says what each endpoint
accepts and returns; this section says **when to call it, in what order, and what to do
with the answer**. Each journey follows one person through one task.

- **Base URL:** `https://kasagadi.ai/api/v1`
- **Credential:** an access token from `POST /auth/login`, sent as
  `Authorization: Bearer <access_token>`. The app ships **no** API key. The partner key
  described in §21 is a different credential for a different audience: never put one in
  the app.
- **Two roles reach the app:** members of the public, and accredited fact-checkers.
  Administrators are refused and told to use the web portal.
- **Three tiers of endpoint.** `/claims`, `/claims/:id`, `/claims/:id/related` and
  `/marketplace/facets` take either credential. `/me`, `/me/notifications`, `/taxonomies`
  and `/auth/logout` take any user token. Everything under `/me/claims` is members only and everything under
  `/fact_checker` is fact-checkers only, each refusing the other role with `403` (§22.16).

### 22.1 The two tokens, in one paragraph

Signing in returns a pair. The **access token** rides on every request and lasts
**30 minutes**. The **refresh token** does nothing but buy a new pair, and lasts
**30 days**. Spending a refresh token invalidates it and issues a new one, so the app
always holds exactly one live pair. Store both in the device keychain, never in plain
preferences, and never write either into a log.

| | Lifetime | Used for |
|---|---|---|
| `access_token` | 30 minutes | Every authenticated request |
| `refresh_token` | 30 days, rolling | Only `POST /auth/refresh` |

### 22.2 Kofi opens the app for the first time

*Kofi has just installed Kasagadi. He hasn't signed in and hasn't tapped anything.*

Before the login screen renders, fetch the configuration. It needs no credential.

```
GET /api/v1/config
```

```json
{ "data": {
  "api_version": "v1",
  "min_supported_app_version": "1.0.0",
  "uploads": {
    "evidence": { "max_bytes": 10485760, "content_types": ["image/jpeg", "…"],
                  "formats_label": "JPG, PNG, WebP, GIF, PDF, DOC, DOCX, MP3, M4A, WAV or OGG" },
    "avatar":   { "max_bytes": 5242880, "content_types": ["image/png", "image/jpeg", "image/webp"] }
  },
  "claim_statuses": ["draft", "pending_assignment", "in_progress", "published", "unpublished"],
  "verdicts": ["verdict_true", "verdict_false", "misleading", "partly_true", "unverifiable"],
  "assignment_statuses": ["pending", "in_progress", "completed", "rejected"],
  "links": { "privacy_policy": "https://…/privacy", "terms": "https://…/terms", "cookies": "https://…/cookies" }
} }
```

**What the app does with it.** Cache it on disk and refresh on each cold start. Drive the
file picker from `uploads.evidence` rather than hard-coding "10MB". Server limits can
change, and a hard-coded copy will drift and reject files the server would accept, or
accept files it rejects. Render verdict and status labels from these lists so a new verdict
type does not require an app release. Compare the running build against
`min_supported_app_version` and prompt an upgrade when it is older.

**Do not** ship a fallback copy of these values that silently replaces a failed call. If
`/config` is unreachable, say so and retry: guessing the limits is how the two drift apart.

### 22.3 Kofi creates a member account

*Kofi wants to submit a claim he saw on WhatsApp, so he needs an account.*

```
POST /api/v1/auth/register
```

```json
{ "name": "Kofi Mensah",
  "email_address": "kofi@example.com",
  "password": "a-strong-password",
  "password_confirmation": "a-strong-password" }
```

`201 Created`:

```json
{ "data": {
  "token_type": "Bearer",
  "access_token": "…", "refresh_token": "…",
  "expires_in": 1800, "refresh_expires_in": 2592000,
  "user": { "id": 42, "name": "Kofi Mensah", "role": "member",
            "profile": { "organization": null, "phone": null, "region": null, "bio": null },
            "abilities": ["marketplace.browse", "chat.use", "claims.create",
                          "claims.submit", "claims.track"] }
} }
```

**There is no email-verification step.** Registration signs Kofi in: the response already
carries a usable pair, so go straight to the member home screen. Do not call `/auth/login`
afterwards.

**Only members self-register.** Fact-checkers arrive by invitation (§22.11), so the app
needs no "sign up as a fact-checker" path.

**Password rules,** worth enforcing in the form so Kofi finds out before the round trip:
at least 8 characters, at most 72 bytes, and `password_confirmation` is **required**:
omitting it is rejected rather than skipping the check.

**When it fails,** the body names the field:

```json
{ "error": { "status": 422, "message": "Validation failed",
             "details": { "email_address": ["has already been taken"] } } }
```

Map `details` onto the form fields. Every key is an attribute name and every value is a
list of messages, so the shape is stable enough to render generically.

### 22.4 Ama signs in, and the app works out where to send her

*Ama already has an account. She may be a member or a fact-checker; the app does not know
which until the server says so.*

```
POST /api/v1/auth/login
{ "email_address": "ama@example.com", "password": "…" }
```

The `200` body is the same envelope as registration, and `data.user.role` is the routing
decision:

| `role` | Send them to |
|---|---|
| `member` | The member home: submit a claim, track claims, browse the marketplace |
| `fact_checker` | The fact-checker home: assignment queue, verdicts |

**Route on `role`, not on `abilities`.** `abilities` is a convenience for deciding which
tabs and buttons to render; it is **not** an authorization mechanism. The server enforces
every rule independently, so hiding a button is a courtesy to the user, not a security
control, and forging the list gains nothing.

**Three ways this fails, and they mean different things:**

| Status | Meaning | What the app should show |
|---|---|---|
| `401` | Wrong email or password | "Try another email address or password." Stay on the form. |
| `403` "…suspended…" | Correct password, account suspended | A dead end. Show the message; offer support contact, not a retry. |
| `403` "Administrators sign in on the web portal." | Correct password, admin account | Point at the website. Do not offer a member screen. |

Note that the password must already be correct before either `403` appears, so these
cannot be used to discover whether an address has an account.

### 22.5 Ama comes back the next morning

*Ama used the app at 9am and opens it again at 2pm. Her access token expired hours ago.*

The first authenticated call returns:

```json
{ "error": { "status": 401, "message": "Invalid or expired access token" } }
```

Do not sign her out. Refresh:

```
POST /api/v1/auth/refresh
{ "refresh_token": "…" }
```

```json
{ "data": { "token_type": "Bearer", "access_token": "…", "refresh_token": "…",
            "expires_in": 1800, "refresh_expires_in": 2592000 } }
```

No `user` object comes back: the app already has it. **Store both new tokens and discard
the old pair immediately**: the refresh token you just spent is already dead.

**Build this as an interceptor,** not as a check on each screen: on any `401`, refresh once
and replay the original request. If the refresh itself returns `401`, the session is
genuinely over. Clear the keychain and show the sign-in screen.

**One trap worth designing around.** Refresh is single-use. If five requests fail with
`401` at once and each fires its own refresh, the first succeeds and the rest present a
spent token and get `401`, which will look like a broken session. **Single-flight the
refresh:** the first caller refreshes, the others wait for it and replay with the new
token.

**The old access token dies with the old refresh token.** Rotation is re-issuance: the same
session row is rewritten, so the access token the app was carrying stops working the moment
the refresh returns, whether or not it had minutes left on it. A live smoke test caught a
client that stored the new refresh token, kept using the old access token, and then failed
`401` on every subsequent request while looking like a server fault. Swap **both** together,
before replaying anything.

### 22.6 Ama forgets her password

*Ama is locked out. She is on a phone, so an emailed link is unreliable: it may open on a
laptop, or in a browser with no route back into the app.*

The flow is a **6-digit code**, in two steps. Step one:

```
POST /api/v1/auth/password_resets
{ "email_address": "ama@example.com" }
```

`202 Accepted`, always:

```json
{ "data": { "message": "If an account exists for that email address, a reset code is on its way." } }
```

**That response is identical whether or not the account exists**, deliberately, so the
endpoint cannot be used to discover who has an account. The app cannot tell either: show
the same "check your email" screen regardless, and never phrase it as confirmation that
the address was found.

Step two, Ama types the code and her new password on one screen:

```
PUT /api/v1/auth/password_resets/000000
{ "reset_code": "482913",
  "email_address": "ama@example.com",
  "password": "her-new-password",
  "password_confirmation": "her-new-password" }
```

**Send the code as `reset_code` in the body.** The URL segment still works and is still the
published shape, but a code in a path is written verbatim into server and proxy access
logs where it cannot be redacted. When `reset_code` is present it wins, so put any
placeholder in the path.

On success:

```json
{ "data": { "message": "Your password has been reset. Please sign in." } }
```

**No tokens come back.** That is deliberate: a reset exists to lock someone else out, so it
destroys every session on every device and the app returns Ama to the sign-in screen. If
she was signed in on a tablet, that session is gone too.

The code **expires in 15 minutes**, works **once**, and dies after **five wrong attempts**.

**Two failure shapes to handle differently:**

- `422` means the password itself was rejected (too short, too long, confirmation mismatch).
  **The code is not spent.** Let her fix the password and submit again with the same code.
- `401` "That reset code is invalid or has expired." means the code was wrong, expired,
  already used, or five attempts burned. Send her back to step one for a fresh code.

### 22.7 Kofi signs out

```
DELETE /api/v1/auth/logout
Authorization: Bearer <access_token>
```

`204 No Content`. This destroys **only this device's** session; if Kofi is signed in on a
tablet, that stays signed in.

**Logout always succeeds**: `204` even for an expired, already-spent or malformed token,
and even with no header at all. An app can never be stuck holding a credential it cannot
discard. It accepts **either** token, so send whichever you have.

Clear the keychain when the response arrives. If the request fails on the network, clear it
anyway and move on: the token is short-lived, and the server-side row is cleaned up on
expiry.

### 22.8 Kofi writes a claim, attaches a photo, and submits it

*Kofi has a screenshot of the WhatsApp broadcast. This is the whole reason he installed the
app.*

Three requests, in this order. There is no signed-upload handshake and no separate commit
step: **the bytes go in the same request that creates the evidence.**

**Step one, the draft.** Nothing is required yet, so a half-written claim survives a bad
connection.

```
POST /api/v1/me/claims
Authorization: Bearer <access_token>

{ "claim": {
  "title": "Free bus fares for students in September",
  "content": "A WhatsApp broadcast says the Ministry has approved free bus fares…",
  "source": "WhatsApp",
  "topics": ["Education"],
  "regions": ["Greater Accra"],
  "source_links": ["https://example.com/post/1"]
} }
```

`201 Created`, and the body is the whole claim with an empty evidence list:

```json
{ "data": { "id": 88, "title": "Free bus fares for students in September",
            "status": "draft", "topics": ["Education"], "regions": ["Greater Accra"],
            "source_links": ["https://example.com/post/1"],
            "cover_image_url": null, "verdict": null,
            "created_at": "2026-08-01T09:12:00Z", "updated_at": "2026-08-01T09:12:00Z",
            "evidences": [] } }
```

Take `topics` and `regions` from `/taxonomies`: send the `value`, show the `label`. Only
`source_links` are checked at this stage, and each must be a valid `http` or `https` link.

**Step two, the photo.** One `multipart/form-data` request:

```bash
curl -X POST https://kasagadi.ai/api/v1/me/claims/88/evidences \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -F "evidence[files][]=@broadcast.jpg" \
  -F "evidence[description]=The message as it arrived"
```

`201 Created`, and the body is the **whole claim again**, so the app redraws its evidence
list from the response and never has to follow a write with a read:

```json
{ "data": { "id": 88, "status": "draft",
  "cover_image_url": "https://kasagadi.ai/rails/active_storage/blobs/redirect/…/broadcast.jpg",
  "evidences": [
    { "id": 15, "description": "The message as it arrived",
      "created_at": "2026-08-01T09:14:00Z",
      "files": [ { "id": 31, "filename": "broadcast.jpg", "content_type": "image/jpeg",
                   "byte_size": 284133,
                   "url": "https://kasagadi.ai/rails/active_storage/blobs/redirect/…/broadcast.jpg" } ] } ] } }
```

Note that `cover_image_url` filled itself in. A claim's cover is promoted from the first
image among its evidence, which is why there is no separate cover upload.

Four things about that request:

- **Repeat `evidence[files][]` once per file.** Several files in one request become one
  evidence row with several files under it.
- **Evidence is added, never replaced.** A member photographs one thing at a time, and a
  phone that loses signal halfway through a set must not wipe what already uploaded. Send a
  second request for the receipt.
- **The server decides what a file really is.** Every upload is identified from its own
  bytes, not from the content type the request declared, so a payload renamed to `.png` is
  refused even though the request called it an image.
- **Read the limits from `/config`**, under `uploads.evidence`: today 10MB a file, and
  "JPG, PNG, WebP, GIF, PDF, DOC, DOCX, MP3, M4A, WAV or OGG". Do not hard-code either.

Refusals are per file and name the file:

```json
{ "error": { "status": 422, "message": "Validation failed",
             "details": { "files": ["\"clip.wav\" is larger than 10MB"] } } }
```

An unsupported format reads `"notes.exe" is not a supported format. Attach JPG, PNG, WebP,
GIF, PDF, DOC, DOCX, MP3, M4A, WAV or OGG`, and a file that is both unsupported and oversized
says both. Sending no usable file at all, a file field submitted with nothing chosen
included, gives `{ "files": ["Attach at least one file."] }`. Omitting the `evidence` part
altogether is a different thing: `400` with `Missing parameter: evidence`. A missing envelope
is a malformed request, not a validation failure about its contents.

**Step three, submit.** No body at all.

```
POST /api/v1/me/claims/88/submit
```

`200`, and `status` is now `pending_assignment`. This is where the strict rules run: title,
content, source, at least one topic, at least one region, **and at least one piece of
evidence with a file actually attached**. Only once all of that passes does the claim change
state, so it can never reach the queue half-finished.

The refusal for the last of those arrives under `base` rather than under a field, because it
is about the claim as a whole:

```json
{ "error": { "status": 422, "message": "Validation failed",
             "details": { "base": ["Please attach at least one piece of evidence with a supporting file."] } } }
```

Render `details.base` as a form-level message. Every other key is an attribute name, so one
renderer covers both.

**Submit once.** The call queues the AI analysis and tells both Kofi and the admin desk. A
second submit answers `422` and `This claim has already been submitted.`

### 22.9 Ama follows a claim from her dashboard to a published verdict

*Ama submitted a claim last week and wants to know where it has got to.*

```
GET /api/v1/me/dashboard
```

```json
{ "data": {
    "filter": "all",
    "counts": { "all": 7, "drafts": 2, "pending": 1, "in_progress": 3, "review": 3, "verified": 3 },
    "claims": [ { "id": 88, "status": "in_progress", "verdict": null } ] },
  "meta": { "page": 1, "per_page": 10, "total_pages": 1, "total_count": 7 } }
```

Three things to know about those numbers:

- **`all` excludes drafts.** That is the web's meaning of the word, and drafts have their own
  tab. `?filter=drafts` is how you get them.
- **`review` is the same number as `in_progress`.** The web dashboard has always offered both
  names for the same tab, and the API keeps both rather than making one platform lie.
- **`data.filter` is the filter that was actually applied.** An unrecognised value falls back
  to `all` rather than being refused, so read it back instead of assuming.

`GET /me/claims` is the same list without the counts, taking the same `filter` names and
paging 15 at a time rather than 10. Neither list carries `evidences`: no list screen shows
them and they would be paid for in queries. `GET /me/claims/88` does carry them.

**How the state reads on screen.** Ama's copy of the claim carries the verdict from the
moment a checker submits it, which is before an admin has approved it. So:

| What Ama should be shown | `status` | `verdict` |
|---|---|---|
| Saved, not sent | `draft` | `null` |
| Waiting for a fact checker | `pending_assignment` | `null` |
| Being checked | `in_progress` | `null` |
| Checked, waiting on approval | `in_progress` | present, `checked_at` null |
| Live on the marketplace | `published` | present, `checked_at` set |

**Do not read "a verdict exists" as "the claim is published."** A verdict awaiting approval
can still be discarded if an admin reassigns the claim, so gate the "fact checked" badge on
`status == "published"` and take the date it went live from `verdict.checked_at`.

**A claim can also vanish.** An admin can unpublish one, and unpublished claims are filtered
out of everything a member reads: it drops off the dashboard, and `GET /me/claims/88` starts
answering `404`. Ama is told, and her notification feed carries it (§22.14), but the app may
well render the `404` first. Treat a `404` on a claim you were already displaying as a stale
screen: return to the list rather than showing an error.

### 22.10 Kofi edits one draft, discards another, and is refused on a third claim

*Kofi has two drafts and one claim he sent yesterday.*

**Editing a draft** is a `PATCH`, and only the keys sent are written, so an app correcting a
title need not resend the body:

```
PATCH /api/v1/me/claims/88
{ "claim": { "title": "Free bus fares for students from September" } }
```

`200` with the whole claim back, evidence included.

**Removing one piece of evidence** is a `DELETE` on the evidence row, and it also answers
with the whole claim rather than `204`, so the app gets back the list it should now be
showing:

```
DELETE /api/v1/me/claims/88/evidences/15
```

The cover image is deliberately **not** cleared when the evidence it came from is removed. It
is a copy taken at promotion time, not a reference to that file, and the web behaves the same
way.

**Discarding a draft** is permanent, and takes its evidence and their uploads with it:

```
DELETE /api/v1/me/claims/88
```

`204 No Content`.

**And the wall.** All three of those are refused the moment the claim leaves `draft`:

```json
{ "error": { "status": 422, "message": "This claim has already been submitted." } }
```

That is a `422` rather than a `403`, and the distinction is real: the claim is still his, he
simply cannot change it any more. It is in the platform's queue and somebody may already be
working on it. Hide the edit and delete controls once `status` is anything but `draft`, and
treat the `422` as the backstop for a stale screen rather than as the primary check.

### 22.11 Akosua the fact-checker signs in

*Akosua has been accredited by Kasagadi. She never registered.*

Fact-checkers are invited by an administrator and accept through the **web** (a 24-hour,
single-use link, §6.2), which creates the account. From then on she uses the same
`POST /auth/login` as everyone else, and `/me` returns:

```json
{ "data": { "role": "fact_checker",
  "profile": { "organization": "FactCheck Ghana", "specialty": "Politics",
               "phone": "0209876543", "region": "Ashanti", "bio": "…", "verified": true },
  "abilities": ["marketplace.browse", "chat.use", "assignments.view", "assignments.start",
                "assignments.reject", "verdicts.submit", "verdicts.edit"] } }
```

Note `profile` carries different keys per role: `specialty` and `verified` exist only for
fact-checkers. Read it by role, not as a fixed shape.

### 22.12 Akosua works through an assignment

*Akosua has been handed a claim. She has not opened it yet.*

```
GET /api/v1/fact_checker/assignments
```

```json
{ "data": [ { "id": 7, "status": "pending", "due_date": "2026-08-10T00:00:00Z",
              "created_at": "2026-08-02T08:00:00Z", "updated_at": "2026-08-02T08:00:00Z",
              "claim": { "id": 88, "title": "Free bus fares for students in September",
                         "status": "in_progress", "verdict": null } } ],
  "meta": { "stats": { "assigned": 3, "in_review": 1, "completed": 12 },
            "q": null, "status": null, "topic": null, "sort": "newest",
            "page": 1, "per_page": 15, "total_pages": 1, "total_count": 4 } }
```

`meta.stats` counts the whole queue rather than the current filter, so a tab does not hide
its own count the moment you select it. `meta` also echoes the filters back, and that matters:
an unrecognised `status` or `sort` is **ignored rather than refused**, so compare what came
back with what you sent before rendering "no results".

The `status` filter takes `pending`, `in_progress` or `completed`. There is deliberately no
`rejected`: an assignment she has handed back has left her queue entirely, so the filter would
only ever return nothing.

**Opening it.** `GET /fact_checker/assignments/7` adds the admin's brief (`notes`), the name
of the member who submitted the claim and nothing else about them, the claim's evidence, her
own working copy of the verdict, and three booleans: `can_start`, `can_reject` and
`verdict_editable`. Drive the buttons from those rather than from `status`.

The payload carries **two different verdicts**, and they are not duplicates.
`data.claim.verdict` is the public finding, the same shape a marketplace reader gets.
`data.verdict` is her working copy: it has the record id, whether it can still be revised,
the documents behind it, and `submitted_at` as distinct from `published_at`.

**Starting the review.** No body.

```
POST /api/v1/fact_checker/assignments/7/start
```

`200`, and `data.status` is now `in_progress`. The claim itself does not move: it has been
`in_progress` since the admin assigned it.

**This is single use.** Starting an assignment that is already started answers
`422` and `This review has already been started.` A double tap or a stale screen posting
again is exactly how that happens, so gate on `can_start` and treat the `422` as the
backstop.

**Writing the verdict.** The verdict and its supporting documents travel together, in one
`multipart/form-data` request:

```bash
curl -X POST https://kasagadi.ai/api/v1/fact_checker/assignments/7/verdict \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -F "verdict[verdict]=misleading" \
  -F "verdict[verdict_summary]=The figure is real but out of date." \
  -F "verdict[research]=Checked against the 2024 budget statement." \
  -F "verdict[documents][]=@budget-statement.pdf"
```

`201 Created`. The assignment is now `completed` and the claim is in front of the owner admin
for approval. `data.verdict.published_at` stays `null`: nothing is live until an admin
approves, and that approval is also when the member is told.

`verdict`, `verdict_summary` and `research` are required; `reference_link` and the documents
are optional. `verdict` must be one of the five values from `/taxonomies`. Anything else comes
back as `{ "verdict": ["is not one of the five verdicts"] }` rather than as a server error,
which is why it is worth driving the picker from `/taxonomies` in the first place.

**One verdict per claim.** A second `POST` answers `422` and
`This claim already has a verdict. Revise it with PATCH instead.`

**Revising** is a `PATCH` on the same path with every part optional, so correcting a summary
need not resend the research she did not touch. Documents sent on a `PATCH` are **added** to
what is attached rather than swapped for it; taking one off is
`DELETE /fact_checker/assignments/7/verdict/documents/{id}`, using the id from
`data.verdict.supporting_documents[].id`. Nothing on the `PATCH` re-announces the verdict to
the admins: it is already in their queue, and only the first submission notifies.

Revision closes when an admin approves:

| Situation | `verdict.editable` | `PATCH` answers |
|---|---|---|
| No verdict written yet | there is no verdict object | `404` `No verdict has been submitted for this claim yet.` |
| Submitted, awaiting approval | `true` | `200` |
| Approved and published | `false` | `422` `This verdict has been approved and can no longer be edited.` |

Read `editable` before offering the edit screen. Once the finding is on the marketplace it
must not change underneath readers, and the same rule blocks pulling a supporting document
off it.

### 22.13 Akosua hands an assignment back

*The claim names an organisation Akosua works for. She cannot be the one to check it.*

```
POST /api/v1/fact_checker/assignments/7/reject
{ "rejection_reason": "I work for the organisation named in this claim." }
```

The reason sits at the top level, not inside an envelope. It is optional, but collect it: it
is what the admin reads when deciding who to hand the claim to next.

This is allowed from a review already in progress as well as from an untouched one, because a
conflict of interest is often only visible once you start reading. It is refused once a
verdict exists:

| Status | Message |
|---|---|
| `422` | `A verdict has already been submitted, so this assignment cannot be handed back.` |
| `422` | `This claim is no longer open for review, so it cannot be handed back.` |

**The `200` is the last look she gets at it.** Handing a claim back returns it to the admin
desk and takes it out of her scope entirely, so the very next `GET
/fact_checker/assignments/7` answers `404`, not `403`, and it is gone from her queue and from
every filter on it. Navigate back to the list on success. Do not refresh the detail screen,
and do not treat that `404` as a fault.

The claim goes back to `pending_assignment`, the assignment is marked `rejected`, and an audit
entry is written, all in one transaction: a half-applied rejection would leave a claim nobody
is working on and nobody has been told about. The member is deliberately **not** notified,
because from their side nothing has changed and the claim is still being worked on.

### 22.14 Kofi and Akosua open their notifications

*Kofi's claim went live overnight. Akosua has a new claim in her queue. Both open the app to
a badge on the bell.*

The feed is the same table the web bell reads, so a notification opened on the phone is read
on the web too. Both roles have one, and the endpoints are identical for each.

**On cold start, and whenever the app comes back to the foreground,** ask for the number
alone. Polling the whole feed to draw a badge is a page of rows paid for one integer.

```
GET /api/v1/me/notifications/unread_count
```

```json
{ "data": { "unread_count": 3 } }
```

**When the bell is tapped,** fetch the feed. Listing marks nothing read.

```
GET /api/v1/me/notifications?page=1&per_page=20
```

```json
{ "data": [
    { "id": 57, "type": "claim_assigned",
      "title": "New claim assigned to you",
      "body": "\"Free bus fares for students in September\" is now in your queue.",
      "read": false, "read_at": null, "created_at": "2026-08-29T14:48:20Z",
      "target": { "type": "assignment", "id": 294 } },
    { "id": 55, "type": "claim_in_review",
      "title": "Your claim has been assigned",
      "body": "A fact checker has been assigned to \"Free bus fares for students in September\".",
      "read": true, "read_at": "2026-08-29T07:12:04Z", "created_at": "2026-08-28T18:11:41Z",
      "target": { "type": "my_claim", "id": 345 } } ],
  "meta": { "filter": "all", "page": 1, "per_page": 20,
            "total_pages": 3, "total_count": 41, "unread_count": 3 } }
```

`meta.unread_count` is the whole feed, not this page, so a pull to refresh updates the badge
without a second request. `?filter=unread` narrows the list for a badge sheet; anything
unrecognised falls back to `all`, and `meta.filter` reports what was actually applied.

**Render `title` and `body` as they arrive.** Both are written for a person, already
localised, and already quote the claim title so the surrounding sentence reads. Do not build
copy from `type`: it is there to group and style rows, not to be spoken.

**Tapping a row does two independent things.** Record the read, and navigate. Neither waits
on the other, and the navigation comes from `target`, not from the response.

```
POST /api/v1/me/notifications/57/read
```

```json
{ "data": { "id": 57, "read": true, "read_at": "2026-08-29T15:02:11Z", "…": "…" },
  "meta": { "unread_count": 2 } }
```

The notification comes back so the row can be redrawn without a follow-up read, with the
fresh badge in `meta`. The call is idempotent: a second one leaves `read_at` where it was,
because the time a notification was first opened is the only thing that column is good for.
There is no `GET /me/notifications/:id`, and none is needed: the app already holds the row.

**`target` is how the app navigates.** It names the subject rather than a route, because a
web path names a screen in a browser and says nothing about which of the app's own screens
to push. Five types, each mapping to exactly one endpoint:

| `target` | Push | Fetch from |
|---|---|---|
| `{"type": "claim", "id": 259}` | the public fact check | `GET /claims/259` |
| `{"type": "my_claim", "id": 345}` | the member's own claim | `GET /me/claims/345` |
| `{"type": "my_claims"}` | the member's claim list | `GET /me/claims` |
| `{"type": "assignment", "id": 294}` | the assignment | `GET /fact_checker/assignments/294` |
| `{"type": "assignments"}` | the queue | `GET /fact_checker/assignments` |

Switch on `type`, and **treat an unrecognised one as no target**, the same as null. A new
notifier can ship without an app release, and an app that crashes on a `type` it has not met
would make every future notifier a coordinated deploy.

**A null `target` means there is nowhere to go.** Render that row as plain text rather than
as something tappable. It happens when the claim behind a notification has since been
deleted, in which case `body` is null too and only `title` survives. The row is still worth
showing: it is the last true thing left to say.

```json
{ "id": 47, "type": "claim_submitted", "title": "Claim received",
  "body": null, "read": true, "target": null }
```

**Clearing the badge** marks every unread notification in the caller's own feed:

```
POST /api/v1/me/notifications/read_all
```

```json
{ "data": { "read_count": 8, "unread_count": 0 } }
```

**A `404` on a read is somebody else's notification**, not a permission error. A feed is
scoped to its owner, and whose a notification is, is not the caller's business.

**A target can go stale.** A claim that was published when the notification fired can be
unpublished by the time it is opened, so `GET /claims/259` answers `404`. Handle it the way
every other stale screen is handled: return to the list rather than showing an error.

**Push notifications do not exist yet.** This is the feed the app reads while it is open.
APNs and FCM come later, and will not change these four endpoints.

### 22.15 Kwame the administrator tries the app

*Kwame runs the platform. He downloads the app and enters his working credentials.*

He gets `403` and "Administrators sign in on the web portal." His password was correct;
mobile simply is not for him. The app should show that message and link to the website,
not offer a retry.

This holds even if his account also has a member record, and it is re-checked on **every**
request, not just at sign-in.

### 22.16 What each role is refused, and why a missing record is a 404

A token belongs to a member or to a fact checker, and the two tiers do not overlap.

| Called by | `/claims`, `/me`, `/taxonomies` | `/me/dashboard`, `/me/claims/*` | `/fact_checker/*` |
|---|---|---|---|
| Member | `200` | `200` | `403` `This area is for fact checkers. Members use the member portal.` |
| Fact checker | `200` | `403` `This area is for members. Fact checkers use the fact checker portal.` | `200` |
| Administrator | `403` | `403` | `403` |

An account that somehow holds both roles counts as a **fact checker**, the same way `/me`
resolves the `role` it hands the app, so the member tier refuses it. Route on `role` from
`/me` and none of these ever appear in normal use.

The role check runs on **every** request, not only at sign-in, so an account whose role
changes mid-session starts being refused straight away.

**A record that is not yours, though, is `404` rather than `403`,** and that is deliberate:

- Another member's claim, and one an admin has unpublished: `404`.
- Another checker's assignment, and one this checker has already handed back: `404`.
- A supporting document belonging to somebody else's verdict: `404`.

A `403` would confirm the record exists, which is a fact the caller is not entitled to. Each
lookup is scoped to the caller's own collection, so a record they may not read is simply
absent from it. The web answers the same lookup with a redirect and an explanation, because a
stale link in an email is worth explaining to a person; the API does not have that luxury.

**So do not render "you do not have permission" for a `404` on one of these paths.** "That
claim is no longer available" is both truer and less alarming. The one place a wrong role
really is a `403` is the tier itself, and that says nothing about any particular record.

### 22.17 Kofi's account is suspended while he is using it

*A moderator suspends Kofi mid-session.*

His tokens stop working immediately: suspension destroys every session, so the next call
returns `401` and the refresh fails too. There is no grace period and no push notification.

**Design for it:** a `401` where the refresh also fails is not a bug and not necessarily an
expiry. Clear credentials, return to sign-in, and let his next sign-in attempt surface the
real reason (`403` with the suspension message).

The same applies if his role changes: an account promoted to administrator, or one that
loses its member record, stops working mid-session by design.

### 22.18 Kofi is on a bad network

*Kofi is on 3G in Tamale. Requests time out; he taps twice.*

- **Retrying `POST /auth/login`** is safe. It creates a second session rather than
  replacing the first, which is fine. Sessions are per device and cheap.
- **Retrying `POST /auth/refresh` with the same token is not.** If the first attempt
  reached the server, the token is spent and the retry returns `401`. Treat a timed-out
  refresh as unknown, not failed: retry once, and on `401` re-authenticate rather than
  looping.
- **Double-tapping "Send code"** is safe. Each request replaces the previous code, so only
  the most recent email works. If Kofi reads the older message, the code is rejected. It is
  worth saying "use the most recent code" on the entry screen.
- **Double-tapping "Register"** returns `422 has already been taken` on the second, not a
  crash. Treat it as "you already have an account" rather than a hard error.

**Rate limits** return `429`. Back off and show a plain message; do not retry in a tight
loop.

| Endpoint | Limit |
|---|---|
| `POST /auth/login` | 10 per 3 minutes, per email address |
| `POST /auth/register` | 5 per hour, per email address |
| `POST /auth/refresh` | 30 per minute, per refresh token |
| `POST /auth/password_resets` | 5 per hour, per email address |
| `PUT /auth/password_resets/:code` | 10 per 15 minutes, per email address |
| Everything else | 300 per minute, per signed-in user |

These are keyed by identity rather than by IP address on purpose: a large share of
Ghanaian mobile traffic leaves through a small number of carrier addresses, and an
IP-keyed limit would have users locking each other out.

### 22.19 Errors, in one shape

Every failure uses the same envelope, so one parser covers the whole API:

```json
{ "error": { "status": 422, "message": "Validation failed",
             "details": { "email_address": ["has already been taken"] } } }
```

`details` appears only on `422`, and maps attribute names to lists of messages. `message`
is written for a person and can be shown as-is.

| Status | Meaning | App behaviour |
|---|---|---|
| `400` | Malformed JSON body | A client bug. Log it; do not retry. |
| `401` | Missing, expired or invalid token | Refresh once, then sign out. |
| `403` | Valid credential, not allowed on mobile | Terminal. Show the message. |
| `404` | No such record | Terminal for that screen. |
| `422` | Validation failed | Render `details` against the form. |
| `429` | Rate limited | Back off, show a plain message. |
| `5xx` | Server fault | Retry with backoff; show a generic failure. |

### 22.20 Checklist before the first build ships

- [ ] Tokens are in the device keychain, never in plain preferences, and never logged.
- [ ] A single-flight refresh interceptor handles `401` and replays the request once.
- [ ] A refresh replaces **both** stored tokens; the old access token is never reused.
- [ ] `role` from `/me` drives navigation; `abilities` only shows and hides controls.
- [ ] Upload limits and vocabularies are read from `/config`, not hard-coded.
- [ ] Topic, region and verdict pickers are built from `/taxonomies`, sending `value` and
      showing `label`.
- [ ] `min_supported_app_version` is checked on cold start.
- [ ] The reset code is sent as `reset_code` in the body, not in the URL.
- [ ] Evidence and verdict uploads are a single `multipart/form-data` request, with
      `evidence[files][]` and `verdict[documents][]` repeated once per file.
- [ ] Every write response is used to redraw the screen, with no follow-up read.
- [ ] Edit and delete controls are hidden once a claim's `status` leaves `draft`.
- [ ] "Fact checked" is gated on `status == "published"`, not on a verdict existing.
- [ ] `can_start`, `can_reject` and `verdict_editable` drive the fact-checker buttons, with
      the `422` treated as the backstop for a stale screen.
- [ ] Rejecting an assignment navigates back to the queue; the detail screen is not
      refreshed, because it now answers `404`.
- [ ] `403` on sign-in is a dead end with the server's message, not a retry loop.
- [ ] `404` on someone else's record is shown as "no longer available", not as a permission
      error.
- [ ] The bell badge is drawn from `/me/notifications/unread_count`, not by fetching a page
      of the feed.
- [ ] Notification rows show `title` and `body` as they arrive; no copy is built from `type`.
- [ ] Tapping a row navigates from `target` and records the read separately, with neither
      waiting on the other.
- [ ] An unrecognised `target.type`, and a null `target`, both render as plain text rather
      than as a tappable row.
- [ ] `422` `details` are rendered per field, and `details.base` as a form-level message.
- [ ] Sign-out clears local credentials even when the request fails.
- [ ] Nothing in the app contains a `kg_live_…` partner key.
