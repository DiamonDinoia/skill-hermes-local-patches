# Host notes for the local Hermes patch set

Recorded 2026-10-20 during the statusbar/resume restore on this host. Extends `patch-review-chain.md` with host-specific facts.

## What breaks when `hermes update` runs

- ~/.hermes/hermes-agent is a git checkout. `hermes update` hard-resets it to upstream and wipes every local patch. On 2026-10-20 both symptoms appeared at once: the statusbar widget `quotabar.mjs` lost its segments, and resume of old sessions failed with `invalid_params (base_url: must be an http(s) URL without credentials, query or fragment)`.
- Fix per chain: reapply the patches, restart the gateway and TUI (patched code loads only then), then run the review chain before pushing the follow-up dotfiles commit.
- As of 2026-10-20 the patch set is 54 modified + 5 untracked files on upstream base `bd3cbe91a90097f9804f16ea5c1768f5cbaaf0b7`. combined.diff and review-request.md live in ~/.hermes/cache/scratch/patch-review/ (scratch prunes after 7 idle days, so regenerate rather than rely on them).

## Combined diff

- Build it from clean state each run: `git diff HEAD`, then for each untracked patch-added file append `git diff --no-index /dev/null <file>`, processed in `git status --short` order. Regenerate and `cmp` against a saved combined.diff before reusing it.

## Kimi routing (the wrapper cannot reach Kimi by itself)

- hermes-adversarial-review hard-codes `--provider openai-codex`. ChatGPT/Codex rejects kimi-k3: `The 'kimi-k3' model is not supported when using Codex with a ChatGPT account.`
- Working route: model id `moonshotai/Kimi-K3` on the flatiron gateway, through custom provider flags `--provider custom --base-url https://inference.flatironinstitute.org/v1 --api-key <contents of ~/.config/fi-llm-token>`. `hermes-dispatch-route models` lists the exact IDs; `latest` has no kimi family.
- The wrapper takes no provider flag, so route it through a scratch PATH shim named `hermes` that rewrites the Kimi argv. Keep the token out of argv (visible in `ps`): pass it by environment variable if the custom provider supports one; if not, accept the brief exposure and say so in the report.
- Kimi reasoning effort should be max (the config's intended effort).

## The wrapper's request-dump gate

- The wrapper's isolation check fails closed without a request dump: exit 3, `no request dump found`. A run with no verdict is not a review round, so Sol must not run. Rerun once per the chain, then stop.
- Dumps exist only when the RUNNING gateway has HERMES_DUMP_REQUESTS=1. The TUI spawns the gateway as `gateway run --quiet --idle-exit`; it inherits the env of the `hermes` launch and exits on idle once every TUI closes.
- Restart with dumps on: close every TUI, `pgrep -af 'gateway run'` (kill the pid only if it lingers), then relaunch with `HERMES_DUMP_REQUESTS=1 hermes`. Confirm: `tr '\0' '\n' < /proc/<pid>/environ | grep -c '^HERMES_DUMP_REQUESTS=1$'` prints 1. A second pgrep hit that is the caller's own shell is not a gateway.
- Dumps write every provider request to disk. Restart Hermes without the variable once the review is done.
- Hand the user the restart commands. The agent never restarts or kills the user's processes.

## Verifying a resume non-interactively

- Run `hermes -r <session_id>` inside a scratch tmux pane, never in the user's TUI: `tmux new-session -d -s resume-check -x 200 -y 50 "bash -lc 'hermes -r <id>'"`. Poll `tmux capture-pane -p -t resume-check` until the transcript renders, then `tmux kill-session -t resume-check`. Check only the new lines in ~/.hermes/logs/gateway.log for `session.resume` and `invalid_params`.

## Open items

- quotabar.mjs renders below the input box although config.yaml says `display.statusbar: top`; the rollout spec placed this widget at the bottom. Unconfirmed which the user wants.
- Give the wrapper `--review-provider/--review-api-key/--review-base-url` (or a HERMES_REVIEW_PROVIDER env var) so the PATH shim is unnecessary. Decision rests with the user.
