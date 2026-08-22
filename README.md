# RDAP — Raven Distributed Agent Protocol

**A2A agent teams authenticated by RVN1 (Raven protocol) identities.**

RDAP wires the [RAVEN](https://github.com/Ahmadreza-Arezehgar/RAVEN) protocol
reference (`raven_protocol`) into Google's **A2A (Agent-to-Agent)** SDK so a
team of AI agents running on different machines can delegate tasks to each
other with cryptographic proof of who sent what.

```
┌──────────────────┐   signed A2A task    ┌──────────────────┐
│  agent "raphael" │ ◄──────────────────► │ agent "donatello"│
│  rvn1qyx0…uxp9   │     (Ed25519/RVN1)   │  rvn1qyw5…fpng   │
└────────┬─────────┘                      └────────┬─────────┘
         │            git-synced shared repo       │
         └──────────────► .team/ ◄─────────────────┘
                          BOARD.md   (task board)
                          journal.md (event log)
                          facts.md   (shared memory)
                          locks/     (file claims)
                          keys/      (device_ed25519.seed, chmod 600)
```

## How it works

1. **Identity** — every node owns an Ed25519 device key. Its RVN1 address is
   derived with the same bech32m + fingerprint rules as RAVEN
   (`rvn1q…` / `rvn1:XXXX-XXXX-…`).
2. **Delegation** — a delegating client signs the canonical task bytes
   (`raven.a2a.delegation.v1` context, length-prefixed per `_canon.lp`) and
   attaches the signature to the A2A message metadata.
3. **Verification** — the receiving executor checks sender address against its
   trusted-peers policy, rejects stale timestamps (>300 s skew) and invalid
   signatures *before* any work happens.
4. **Coordination** — agents share one git repo: board, journal, facts and
   advisory file locks keep humans (and other agents) in the loop.
5. **Brains** — `provider=openai` runs a ReAct-style tool loop against any
   OpenAI-compatible endpoint; anything else uses the deterministic keyless
   EchoBrain (perfect for CI and demos).

## Quickstart (wizard — 4 commands total)

```bash
git clone https://github.com/Ahmadreza-Arezehgar/raven-distributed-agent-protocol rdap
cd rdap
./rdap init        # asks your agent's name once → prints your INVITE line
```

Send your `RDAP1 …` invite line to a teammate (iMessage, AirDrop, whatever),
paste theirs into:

```bash
./rdap trust 'RDAP1 donatello rvn1q… efd8b6…'
./rdap start       # auto-detects LAN IP, picks a free port, enforces signatures
./rdap ask "build the login API"
```

That's it. `init` and `trust` are one-time per machine; day-to-day is just
`start` + `ask`. State lives in `rdap.json`, `peers.json` and `team-repo/`
next to the script; private keys never leave `team-repo/.team/keys/`
(gitignored, chmod 600).

## Working as a team (group chat + unified goal)

```bash
# the ONE mission every agent serves:
./rdap goal "Build the Raven demo app: login page, settings screen, tests"

# group chat — @tag an agent, or @all to broadcast:
./rdap say "@raphael build the login API endpoint"
./rdap say "@donatello review the login flow"
./rdap say "@all standup now!"

# read the shared thread (synced between machines via git):
./rdap chat

# pick up answers that arrived while you were offline:
./rdap replies
```

Every delegated task is framed by the TEAM GOAL before it reaches a brain,
and agents post their results back into the shared thread.

## Choosing brains (open-source models)

```bash
./rdap model                 # interactive menu (local Ollama / Groq / OpenRouter)
./rdap model openai llama3.2 --base-url http://localhost:11434/v1   # Ollama
brew install ollama && ollama serve & && ollama pull llama3.2        # runtime
```

## Discovery & mesh transports

| Command | What it does |
|---|---|
| `./rdap discover` | find nearby agents on this LAN via mDNS |
| `./rdap discover --trust 1` | auto-trust (fetches identity incl. mesh address) |
| `./rdap ping <url>` | check reachability of a node |
| `./rdap invite --port 9001` | print your invite with URL for remote peers |
| `./rdap mesh-build` | build the raven-swarm mailbox binary (needs Rust once) |

Transport ladder per message (automatic): **T1** direct A2A → **T3** Raven
swarm mailbox (libp2p) → **T4** git relay. Internet down but Wi-Fi alive?
T1 still works over LAN. Peer fully offline? T4 holds the signed task until
they sync, then it drains automatically.

### Advanced CLI

<details>
<summary>python -m team_agents …</summary>

```bash
# identity of this machine's agent
.venv/bin/python -m team_agents id --keys-dir ./repo/.team/keys

# run a node that only accepts signed tasks from trusted peers
.venv/bin/python -m team_agents serve --name raphael --role "backend" \
    --port 9001 --repo ./repo --require-signed --peers peers.json

# delegate a signed task from another repo/machine
.venv/bin/python -m team_agents send --url http://127.0.0.1:9001 \
    --text "implement the login endpoint" \
    --keys-dir ./other-repo/.team/keys
```

`peers.json` maps RVN1 addresses to Ed25519 public keys:

```json
{
  "rvn1qyw5g05kce8xtjmnvhxynckxejy0s3k5ducvfpng": "355b7184…5039"
}
```

or with aliases:

```json
{ "donatello": { "address": "rvn1qyw5…", "pubkey": "355b71…" } }
```

</details>

## Layout

| Path | Purpose |
|---|---|
| `team_agents/server.py` | A2A server: Agent Card, JSON-RPC, `/raven/identity`, bearer auth |
| `team_agents/executor.py` | Task lifecycle + RVN1 delegation verification |
| `team_agents/raven_identity.py` | Key management, signing, trust policy |
| `team_agents/client.py` | Signed delegation client (uses the A2A SDK client) |
| `team_agents/memory.py` | Git-backed board/journal/facts/locks |
| `team_agents/tools.py` | ToolBox exposed to the LLM brain |
| `protocol/reference/raven_protocol` | Vendored copy of the RVN1 reference — source of truth lives in [RAVEN](https://github.com/Ahmadreza-Arezehgar/RAVEN) |

## Security notes

Private keys never leave `<repo>/.team/keys/device_ed25519.seed` (mode 600)
and are gitignored by default. Signatures cover the exact task text; any
mutation invalidates them. Replay window is bounded by the ±300 s timestamp
check. For hostile networks add nonces or run behind mTLS/bearer auth
(`--token`).

## Configuration

Zero-config by default. Customize via environment variables:

| Variable | Default | Purpose |
|---|---|---|
| `RDAP_HOME` | `~/rdap` | root for state, keys, `bin/`, cloned sources |
| `RDAP_POLL` | `20` | seconds between mesh/git inbox drains (`./rdap start --poll 8`) |
| `RDAP_SWARM_BIN` | auto-discovered | explicit path to the raven-swarm mailbox binary |
| `NO_COLOR` | unset | disable all terminal colors |

Any OpenAI-compatible endpoint works as a brain — set it with
`./rdap model <provider> <model> --base-url <url>` (OpenRouter, Groq,
vLLM, LM Studio, Ollama …).

## Status

Lab-ready. Multi-node delegation, rejection paths and git-backed memory are
covered by end-to-end smoke tests on macOS (Python 3.14, a2a-sdk 1.1.2).

Licensed under AGPL-3.0, matching the RAVEN core.
