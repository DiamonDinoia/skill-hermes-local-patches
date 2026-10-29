---
name: hermes-local-patches
description: "Use when changing a local Hermes customization, or when `hermes update` or the reapply hook fails."
version: 2.0.0
---

# Hermes local customizations

The live install at `~/.hermes/hermes-agent` is an upstream git checkout. `hermes update` resets it to upstream. Every local customization lives in `~/dotfiles/hermes` and deploys through stow. The dotfiles copy, not the live tree, is the source of truth.

## Where a customization goes

Use the first route that can carry the change. Each later route costs more on every update.

1. A config key in `~/.hermes/config.yaml` (`hermes config set`). Search `hermes_cli/config_defaults.py` first.
2. A user plugin in `dotfiles/hermes/.hermes/plugins/<name>/` (`plugin.yaml`, `__init__.py` with `register(ctx)`). The plugin API is in `hermes_cli/plugins.py`: `register_hook` (names in `VALID_HOOKS`), `register_middleware` (`hermes_cli/middleware.py`), `register_system_prompt_section` (4000-char cap), `register_delegation_provider`, `register_tool`, `register_command`. Enable it in `plugins.enabled`.
3. A TUI user widget in `dotfiles/hermes/.hermes/tui-widgets/*.mjs`. `ui-tui/src/sdk/userWidgets.ts` loads every file there.
4. A hook point in core: a call of one or two lines into a new file that the patch adds. New files never conflict.
5. A change to upstream lines, as the last option.

Routes 4 and 5 go into the one patch file `dotfiles/hermes/.hermes/patches/local.patch`. Keep its changed lines in upstream files near 200. Count them with:

    git diff --numstat <base> | while read a d f; do git cat-file -e <base>:"$f" 2>/dev/null && echo "$((a+d)) $f"; done | sort -rn

A route-4 or route-5 change gets a test that fails when the change is removed. Put new tests in new files.

Before writing a new change, prove the trigger with the real handlers under the live config, read-only. Check whether a config key removes the symptom at zero code cost. When a probe shows a change is no longer needed, drop it and name the upstream or config fact that made it dead.

The TUI gets `subagent.list` from the messaging gateway, which projects the roster with its own field list in `gateway/session_ancillary.py`, not the one in `tui_gateway/methods_subagents.py`. A new roster field goes into both. A test keeps them equal.

In `_REASONING_CARRIER_ROUTES` (`agent/message_sanitization.py`) the first matching row wins. The flatiron host row must stay before the `_LOCAL_CARRIERS` row, which claims `custom`.

## Current customizations (2026-10-29, base fcc26fbd9f)

- Plugin `delegation-tiers`: `delegation.backend: tiers` routes each `delegate_task` task by its optional `tier` (model, provider, reasoning effort, fallback, extra skills from `delegation.tiers`). Every child also gets `skills.auto_load`. If the plugin breaks, `hermes config set delegation.backend default` restores the stock route.
- Widget `quotabar.mjs`: the statusline rows, `/quota` and `/hints`. `node quotabar.mjs` runs its self-check and prints `quotabar self-check ok`.
- `local.patch`: route skills into the child prompt, `delegation.nest_providers` gate, last tool preview on the subagent record, request-dump warning log level, one flatiron reasoning carrier row (without it Kimi-K3 gets HTTP 400 `duplicate field reasoning`), durable-write symlink fix, scratch retention 7 days, resume billing-stamp fix, the subagent roster with tier and model (subagent progress events also carry the tier), the agent chat view in the agents overlay, the two Ctrl+T fixes, and the CLI chrome skins. The TUI parts live mostly in the new files `ui-tui/src/app/cliChrome.ts`, `components/cliChrome.tsx` and `components/agentChat.tsx`.

## The reapply hook

`dotfiles/hermes/.local/bin/hermes-reapply-patches [CLONE]` runs after each successful `hermes update`, through the `~/.local/bin/hermes` wrapper. It applies `local.patch` with plain `git apply`, which is all-or-nothing, then runs `npm ci` and `npm run build` in `ui-tui`.

- Exit 0: applied, or `skipped: local.patch (already applied)`.
- Exit 1: the patch does not fit the new base, and the tree stays stock. Hermes still runs without the patch-carried features. The plugin still routes tiers, but core drops the tier `skills`. The widget refuses to load and names `local.patch` in the transcript. Rebase the patch (below), then rerun the hook. A failed ui-tui rebuild also exits 1.
- Exit 2: CLONE is not a git checkout.

The first line of `local.patch` is `Upstream base: <sha>`. `git apply` skips text before the first `diff --git`.

## Running `hermes update` non-interactively

Dry run first (`hermes update --dry-run`), then `hermes update --yes --keep-stash --no-gateway-restart`. Without `--yes` the command waits on a prompt with no error.

The updater stashes local changes. Treat the stash as a read-only backup. Never pop it onto the new base. Rebuild from `local.patch`.

## After a successful update

Make sure that the base moved (`git log --oneline -3`) and that the hook printed `applied:`. Then run, and judge each by stdout and exit status: a second `hermes update` (must print up to date), `hermes doctor`, one `hermes chat -q pong -Q` for each configured provider, `hermes mcp list --check`, `hermes skills list`, and the gateway status command. Then `hermes skills update`, because a code update does not refresh managed skills.

The config migration changes only `_config_version` in config.yaml. Diff config.yaml before and after. Report any other change. Never revert user settings.

## Rebase `local.patch` onto a new upstream commit

1. Never edit the live clone. Use `git clone --shared ~/.hermes/hermes-agent <dir>` with `<dir>` outside `~/.hermes`. A linked worktree under `~/.hermes` trips `tests/home_io_guard.py` and gives false failures.
2. `git checkout <new-base>`, then `git apply --3way <local.patch>`. Resolve each conflict and keep both the upstream change and the purpose of the local change. Commit each logical change on its own (`git commit --no-verify`, because the clone has no node_modules for the eslint hook).
3. Drop a change that upstream now covers. Move a change to an earlier route when the new base gives one.
4. When a change touches the input of a generated file, rerun the generator (`scripts/gen_gateway_contracts.py`). Never edit a generated file by hand.
5. Run the gate (below) on the clone and on a clean clone at `<new-base>`. The patched run may fail only tests that also fail on the clean base.
6. Regenerate the file. Never hand-edit the old patch:

       { echo "Upstream base: $(git rev-parse <new-base>)"; echo; git diff --binary --full-index <new-base> HEAD; } > local.patch

7. Fit check: in a fresh clone at the current origin tip, run the hook with that clone as CLONE. Fetch just before. `hermes update` lands on the tip at the time it runs, not on the checked one.

## The gate

- Test venv: the newest match of `~/.hermes/installs/*/test-environment/gen-*/venv/bin/python` (`ls -td`). Use it read-only.
- Run pytest as `env -u HERMES_SINGLE_QUERY_SESSION <venv> -m pytest -p no:cacheprovider`. The variable leaks into subprocess tests and breaks the delegate tests.
- Python files: every test file the patch adds or changes (`git diff --name-only --diff-filter=AM <base> -- 'tests/*.py'`), plus `tests/tools/test_delegation_backends.py`, `tests/tui_gateway/contracts/test_generated.py`, and the plugin's `tests/` directory.
- TUI: link `node_modules` and `ui-tui/node_modules` from the live clone, then `env -u NODE_ENV npm run typecheck` and `npx vitest run` on the changed `src/__tests__` files plus `src/__tests__/useInputHandlers.test.ts`.
- Widget: `node quotabar.mjs`.
- Run every step under `nice -n 19 ionice -c 3` and `taskset` off the measurement cores.
- Retention tests take the age from `SCRATCH_MAX_IDLE_HOURS`, never from a fixed number of hours.

## Apply to the live install

1. Reverse the old patch: `git apply -R --check` then `git apply -R` with the old `local.patch`.
2. Run the hook with the new `local.patch`, or `git apply` plus `npm run build` in `ui-tui`.
3. Copy the files into `~/dotfiles/hermes`. Deploy with `~/dotfiles/setup.sh --restow-configs`. Never make symlinks by hand. Commit only the hermes paths, because the dotfiles tree carries unrelated user edits.
4. Restart the gateway only with user consent. `hermes gateway restart --yes` interrupts live turns, in-flight compaction included, after its drain wait. Without a supervisor it runs the new gateway in the caller's shell. A TUI-spawned gateway (`--idle-exit`) keeps the old Python modules until it exits: close every TUI, wait for the gateway process to end, then start `hermes`. Running TUIs keep the old build until the user restarts them.
5. Check live: open `hermes` in a detached tmux session, send keys, and read `tmux capture-pane -p`. Count new `duplicate field` lines in `~/.hermes/logs/errors.log` after the restart.

## A new error right after a rebuild

A rebuild exercises cold paths that the long-running process never hit. Diff every file on the error path against the upstream base and grep `local.patch` for its symbols before writing a fix. An empty diff and no patch hit mean the error is upstream.

A refused `session.resume` with `invalid_params field=base_url` on a subscription provider comes from `gateway/session_local_migration.py`, which copies a `process://` billing stamp into the launch base_url. `local.patch` drops stamps that the validator refuses. Never loosen the validator: it is the trust boundary. Log the scheme only, never the value.

## Roll out to other hosts

- Roll out only after the local gate passes.
- A host on a different base commit gets `hermes update` first, then the hook. Never force a patch built for one base onto a host at another base.
- Read the channel from the host's own config (`update.installs.<sha16>.channel`). A new host needs `hermes update --set-channel main` before the first gated update.
- Drive remote hosts with `ssh <host> 'bash -lc "..."'`. Pull dotfiles, run `setup.sh --restow-configs`, then run the hook.
- A remote tree with the old 10-patch set: reverse those patches with the dotfiles content of the old commit before the pull, or `git reset --hard` to the tracked base. Then pull dotfiles and run the hook.

The review chain for patch changes is in `references/patch-review-chain.md`. Host facts are in `references/host-notes.md`.
