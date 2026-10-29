# Patch review chain

The chain for a patch-stack change: a fresh blind Kimi review of the combined stack diff against the live base. Only on PASS or PASS-WITH-NITS does Sol review the same diff. A wrapper run that produced no verdict is not a review round.

## Review tree

Run the review in a throwaway repo with neutral paths, for example `router/` and `skill/`, so the diff does not name the live files. The baseline commit holds a byte-copy of each live file. The brief's step 0 runs `cmp` of each tree file against its staged copy and stops on any mismatch. Pin the stack tip and say so in the brief; the child proves it with `rev-parse HEAD`.

## Wrapper contract

`hermes-adversarial-review` exit codes: 0 PASS or PASS-WITH-NITS, 1 FAIL, 2 run error, 3 isolation breach. Exit 2 means the model gave no verdict: fix the wrapper and rerun; the run does not count. An exit 3 with "no request dump found" means the serving gateway started without `HERMES_DUMP_REQUESTS=1`; dumps are written only when the variable is in the gateway's own environment. Check the daemon: `tr '\0' '\n' < /proc/<gateway-pid>/environ | grep -c '^HERMES_DUMP_REQUESTS=1$'` (pid from `pgrep -f 'gateway run'`). A count of 0 is deterministic, rerun never helps: stop and have the user restart the gateway with the variable set. Never kill the daemon. Only with the variable present is one rerun worth attempting. The verdict sits in the review file the wrapper writes, not in the launcher notice.

## Kimi round through the flatiron proxy

`hermes-adversarial-review` hard-codes `--provider openai-codex`, and a ChatGPT Codex subscription refuses Kimi (HTTP 400, model not supported). Run the Kimi round through a shim `hermes` placed earlier on PATH that rewrites the Kimi invocation to `--provider custom --base-url https://inference.flatironinstitute.org/v1 --api-key <token> --model moonshotai/Kimi-K3`, the token read from `~/.config/fi-llm-token` at exec time and never written into a brief.

`--api-key` on argv is visible in `ps` for the run's duration. Exporting the key's environment variable in the review client does not remove it: safe-mode turns resolve provider config inside the gateway, whose environment was fixed at its start, and secret-named variables are stripped from client_env, so the turn still sends the placeholder key and gets a 401. The clean route is to start the gateway with the key in its own environment, then drop `--api-key` from the shim.

## Launch check

The job log's first line is the verbatim launched command. Compare it with the briefed command before accepting the result. An unexpanded variable leaves a placeholder word in argv, the wrapper still exits 0, and the review ran the wrong model or defaults.

## Triage before fix

Every finding gets a read-only triage before any fix brief:

- Verdict: REAL, FALSE or PARTLY, with file:line, one quoted line, and a probe for every REAL.
- Origin: introduced by the last fix round (diff the pre-fix stack against the current one), pre-existing in the patches, or upstream. Read the upstream copy with `git show <upstream-sha>:<path>`.
- After a rebase onto a new upstream base, check whether the rebase introduced a finding by comparing its cited line between the old and new dotfiles versions of `local.patch` (`git -C ~/dotfiles show <old-sha>:<patch path>`). A text-identical line in both means the defect predates the rebase; report that, since it widens the blame window to every earlier review that passed it.
- Overlap with earlier review findings, by finding number.
- Impact class: A is a wrong result or data exposure in ordinary use; B needs an unusual setup; C is cosmetic or test-only.
- Fix size in lines and the route (plugin, widget or `local.patch`).

The fix round covers class A plus every finding the previous round introduced. The rest ships as documented limits.

A claimed race gets a forced-interleaving probe that drives the real entry points, not mocks. It prints expected, observed and match for each case. Serial-order controls before and after the window must deny. The report names which side reads whose data.

Probes run read-only against the live tree and make no model calls. When the code under test writes files (publish paths, session stores), run it with `HERMES_HOME` pointed at a scratch directory so no probe output lands in the real home.

## Fix round mechanics

- Back up first: `git branch backup/<name> HEAD`.
- Each fix is its own commit with subject `fixup! <subject of the commit it fixes>`. Commit with `--no-verify`.
- Fold with `GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash <upstream-base>`. The stack keeps its commit count, and no fixup subjects remain.
- Regenerate `local.patch` from `git diff --binary --full-index <upstream-base> HEAD` (SKILL.md, step 6 of the rebase). Report which files differ from the previous patch.
- Fit check: run the reapply hook on a fresh clone at the upstream base. `diff -r --exclude=.git --exclude=node_modules --exclude=__pycache__` against the stack tree must show 0 mismatches. Remove the clone after.
- Rerun the gates at the new tip: the pytest subset with the delegate release test file first, the TUI battery, and the triage probes. The race probe must now deny the window case.
