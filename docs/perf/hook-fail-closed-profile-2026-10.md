# Hook timing profile for the fail-closed gates: October 2026

## Scope and method

This profile supports me2resh/apexyard#1623.
It sets the `timeout` value for two hook entries in `.claude/settings.json`:
the `dispatch-bash.sh` entry and the `require-active-ticket.sh` entry.
Both entries also set `onFailure` to `block`.
See `docs/hooks-fail-closed.md` for the behavior.

The host is macOS 27.0.1 on arm64, with `/bin/bash` 3.2.57.
Claude Code is version 2.1.296.
The repository base is `db03eb0` on `dev`.
All times are in milliseconds.
p50 is the median. p95 is the 95th percentile.
The percentile method is nearest rank:
sort the n samples and take the value at rank ceil(p x n).
`bin/hook-timing-profile.py` computes the numbers.

### Source 1: real Claude Code sessions with `--debug-file`

`claude --debug-file <path>` records one line for each hook run:

```text
Hooks: PreToolUse:Bash <command> finished with status 0 (547ms)
```

The duration in that line is the wall time of the complete hook command,
including the `bash -c` wrapper from `.claude/settings.json`.
This source is the real Claude Code measurement.

Two headless sessions ran in the repository with `claude -p --model haiku`.
The prompt told the model to make one tool call at a time, never in parallel.
The Bash session made 112 calls from eight read-only commands.
The Write session made 64 writes of new files to a path that the ticket gate
does not exempt, so each call took the full gate path.
An active ticket marker was present.

`bin/hook-timing-profile.py parse <debug log> <script>` read the two logs.

### Source 2: a faithful harness for routes that a short session does not reach

A model session cannot safely run `git push` or a merge.
For those routes, `bin/hook-timing-profile.py run` pipes a payload into the
exact wrapper command from `.claude/settings.json` and times each call.
The command in the payload is never run.
Only the hooks see it.
This harness measures the same wrapper and script as Claude Code.
It does not include the time that Claude Code spends to start the hook.
The merge and PR routes call the tracker over the network, so they vary most.

## Results

| Hook | Route | Source | n | p50 | p95 | max |
|------|-------|--------|---|-----|-----|-----|
| `dispatch-bash.sh` | Bash, read-only commands | `--debug-file` | 112 | 534 | 577 | 872 |
| `require-active-ticket.sh` | Write, in-tree path | `--debug-file` | 64 | 192 | 207 | 223 |
| `require-migration-ticket.sh` | Write, in-tree path (same entry group, for context) | `--debug-file` | 64 | 122 | 138 | 156 |
| `dispatch-bash.sh` | `echo hello` | harness | 30 | 497 | 548 | 569 |
| `dispatch-bash.sh` | `git status --short` | harness | 30 | 503 | 589 | 609 |
| `dispatch-bash.sh` | `git commit -m ...` | harness | 30 | 762 | 834 | 2184 |
| `dispatch-bash.sh` | `git push origin HEAD` | harness | 30 | 928 | 1001 | 1354 |
| `dispatch-bash.sh` | `gh pr create ...` | harness | 30 | 1956 | 2226 | 2309 |
| `dispatch-bash.sh` | `gh pr merge N --squash` | harness | 30 | 2607 | 2748 | 2883 |

The ticket gate runs inside `dispatch-bash.sh` for every Bash call.
So the `dispatch-bash.sh` rows already include the ticket gate time on the Bash path.

### Linux addendum (2026-10-18, samahali/apexyard#3)

A Linux machine (kernel 6.8, bash, `gh` 2.102.0) hit the 20 second timeout on every merge.
The merge route there calls `gh` 10 times in series.
Each call took 0.9 to 1.9 seconds, because DNS lookup alone took 0.24 to 0.77 seconds.
The table shows 5 runs of the merge route against one merged PR.
The four merge gates ran directly with the payload, and the `gh` calls were real.
The always-on part is the harness `git status --short` route on the same machine.

| Part | n | Min | Max |
|------|---|-----|-----|
| Always-on gates (`git status --short`, p50 1190) | 5 | - | 1213 |
| `block-unreviewed-merge.sh` | 5 | 3730 | 5203 |
| `require-design-review-for-ui.sh` | 5 | 3709 | 4644 |
| `block-merge-on-red-ci.sh` | 5 | 6919 | 8499 |
| `require-architecture-review.sh` | 5 | 3881 | 4530 |
| Four merge gates, total per run | 5 | 18642 | 22280 |
| Merge route, worst run plus always-on p95 | 5 | - | 23493 |

With n = 5, the p95 is the worst run: 23493 ms.
This does not include Claude Code's own hook start time.

## Timeout rule and chosen values

The rule is: timeout >= 5 x p95, and never under 10 seconds.
Use the slowest route p95 of the entry, not the median route.
The fail-closed gate blocks the call when the timeout ends it.
A timeout that is too short would block valid merges.

| Entry | Slowest p95 | 5 x p95 | Floor | Chosen timeout |
|-------|-------------|---------|-------|----------------|
| `dispatch-bash.sh` (Bash matcher) | 23493 (merge route, Linux addendum) | 117465 | 10000 | 120 seconds |
| `require-active-ticket.sh` (Edit, Write, MultiEdit matcher) | 207 | 1035 | 10000 | 10 seconds |

The `dispatch-bash.sh` value was 20 seconds, from the macOS p95 of 2748 ms.
The Linux addendum p95 of 23493 ms is the slowest route recorded, so the rule now uses it.
The `dispatch-bash.sh` value is 120 seconds, above the 117.47 second minimum.
A merge that waits longer than 120 seconds for the tracker fails closed.
A hung gate holds a call for up to 120 seconds before Claude Code blocks it.
The operator can retry it.

`.claude/hooks/tests/test_settings_fail_closed.sh` reads the lines below and
checks the two timeouts in `.claude/settings.json` against them.

```text
profile: dispatch-bash.sh p95_ms=23493
profile: require-active-ticket.sh p95_ms=207
```

## Limits of this profile

- The main table comes from one macOS machine. The Linux addendum comes from one Linux machine with n = 5.
- The harness routes do not include Claude Code's own hook start time.
- A sandbox without network access makes the tracker routes fail fast, not slow.
- A hook speed-up is outside this profile. Issue #1613 tracks hook speed-ups.
  A speed-up lowers the p95 and does not require a timeout change.
