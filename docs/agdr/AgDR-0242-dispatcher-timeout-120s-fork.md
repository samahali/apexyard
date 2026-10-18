---
id: AgDR-0242
timestamp: 2026-10-18T16:49:25Z
agent: claude
model: claude-opus-5-5
session: f8214bfc-b077-4631-8d6a-d917b9d40da9
trigger: user-prompt
status: executed
category: security
---

<!-- When this template creates an artifact, use the controlled technical writing profile in .claude/rules/writing-standard.md. -->

# Raise the dispatcher timeout to 120 seconds in this fork

> In the context of the fail-closed Bash dispatcher from AgDR-0228, facing a merge route that takes up to 23.5 seconds on Linux against a 20-second timeout, I decided to set the `dispatch-bash.sh` entry `timeout` to 120 seconds in this fork, to achieve merges that complete while the gate still fails closed, accepting that a hung gate holds a Bash call for up to 120 seconds and that this fork's `.claude/settings.json` differs from upstream.

## Context

- Parent record: AgDR-0228. It set `onFailure: "block"` and a 20-second `timeout` on the `dispatch-bash.sh` entry. It derived 20 seconds from a macOS p95 of 2748 ms on the merge route. This record supersedes only that timeout value, and only in this fork. AgDR-0228 stays unedited.
- On a Linux machine, every merge timed out at 20 seconds and failed closed. The PR #2 merge failed twice on 2026-10-18. Issue #3 records the investigation.
- The merge route calls `gh` 10 times in series across four merge gates. Each call takes 0.9 to 1.9 seconds. DNS lookup alone takes 0.24 to 0.77 seconds per call. The investigation ruled out `gh` auth, the keyring, the hook environment, hung calls, and a proxy.
- Five runs gave a four-gate total of 18.6 to 22.3 seconds. The always-on gates add up to 1.2 seconds. The recorded p95 is 23493 ms. The rule in `docs/perf/hook-fail-closed-profile-2026-10.md` is timeout >= 5 x p95, so the minimum is 117.465 seconds.
- `.claude/settings.json` is a trust-chain file. `onFailure` must stay `block`.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| Keep 20 seconds | Matches upstream. A hung gate blocks quickly. | Every merge on this machine fails closed. The operator must merge in the GitHub UI, outside the gates. |
| Set 120 seconds (chosen) | Meets the 5 x p95 rule. Merges complete. `onFailure` and every gate stay unchanged. One-line change. | A hung gate holds a Bash call for up to 120 seconds. The fork differs from upstream. The margin over the rule minimum is 2.2%. |
| Remove the 4 duplicate `gh` calls upstream | Fixes the cause. Saves about 7 seconds per merge for every adopter. | The change is in upstream hook code, outside this fork's control. A 7-second saving alone does not restore a 5 x p95 margin under 20 seconds. |
| Set a per-platform timeout | Keeps 20 seconds on fast machines. | `.claude/settings.json` has no per-platform field. It would need a wrapper or a generated settings file, which adds trust-chain code. |

## Decision

Chosen: **Set 120 seconds**, because it is the smallest change that makes merges complete on this machine and it meets the profile rule. It keeps the fail-closed property: `onFailure` stays `block`, every merge gate still runs, and a crash or a timeout still blocks the call. The other options either keep merges broken or need code outside this fork's control.

## Consequences

- Merges through Claude Code complete on this machine. A merge that waits longer than 120 seconds still fails closed.
- A hung gate holds any Bash call for up to 120 seconds before Claude Code blocks it.
- This fork's `.claude/settings.json` differs from upstream on one line. A later `/update` can raise a conflict on that line. Keep 120 seconds unless the upstream value is at least as large.
- The margin over the 117.465-second minimum is small. A slower re-profile can fail `test_settings_fail_closed.sh`. The next step is to remove the duplicate `gh` calls upstream, not to raise the timeout again.
- This record uses a fork-local ID. An upstream AgDR with the same number can appear later. Rename this file if `/update` reports a clash.

## Artifacts

- Issue: samahali/apexyard#3
- PR: samahali/apexyard#4
- Parent: `docs/agdr/AgDR-0228-fail-closed-dispatcher-and-ticket-first.md`
- Profile: `docs/perf/hook-fail-closed-profile-2026-10.md` § "Linux addendum"
