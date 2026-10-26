---
name: hermes-local-patches
description: "Use when rebasing or rolling out the local Hermes patches."
version: 1.0.0
---

# Hermes local patches

The live install at `~/.hermes/hermes-agent` tracks upstream plus a set of local patch files. The patch set, not the live tree, is the source of truth.

2026-10-19: the scratch prune is now a proper patch, scratch-no-prune.patch, rolled through this flow. SCRATCH_MAX_IDLE_HOURS = 24 * 7. The reason: the user keeps scratch worktrees and logs across multi-day work.

## Layout

- Patch files: `dotfiles/hermes/.hermes/patches/*.patch`.
- Apply order: the list inside `dotfiles/hermes/.local/bin/hermes-reapply-patches`, which `hermes update` runs after each update. Later patches may touch lines earlier ones added. Keep the hook list and the directory in sync. Never reorder without testing.
- The hook skips already-applied patches. A fully patched tree prints one `skipped:` per patch when the hook runs.

## Running `hermes update` non-interactively

- Dry run first (`hermes update --dry-run`), then `hermes update --yes --keep-stash --no-gateway-restart`. Without `--yes` the command waits on an approval prompt and a background job sits pending with no error, so any unattended run needs the flag set explicitly.

## After a successful update

Confirm the base moved (`git log --oneline -3`, `hermes --version` matches the target tag) and the hook printed its `applied:` then `skipped:` lines; then run the battery before telling the user to restart: a second `hermes update` (must print up to date, catches a half-applied fast-forward), `hermes doctor` on the default profile, one `hermes chat -q pong -Q` per configured provider with the provider and a known-good model pinned, `hermes mcp list --check`, `hermes skills list`, and the gateway status command. Stdout and exit codes decide; a completion banner does not.

A code update does not refresh managed skills. Follow with `hermes skills update`; verify with `hermes skills list` that zero entries still show an available update, since the summary count and the per-skill rows can disagree.

The live clone has no node_modules, so `tests_tui` fails at import in any pytest run there; exclude that directory from the update gate and rely on the updater's own TUI build step plus `typecheck` instead.

## Rebase onto a new upstream commit

1. Never edit the live clone. Run `git -C ~/.hermes/hermes-agent worktree add <scratch> <new-base>` and work there.
2. Apply each patch with `git apply --3way` in hook order, one commit per patch. Resolve conflicts keeping both intents: upstream change and patch purpose.
3. Commit each applied patch with `git commit --no-verify`. The worktree has no node_modules, so the lefthook pre-commit (eslint) fails on TUI-touching patches even when the tree is correct.
4. Drop a patch that resolves to an empty diff from the hook list; its feature landed upstream. Report the removal.
5. Regenerate each patch with `git diff` between its commit and its parent. Never edit the old patch file by hand.
6. Make sure the worktree equals `git diff <new-base>` of the intended combined change before touching anything else.
7. Add generated-artifact currency to the gate. When a patch touches the input of a checked-in generated file (a contract OpenRPC json, a generated TypeScript file), the rebase reruns the generator on the patched tree, diffs its output against the committed artifacts, and folds any drift into the patch that owns the input with `git commit --fixup` plus `GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash` at the stack base. The patch header then keeps matching the tree. A currency test (`test_generated_files_are_current` and its siblings) is green only when those artifacts sit in the patch, so run it as the gate's first test.

If origin/main moved since the rebase base, gate the rollout before touching the live tree: in a detached scratch worktree at the new origin tip, run `git apply --check` then `git apply` for each patch in hook order, so each later patch is checked against the lines the earlier patches added; a --check-only pass reports false conflicts where a patch anchors on text another patch adds. Count how many of the intervening upstream commits touch the patched paths; when the count is zero the rebase applies as-is and later fixup work rebases without conflicts. Updates move fast: a 74-commit gap applied cleanly when no commit touched patched files. Fetch immediately before this check and pin the resolved tip in the report: main moves under you, and the `hermes update` that follows lands on whatever the tip is at that moment, not on the checked one. The check's step 0 is `sha256sum -c manifest.txt`, and the report quotes that output, the stack tip and the checked origin tip. A fit check that cannot prove which patch set and base it ran against counts for nothing: one such check passed the previous generation's patches at the previous base while reporting the current names, and a later cleanup job had deleted the patch dir, so the contradiction settled only by a rerun.

## Testing

- Measure a baseline at the new base with no patches; upstream ships failing tests. The patched run may only fail a subset of the baseline's failures. Gate on "no new failures", not "exit 0".
- The test venv is the newest match of `~/.hermes/installs/*/test-environment/gen-*/venv/bin/python`. Resolve the glob before dispatch. Several installs can match. Use the venv read-only. The runtime venv has no pytest.
- Run pytest under `env -u HERMES_SINGLE_QUERY_SESSION`. The variable leaks into subprocess tests and breaks the delegate tool's test file.
- Spell the invocation out in every delegate or one-shot harness brief. Run the tests with HERMES_SINGLE_QUERY_SESSION unset (prefix `env -u HERMES_SINGLE_QUERY_SESSION`), with the test venv python, and with the new and old test files in ONE pytest run:

      env -u HERMES_SINGLE_QUERY_SESSION <venv>/bin/python -m pytest tests/<new_test_file>.py tests/<old_test_file>.py

  <new_test_file> is the patch's new test file; <old_test_file> is the pre-existing test file it extends or replaces. <venv> is the resolved test venv from the line above. If the run includes tests/tools/test_delegate.py, the run_agent-first rule below wins over this order: put the run_agent-importing file first and the rest after it.

  A split run once produced a wrong first conclusion ("live-tree run omitted my test file"), because delegate test failures depend on which files collect together.
- In a linked worktree made from the live clone, tests that import `run_agent` fail at `tests/home_io_guard.py:160` (TEST BUG: file I/O against the REAL hermes home). The cause, not a patch defect: a linked worktree's `.git` file resolves into the live clone's gitdir, which sits outside the worktree's allowlist root. Job-local venv installs and fixtures also land under that gitdir. Two results follow. Gating on any FAIL set in a worktree is meaningless unless the clean base worktree shows the same set; compare always. And a worktree run where all tests pass is not proof for the live tree: the rollout reruns the targeted subset on the live install after applying, because the guard there can also trip on an isolated run of one file.
- Running `tests/tools/test_delegate.py` standalone trips the guard even in the live install (~36 failures in the same file; the full batch shows only the known failures). When a run must cover that file alone, put a run_agent-importing test file first (e.g. tests/tools/test_delegate_child_transcript_release.py); it imports run_agent before the guard installs and the following files run clean.
- Tests that exercise retention settings fixture the age from the retention constant (read SCRATCH_MAX_IDLE_HOURS, never a hard-coded number of hours), and assert entries younger than the window survive. A retention change otherwise re-breaks a whole test file whose fixtures aged below the new window.
- Run npm under `env -u NODE_ENV`. Run `build:ink` before `test`, and run `typecheck` too.
- Run everything under `nice -n 19 ionice -c 3`.
- Gate narrowly: run only the files that import the changed module (`rg -l '<module-stem>' tests --glob 'test_*.py'`, minus tests/agent/lsp) in the patched tree and the clean-base tree in parallel, then compare FAIL sets. The full tests/tools+tests/agent double run is ~1,800 files and takes 75+ minutes under nice/ionice/taskset on the laptop, and an unbounded run can stall with no log growth for 20+ minutes. `--timeout` failed to install in the test venv, so watch the log mtime and kill a run that stops growing; rerun suspect ids alone in both trees. Start gates with `-v --tb=line` (not `-q`) so failing test ids stream into the log and trees can be compared before either run ends.

## Apply to the live install

1. Run `git apply --check` then `git apply`, one patch per call, in hook order.
2. Rerun the targeted pytest subset and typecheck in the live tree.
3. Running sessions keep the old build. Tell the user to restart. Never restart their processes.
4. Copy the regenerated patches into the dotfiles patch dir, commit only those files by name (the dotfiles tree carries unrelated user edits), and push in the same rollout. A stale dotfiles patch set silently re-applies old code after each update. Before pushing, `git ls-remote origin` and list the range remote..local: a fast-forward push carries every unpushed local commit, not just the patch commit. Scan the range's added lines for key/token/secret strings first.
5. Give patch changes the review chain on the combined diff against the live base. First round runs on the top-ranked healthy flatiron model per the tiered-subagent-dispatch routing table, refreshed with `models --refresh`, never a fixed model name; Sol runs last and only when the first round passes. A review against an older base does not count. A subagent review counts only when it proves the intended model served it (state.db session row or the job's model line): a tier-pinned model the provider rejects falls back silently to a paid Claude model. The full chain procedure is in references/patch-review-chain.md.
6. Warning on running-session state: the process the user is in keeps the old build. Tell the user to restart once; never restart their processes. A subagent's background processes are killed when the subagent exits: a probe or test that starts Hermes code can exit green and still leave a gateway child that dies silently a moment later. Have the child name the pids it spawned, kill and `wait` them inside its own turn, then run a pgrep/socket check for every candidate (`hermes gateway*`, 'gateway run') and report zero hits before the verdict.

## After `hermes update` fails to restore local changes

- The updater stashes local modifications and resets the tree on conflict. Treat the stash as a read-only backup. Never pop or apply it onto the new base. Rebuild from the patch files.
- `hermes update` runs stash, pull, dependency install, TUI build, stash restore. With `--keep-stash` it parks the stash and the wrapper runs `hermes-reapply-patches` itself: expect one `applied:` line per patch inside the update output, then one `skipped:` per patch on a manual rerun.
- The update's config migration rewrites `_config_version` in config.yaml and nothing else. Diff config.yaml before and after; report any other change, never revert user settings.
- The stash does not capture untracked files. Files added by patches stay untracked, and re-applying a patch that adds a file fails with "already exists". Delete those files first, and only after byte-comparing each one against the patch's output.
- The hook's per-patch "already applied" reverse check fails when a later patch edited the same lines. An applied-state check must process the stack from last patch to first. Validate the hook against a tree with the full set applied before trusting `skipped:` output.

## A new error right after a rebuild

A rebuild or restart exercises cold paths the long-running process never hit (cold session resume, reconnect). Attribute the error before writing a fix: diff every file on the error path against the upstream base and grep both patch sets for the path's symbols. An empty diff to the base plus no patch hit means the error predates the patch set; a TUI rebuild that only restarts the process can trigger a latent upstream defect.

On a subscription provider, a refused `session.resume` with `invalid_params field=base_url` comes from `gateway/session_local_migration.py`. It copies the session row's billing_base_url (a `process://` usage stamp) into the launch base_url, and the http(s)-only validator in `gateway/session_policy.py` refuses it. The live chat keeps running; only the cold resume of that session fails, once per reconnect. Fix at the stamp copy, never by loosening the validator: it is the trust boundary. The copy probes the validator's own predicate (`_apply_launch_overrides({'base_url': stamp}, {})`) and keeps only stamps it accepts; refused stamps (`process://`, scheme-less `x.example/v1`, `https://user:pw@x`, empty, None) drop with one debug line naming the scheme only, never the full value, which can carry credentials. Uppercase `HTTPS://` is accepted. Refusal always falls back to the profile endpoint. Any debug or info line added next to the stamp copy logs the scheme only, never the value, which can carry credentials; the same holds for hook-capture code (log the plugin-name set, never the hook text).

Verify a resume fix without spending a model turn: open `hermes -r <session-id>` in a detached tmux pane, send no prompt, capture the pane with `tmux capture-pane -p`, and grep only the gateway.log lines appended during the run for `session.resume` and `invalid_params`. Earlier messages visible in the pane plus no new refused line is a pass. The capture wraps and truncates long lines; confirm any line that matters against the session file, not the capture.

A resumed session's standing-guidance rebuild in `tools/delegate_tool_progress.py` fires plugin hooks itself; a parent with a pre_llm_call plugin then pays the hook twice per turn, and silencing the rebuild drops the hook half for parents resumed before any real turn. The fix captures the hook output during the parent's real turns (stored from `agent/turn_context.py`) and the rebuild reuses it, firing 0 hooks. Before the parent's first real turn after a resume no stored text exists, so the rebuild marks guidance not-ready and a spawn in that window ships the skills half only. A skill-less parent treats an empty skills half as resolved and publishes ready, or it rebuilds on every spawn.

A failing-fix table that flips one row often flips a second: the sanitizer's "duplicate field" matcher picks its strip set from anywhere `reasoning_content` appears in the provider error, and echoing the request payload back into the error makes it strip on an unrelated field's duplicate. Fail closed by parsing the field name only from the error's own `duplicate field \`X\`` clause (serde backtick form; accept "X" and 'X' too). Strip X only when X is a reasoning carrier the module already knows; return frozenset() for any other name, a nameless clause, or a name seen only in echoed text, and never scan the echoed payload for this error class. A fix that widens frozenset() to every no-alias error also stops stripping a genuinely named `reasoning_details`; the clause parse covers both.

## Roll out to other hosts

- Roll out only after local tests are green and the review chain ran.
- A seeded install has no channel record in its config and defaults to stable.
- Read an existing host's channel and target from its live record `update.installs.<sha16>.channel` in that host's own config.yaml, where sha16 is the first 16 hex chars of the sha256 of the install path; then fetch and resolve the target ref live. A channel or target from an earlier report is stale evidence: a job reported both hosts on stable while both held main records and the main tip had already moved past the rebase base on both. On a new host, run `hermes update --set-channel main` from inside the install before the first gated update: the update wrapper runs the reapply hook after any `update` subcommand that exits 0, including one that pulls nothing, so the patch set must already fit the base the hook runs against.
- An unattended apply is plain `git apply` behind a `git apply --check` pre-pass over every pending patch, aborting with the tree untouched. `git apply --3way` in automation is not atomic: when the preimage blobs exist it stages what fits and leaves conflict markers on failure, when they do not it fails whole, so its outcome silently depends on the object db.
- A host on a different base commit gets `hermes update` first, then the hook. Never force a patch set built for one base onto a host at another.
- Drive remote hosts through `ssh <host> 'bash -lc "..."'` and run the same hook. Expect one `skipped:` or `applied:` line per patch.
- Verify the remote tree matches the laptop reference: sha256 every patch-touched path on both hosts, excluding the paths upstream commits in <base>..HEAD touched (`git log --format= --name-only <base>..HEAD | sort -u`). Mismatch count must be 0; the excluded count goes in the report.
- A remote `hermes update` while the tree is half-patched leaves a broken state: reset to the tracked base commit first (`git reset --hard <base>`), gate the patch set against the real origin tip in a detached worktree, then run the update. Never force the hook onto the half-patched tree.
- A remote dotfiles tree can diverge from origin (an unpushed rebase of the patch commits onto the host's current base). Back up first (`git -C ~/dotfiles branch backup/<host>-<sha>` on the remote), reverse-apply the OLD patches in the Hermes tree while dotfiles still sits on that old sha, then `git fetch origin && git reset --hard <rebased-sha>` (never pull), then fast-forward Hermes and run the hook. The reverse-apply must use the patch content at the old sha, not the rebased content.
- Patch-added test files that later patch generations dropped stay untracked after a reverse-apply. Before the fast-forward, list untracked paths, drop each one in the new header's deletion list, and keep the rest only after confirming they exist neither in the new upstream commit nor in the new patch set.
