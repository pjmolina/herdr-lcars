# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

LCARS for Herdr is a **Herdr plugin** (`herdr-plugin.toml`) that runs a local Node bridge serving a live dashboard for a fleet of AI agents (Claude Code, Codex, OpenCode…), per-account quota tracking, and context-preserving engine handoff. Plain ESM JavaScript, **no runtime dependencies and no build step** (Node >= 22). Source comments and some error strings are in Spanish; user-facing text must exist in both Spanish and English (`public/i18n.js`).

## Commands

```sh
npm start                      # node bin/lcars-bridge.mjs (add --open, --port, --host, --socket)
npm run check                  # node --check on every .js/.mjs, manifest sanity, forbids inline <script> in public/*.html
npm run check:shell            # shellcheck bin/plugin bin/lcars-statusline (needs shellcheck installed)
npm test                       # node --test tests/*.test.mjs tests/*/*.test.mjs
npm run release:check          # check + check:shell + test + npm pack --dry-run (run before a PR)

node --test tests/http.test.mjs                        # single test file
node --test --test-name-pattern="gzip" tests/http.test.mjs   # single test by name
```

Dashboards without a live Herdr/fleet: `http://127.0.0.1:4700/msd.html?demo=200` (or `/?demo=200`). `tests/http.test.mjs` starts the server on a free port with no Herdr.

macOS/Linux use the POSIX shell launchers (`bin/plugin`, `bin/lcars-statusline`); Windows (preview) uses Node twins (`bin/plugin.mjs` over `server/launcher.mjs`, `bin/lcars-statusline.mjs` over `writeStatusDrop` in `server/statusdrop.mjs`). Behavior changes to a launcher must be made in both. Herdr requires action/pane ids to be unique per plugin even across disjoint `platforms`, so each Windows twin in `herdr-plugin.toml` has a `-win` id suffix (`[[startup]]` has no id, so it is simply duplicated). To exercise the real integration on Windows: `herdr plugin action invoke dev.jlcases.herdr-lcars.ping-win`, then `herdr plugin log list --plugin dev.jlcases.herdr-lcars`. Keep the Node side portable:
- Compare or key on paths only through `server/paths.mjs` (`samePath`, `pathKey`, `isInsidePath`, `realPath`); git prints `C:/x`, Herdr/Codex may print `c:\X`.
- On Windows the Herdr socket is a named pipe named `\\.\pipe\` + the value of `HERDR_SOCKET_PATH` (which is a marker file, not a socket); `toEndpoint` in `server/herdr.mjs` does this.
- Pass `windowsHide: true` to every `execFile`/`spawn`; replace files with `renameReplace` (`server/atomic-rename.mjs`); open URLs via `openCommand` (`server/platform.mjs`).
- Tests must not assume POSIX modes or symlinks: use `expectMode`/`expectPrivate`/`symlinkOrSkip` from `tests/helpers.mjs`.

## Architecture

**Data flow.** `server/index.mjs` is the single HTTP server: static files from `public/`, SSE (`/events`), an OTLP HTTP/JSON receiver (`/v1/logs|metrics|traces`), and control endpoints (`/api/focus|read|session|context|remember|handoff`). It merges several sources into one state that is pushed to the browser:
- Herdr socket (`server/herdr.mjs`): `session.snapshot` polled every 1s + per-pane event subscriptions.
- Claude Code OTLP telemetry (`otlp.mjs` → `telemetry.mjs` `TelemetryStore`), with fidelity tracked in the `SOURCES` table.
- Fallbacks when a session doesn't export OTLP: transcript JSONL (`transcripts.mjs`, `claude.mjs`, `jsonl.mjs`), statusline drops (`statusdrop.mjs`, written by `bin/lcars-statusline`), `subagents.mjs`.
- CLI adapters in `server/adapters/` (Codex rollouts, OpenCode SQLite, Claude usage endpoint). **Adding a CLI** = one adapter file with `static kind` and `sync(agents)` returning `paneId → sessionId`, then add it to `ADAPTERS` in `server/index.mjs`.
- Quotas are **per account**, not global (`accounts.mjs`, `account-profiles.mjs`). Window consumption is monotonic within the same reset instant, and a missing/expired window shows "no signal", never "full tank". The UI shows percentage *remaining*.

**Context memory / handoff (`server/context/`)** is hexagonal and is the part that needs multiple files to understand:
- `domain/` is pure (identity, events, folded versioned record, summary) — it must not import `node:fs`, child_process or Herdr.
- `ports.mjs` defines contracts; `usecases/` depend only on ports; `adapters/` are the concrete git/fs/Herdr/thread-reader implementations; `composition.mjs` is the **only** module that knows concrete adapters; `registry.mjs` maps engines to thread readers; `scheduler.mjs` decides *when* to observe (worker pool of 6, ~30s interval).
- Tests wire use cases to in-memory doubles from `tests/context/fakes.mjs`.
- Touched files come from `git status --porcelain=v2 -z`, not from engine tool calls, so memory works for engines whose transcripts can't be read. Records are keyed by (repo, branch, checkout), stored outside the repo with `0700/0600` perms, atomic writes under a per-context lock.
- Handoff never closes/modifies the source pane, is idempotent via `request_id`, and advances lineage only after confirmed delivery. Handoff targets come from the installed-engine catalog, never from the request body.

**Frontend (`public/`)**: two views — MSD ship view (`msd.html`/`msd.js`, for 32:9 displays; geometry from `ship.json`, regenerated from patent drawings by `tools/trace-ship.py`) and compact deck (`index.html`/`app.js`). Shared logic lives in `shared.js`, translations in `i18n.js`, handoff/focus styling in `context.css`. No inline JS (CSP `script-src 'self'`, enforced by `npm run check`).

## Invariants to preserve

- Server binds loopback only; **every** request needs a loopback `Host` and, if present, a same-port loopback `Origin` (`sameOrigin` in `server/index.mjs`). No CORS headers. Control endpoints require `application/json`, ≤ 64 KiB, and only act on panes/dirs Herdr reports as active.
- Bound and validate all external input at its trust boundary; escape all terminal titles, paths, labels, tool names and errors before inserting into the DOM. Never log or persist tokens; the Claude usage reader uses a fixed HTTPS URL, no redirects. Don't weaken a security limit to make a test pass.
- One behavior, one implementation: share frontend product logic via `public/shared.js`.
- State colors are semantic and distinct: completed = green, waiting = orange, blocked = red, working = amber. Text never below 13 px.
- Every manifest command starts with `sh` because Herdr runs under launchd with a minimal `PATH`; `bin/plugin` locates Node (nvm/fnm/volta/asdf/Homebrew).
- Never commit local account catalogs, credentials, terminal output, context records, or bridge logs.
- `NOTICE.md` covers the unofficial fan-made/trademark position; keep the disclaimer intact.
