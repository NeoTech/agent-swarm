---
name: cloudflare-broker
description: Build, test and deploy the coor broker (TypeScript Cloudflare Worker with Durable Objects) and the Rust coor CLI of NeoTech/agent-swarm. Use before you run a test, build the Worker, or deploy to a Cloudflare account.
---

# cloudflare-broker

The design is in NeoTech/agent-swarm#11 ("Research 1/7" to "7/7"). The layout plan is in `AGENTS.md`. Update this skill when issue #1 fixes the layout.

## Design facts (verify against the research comments)

- One Durable Object for each channel (a channel is a name such as `NeoTech/Camcraft`). It stores the message log with a sequence number, and holds the WebSocket connections.
- A Worker cannot accept inbound public TCP. The transport is WebSocket plus HTTPS. ZWS (RFC 45) is a later option.
- `coor monitor` prints one line for each message, remembers the last sequence number in `~/.coor/`, and replays after a restart. `--once` exits on the first message.
- Claude Code sessions receive the lines through the Monitor tool. Restart it at least every 30 minutes.
- Settings are in `~/.coor/config.toml` (Worker address, channel). Secrets are in `~/.coor/credentials`, file mode 0600. Never print them.
- The free Workers plan is enough for about 10 agents (an estimate, not measured).

## Commands (adjust to the real layout)

```bash
# Worker: local tests, no Cloudflare account needed
cd worker && npm ci && npm test          # Miniflare / workerd
cd worker && npx tsc --noEmit            # type check

# CLI: touched tests only
cargo test -p coor --lib                 # unit tests
cargo test -p coor --test <name>         # one integration test
cargo clippy -p coor --all-targets -- -D warnings
```

## Deploy

Deploy only when the issue says so and the maintainer named the account (#4). Never deploy from an agent session to the maintainer's production account without an `ASK:` answer.

- `coor install` calls the Cloudflare REST API with an API token the user supplies. The token goes into `~/.coor/credentials` or a Worker secret. It never goes into a log, an issue or a PR.
- The upload of a Durable Object Worker through the REST API without `wrangler` is not tested yet. Prove it against a throwaway account first and record the result on issue #4.
- Use a separate throwaway Worker name and channel for each test run. Delete them at the end.

## Never

- Put a token, the admin secret or a Cloudflare API token into a file that is committed, into a test fixture or into output.
- Run a test against the maintainer's account.
- Add a GitHub Actions trigger (`push`, `pull_request`, `schedule`) without the maintainer's approval.
