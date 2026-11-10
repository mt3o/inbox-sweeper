# Inbox Sweeper — engineering principles & standards

Oct 10, 2026 · @Teodor

The principles this codebase follows. The guiding frame is **hexagonal architecture**: a pure core engine surrounded by adapters, so the same engine runs behind the Tauri desktop shell and the headless self-hosted HTTP layer without change.

## Architecture: hexagonal (ports & adapters)

The core engine is pure domain logic and depends on nothing outside itself. Everything external reaches it through a **port** (an interface the core defines) and is supplied by an **adapter** (an implementation the core never imports).

- **The core owns the interfaces.** The engine declares ports like `MailStore`, `MessageSource`, `Classifier`, `ActionQueue`. It calls those interfaces, never a concrete library.
- **Adapters depend inward, never the reverse.** ImapFlow is an adapter behind `MessageSource`; `bun:sqlite` is an adapter behind `MailStore`; the HTTP layer and the Tauri sidecar are *driving* adapters that call into the core. The core imports none of them.
- **Why it matters here:** the same engine runs behind the Tauri desktop shell and the headless self-hosted HTTP layer unchanged. Swapping IMAP library, database, or transport touches one adapter, not the domain.
- **Testability falls out of it.** The core is tested against in-memory fakes of its ports — no live IMAP server, no disk — so bundle logic and classification are fast, deterministic unit tests.
- **Rule of thumb:** if a file imports `imapflow`, `bun:sqlite`, or anything about HTTP or windows, it is an adapter and lives at the edge. If it reasons about messages, bundles, or rules, it is core and imports only other core.

## SOLID, applied to this engine

- **Single responsibility** — the indexer indexes, the classifier classifies, the action queue schedules actions and the action pipeline runs them. A bundle-grouping change never touches IMAP fetch code. Each module has one reason to change.
- **Open/closed** — new classification backends (local naive Bayes, Ollama, Claude Haiku) are added as new `Classifier` implementations, not by editing an existing one. The escalation logic composes them; it doesn't grow a switch statement per backend.
- **Liskov substitution** — any `MailStore` or `MessageSource` adapter is interchangeable. The in-memory test fake and the real `bun:sqlite` / ImapFlow adapters honour the same contract, including error and empty-result behaviour, so the core can't tell them apart.
- **Interface segregation** — keep ports narrow. A read-only bundle view depends on a `BundleQuery` port, not on the full `MailStore` with its write and delete methods. Consumers see only what they use.
- **Dependency inversion** — the core defines the abstractions; adapters implement them. Concrete libraries are injected at the composition root (the Tauri sidecar entry, or the HTTP server bootstrap), never reached for inside the domain.

## Local-first & data

- **Nothing leaves the machine by default.** All indexing, grouping and the default classifier run locally. Data crosses the network only when the user explicitly enables a cloud engine (Ollama on another host, or Claude Haiku via their own API key).
- **When cloud is on, show exactly what is sent.** The classification guardrails spell out the payload — sender, subject, first 500 characters — with redaction and a spend cap. No silent exfiltration.
- **The local index is a cache, not the source of truth.** The IMAP account is authoritative; the SQLite index can be rebuilt from it at any time. Design every store operation to survive a wipe-and-reindex.
- **Credentials stay local and least-exposed.** IMAP credentials and any API keys live on the machine; optional KeePass lookups unlock once per session and are never written back, and a password reaches the clipboard only when the user clicks Copy. The app ships no OAuth client ID or secret, and borrows none: sign-in is an app password, or the user's own OAuth client (the only option for Outlook.com). Microsoft access tokens last about an hour, so refresh is routine and silent.
- **Don't render untrusted content; hand it off.** The app never renders email HTML or opens attachment formats itself. Full messages open in Gmail or the system mail client; attachments open in the system's default app. The in-app preview is plain text extracted from the body. We don't build our own sandbox for hostile content: apps that specialise in it do it better.
- **Own the data format.** The index schema and any local backup archive are documented and restorable without the app — the user is never locked in.

## Safety & reversibility

- **Preview before acting.** Every bulk action shows a confirm sheet with the exact count and any warnings before anything is touched. No action runs blind.
- **Undo is the default.** A 10-second undo toast follows every action; the Activity log lets the user reverse earlier actions. Removed mail can be backed up (disk, S3, SFTP, WebDAV) so a single message restores without unpacking the whole archive. Backup is bounded: by default bodies under a user-set size limit, attachments as placeholders, everything else only on request, throttled to provider download caps. Deleting anything not fully backed up requires the user's explicit acceptance.
- **Permanent deletion is deliberate.** Irreversible deletes require typing the message count — friction proportional to the risk.
- **Fail safe, not silent.** An interrupted bulk run leaves a consistent state and a resumable queue; a partial action is logged, never half-applied with no record.
- **The destructive path is the exception, routed through one place.** All mutations go through the `ActionQueue` port, so preview, logging, undo and backup are enforced in one spot rather than sprinkled across callers.

## Code & testing standards

- **TypeScript strict, everywhere.** `strict: true`, no implicit `any`, no non-null assertions to paper over gaps. Ports are typed interfaces; adapters satisfy them at compile time.
- **Domain types, not primitives.** Model `Uid`, `FolderPath`, `BundleKey` as distinct types rather than passing bare strings and numbers around the core.
- **Test the core against fakes, the adapters against reality.** Fast deterministic unit tests for bundle and classification logic using in-memory port fakes; a smaller set of integration tests exercises the ImapFlow and `bun:sqlite` adapters against real targets.
- **Use Bun's native tooling.** `bun test` as the runner, `bun:sqlite` for storage, `bun build --compile` for the sidecar binary — fewer dependencies, one toolchain.
- **Errors are values at the edge.** Adapters translate library and network failures into the domain's error shapes; the core never catches an ImapFlow-specific exception.
- **Keep the composition root thin.** Wiring (which adapter implements which port) lives only at the entry points — Tauri sidecar and HTTP bootstrap — and nowhere else.
- **Mail content is always text in the UI.** Svelte escapes `{text}` by default; raw HTML via `{@html}` is banned with the `svelte/no-at-html-tags` lint rule at error level. HTML bodies become text in the engine (`html-to-text`, input size capped) before the UI sees them.
- **Build a Storybook for the UI.** Every reusable UI element gets a story, across the Green, Amber and Paper themes. It doubles as the living design-system reference and lets components be developed in isolation from the engine.
- **Regression-test components in the Storybook.** Visual/snapshot regression tests run against the stories so theme and layout changes to any element are caught before they ship.
- **Playwright for end-to-end integration tests.** Drive the running application through the UI to exercise whole flows — scan, bundle, bulk action, undo — not just units.
- **Back those runs with an in-memory IMAP fixture.** Provide a test adapter implementing the `MessageSource` (IMAP) port that serves a seeded in-memory mailbox, so Playwright runs are fast, deterministic and need no live server. Because it honours the same port contract as the ImapFlow adapter, the app under test can't tell them apart.
- **Wrap the Playwright run in a helper script.** One command that starts the app with the in-memory IMAP fixture, seeds the mailbox, runs the suite, then tears everything down and cleans up — so the whole cycle is repeatable and identical locally and in CI, with no manual setup or leftover state.

## Engineering log (for the LLM)

After every implementation or bug-fixing session, write notes before finishing. The log is a durable record we review from time to time, and the raw material for tooling that makes future work easier.

- **Record what was done and concluded.** A short entry per session: what was built or fixed, and the conclusions reached.
- **Capture the hard parts.** Difficult steps that had to be resolved, the tensions and dead ends hit along the way, and how they were worked around — so the same wall isn't hit twice.
- **Note observations worth keeping.** Anything learned about the codebase, the tools, or the IMAP/Bun edges that future sessions should know.
- **Promote single-use scripts into repeatable tools.** When a one-off script or command proves useful, flag it in the log; periodically turn those into committed helper tools so the work carries over between coding sessions instead of being re-derived.
- **Review periodically.** We read back through the log every so often to spot recurring friction and decide what to automate next.

## Data access & migrations

The index is SQLite via `bun:sqlite`. Queries are built with **Kysely** used as a typed query builder, and migrations run through Kysely's own migration runner so the schema has a single owner and the query types stay in sync with it.

- **Queries live behind named methods on named repository classes.** Kysely never leaks past the data layer. Each query is a method — `BundleRepository.bundlesBySender()`, `MessageRepository.byUid()` — so every query has exactly one home, the whole data-access surface reads off the method names, and index tuning for a bundle query has a single place to go. These repository classes are the `bun:sqlite` adapter implementing the `MailStore` and `BundleQuery` ports; the core calls the interface and never knows Kysely exists.
- **Kysely is a query builder, not an ORM.** Chosen deliberately over a full ORM (Prisma, TypeORM) because the performance-sensitive work is analytical `GROUP BY` aggregation over millions of rows — counts, sizes, unread share by sender/domain/list-id — where you want control of exactly which index is hit. Kysely keeps that essentially SQL while adding type safety and a migration runner.
- **Migrations are append-only and never edited once written.** A written migration is immutable; corrections are new migrations. This is what makes the from-zero test path trustworthy.
- **Migration names are full timestamps, not sequential integers.** Name each `YYYYMMDDTHHMMSS_description` (e.g. `20261010T143052_add_listid_column`). Two sessions can never mint the same identifier, and the names still sort chronologically — the exact order they must apply in. Sequential integers would race.
- **Migrate in place; never wipe-and-reindex as the upgrade path.** A large inbox (100 GB) makes rebuilding the cache from IMAP prohibitively slow, so schema changes preserve the expensive tier — the data fetched over IMAP (envelopes, headers, list-id, sizes). Add columns and backfill from what's already stored, build indexes in place. Wipe-and-reindex stays a last-resort recovery path, not a migration.
- **Two tiers migrate differently.** The *expensive tier* (IMAP-fetched) must be migrated in place. The *cheap tier* (locally derived — bundle groupings, classifier state, aggregates) can be dropped and recomputed without touching IMAP, which is often simpler than migrating it.
- **Only one process migrates at a time.** Migrations run on startup before the engine serves. Each runs in a transaction; SQLite's write lock means a second process blocks rather than corrupting. Make it robust by re-checking the schema version inside the transaction and taking an explicit application-level startup lock, so concurrent engine processes wait rather than racing.
- **Prove it from zero on every run.** The Playwright helper spins up a fresh database and runs every migration from empty, so a collision or broken step fails CI immediately rather than on a user's machine.

## Configuration

- **Use the `config-layers` library for all configuration.** Configuration is assembled from a series of config objects (layers), each filling in the properties it knows, passed to the library in priority order. A lookup reads from the highest-priority layer first and falls back to the next layer down when a key is absent, so defaults, file config, environment and user overrides stack cleanly without hand-rolled merging. This is a fixed requirement — the engine does not read config any other way.
- **Why it fits here.** The app already has naturally layered settings — built-in defaults, the user's saved settings, per-account overrides, and (in self-hosted mode) environment variables. Layered priority with fallback expresses exactly that, and keeps config resolution in one well-understood place rather than scattered conditionals.

## Actions: a phased pipeline, not the message bus

- **Action** — a mailbox change the user has confirmed (archive, delete, move, …), recorded as a job before anything runs, so it can be previewed, logged, resumed and undone. Undo is itself an action.
- **Action queue** (`ActionQueue`) — the port through which every mailbox change goes; callers only ever enqueue actions here. It decides *what runs when*: one ordered queue per account, large actions split into chunks, small actions slipped in between chunks, progress saved, resume after a crash. It hands each chunk to the action pipeline and never touches the mailbox itself.
- **Action pipeline** (`ActionPipeline`) — runs *one chunk* through Check → Prepare → Execute → Settle, with the hooks registered for each phase. It is reachable only from the action queue, so the queue stays the single way in. Its Execute step is the only code that changes the mailbox.

Anything that changes the mailbox is scheduled by the `ActionQueue` and run, one chunk at a time, by the `ActionPipeline` as a series of awaited phases. The message bus only announces what already happened. A bus can't wait for a listener, let one cancel an action, or survive a crash, and undo needs all three: an undo record written after the fact can be lost while the mail has already moved.

**The pipeline: one chunk, four phases.**

- **Four phases around one executor, each finished before the next starts.** A phase exists only because a failure in it means something different:
    - **Check** — protections, spend cap, "is the preview still accurate?". A failure vetoes the action; nothing has happened yet.
    - **Prepare** — the undo record (folder, flags, labels, identity key) and the job marked as started. A failure aborts; the mailbox is untouched.
    - **Execute** — the IMAP change itself, done by the pipeline's executor alone. Hooks never change the mailbox, so the destructive path stays in one place.
    - **Settle** — local index, bundle counts, activity log. A failure is retried, never rolled back: the mailbox is already right.
- **New behaviour is a new hook, not a change to the action code.** Undo is a Prepare hook; the action code never knows it exists. Hooks are registered by name, so a feature plugs in without editing the pipeline.
- **Every blocking hook has a timeout.** In Check and Prepare a timeout counts as a failure, so the action is cancelled — the safe direction.
- **The pipeline reports, the queue decides.** The pipeline reports each finished phase and, on failure, which phase failed. What happens next (cancel, resume, retry) is the queue's call.

**The queue: what runs when.**

- **The barrier is per action, not app-wide.** Each account has its own execution queue: changes to one mailbox run in order, and accounts don't block each other.
- **Large jobs run in chunks, and chunk size is configurable.** Each chunk is one pipeline run, so it goes through every phase and Check runs again before each chunk: messages a rule already moved are skipped, not acted on from a stale preview. A small job can slip in between chunks of a large one.
- **Progress is saved to the job's row in SQLite.** The row keeps the job's place in the queue apart from each chunk's last finished phase. After a crash, the queue resumes the job from there instead of losing the barrier.
- **Follow-up work runs in the background and is tracked as owed.** Full-content backup, classifier training and rule suggestions don't hold up the user. Empty trash and account removal wait until the owed work for those messages is done.

**Both.**

- **Undo is itself an action**: queued like any other and run through the same pipeline; its Check asks whether the messages can still be recovered.
- **Open:** the pipeline is expected to use `@mt3o/middleware-pipe` (`composeStack`, onion model, one run per chunk), pending its 1.1.0 fix. See `open-questions.md`, item 2. The queue doesn't depend on it.

## Internal communication: message bus

- **The bus carries notifications only.** After Settle, the pipeline announces what happened ("1,812 messages archived"). The UI, stats and mascot react. A listener can't delay or cancel anything, and a failing listener affects only itself.
- **Components announce, they don't call each other for reactions.** The indexer, classifier and queue post events without knowing or caring who reacts.
- **It fits the ports model.** The bus is an adapter behind a port; the core emits domain events and depends on no concrete bus implementation.

## Message identity & sync strategy

A message must be recognised as the same message across re-indexes and across servers, or bundles double-count and undo restores to the wrong place. IMAP makes this harder than it looks, so identity is layered, and the local cache — not the server — is the arbiter.

- **UID + UIDVALIDITY is the primary key.** A message is keyed by its IMAP UID together with the folder's UIDVALIDITY value. UIDs are unique and stable *within* a folder only as long as UIDVALIDITY is unchanged; when a server resets UIDVALIDITY, the old UIDs are void and the folder is re-anchored. Always store the pair, never a bare UID.
- **Gmail: `X-GM-MSGID`.** Google documents it (Gmail IMAP extension `X-GM-EXT-1`) as a 64-bit ID that identifies a message across all folders, usable in FETCH and SEARCH; it is the decimal form of the hex ID in the Gmail web UI. The engine spike checks it once on a real mailbox. It is the identity key for Gmail accounts, ahead of Message-Id, and what makes "delete removes it everywhere" safe with labels as folders.
- **Message-Id is the fallback for re-anchoring.** When UIDVALIDITY changes, or a message moves between folders or servers, the `Message-Id` header re-identifies it. It is a fallback, not the primary key: some servers rewrite `Message-Id`, so it can collide or go missing, and it must never be trusted as the sole identity.
- **A content checksum is a soft dedup signal, never a hard key.** A canonical checksum over normalised `Date` / `From` / `To` / `Subject` / `Message-Id` plus the decoded body (excluding `Received` lines, flags and raw MIME framing) helps spot duplicates and re-anchor when both UID and `Message-Id` fail. It is only ever a signal — it informs a dedup decision, it never by itself keys or destroys a message.
- **The local cache is the dedup source of truth.** Identity resolution reads the local index, not the server, so re-runs are idempotent and a flapping server can't manufacture duplicates. This mirrors what imapsync learned the hard way: `--useuid --nousecache` produces duplicates across runs, so a cache is mandatory, not optional.
- **Flags are always resynced.** On each pass, message flags (seen, flagged, answered, etc.) are re-read and reconciled from the server, which stays authoritative for flag state. We never assume cached flags are current.
- **CONDSTORE / QRESYNC are opportunistic, never assumed.** When the server advertises them, use them to fetch only changed messages since the last known modseq — a real speed-up on large mailboxes. When it doesn't, fall back to re-scanning against the cache. The design never *requires* these extensions to exist.

## Dedup gate: cuckoo filter in front of SQLite

Deduplication checks "have we already seen this message identity?" on a hot path over millions of rows. A cuckoo filter sits in front of the exact store as a cheap in-memory gate, but it is never allowed to make a destructive decision.

- **The filter is a fast negative gate only.** A cuckoo filter answers "probably seen" or "definitely not seen" from memory, with no disk hit. "Definitely not seen" is exact — it skips the SQLite lookup entirely, which is where the speed comes from. "Probably seen" is where the filter can be wrong (a false positive), so it is only ever a hint to go check.
- **SQLite is always authoritative.** Every "probably seen" is confirmed against the index before anything acts on it. The filter narrows the work; the exact store makes the call. A false positive costs one wasted lookup — never a wrong outcome.
- **The filter never decides a destructive action.** No message is deleted, merged or skipped on the filter's word alone. Cheap gate, authoritative store, exact decision — in that order, always.
- **Cuckoo over Bloom because dedup needs deletion.** As messages leave the cache the filter must forget them, and a plain Bloom filter can't delete without injecting false negatives — fatal for a gate that must never wrongly say "not seen". A cuckoo filter removes a whole fingerprint cleanly, keeps two cache-local probes, and is smaller than Bloom below roughly 3% false-positive rate. (A quotient filter is the alternative if cache locality ever dominates; it's fiddlier to implement.)
