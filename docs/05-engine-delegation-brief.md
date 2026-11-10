# Delegation: Inbox Sweeper engine spike — Bun + ImapFlow

Oct 10, 2026 · @Teodor

**For:** Claude Code. **Goal:** stand up the core engine for Inbox Sweeper — a local-first, any-IMAP bulk inbox cleaner — on **Bun**, proving the IMAP indexing and SQLite storage path end to end before the UI is built. **Done when:** the engine connects to a real IMAP account, indexes messages into SQLite, groups them into sender/domain/subject bundles, and a smoke test confirms ImapFlow works under Bun.

## Runtime and delivery decisions (already settled)

- **Runtime:** Bun, not plain Node — faster startup, built-in `bun:sqlite`, `bun build --compile` gives a single self-contained binary.
- **Engine:** one TypeScript service holding the IMAP indexer, the naive-Bayes classifier, and the bulk-action queue. Runs identically inside the desktop app and in self-hosted mode.
- **Desktop shell:** Tauri (native window chrome), the Bun engine as a compiled sidecar binary.
- **Self-hosted mode:** the same engine headless behind a thin HTTP layer, same web frontend served in a browser.
- **Scope of this spike:** the engine only — no UI, no Tauri shell yet.

## IMAP library: ImapFlow

- Use **ImapFlow** (`imapflow`). Its ES module build officially runs on Bun, tested against the latest release. Modern Promise-based API, full Gmail label support (we treat Gmail labels as folders), mailbox locking for safe concurrent access.
- **Known Bun caveat to handle:** STARTTLS fails on Bun when the host is a bare IP literal — Bun throws on `servername: false`, which Node tolerates. Workaround: pass `servername: 'localhost'` (or omit it). Real accounts connect by hostname with implicit TLS on port 993, so this is unlikely to bite in practice, but the smoke test should cover it.

## Tasks

1. **Smoke test first.** Minimal Bun script using ImapFlow against a real IMAP account (hostname, implicit TLS, port 993): connect, list folders, fetch a handful of envelopes. Sign in with an **app password** (the app's default; OAuth is only via the user's own client ID, later). Confirm it runs clean under the latest Bun. Note the version used.
   - **Gmail checks, record the results:** does the server advertise `X-GM-EXT-1`, and does `X-GM-MSGID` stay the same for one message seen under two labels? If yes, it becomes the Gmail identity key (doc 03, Message identity). Also check whether partial body fetch with `BODY.PEEK[...]<0.4096>` works through ImapFlow, for the preview peek, and that it leaves `\Seen` unset.
2. **SQLite schema** via `bun:sqlite` (Kysely, timestamp-named migrations, per doc 03): a messages table (uid + uidvalidity, folder, Message-Id, Gmail `X-GM-MSGID` when available, from, subject, date, size, flags, and the headers `List-Id`, `List-Unsubscribe`, `List-Unsubscribe-Post` and `Authentication-Results`), indexed for grouping by sender / domain / subject / size / age. Store the unsubscribe headers so a bundle knows which methods it offers (one-click, web link, mailto), and the DKIM result so unsubscribe can be offered only to authenticated senders. Design for incremental re-index, not full rescans.
3. **Indexer:** connect, walk folders, fetch envelopes + the headers listed above (not full bodies), write to SQLite. Handle Gmail labels as folders — a message with multiple labels appears under each. Headers only keeps indexing far below Gmail's ~2,500 MB/day IMAP download cap; never fetch bodies or attachments here.
4. **Bundles query:** build the sender / domain / subject / list / size / age groupings from SQLite, each with count, total size, unread share.
5. **Leave stubs** for the naive-Bayes classifier and the bulk-action queue — interfaces only, so the UI work can start against them later. The action side is two stubs, as defined in doc 03 ("Actions"): `ActionQueue` (one queue per account, configurable chunk size, progress saved to the job row, resume after a crash) and `ActionPipeline` (one chunk through Check → Prepare → Execute → Settle, hooks registered by name, reports each finished phase and which phase failed). Callers only see `ActionQueue`; only `ActionQueue` calls `ActionPipeline`. Build the pipeline on `@mt3o/middleware-pipe` `^1.1.0` (`composeStack`, one run per chunk); hooks such as undo register through its registry. The queue doesn't import middleware-pipe. **Precondition, check before writing any engine code:** 1.1.0 fixes both defects (reaching the end of the chain must fail, not return `{}`; `composeGenStack` must not lose errors). For this spike a local build is enough: if npm doesn't have 1.1.0 yet, install the `npm pack` tarball from the middlewares repo (`mt3o-middleware-pipe-1.1.0.tgz`). Note in the deliverables that the dependency is a local tarball, since it must move to the published `^1.1.0` before anything past the spike. If neither the published package nor a local 1.1.0 build has the fixes, stop and report back instead of working around it.

## Deliverables back to me

- The smoke-test result and the Bun version it passed on.
- The SQLite schema and the bundle queries.
- Any ImapFlow-on-Bun rough edges beyond the STARTTLS one.
- A note on whether `bun build --compile` produces a working binary with the IMAP + SQLite deps bundled.

## Model tiering (how I plan work)

- [ ] Smoke test + schema design — **Sonnet** (moderate, some judgement).
- [ ] Boilerplate indexer wiring and the stub interfaces — **Haiku** (simple, low-impact).
- [ ] Incremental re-index strategy and the bundle query design — **Opus** (the decisions that shape everything downstream).

## Open, for later (not this spike)

- Naive-Bayes learner, Ollama/Haiku escalation path, the bulk-action queue and undo/backup layer, the Tauri shell, the web frontend.
- Unsubscribe execution (one-click HTTPS request, mailto hand-off), the preview peek, and the size-limited backup. The spike only stores what they need.
- OAuth via the user's own client ID.
