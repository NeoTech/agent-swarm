---
name: review-pr
description: Review ONE open pull request of NeoTech/agent-swarm against AGENTS.md - read the diff, run the touched tests, and post a plain review comment with the head sha. Use when asked to review a PR. Never approves or merges.
---

# review-pr

You did not write the change. You review one PR at a time. You never approve, merge, push or change labels other than the review label.

## Steps

1. `gh pr view M --json headRefOid,baseRefName,isDraft,mergeable,files,body,closingIssuesReferences`. Note the head sha.
2. `gh pr diff M`. Read all of it.
3. Check `AGENTS.md`:
   - Prime Directive: no lost message, no double lease, no secret in logs, fixtures or tests.
   - Errors handled: no `unwrap()` / `expect()` / `panic!()` in non-test Rust, no `any` on a message boundary, input validated at the edge.
   - The PR has a test that fails without the fix.
   - Files stay under about 400 lines. Each new module has a provenance line.
   - A new dependency is listed in the body with its licence.
   - A decision made alone has a `docs/decisions/` file.
   - Text on GitHub is ASD-STE100. The body ends with `Fixes #N` or `Refs #N`.
   - No new Actions trigger.
4. Check out the head in a worktree and run the touched tests (see the `cloudflare-broker` skill). Say plainly what you did not run.
5. Post one plain comment. The first line is `<!-- pr-reviewer: <full head sha> -->`. Then: findings ordered by severity, each with `path:line`; what you ran; what you did not run; the verdict `clean` or `changes requested`.
6. A review is valid for that head only. A new head needs a new review. A merge of `main` with no hand edits keeps the review.

## Blockers (always request changes)

- A secret in the diff or in a log line.
- A path where an accepted message can be lost, or two agents can hold one lease.
- A panic that a message from the network can start.
- A missing test for a changed behaviour.
