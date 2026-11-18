# AGENTS.md

## What This Is

`agent-swarm` builds `coor`: a small coordination broker for concurrent coding
agents. It has two parts:

- **The broker.** A TypeScript Cloudflare Worker with one Durable Object for
  each channel. It keeps a message log and streams messages over WebSocket.
  It also answers on HTTPS.
- **The CLI.** A Rust program, `coor`. It runs in two modes: `coor monitor`
  (long-running, one line for each message) and `coor send <message>` (post
  one message). `coor install` deploys the broker to the user's own
  Cloudflare account.

The first customer is NeoTech/Camcraft. Its agents (coordinator, workers,
scouts and the review session) must claim issues, hold the review lock and
send messages to each other without the user as relay. Camcraft stays
the record: leases mirror to GitHub claims and labels, never the other way.

The work started as NeoTech/Camcraft#1422. The research is in
NeoTech/agent-swarm#11 (comments "Research 1/7" to "7/7"). Read it before you
change a design decision. `coor` is the working name. The maintainer decides
the final name (#1).

This repository is separate from Camcraft. It has no xtask, no Camcraft gate
and no Camcraft release. Camcraft rules do not apply here unless this file
says so.

## How to read this file

This file holds the rules. A procedure that one task uses is in a skill in
`.claude/skills/`. Load the skill (Skill tool) before you do the task. If a
skill and this file disagree, this file is correct.

| Task | Skill |
|---|---|
| Read, file, label, comment on or close an issue | `gh-issues` |
| Work one issue from claim to draft PR | `work-issue` |
| Review a pull request | `review-pr` |
| Build, test or deploy the Worker or the CLI | `cloudflare-broker` |

## Autonomous Work Law

The maintainer reviews later. The work does not interrupt him.

- Work the open issues of this repository (Issue Law). Do not ask for a
  decision that has a recommended option: take it and record it.
- Record each decision taken alone in `docs/decisions/<issue>-<slug>.md`:
  what was chosen, why, and what was dropped.
- A decision that only the maintainer can make is a comment on the issue that
  starts with `ASK:`. Label the issue `blocked`. Take the next issue.
- The decisions that wait for the maintainer now: the repository and tool
  name (#1), the transport choice (#2), the Cloudflare account and the admin
  token holder (#4), the GitHub App and its secrets (#8).
- Subagents do testing and review. Never give one agent both a code change
  and the verification of it.

## Issue Law

Work is tracked as GitHub issues of this repository, read and written with
`gh`.

- **Pick** an open issue without `claimed`, `blocked` and `needs-research`.
  Take the lowest `P` number first, then the oldest.
- **Priorities.** `P1` is unsafe or wrong behaviour: lost or duplicated
  messages, a token leak, a lease that two agents hold. `P2` is a broken main.
  `P3` is wrong behaviour with a workaround. `P4` is work the maintainer asked
  for. `P6` is an accepted enhancement. `P7` is polish. An agent that files an
  issue sets exactly one `P*` label, one `effort:*` label and `source:agent`.
- **Effort labels.** `effort:low` is one file with no new logic.
  `effort:medium` is one module with a known pattern. `effort:high` is
  several modules or new behaviour with tests. `effort:max` is a new protocol
  or open design. Anything that changes auth, tokens or message delivery is
  `effort:high` or more.
- **Claim before any code.** Run `gh issue edit N --add-label claimed
  --add-assignee @me`. Then comment `claimed: <agent name> · <UTC time> ·
  branch <name> · plan: <one line>`. Do not work an issue that has `claimed`
  unless the claim is more than 4 hours old and the claimant made no later
  comment. Then comment `reclaimed from <old claimant> (stale)`.
- **One issue, one branch**, `issue/N-<slug>`, from the latest `main`. Never
  commit to `main` for an issue. Never force-push.
- **Report on the issue.** A finding, a held patch, a blocker or a decision
  goes into an issue comment. Blocked work: replace `claimed` with `blocked`
  and say what blocks it.
- **Giving up.** Remove `claimed`, comment what you learned, name a pushed
  branch that holds work, and leave the issue open.
- **Never close by hand without a commit.** The exceptions are a duplicate,
  work already done (name the commit) and a decided won't-do (give the
  reason).
- **Language.** Write each issue, comment and PR description in ASD-STE100
  Simplified Technical English: short sentences, active voice, one word for
  one meaning, no idioms. Do not change code, commands, paths or log output.
- **Cross-repository references** use the full form: `NeoTech/Camcraft#681`.
  A bare `#N` means this repository.

## Pull Request Law

- Done on a branch means pushed with a PR, never merged by its author.
  Open it as a draft: `gh pr create --draft --base main --head
  issue/N-<slug>`. The body says what changed, the tests that pin it, what
  was not exercised and ends with `Fixes #N` (`Refs #N` for part work).
- A reviewer that did not write the change reviews each PR. Merges are merge
  commits only (`gh pr merge M --merge --delete-branch`). Never squash or
  rebase.
- `main` takes changes only through a PR once the CI workflow exists. Until
  then the maintainer sets the branch rule.
- Never add a GitHub Actions trigger (`push`, `pull_request`, `schedule`)
  without the maintainer's approval. Budget the billed minutes.

## Prime Directive: Never Lose, Never Duplicate, Never Leak

A broker that agents trust must be correct or visibly refuse.

- **No lost message.** A message accepted by the broker is in the log with a
  sequence number before the sender gets an answer. `coor monitor` remembers
  its position and replays from it after a restart.
- **No double lease.** At any time at most one agent holds the lease of an
  issue. A lease expires when its heartbeat stops. The expiry time comes from
  the broker clock only.
- **No secret in the log.** Tokens, the admin secret and Cloudflare API tokens
  never appear in a message, a log line, an issue, a PR or a test fixture.
  Store them only in `~/.coor/credentials` (file mode 0600) or in Worker
  secrets.
- Handle errors or propagate them. In Rust, `unwrap()`, `expect()` and
  `panic!()` are forbidden outside tests. In TypeScript, no unhandled promise
  and no `any` on a message boundary. Validate every message at the edge.
- Degenerate input (huge message, bad UTF-8, replayed token, unknown channel)
  gives a clean error. It does not crash or hang.
- Log diagnostics. Never hide them.

## Layout

The layout is decided in issue #1. Until then, use this plan:

| Path | Content |
|---|---|
| `worker/` | TypeScript Worker, Durable Object, `wrangler.toml` for local tests only |
| `cli/` | Rust crate `coor`: `monitor`, `send`, `install`, `next` |
| `docs/decisions/` | decision records |
| `docs/protocol.md` | the message and lease wire format |
| `.claude/skills/` | the skills of the table above |

- **400-line files.** No source file grows past about 400 lines. Split it
  along the domain seam.
- **Stays buildable everywhere.** The CLI builds on Windows, Linux and macOS
  from the same source. No platform-specific code in core paths.
- The installer uses the Cloudflare REST API. It does not need `wrangler`
  (NeoTech/agent-swarm#4). This is not verified yet: prove it with a test
  against a throwaway account before you rely on it.

## Testing

- Each module has its own tests. A bug fix has a test that fails before the
  fix.
- The Worker is tested with Miniflare or `workerd` locally. No test talks to a
  real Cloudflare account unless the issue says so and the account is a
  throwaway account.
- A test that needs the network is `#[ignore]`d (Rust) or tagged `integration`
  (TypeScript), and runs only on request.
- Test the change, not everything: run the tests that pin the fix first.
- Property and fuzz tests cover the message parser and the lease table.

## Commits

[Conventional Commits](https://github.com/qoomon/conventional-commits):
`<type>(<scope>): <subject>`. Types: `feat`, `fix`, `perf`, `refactor`,
`test`, `docs`, `build`, `ci`, `chore`. A breaking change says so (`feat!:`).
The body of the fixing commit ends with `Fixes #N`.

Versioning is semantic. The maintainer starts a release and a tag. Agents do
not.

## Clean-Room and Licence

- Use only public documentation and standards: Cloudflare docs, RFC 6455
  (WebSocket), ZWS (RFC 45) and the GitHub API docs.
- Do not copy the code of another coordination tool. Each module has a
  one-line provenance comment: what it implements and from which public
  source.
- Add a dependency only when the issue needs it. List each new dependency in
  the PR body with its licence.

## Privacy

Treat information about the user and their projects as private. Never put a
name, email address or project detail into a request to an outside service,
a header, a URL or a payload. Use a generic client name.
