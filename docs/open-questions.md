# Open questions — design review of the Inbox Sweeper docs

Started 2026-10-10. We are reviewing the design docs (product features, design
system, engineering principles, frontend stack, engine spike brief) one problem
at a time, before any engine code is written.

## Decided

### 1. Changelog: one file per entry
The single YAML file conflicts whenever two branches append an entry. Tested:
two appends at the end of the file conflict. Instead, each entry is its own file,
`changelog/<YYYYMMDDTHHMMSS>-<slug>.yml`, holding the same one-line entry
(date, plain-English description, commit). A build step gathers and sorts them
for the "What's new" modal. The timestamp naming matches the migration naming
rule, for the same reason.

### 3. Unsubscribe without sending mail (no SMTP)
The app never sends mail. Unsubscribe methods, in order:
1. **One-click (RFC 8058):** an HTTPS POST to the `List-Unsubscribe` URL when
   `List-Unsubscribe-Post: List-Unsubscribe=One-Click` is present. Bulk senders
   have had to support it for Gmail and Yahoo since 2024, so it should cover most
   newsletters. That is from memory and not measured on a real mailbox.
2. **Email-only lists:** hand the `mailto:` to the system mail client
   (self-hosted: the browser's mailto handler), pre-filled. The user sends it.
3. **Web link:** open it in the browser, as before.
4. **Block:** a rule, as before.

"Confirmed quiet / Still sending" verification watches incoming mail. No sending
needed. SMTP is not planned, not even later.

**Changes this makes to the engine plan (doc 05):**
- The indexer also fetches `List-Unsubscribe`, `List-Unsubscribe-Post` and
  `Authentication-Results`, alongside `List-Id`, and stores them so bundles know
  which methods they support.
- Unsubscribe is offered only when the sender passes DKIM aligned with its From
  domain (per `Authentication-Results`). Otherwise the app suggests Block.
  Unsubscribing from real spam only confirms the address is live.

**Changes to the features doc (doc 01):** remove "an email to the list's
unsubscribe address (sent from the user's account after confirming)" and replace
it with the mailto hand-off. Add the DKIM gate to the Unsubscribe section.

### 4. Authentication: no shipped OAuth secrets
- **Primary: app password** (Gmail requires 2-Step Verification). This is the
  path imapsync documents first for personal Gmail. It doesn't work for
  work/school accounts, Advanced Protection, or security-key-only 2SV.
- **Secondary: bring your own OAuth client.** The user creates a client in
  their own Google Cloud (or Azure) project and enters the client ID in
  Settings. Each user is "personal use", so there's no verification and no
  100-user cap.
- **The app never ships its own client ID or secret.** That rules out Google
  verification entirely.
- Re-auth must be a clear flow: Gmail-scoped tokens die on a password change.
- **The principle (confirmed 2026-10-10):** the app never depends on a
  registration we own. The reason isn't only Google's verification cost, so
  it applies to Microsoft too, even though Microsoft would let us ship an
  unverified public client.
- **Outlook.com is bring-your-own only.** Microsoft turned off Basic auth for
  personal accounts (outlook.com, hotmail, live, msn) on 16 Sept 2024, and app
  passwords are not a workaround (support.microsoft.com, "Outlook and other
  apps are unable to connect to Outlook.com when using Basic authentication").
  So Outlook users need their own Azure app registration. **Cost, accepted:**
  in practice the app is Gmail-first, and Outlook is for power users only.
  Access tokens last about an hour, so silent refresh must be routine (and
  Microsoft issues a new refresh token on every refresh).
- **No borrowed client IDs.** imapsync's `oauth2_imap` uses Thunderbird's
  client ID by default, for Gmail and Office365 alike. We will not do that,
  or anything like it: it means presenting as another app.
  (imapsync.lamiral.info/oauth2/oauth2_imap/README_oauth2.txt. Its Office365
  FAQ still says app passwords work for personal accounts; that text dates
  from Sept 2024 and contradicts Microsoft.)

Sources (fetched 2026-10-10):
- IMAP needs the restricted scope `https://mail.google.com/`
  (developers.google.com/workspace/gmail/imap/xoauth2-protocol).
- Tokens of apps in "Testing" status expire after 7 days
  (developers.google.com/identity/protocols/oauth2).
- App-password limits: support.google.com/accounts/answer/185833.
- imapsync's Gmail approach: imapsync.lamiral.info/FAQ.d/FAQ.Gmail.txt and
  README_oauth2.txt (its Gmail OAuth client is unverified: "forget it for now").

### 5. Backup scope: never fetch everything by default
Gmail caps IMAP downloads at 2,500 MB/day (imapsync FAQ), so full-content backup
must be opt-in and bounded.
- **Default backup:** metadata plus the message body for mails **under a size
  threshold** (user-configurable). Attachments are omitted and stored as
  placeholders: name, type, size and MIME part id, so they can be identified
  and fetched later while the message still exists.
- **Over-threshold mails** are skipped from content backup (metadata only).
- **Attachments are downloaded only on request,** per attachment.
- **The user can choose,** per action or as a setting, to also fetch:
  - full contents of over-threshold mails, and/or
  - attachments.
  The app shows an estimate. Large fetches throttle and may span days under
  Gmail's cap; show "day 1 of N".
- **Deleting without a full backup needs explicit acceptance.** If a selection
  includes messages whose content or attachments aren't backed up, the confirm
  sheet says so plainly, e.g. "214 messages will be deleted without a full
  backup: 12 over the size limit, 190 attachments not saved". The user must
  accept it, or choose to fetch first. Show the same warning again at Empty
  trash, because that is when it becomes irreversible.
- A restored message whose attachments were never saved is marked as such.

### 6. No in-app HTML reader: hand off instead
**Principle:** we don't build our own security mechanisms for hostile content.
Rendering is left to apps that already do it well.

Evidence that rendering in-app is risky:
- Tauri docs: "On Linux and Android, Tauri is unable to distinguish between
  requests from an embedded `<iframe>` and the window itself."
- CVE-2024-35222: iframes could reach Tauri IPC.
- CVE-2024-4367: pdf.js arbitrary JavaScript execution, which becomes native
  code execution in unsandboxed apps.

Decisions:
- **Full email:** "Open in Gmail" for Gmail accounts, via the documented
  `rfc822msgid:` search operator
  (`https://mail.google.com/mail/?authuser=<address>#search/rfc822msgid%3A<id>`;
  the URL shape is from memory, so test it once).
- **Other accounts:** export `.eml` and open it in the system mail app. If none
  is installed, that's the user's problem, not ours.
- **Opening in Gmail marks the mail read.** Accepted: reading a message is a
  signal not to bulk-delete its bundle, so the open-rate effect is fine.
- **Attachments** open in the system default app. The app never parses PDF,
  Office or image formats.
- **KeePass stays, with no automatic actions.** When an encrypted attachment
  has a matching KeePass entry, the app shows a small note: "A saved password
  for this file was found in your KeePass database." Next to it is a **Copy
  password** button. Nothing is written to the clipboard until the user clicks
  it. The clipboard clears ~30s after the copy, as KeePass itself does. Opening
  the file in the system viewer is a separate action, and the user pastes the
  password there themselves. No autofill and no chaining of steps on the user's
  behalf. The DB is unlocked once per session and never written back. No
  decrypted copy is written to disk.
- **Preview: keep it simple.** The app is a cleanup aid, not a mail client.
  Reading mail belongs in a mail client. The preview is a short peek:
  - fetch the start of the body with `BODY.PEEK` (doesn't mark it read)
  - convert HTML to text with `html-to-text`
  - show it as plain text (Svelte escaping; `{@html}` banned by the
    `svelte/no-at-html-tags` lint rule)
  - cap the input size
  No rendering, no extras. A separate rendering webview was considered and
  rejected: it would mean building our own security (sanitizer, CSP,
  capabilities, navigation hook) across three webview engines. The RFC 8970
  server preview is an optional speed-up, not required.
- **Doc 01 changes:** remove "full reader with its real HTML" and the in-app
  local viewer. Rename to "Open in Gmail / Open in mail app". Rewrite the
  attachment/KeePass paragraph to the clipboard flow.

### 7. Mascot art is a separate project
- The production pipeline (character sheet, references, 3D model, rig, render
  and export scripts) lives in a separate private project, not in this repo.
  This repo keeps the behaviour spec and the **asset contract** (doc 02, Mascot
  section) plus the exported files under `assets/mascot/`.
- **Sprite sheets, not video:** PNG strips animated with CSS `steps()`, with
  transparency for the peeks.
- **Themed at runtime:** frames use five grey levels, one per shade role, and
  the app maps them to theme tokens. One export serves every theme.
- **Open risk, being tested in the mascot project:** a smooth downscale
  creates in-between greys that runtime theming may colour wrongly. The grey
  levels or the downscale rule may change; the contract here is the source of
  truth.

## Open points

- **Licensing.** The app code is MIT. The mascot art (model, renders, sprites)
  is proprietary and copyrighted, all rights reserved, like Firefox's logo next
  to its open code. Needs:
  - an `assets/mascot/LICENSE` (the contract already reserves the spot);
  - a sentence in the root licence or README saying the MIT grant doesn't
    cover `assets/mascot/`.
  Low priority, to sort out before the first public release.

## To verify
- **Gmail identity:** `X-GM-MSGID` is confirmed by Google's docs
  (developers.google.com/gmail/imap/imap-extensions): a 64-bit ID, unique
  per message across folders, usable in FETCH and in SEARCH / UID SEARCH. It
  is the decimal form of the hex ID used by the Gmail web UI and the Gmail
  API. Still check it once in the engine spike (doc 05).
- **Open in Gmail by hex ID:** because of the above, decision 6 could link
  straight to the message with the hex form of `X-GM-MSGID` instead of the
  `rfc822msgid:` search, which avoids search misses on odd Message-IDs. Test
  both URL shapes in the spike and keep whichever works.

## Decided, on one condition: middleware-pipe 1.1.0

**Decision (2026-10-10):** proceed as if middleware-pipe 1.1.0 has landed.
(Planned as 2.0.0; shipping as a minor version instead, decided 2026-10-28. Depend on
`^1.1.0`: `^1.0.0` would still accept the old behaviour.)
The fix is expected before coding starts here. The action pipeline is built
on it, as proposed below.

**Gate:** at the start of the implementation phase, before any engine code,
check that 1.1.0 has both defects fixed. If it doesn't, stop and reopen this
item. Don't work around it. Doc 05 task 5 carries the same check.
- **Engine spike (loosened 2026-11-10):** a local build of 1.1.0 is enough.
  Install the tarball made by `npm pack` in the middlewares repo
  (`mt3o-middleware-pipe-1.1.0.tgz`), built from `main` with the fix merged.
  It passes lint and all 70 tests.
- **Anything past the spike** (a merge to `main` that others build on, or a
  release) needs 1.1.0 **published on npm**. Then swap the tarball dependency
  for `^1.1.0` from the registry.

### 2. Undo must be recorded before an action runs, not after (decided)
**Problem:** the engineering doc makes undo a bus subscriber that reacts after an
action. If it fails, lags, or the app dies mid-run, the mail has moved and
nothing can reverse it.

**Direction (decided):**
- Split into three mechanisms.
  - **Action pipeline** (`ActionPipeline`): Check → Prepare → Execute → Settle
    for one chunk, each awaited before the next.
  - **Action queue** (`ActionQueue`): schedules actions per account, splits
    them into chunks and runs the pipeline once per chunk. Split from the
    pipeline on 2026-10-30, so a hook author never touches scheduling and the
    job row keeps queue position apart from each chunk's last finished phase.
  - **Message bus**: notification only (UI refresh, stats, mascot), after
    Settle, never able to block or veto.
- Phases, and what a failure means in each:
  - **Check** (protections, spend cap, stale preview): veto.
  - **Prepare** (undo record, job marked started): abort.
  - **Execute** (the IMAP change, done by the pipeline's executor alone; no
    hooks may change the mailbox): stop, then resume.
  - **Settle** (index, bundle counts, activity log): retry, never roll back.
  - **Follow-up** (full backup, training): owed work, run in the background.
    Empty trash and account removal wait for it.
- The barrier is per action, not app-wide. One execution queue per account, so
  changes to one mailbox run in order and accounts don't block each other.
- Large jobs run in **chunks, size configurable** (default about 500). Check
  runs again before each chunk. A small job can slip in between chunks of a big
  one.
- Phase progress is saved to the job row in SQLite, so a crash resumes instead
  of losing the barrier.
- Every blocking hook has a timeout, and a timeout counts as failure.
- Undo is itself an action through the same pipeline.

**Implementation candidate:** `@mt3o/middleware-pipe` (`composeStack`, onion
model) for each chunk's Check → Prepare → Execute → Settle run. Hooks are added
through the registry, so the action code never knows undo exists.

**Blocked on:** the library fix handed off on 2026-10-10. Two defects to fix
first:
- Reaching the end of the chain quietly returns `{}`, so a missing executor
  "succeeds".
- `composeGenStack` loses errors.

Handoff doc: https://claude.ai/code/artifact/53296a6d-c421-4b68-a371-b805f96ce79f
Checked 2026-10-10: still 1.0.0, locally and on npm.
Checked 2026-10-28: fix is in PR https://github.com/mt3o/middlewares/pull/1
(open, version 1.1.0); npm still has 1.0.0 only.
Checked 2026-11-10: PR #1 merged to `main`, version 1.1.0, built locally
(lint clean, 70/70 tests). Not tagged, not on npm yet.

## Still to review

Nothing left, apart from licensing before the first public release. Item 2's
gate is checked when implementation starts.

## Done on disk (2026-10-10)
- Docs 01–05 are in `docs/`. The mascot-pipeline handoff and the filter
  lessons were left out.
- **Problem 7 fixed:** "Message identity & sync strategy" and "Dedup gate" are
  top-level sections again in doc 03.
- **Doc 03 now has "Actions: a phased pipeline, not the message bus"**
  (Check → Prepare → Execute → Settle, follow-up, per-account queue,
  configurable chunks, saved progress, timeouts). The bus section is cut down
  to notifications only. Item 2's implementation choice (middleware-pipe)
  stays open.
- **Decisions 1, 3, 4, 5 and 6 applied to the doc text:**
  - **Doc 01:** changelog files, unsubscribe without sending plus the DKIM
    gate, app password or own OAuth client, bounded backup with explicit
    acceptance, plain-text peek, Open in Gmail / mail app, attachments in
    system apps, KeePass Copy button, chunked jobs.
  - **Doc 02:** component table.
  - **Doc 03:** credentials, "don't render untrusted content", text-only mail
    content plus the lint rule, bounded backup, `X-GM-MSGID` marked to verify.
  - **Doc 05:** extra headers, app-password smoke test, Gmail checks, peek
    fetch, phased action-queue stub.
- **Item 2's implementation choice is in doc 05 task 5:** middleware-pipe
  `^1.1.0`, behind the 1.1.0 gate (was 2.x / 2.0.0 until 2026-10-28).
- **Tool names in the docs are fine** (e.g. the 3D tool in doc 02's mascot
  pipeline). What must not appear is how our own tooling is set up: keys,
  hosts, integrations. Doc 02's "Reference images" line was reworded to
  "AI-generated concept art, kept outside this repo".
