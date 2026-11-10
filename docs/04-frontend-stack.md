# Frontend stack: Mochi vs Vite + router

Oct 10, 2026 · @Teodor

Which foundation the Svelte frontend rides on — the young, Bun-native **Mochi** framework, or the proven **Vite + standalone router** recipe. Both run Svelte; the choice is about maturity versus fit with the Bun engine we already picked.

## The decision in one line

For a **desktop-first app in a Tauri shell**, the frontend mostly renders locally and is not a content website chasing first-paint SEO — so Mochi's headline strength (server-side rendering with zero-JS islands) buys less than it would for a public site, while its youth is a real risk. **Vite + a small router is the safe default.** Mochi becomes genuinely attractive for the **self-hosted mode**, where its Bun-native SSR, built-in WebSocket/SSE and `bun:sqlite` alignment match our engine exactly — so the real question is whether both modes share one frontend foundation, or the desktop and self-hosted hosts diverge.

## Side by side

| Axis | Mochi | Vite + router |
| --- | --- | --- |
| JS shipped by default | Zero — opt-in hydration per component (islands) | Full SPA bundle, everything hydrated |
| Rendering | Server-first SSR + islands | Client-side SPA |
| Runtime | Bun only | Any (Vite is runtime-agnostic) |
| Routing | Built-in route objects | Add a small router (svelte-spa-router, etc.) |
| WebSocket / SSE | Built in, Bun-native | Not included — add your own |
| Bundler | Bun | Vite |
| Maturity | Young, 0.x, small ecosystem | Battle-tested, huge ecosystem |
| Storybook / Playwright | Works (Svelte + browser-driven) | Works, deepest tooling support |
| Best for | Content sites, SSR, performance | SPAs, broad deployment, stability |

## For Inbox Sweeper specifically

**In the Tauri desktop shell — edge goes to Vite + router.**

- The UI loads from local files inside a native window; there's no network round-trip to optimise and no SEO to chase, so SSR's main payoff is muted.
- Tauri's recommended, documented path is Vite in SPA mode — fewer surprises, and the Svelte plugin gives hot reload and production builds out of the box.
- A dashboard of mostly-static bundle rows is still cheap to hydrate in Svelte; islands are a smaller win here than on a content site.

**In self-hosted mode — edge goes to Mochi.**

- Served from a server over the network, so SSR and zero-JS islands genuinely cut load on a kitchen-terminal-class browser.
- Bun-native, matching the engine runtime exactly; its built-in WebSocket/SSE would carry live scan progress and queue updates with no extra library.
- Optional `bun:sqlite` is the same storage primitive the engine already uses.

**The tension:** picking Vite for desktop and Mochi for self-hosted means two frontend foundations to maintain. One shared foundation is simpler even if it's not optimal for both hosts.

## Recommendation

**Start on Vite + a small router, and keep the UI framework-light so a later Mochi move stays cheap.** It's the proven path, it's Tauri's blessed setup, and it de-risks the desktop app that ships first. Mochi is young enough that betting the primary deliverable on a 0.x framework isn't worth it yet.

To keep the door open for Mochi in self-hosted mode:

- Keep components presentational and free of router- or framework-specific coupling, so they can be mounted under either host.
- Hold routing behind a thin wrapper rather than scattering router calls through the UI.
- Reassess Mochi when it reaches 1.0, or when the self-hosted mode becomes a priority and its SSR + WebSocket story pays for itself.

**Decided:** both applications share the same frontend code, so it must run under either host. That settles the open question in favour of one shared foundation — which makes the framework-neutrality above a hard requirement, not a nicety: build on Vite + a small router from day one, keep components presentational and free of framework-specific coupling, and hold routing behind a thin wrapper, so the one shared codebase can later be served under Mochi in self-hosted mode without a rewrite.
