# opencode Adapter

apexyard's governance was built for Claude Code first: `CLAUDE.md` is auto-loaded at session start, `.claude/hooks/*.sh` mechanically enforce the merge gate / ticket-first / secrets-scan rules, and `.claude/skills/*.md` become typed slash commands. [opencode](https://opencode.ai) is a different shape — a plugin-extensible agent CLI with a `tool.execute.before` lifecycle hook, no built-in slash-command runner, and no Claude-Code-style hook wiring of its own. **apexyard reaches opencode through a plugin that shells out to the same, unmodified `.claude/hooks/*.sh` scripts Claude Code uses** — one governance implementation, enforced under two harnesses.

Decision record: [`AgDR-0092`](agdr/AgDR-0092-opencode-gate-adapter.md). Promoted from spike [me2resh/apexyard#816](https://github.com/me2resh/apexyard/issues/816) (verdict: VIABLE, proven fully live). Feature: me2resh/apexyard#821.

## What this enforces

The adapter (`harness-adapters/opencode/src/gate-dispatcher.ts`) does **not** hand-maintain a gate list. At plugin load, it reads the ops fork's `.claude/settings.json` and derives the full `PreToolUse` gate table from it — every hook wired to the `Bash`, `Edit`, `Write`, `MultiEdit`, `Read`, `Glob`, or `Grep` matchers, including the multi-tracker merge-shape variants (`gh pr merge`, `gh api .../merge`, `glab mr merge`, `glab api`, `tracker_pr_merge`). This was originally a deliberate upgrade over the sibling pi adapter (`harness-adapters/pi/`), which shipped a hand-written, curated `DEFAULT_GATES` table — pi has since converged to the same settings.json-derived pattern (#840 C5; see `harness-adapters/_shared/derive-gates-core.ts`, the parsing core the two adapters now share) — see `harness-adapters/opencode/src/derive-gates.ts` for the full rationale, referred to throughout this framework as "the #730 convergence" (the design review on the Codex adapter, #730, first named the risk that a hand-maintained gate table can silently drift from `.claude/settings.json`).

Concretely, this means the opencode adapter enforces (non-exhaustively — the list is whatever `.claude/settings.json` wires, not a fixed set): the two-marker merge gate (`block-unreviewed-merge.sh`), red-CI merge blocking (`block-merge-on-red-ci.sh`), design and architecture review gates (`require-design-review-for-ui.sh`, `require-architecture-review.sh`), secrets scanning (`check-secrets.sh`), leak protection (`block-private-refs-in-public-repos.sh`), ticket-first and migration-ticket-first edit blocking (`require-active-ticket.sh`, `require-migration-ticket.sh`), and every other blocking `PreToolUse` hook the framework ships or an adopter adds — automatically, with zero adapter changes required when a new gate is wired.

## Install

```bash
cd harness-adapters/opencode
npm install
cd ../..
bash bin/install-opencode-adapter.sh   # writes .opencode/plugins/apexyard/ in your CWD
```

**Install at the ops-fork root** (run the script from `/home/bara/apexyard`, not from inside a workspace project). opencode discovers `.opencode/plugins/` directories by walking up from the session directory through every ancestor — verified live: sessions in `workspace/alfrouting/` and `workspace/alfdocument/` both load the single fork-root copy. One canonical install serves the whole portfolio; per-project installs drift (this already happened once: a stale V1 copy in one project kept failing while the fork copy had moved on) and every extra copy re-runs each gate once per tool call.

**No `opencode.json` entry is needed.** Plugins under `.opencode/plugins/` load automatically, and no `plugins` key (V2's array shape — not the `{name: path}` object map some notes suggest) needs to point at the adapter. A `plugins` array entry is only useful for loading the adapter from a non-discovered location.

**The discovery directory is PLURAL — `.opencode/plugins/` — not `.opencode/plugin/`.** An earlier version of this doc claimed opencode's discovery glob accepted either form; that was wrong, confirmed live against opencode 1.17.16 in [me2resh/apexyard#844](https://github.com/me2resh/apexyard/issues/844) — the singular form silently loads nothing.

**Every adapter file must live inside one subdirectory, never flat in the discovery dir.** opencode's loader invokes EVERY exported function of a discovered file as a candidate plugin factory, not only its default export. `gate-dispatcher.ts` (the adapter's real implementation) exports several named helpers alongside its default plugin — placed directly in `.opencode/plugins/`, opencode calls each of those as if it might be a plugin and crashes the whole server at startup (`Error: {"name":"UnknownError","message":"Unexpected server error"}`; also #844, live). `bin/install-opencode-adapter.sh` avoids this by construction: it writes `.opencode/plugins/apexyard/index.ts` — a re-export shim whose ONLY export is the default plugin — plus its helper modules as siblings inside that same subdirectory. opencode's subdirectory discovery only loads `index.ts` per subdir (verified live), so the helpers are never independently visited. See `harness-adapters/opencode/src/index.ts`'s header comment for the full mechanism, and `harness-adapters/opencode/README.md` for the script's `--target-dir`/`--root` options and the alternative `opencode.json` `plugin` config array install path.

## How it works

1. opencode loads the plugin's default-exported object (`{id: "apexyard", setup, server}` — the dual V1+V2 entrypoint). On 2.x, `setup` resolves the apexyard ops root (an `.apexyard-fork` marker or the legacy `onboarding.yaml` + `apexyard.projects.yaml` pair, walked up from `location.directory`, with `location.project.canonical` as fallback; or an explicit `APEXYARD_OPS_ROOT` override), derives the gate table from that ops root's `.claude/settings.json`, and registers `ctx.tool.hook("execute.before", handler)`. On 1.18.29+, `server()` returns the same gates behind the V1 `(input, output)` envelope. The installed copy has zero runtime imports (every SDK import is `import type`, erased at load) — a runtime SDK import is what broke the first V2 port live (`Cannot find package '@opencode/plugin'`, reproduced locally), because the installed copy ships with no `node_modules` beside it.
2. On every `tool.execute.before` / `execute.before` event, the plugin checks the tool call against the derived gate table. A matching gate runs its bash hook as a real subprocess with the reconstructed Claude-Code-shaped stdin (`{tool_name, tool_input}`). The V1 and V2 envelopes differ and both are normalized: 1.x wraps arguments in `{args}` and reports the bash tool as `"bash"` with file paths as `filePath`; 2.x passes arguments directly and reports the same tool as `"shell"` with file paths as `path` (both proven from real session data, not docs — see `derive-gates.ts`). `Bash` gates are wired to both ids; argument shapes are unwrapped then mapped to `{command}` / `{file_path}`.
3. Exit code `2` → the plugin **throws** (opencode's only way to deny a tool call inside `tool.execute.before` — there's no `{block: true}` return the way pi has; third-party reports confirm throw-to-deny holds on 2.x). Any other numeric exit code → allow. No numeric exit status at all (spawn failure, output over the buffer ceiling, signal kill) → the plugin throws and **fails closed**, naming the hook and the underlying error rather than silently letting the tool call through.

## Testing

```bash
cd harness-adapters/opencode
npm test               # hermetic unit tests: settings.json-derived gate matching,
                        # stdin construction, exit-2 -> throw mapping (subprocess exec mocked)
npm run typecheck      # verifies field-name assumptions against opencode's real, installed types
bash test/smoke-block-unreviewed-merge.sh   # real hook, real exec, fixture ops root, negative control
```

## What is proven, and what remains — the honest breakdown

| Claim | Status |
|-------|--------|
| opencode's plugin API surface: V2 `{id, setup}` + `ctx.tool.hook("execute.before")` with `{tool, input}` (input unwrapped); V1 `(input, output)` with `{args}` wrapping | **Verified against the real, installed `@opencode/plugin@2.4.1` and `@opencode-ai/plugin` packages** — type-only imports in `src/`, re-verified by `npm run typecheck` on every run. The 2.x shell-tool id (`"shell"`, not `"bash"`) and file-path field (`path`, not `filePath`) are proven from real session data (`session_part` vs `part` tables), not docs alone. |
| A V1-shaped default export does not run in V2 | **Verified live, twice**: every pre-port copy failed with `PluginModule.LoadError: Plugin must export a default definition with an id and an effect or setup function` (40+ occurrences in the server log), and a local reproduction shows the mechanism — a runtime `import { Plugin } from "@opencode/plugin"` throws `Cannot find package` in the installed copy (no `node_modules` beside it), leaving no usable default export. The dual `{id, setup, server}` literal (no runtime SDK import — `Plugin.define()` is the identity function) loads clean: zero load failures after reinstall, confirmed in the live server log. |
| The gate table matches `.claude/settings.json`'s real wiring, including multi-tracker merge-shape globs | **Verified** — `derive-gates.test.ts` parses this repo's own real `.claude/settings.json` (not just a hand-written fixture) and asserts every named gate, plus the merge-gate's five glob variants, are derived correctly — on both the `bash` and `shell` wires. |
| A real, unmodified bash hook (`block-unreviewed-merge.sh`) blocks a fabricated merge command when driven through the full dispatcher pipeline (settings.json parse → gate match → real subprocess exec → exit-code mapping) | **Verified live** — `test/smoke-block-unreviewed-merge.sh` runs this against a fixture ops root with a real copy of `.claude/hooks/`, with a negative control (no hook scripts present) proving the block is hook-driven, not unconditional. Additionally verified against the REAL fork (not a fixture): the installed copy's `setup` + registered V2 handler blocks a fabricated V2 `shell` merge event via the real `require-active-ticket.sh`, with the hook's own reason text surfaced. |
| A real, model-driven **opencode** agent turn invoking `tool.execute.before` with a matching event shape | **Proven once, live, by spike #816** (opencode ships a free no-API-key model, `opencode/big-pickle`; a real model turn attempting `gh pr merge` was blocked by the unmodified `block-unreviewed-merge.sh`, with a negative control — corrupting the plugin — confirming the block came from the plugin). **Not re-verified against this shipped, settings.json-derived dispatcher** — this build's environment had no opencode model credentials available to re-run that live check against the final code. The smoke script above re-proves everything downstream of opencode's own dispatch; the one hop it cannot re-prove is opencode's internal event-dispatch behavior itself. |
| The upstream `batch` tool bypass | **Not fixed, not ours to fix.** opencode's `batch` tool bypasses `Plugin.trigger()` entirely ([anomalyco/opencode#5894](https://github.com/anomalyco/opencode/issues/5894)) — a batched tool call skips every plugin hook, including this dispatcher. Tracked here for visibility. |

### Closing the live-verification gap

Once opencode model credentials are available: load the plugin in a real opencode session pointed at an apexyard ops fork, prompt a model turn that attempts an ungated `gh pr merge`, and confirm the tool call is refused with `block-unreviewed-merge.sh`'s own reason text surfaced to the model — the same verification spike #816 already performed once against the throwaway prototype plugin, re-run here against the shipped, settings.json-derived dispatcher.

## Relationship to the pi and Codex adapters

| Adapter | Gate source | Mechanism |
|---------|-------------|-----------|
| `harness-adapters/pi/` | **Derived from `.claude/settings.json` per `tool_call` event** (AgDR-0082's "Update (GH-840)") | Dispatcher extension, `tool_call` event, `{block, reason}` return |
| `bin/sync-codex-adapter.sh` (Codex) | Generated from `.claude/settings.json` at build time (AgDR-0088) | Static `.codex/hooks.json`, predicates compiled into the generated command |
| `harness-adapters/opencode/` (this adapter) | **Derived from `.claude/settings.json` at plugin load time** | Dispatcher plugin, dual entrypoint (`setup` + `ctx.tool.hook("execute.before")` on 2.x, `server()` returning `tool.execute.before` on 1.18.29+), throw-to-deny |

All three converge on the same underlying principle established by AgDR-0086 ("hooks stay bash, not ported"): `.claude/hooks/*.sh` remains the single source of truth for every gate decision, and every harness reaches it through a thin transport layer rather than a reimplementation. This adapter's contribution — closing the "gate table can drift from settings.json" gap pi originally carried — has since been ported to pi as well (#840 C5), sharing the settings.json-parsing core (`harness-adapters/_shared/derive-gates-core.ts`) between the two; each adapter keeps its own runtime-specific tool-vocabulary translation and `buildToolInput`.
