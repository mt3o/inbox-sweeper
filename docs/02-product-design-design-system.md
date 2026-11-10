# Inbox Sweeper — Design System

Oct 10, 2026 · @Teodor

A Fallout-style terminal look for a power tool, built on framework-agnostic tokens so the tech stack stays open.

## Direction

A Fallout Pip-Boy terminal: phosphor text glowing on near-black, boxy framed panels, monospace type, a faint CRT scanline wash. The theme is the skin; underneath it is a dense, fast, keyboard-driven tool.

Principles, in priority order:

1. **Scanning beats beauty.** Long lists must be read fast. The CRT glow and scanlines never reduce contrast or legibility; if they fight the data, the data wins.
2. **Theme is a token set, not hard-coded.** Every color, glow and texture is a variable. Green and amber are swappable accents; the structure stays identical.
3. **Keyboard first.** Every control has a visible shortcut and a clear focus state, styled like a terminal cursor.
4. **Honest states.** Destructive actions, protected mail and running jobs read differently at a glance, by color and frame, not by a buried label.
5. **Framework-agnostic.** Tokens as CSS custom properties; components defined by behavior and structure, so the tech stack is free to choose later.

## Color tokens

Three themes share one token set: Green (default), Amber, and Paper (light, non-terminal). Values are the default green theme; the amber theme overrides only the accent ramp.

| Token | Green (default) | Role |
| --- | --- | --- |
| `--bg` | #0A0F0A | App background, near-black |
| `--bg-panel` | #0E160E | Panel and table background |
| `--bg-raised` | #142014 | Selected row, hover, input |
| `--border` | #1F3A1F | Panel frames, grid lines |
| `--text` | #33FF66 | Primary phosphor text |
| `--text-dim` | #1F9A3D | Secondary text, metadata |
| `--text-faint` | #146628 | Disabled, placeholders |
| `--accent` | #39FF88 | Focus, active, primary action |
| `--glow` | rgba(51,255,102,0.35) | Text-shadow / box glow |
| `--good` | #39FF88 | Confirmed, safe, kept |
| `--warn` | #FFD24A | Protected mail, caution |
| `--danger` | #FF5C4A | Delete, block, destructive |
| `--info` | #4AC3FF | Running jobs, neutral notices |

**Amber theme** overrides: `--text` #FFB642, `--text-dim` #B37A1E, `--accent` #FFC861, `--glow` amber. Base, border and status stay, except `--good` shifts toward the accent so "kept" reads warm.

**Paper theme** is white and dark gray with no glow or scanlines. It doubles as the high-contrast option for long sessions. Overrides: `--bg` #FFFFFF, `--bg-panel` #F5F5F5, `--bg-raised` #EAEAEA, `--border` #C8C8C8, `--text` #1E1E1E, `--text-dim` #555555, `--text-faint` #8A8A8A, `--accent` #2D7A3E, `--glow` none; status colors darken for contrast on white: `--good` #1E7A35, `--warn` #9A6A00, `--danger` #C0392B, `--info` #1F6FA8.

**Rules:** text on any background holds at least WCAG AA contrast with the glow off; status colors never carry meaning alone, always with an icon or frame; the accent is used sparingly so focus actually stands out.

## Typography

Monospace everywhere; it reinforces the terminal and keeps columns aligned, which helps scanning.

- **Family:** a legible mono stack — JetBrains Mono or IBM Plex Mono, falling back to ui-monospace. One family, no serif or display face.
- **Scale (compact base 13px):** 11 caption, 13 body, 15 strong, 18 panel title, 24 screen title. Line height 1.4 for lists, 1.5 for prose.
- **Weights:** regular 400 and bold 600 only. Bold marks headers, sender names and counts; never italics (poor in mono, weak on CRT).
- **Case:** panel titles and labels in uppercase with slight letter-spacing, terminal-style; data stays normal case so addresses and subjects read true.
- **Numbers:** tabular figures so counts and sizes line up in columns.

## Spacing, grid and texture

- **Spacing scale:** 4, 8, 12, 16, 24, 32. Compact is the default; dense table rows are 28px tall, comfortable mode bumps to 36px.
- **Grid:** a fixed left rail (navigation, 220px), a main column, and an optional right panel (preview or detail, 380px). Panels are framed boxes with a 1px border and an uppercase title bar.
- **Radius:** 2px. Nearly square — CRT hardware, not rounded modern cards.
- **Borders:** 1px solid `--border`; active or focused panels switch the frame to `--accent` with glow. Double-line ASCII-style frames are an option for primary panels.
- **Elevation:** no drop shadows. Depth is shown by border brightness and a faint inner glow, not blur.

**CRT effects (on by default in Green and Amber, switchable in Settings, never used in Paper):**

- **Scanlines:** a subtle horizontal line overlay at low opacity.
- **Text glow:** `text-shadow` from `--glow` on primary text; reduced automatically in comfortable/high-contrast mode.
- **Flicker and vignette:** a barely-there flicker and screen-edge darkening. Respects `prefers-reduced-motion` — off means fully off.

A single **Reduce effects** switch strips scanlines, glow and flicker for maximum legibility while keeping the color theme.

## Components

A small set of terminal-styled components covers all nine screens from the Features tab.

| Component | What it is | Used on |
| --- | --- | --- |
| Panel | Framed box, uppercase title bar, optional double border | Everywhere |
| Data table | Dense rows, tabular numbers, sortable headers, checkbox + shift-select, sparkline cell | Bundles, Activity, Unsubscribe, Rules |
| Bundle row | Name, count, size, unread, open-rate, sparkline, suggestion badge, confidence dot | Bundles |
| Suggestion card | Framed callout with a headline count and one action button | Dashboard |
| Stat tile | Big tabular number + label, terminal readout style | Dashboard, Activity |
| Chart | Scanline-styled bars/lines in accent color, monospace axes | Dashboard |
| Toolbar / filter bar | Inline inputs, dropdowns, chips, each with a key hint | Bundles, Activity |
| Confirm sheet | Modal panel: counts, sample subjects, warnings (incl. "not fully backed up", which must be accepted), confirm/cancel | All bulk actions |
| Toast | Bottom readout line with an Undo countdown | After actions |
| Message preview | Headers and a plain-text peek of the body (no HTML rendering), Open in Gmail / Open in mail app buttons, attachment list | Bundle detail |
| Attachment item | Name, type, size, Open button (system app); lock icon when encrypted; when KeePass has a match, a small note and a Copy password button | Bundle detail |
| Train card | Big Keep / Clean buttons, category picker, "Why?" popover | Train |
| Confidence dot | Solid = confident, hollow = unsure, color = Keep/Clean | Bundles, messages |
| Rule editor | Condition rows (AND/OR), action picker, dry-run result | Rules |
| Progress bar | Terminal loading bar with count and ETA | Scan, large jobs |
| Status chip | Category, unsubscribe status, job status; color + glyph | Many |
| Shortcut hint | Small bracketed key label `[x]` beside controls | Everywhere |
| Toggle / switch | Square terminal switch with ON/OFF text | Settings, protections |
| Nav rail | Vertical list of screens with glyphs and key hints | App frame |

Each is defined by structure and behavior, not a library, so any framework can implement them against the tokens.

## Interaction, motion and accessibility

- **Focus:** a blinking block cursor or a bright accent frame marks the focused element, like a terminal caret. Always visible, never removed for mouse users.
- **Keyboard:** every action has a shortcut shown as `[x]`. A `?` overlay lists all shortcuts. Arrow/j-k navigation through lists; space to select; enter to open.
- **Motion:** minimal and mechanical — instant state changes, a quick "type-on" reveal for new readouts, a sweep for progress bars. All motion respects `prefers-reduced-motion`.
- **Feedback:** actions echo to a status line at the bottom ("ARCHIVED 1,812 MESSAGES — UNDO \[z\]"), matching the terminal metaphor.
- **Destructive guard:** delete and block use `--danger` framing and require the confirm sheet; permanent deletion requires typing the count.
- **Accessibility:** the Reduce effects mode, AA contrast with glow off, full keyboard operability, screen-reader labels on every control and chip, and status never by color alone. The terminal theme is cosmetic and fully removable.

## Mascot

A small pixel-art kitten plays the part Vault Boy plays in Fallout: it shows up in empty states, progress and success moments, and reacts to what the user is doing.

**Style:** cute, anime proportions (big head, big eyes), pixel-art feel but softly stylized rather than strict pixel art, 2–6 stepped frames per animation. Colored only from the active theme's tokens: phosphor green, amber, or dark gray on Paper.

| Pose | Where it appears |
| --- | --- |
| Carrying a box of letters | Account scan, sync, large background jobs, next to the progress bar |
| Sweeping the floor | While a bulk cleanup runs; the "done" readout afterwards |
| Sleeping | Empty states: inbox clean, no suggestions, nothing to train |
| Watching the mouse | Idle on the dashboard; eyes follow the cursor |
| Playing with a ball | Idle after a while; the Train screen when the queue is empty |

**Ball easter egg:** in the playing pose the ball can be grabbed and thrown anywhere in the window; the kitten chases it, bats it around and brings it back. Disabled under Reduce effects and reduced motion.

**Favicon and app icon:** the kitten with an envelope in its mouth.

**Calendar easter eggs** swap the mascot (and favicon where noted):

| When | Mascot |
| --- | --- |
| First Monday of the month | Making a mess in a room full of envelopes (also the favicon) |
| Every Friday | Drinking coffee |
| Second Wednesday of the month | Watching out of a window |
| App opened between 19:55 and 20:05 | Eating dinner from a bowl |

**Rules:** the mascot is decorative — hidden from screen readers, never covers data or controls, and never delays an action. Settings offer On, Quiet (empty states only) and Off.

**Presence gradient.** The kitten is on every screen but never in the way: busier screens get a quieter posture, not just a smaller sprite.

- **Busy screens** (Bundles, Bundle detail, Rules): peeking from the edge of an existing widget — a panel, row, card or the nav rail. Only eyes, top of head and paws show; it reacts from there (ears, a glance, a slow blink).
- **Medium screens** (Unsubscribe, Activity, Settings): sitting at an edge, whole body visible, still.
- **Calm and empty screens** (Dashboard idle, Train empty, empty states): fully out, full animation.
- **Peek art** comes as two transparent cutouts anchored to a real widget edge: **top edge** (rising from below) and **side edge** (leaning in from left or right).

**Where the art comes from.** The kitten is produced in a separate, private mascot project (3D model, rig, renders). This repo only holds the exported files and the contract below. The mascot art is proprietary; see the licensing note in `open-questions.md`.

**Asset contract.** What the app expects from the mascot project:

- **Format:** one PNG sprite sheet per pose and size, frames in a single horizontal strip, transparent background. Animated with CSS `steps()`. No video.
- **Colour by role, themed at runtime.** Frames are greyscale, and each grey level stands for one shade role. The app maps the roles to the active theme's tokens at runtime (e.g. an SVG `feComponentTransfer` table), so one set of frames serves Green, Amber and Paper, and a new theme needs no re-render. Proposed levels, to confirm on a test render:

    | Role | Grey (0–255) |
    | --- | --- |
    | Outline | 0 |
    | Pupil | 40 |
    | Shade | 110 |
    | Fur | 170 |
    | Highlight | 235 |

- **Sizes:** 96px (empty-state hero), 64px (inline), 48px (tablet/phone), each at 1x and 2x.
- **Manifest:** a `<pose>.json` next to each sheet with frame count, frame duration (ms), loop, and for peeks the anchor edge (`top` or `side`).
- **Layout:** `assets/mascot/<pose>@<size>.png`, `<pose>@<size>@2x.png`, `<pose>.json`, plus `assets/mascot/LICENSE`.
- **Poses** (slugs): `carry-box`, `sweep`, `sleep`, `watch`, `play`, `peek-top`, `peek-side`, `coffee`, `window`, `dinner`, `mess`. `watch` needs its eyes as a separate overlay with directional frames so they can follow the cursor. `play` needs the ball as a separate sprite so it can be thrown. Favicon and app icon (kitten with an envelope) ship as standard icon files.

**To explore further:**

- [ ] Name for the kitten
- [ ] Frame counts and idle timing before poses change
- [ ] Reactions to user actions (e.g. a big delete, unsubscribe success, undo)
- [ ] More calendar or seasonal eggs

## Decisions and next

- **Default theme:** Green. Amber and a white/dark-gray Paper theme also ship.
- **CRT effects:** on by default.
- **Desktop window:** native window chrome, with the theme inside.
- **High-contrast option:** yes — the Paper theme serves as it.
- **Mascot:** pixel-art anime kitten, specified in the Mascot section.

**Next:** tech stack — how this is delivered as a desktop app with an optional self-hosted mode, the local IMAP indexer, the naive-Bayes learner, and the optional Ollama/Haiku escalation.
