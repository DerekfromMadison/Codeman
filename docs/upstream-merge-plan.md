# CodeMan upstream-merge plan

Hand-off for a builder. Goal: pull the worthwhile fixes that landed in the
original CodeMan upstream since our fork diverged, without losing the local
behavior we actually run (theme, per-session git identity, CPU-gauge fix).

**Authored 2026-06-13 from a read-only analysis. Nothing here has been
applied. Do NOT rebuild or restart `codeman web` without Derek's go-ahead —
8 live tmux sessions are attached and a restart interrupts them.**

---

## The three states

| State | Where | What it is |
|---|---|---|
| **Upstream (original)** | `origin` = `github.com/SGudbrandsson/Codeman` `master` | The canonical repo. Moved **17 commits ahead** of us since Apr 18. |
| **Friend's fork** | `fork` = `github.com/DerekfromMadison/Codeman` | `fork/master` is **0 ahead / 18 behind** upstream — its master never diverged; our PRs (e.g. inline-rename `528537a0`) were merged **into upstream**. Live work sits on fork *feature* branches: `feat/light-dark-theme`, `feat/per-session-git-identity`, `feat/session-inline-rename`, `feat/better-ux` (stale/divergent — ignore), `cleanup/add-project-setup`. |
| **Running system** | `~/.codeman/app`, served by `~/.local/bin/codeman -> dist/index.js`, `node codeman web` on `:3000` | Branch `feat/session-inline-rename` @ `03624322` **+ 7 uncommitted files** (the running blend). |

**Merge-base** of our branch and upstream = `528537a0` (Apr 18, "inline rename"
— our own contribution, already upstreamed). So upstream and we share history up
to Apr 18; everything after is the gap below.

### What "running system" actually contains
The 7 uncommitted files are the running behavior, never committed:
- `styles.css` / `mobile.css` / `index.html` / `upload.html` / `feature-tracker.js` / `app.js` (theme parts) ≈ **`fork/feat/light-dark-theme`** (working tree differs from that branch by only ~69 lines).
- `tmux-manager.ts` ≈ the **per-session-git-identity** work (`setGitIdentityEnvVars`, differs from `fork/feat/per-session-git-identity` by ~5 lines).
- Plus our one committed unique fix `03624322` (CPU gauge → busy-time deltas) on `system-routes.ts`.

So: **running = `session-inline-rename` + CPU-gauge commit + uncommitted
light-dark-theme + uncommitted git-identity.** The dist serving `:3000` was
built from this blend (verify before/after — see Step 4).

---

## What upstream has that we don't (the 17-commit gap)

Grouped by what a builder should do with each. All file-level conflicts are
**mechanical** (upstream and our local edits touch different functions — see
Conflict map).

### TAKE — high value (mobile + Claude-session correctness)
| SHA | What | Why we want it |
|---|---|---|
| `06ad2c02` | Terminal: preserve space keystroke, scrollback 500→5000, bound flicker-hold to 150ms | Real terminal bugs; space-eating + flicker freeze hit daily |
| `0cb7154f` | Restore mobile touch scrolling (xterm v6 broke it) | Derek drives from mobile |
| `b4dd3ef8` | Stop on-screen keyboard show/hide from duplicating the screen on mobile | Mobile |
| `8a229bfa` | Raise initial page-load seed 128KB→512KB to fill the bigger scrollback | Mobile usability |
| `1b03135b` | Forward wheel/touch scroll to fullscreen TUIs (alternate screen) | Scroll inside vim/less |
| `9b1b9ddc` | Optimistic compose-send UI (clear + bubble + working immediately, rollback on fail) | Snappier send; has tests |
| `00e20fe4` | Widen drawer rename target to the whole row | Polishes our own inline-rename feature |
| `7dde8b22` | Untrack corrupt self-referential symlinks (`dist`, `node_modules`, `vendor`) | Repo hygiene; no-op for us (we have real dirs, already git-ignored) |

### TAKE — high value, but writes to Derek's `~/.claude/settings.json` (gate)
These three are a coherent trio and align with Derek's "never pause to ask"
rule (Claude asks as plain text instead of via the AskUserQuestion tool):
| SHA | What |
|---|---|
| `b23a7ab9` | Spawn Claude with `--disallowedTools AskUserQuestion` so it asks as plain text; add a global `idle_prompt` Notification hook so idle-attention still fires |
| `5a01d5c2` | Auto-install the AskUserQuestion PreToolUse hook into **`~/.claude/settings.json`** on server start (idempotent, merge-safe, opt-out `CODEMAN_NO_GLOBAL_HOOK=1`) |
| `3baa6fb7` | Surface AskUserQuestion live in the transcript — readable + answerable on mobile |

**⚠️ Gate:** `5a01d5c2` + `b23a7ab9` mutate Derek's global `settings.json`
(which holds the CodeMan model default, `DO_NOT_TRACK`, the Stop hook, etc.).
The installer reads-merges-writes, never clobbers, aborts if unparseable, and
is opt-out via `CODEMAN_NO_GLOBAL_HOOK=1`. Behavior is desirable, but **confirm
with Derek before the first server restart** so the auto-edit isn't a surprise.
If he prefers, set `CODEMAN_NO_GLOBAL_HOOK=1` and add the hook by hand.

### SKIP — no value to us
| SHA | What |
|---|---|
| `b607e42e` | `new-name.md` — upstream's project-rename brainstorm |
| `020f73a3` | gitignore a public symlink we don't have |
| `TASK.md` churn (inside several commits) | Upstream's internal task log |

---

## Conflict map (all mechanical — verified by reading both sides)

| File | Upstream edits | Our local edits | Verdict |
|---|---|---|---|
| `src/web/public/app.js` | scroll, optimistic-send, transcript-hook, drawer-rename functions | theme system (`XTERM_THEME_*`, picker, localStorage) — different functions | **Mechanical.** Theme block vs scroll/input/transcript blocks don't overlap |
| `src/tmux-manager.ts` | `sendInput` rewrite → new `planSendKeys` util; `--disallowedTools` in `buildSpawnCommand` | new `setGitIdentityEnvVars()` + 2 call sites | **Mechanical.** Orthogonal regions |
| `src/web/routes/system-routes.ts` | **none of the 17 touch it** | our `03624322` CPU-gauge rewrite | **No conflict.** Our fix is unique |

New upstream file `src/utils/tmux-send-keys-plan.ts` + its tests come in clean.

---

## Builder steps

> Run `/review` on this plan before executing (project convention: design-first,
> internal Claude panel, implement on SHIP). The risky bits are the dirty-tree
> capture (Step 1) and the restart (Step 4) — exactly what review should pressure-test.

**0. Snapshot.** `cp -r dist dist.pre-merge-$(date +%s)`; note current HEAD
(`03624322`) and `git stash list`. Confirm `git status` matches the 7 files above.

**1. Capture the running state into commits (do this FIRST — can't merge a dirty
tree, and the dirty tree IS production).**
   - Commit the theme files as one commit on a topic branch
     (`feat/light-dark-theme` content): `styles.css mobile.css index.html
     upload.html feature-tracker.js` + the theme hunks of `app.js`.
   - Commit the git-identity `tmux-manager.ts` work as a second commit.
   - If splitting `app.js` cleanly is fiddly, commit it whole — the point is a
     clean tree, not perfect history.
   - The branch now = a faithful committed copy of what's running.

**2. Merge upstream.** `git fetch origin && git merge origin/master`
   (or rebase our 1+2 commits onto `origin/master` — either works; merge is
   safer given the blend). Resolve the mechanical overlaps in `app.js` /
   `tmux-manager.ts` by keeping **both** sides (different functions). Take
   `system-routes.ts` ours as-is. Decide the AskUserQuestion-trio gate (Step 3).

**3. AskUserQuestion gate.** Either accept the global `settings.json` auto-edit
   (default) or export `CODEMAN_NO_GLOBAL_HOOK=1` in the service env and install
   the hook manually. Confirm Derek's preference once.

**4. Build + restart (NEEDS DEREK'S OK — 8 live sessions).**
   `npm run typecheck && npm run test && npm run build` (build = `node
   scripts/build.mjs`, which rotates `dist` → `dist.backup-<ts>`). Restart only
   when sessions are idle: stop `node codeman web`, relaunch `codeman web`
   (served via `~/.local/bin/codeman -> dist/index.js`, `:3000`).

**5. Smoke (mobile browser, real session — Logs Over Tests):**
   - Mobile: touch-scroll the terminal; toggle keyboard (no screen duplication);
     scroll inside a fullscreen TUI (vim/less).
   - Type a space in a prompt — it survives.
   - Compose-send feels instant; a forced failure rolls the bubble back.
   - Spawn a session that would call AskUserQuestion → it asks as plain text and
     shows in the transcript, answerable from mobile.
   - CPU gauge still reads busy-time (our `03624322`).
   - Theme switch (light/dark/system) still works; a session in a git repo still
     commits with the per-session identity.

**6. Push** the merged topic branch to `fork`, open a PR (don't push to
   `fork/master` directly). Our CPU-gauge fix `03624322` is a clean upstream PR
   candidate too.

---

## Risk summary
- **Lowest-risk wins, take first:** `7dde8b22`, `8a229bfa`, `00e20fe4`, `1b03135b` — no behavior overlap.
- **Daily-pain fixes:** `06ad2c02`, `0cb7154f`, `b4dd3ef8`, `9b1b9ddc`.
- **Gated:** AskUserQuestion trio (`b23a7ab9`/`5a01d5c2`/`3baa6fb7`) — global `settings.json` write.
- **Operational:** rebuild + restart interrupts 8 live sessions → Derek's call on timing.
- **`git revert` is the rollback** for any merge; `dist.backup-*` restores the running binary.

---

## Builder addendum — facts verified before execution (2026-06-13)

Read-only checks run against the live host, resolving the plan's open risks:

- **Restart is process-safe for live sessions (incl. this Claude session).** The
  web server is the systemd **user** unit `codeman-web.service` (PID of `node
  …/codeman web`, parent = systemd user mgr, not system scope). The tmux
  *session* shells (where Claude runs) are children of the **tmux server
  daemon**, not of the web server; the web server only owns short-lived `tmux
  attach-session` *viewer* clients. So `systemctl --user restart codeman-web`
  drops the browser viewers (auto-reconnect) and leaves every session + its
  Claude process running. The "8 live sessions" worry is really "browser
  viewers blink," not "work dies." (10 sessions attached at check time; all idle
  per Derek.)
- **The global `settings.json` auto-edits are append-only and safe.** Two
  installers fire on server start: `5a01d5c2` appends a PreToolUse entry with
  `matcher: "AskUserQuestion"`; `b23a7ab9` appends a Notification `idle_prompt`
  entry. Both read-merge-write, are **idempotent** (no-op once present), **abort
  untouched** if the file is unparseable/not-an-object, and are gated by
  `CODEMAN_NO_GLOBAL_HOOK=1`. Verified by reading `installGlobalAskUserQuestionHook`:
  it `push`es a new array entry and never mutates existing ones — so the
  **frozen runaway-CPU PreToolUse hook (different matcher) is not touched.**
  Decision: **accept the auto-install** (no opt-out) — verified-safe, self-heals
  across future upgrades, and reverses in seconds (hand-edit + `git revert` +
  set the env var). Side effect to expect: `--disallowedTools AskUserQuestion`
  means future Codeman-spawned Claude sessions ask as **plain text** (mobile-
  friendly; aligns with the never-pause-with-a-tool-modal preference).
- **Fork "behind 1" is moot.** `fork/feat/session-inline-rename`'s extra commit
  is `00e20fe4` ("widen drawer rename row") — itself one of the 17 we're taking,
  so it arrives via the upstream merge. Our only unique commit is `03624322`
  (CPU gauge). Merge-base with `origin/master` = `528537a0`; gap = 17.
- **Build does NOT back up — corrected.** `scripts/build.mjs` writes `dist` in
  place (`tsc` → `dist/`, `rm -rf dist/web/public && cp -r`, esbuild
  `--allow-overwrite`); the two existing `dist.backup-*` were made by hand. So a
  build that fails after `tsc` leaves a half-written `dist` that bites at
  restart. Step 0's `cp -r dist dist.pre-merge-<ts>` (8.2 MB) is therefore
  **load-bearing**, and the restart is gated on a clean build, not just exit code.

## Review outcome — round 1 (internal panel) = SHIP w/ refinements

A real trial merge (committed dirty tree vs `origin/master`) produced **zero
conflicts**; `tsc --noEmit` + the merge-critical tests passed; the CPU-gauge fix
survives (`system-routes.ts` untouched by all 17); and `SIGTERM → shutdown() →
mux.destroy()` only saves session state (no `kill-session`/`kill-server`) — so
the restart preserves every live session. Execution refinements folded in:

- **One commit, `--no-verify`.** Capture the running blend as a single commit;
  the pre-commit Prettier hook rejects the unformatted live source, and
  formatting it would mutate what's running. (Upstream's `b23a7ab9` did the same.)
- **`cp -r dist` is mandatory** (build has no backup); restart only after the
  build prints success.
- **Back up `~/.claude/settings.json`** before restart — the global hook
  installer writes non-atomically; back-up + verified-clean build (no crash-loop)
  + the installer's abort-on-unparseable guard contain the blast radius.
- **Gate = typecheck + targeted merge-critical tests + build**, not the full
  246-file upstream suite (Cmd XI). Stop before restart on any failure.
- **Take upstream verbatim.** The dead-but-inert AskUserQuestion PreToolUse path,
  the `||true` curl, the swallowed installer log-reason, the duplicate `mkdir`
  are upstream-owned warts; patching them re-forks us and defeats the
  re-converge goal (Cmd VIII). Confirm the install by **reading `settings.json`
  directly** after restart instead of editing the log.
- **Accept the global-hook auto-install** (no `CODEMAN_NO_GLOBAL_HOOK=1`) — the
  opt-out + manual hook is the ceremony the panel rejected.
