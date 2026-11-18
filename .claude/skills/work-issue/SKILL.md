---
name: work-issue
description: Work ONE issue of NeoTech/agent-swarm end to end - check the claim, claim it, branch from the latest main, fix it with a pinning test, run the tests, push and open a draft PR, move the issue to in-review. Use when you pick an issue to build.
---

# work-issue

The rules are in `AGENTS.md` ("Issue Law", "Pull Request Law"). If this skill and `AGENTS.md` disagree, `AGENTS.md` is correct. You never merge and never review your own work.

## Steps

1. **Pick.** `gh issue list --state open --limit 100 --search "-label:claimed -label:blocked -label:needs-research" --json number,title,labels,createdAt`. Take the lowest `P*` number, then the oldest. An issue without a `P*` label is not triaged: set one first.
2. **Read the issue and all its comments.** `gh issue view N --comments`. Read the design comment it names ("Research n/7" in #11).
3. **Check the claim.** A live `claimed` label or a claim comment less than 4 hours old means the issue is taken. Pick the next one.
4. **Claim.** `gh issue edit N --add-label claimed --add-assignee @me`, then comment `claimed: <agent> · <UTC time> · branch issue/N-<slug> · plan: <one line>`.
5. **Branch.** `git fetch origin && git switch -c issue/N-<slug> origin/main`. In a shared checkout use a worktree: `git worktree add --no-track -b issue/N-<slug> ../wt-N origin/main`.
6. **Write the test first.** The test fails for the right reason before you change code. Keep each file under about 400 lines.
7. **Fix.** Handle errors. No `unwrap()` in Rust outside tests. Validate input at the edge.
8. **Run the touched tests.** Not the whole suite. See the `cloudflare-broker` skill for the commands. Two failures of the same step: stop and escalate (step 11).
9. **Record decisions.** Any choice you made alone goes in `docs/decisions/N-<slug>.md`.
10. **Push and open a draft PR.** `git push -u origin issue/N-<slug>`, then `gh pr create --draft --base main --head issue/N-<slug> --title "<conventional subject>" --body-file - <<'EOF' ... EOF`. The body, in ASD-STE100, says what changed, the pinning tests, what was not exercised, and ends with `Fixes #N`. Then replace `claimed` with `in-review` and comment `fix on issue/N-<slug> · PR #M · pinned by <tests>`.
11. **Escalate, never silently.** If the work is bigger than its `effort:*` label, comment what you learned and where held work is, raise the label one step, remove `claimed`. One escalation for each run.

## Never

- Merge, mark a PR ready, force-push, or commit to `main`.
- Put a token, the admin secret or a Cloudflare API token into any file, comment or log.
- Close an issue by hand without a commit.
