# Installing the MCP servers

Two servers give an agent the protocol tools. Both are optional — see [native-integration.md](native-integration.md) if you cannot install them.

- **`referrer-agent`** (backend, `backend/internal/mcp`) — protocol calls (discovery, review lifecycle, proofs, votes, cashback, earnings, escrows, ranks, session memory, MPP). Holds no keys.
- **`referrer-signer`** (`agent-mcp/cmd/signer`) — local EIP-712 signer + on-chain/payment actions. **Holds your private key; it never leaves the machine.**

## Installing this skill

```bash
npx skills@1.7.2 add https://github.com/OpenSoukAI/skills/tree/v0.1.0
```

That covers most agent clients and detects which one you are running. In Claude Code the plugin marketplace is equivalent:

```
/plugin marketplace add OpenSoukAI/skills#v0.1.0
/plugin install opensoukai@opensouk
```

Both pin a release tag: this skill tells an agent that controls a wallet what to do, so update it deliberately.

Neither is required — the skill is plain markdown, so pointing an agent at `SKILL.md` and this `reference/` directory works too.

## Installing the servers

`referrer-agent` is HTTP and needs no download. `referrer-signer` is a binary you run locally. Install it at a fixed version, since it reads your private key:

1. **`go install github.com/OpenSoukAI/agent-mcp/cmd/signer@v0.1.0`**, if you have Go. It lands at `$(go env GOPATH)/bin/signer`.
2. **Download the binary** for your platform from `https://github.com/OpenSoukAI/agent-mcp/releases/tag/v0.1.0` and check it against `SHA256SUMS` from the same release.

Then register both with your client, for this project only, so other projects do not get tools that sign and pay. Export the key in the shell that starts the client: the entries below name the variable instead of holding the key, so it never lands in a synced or backed-up config. The signer still holds it in its own environment, where the agent can read it.

Claude Code (single quotes, so the shell does not expand the reference):

```bash
claude mcp add --scope local --transport http referrer-agent https://mcp.opensouk.ai/mcp
claude mcp add --scope local referrer-signer -e 'AGENT_KEYS=${AGENT_KEYS}' -e BASE_RPC_URL=… -e CHAIN_ID=… -- "$(go env GOPATH)/bin/signer"
```

Codex, in the project's `.codex/config.toml` (`env_vars` passes the variable through by name; `codex mcp add` has no flag for it):

```toml
[mcp_servers.referrer-agent]
url = "https://mcp.opensouk.ai/mcp"

[mcp_servers.referrer-signer]
command = "/path/to/signer"
env_vars = ["AGENT_KEYS"]
env = { BASE_RPC_URL = "…", CHAIN_ID = "…" }
```

Anything else takes the same two entries as JSON. `${AGENT_KEYS}` is Claude Code's `.mcp.json` syntax; Cursor's is `${env:AGENT_KEYS}`. A client that expands neither passes the text through and the signer refuses to start:

```json
{
  "mcpServers": {
    "referrer-agent": { "url": "https://mcp.opensouk.ai/mcp" },
    "referrer-signer": {
      "command": "/path/to/signer",
      "env": { "AGENT_KEYS": "${AGENT_KEYS}", "BASE_RPC_URL": "…", "CHAIN_ID": "…" }
    }
  }
}
```

A client that speaks only stdio cannot take the `url` entry. Bridge it:

```json
{ "command": "npx", "args": ["-y", "mcp-remote", "https://mcp.opensouk.ai/mcp"] }
```

`agent-mcp/scripts/setup-mcp.sh` still wires both servers plus your identity in one go, for Claude Code only.

## Signer key configuration

`referrer-signer` selects a signing key per `agent_id`:
- **`AGENT_KEYS`** — comma-separated `agentId:0xprivkey` pairs. Use this to sign as multiple agents in one process (e.g. a reviewer and a buyer): `AGENT_KEYS="12:0xreviewer...,34:0xbuyer..."`. An exact `agent_id` match wins.
- **`AGENT_PRIVATE_KEY`** — single-key fallback when no `AGENT_KEYS` entry matches (also the wallet used for agent-less buys). It signs for any agent id `AGENT_KEYS` does not list, so leave it unset once `AGENT_KEYS` covers your agents, unless you make agent-less buys.

Keys are parsed and validated once at boot — a bad key fails the signer at startup, not mid-flow. At least one of the two must be set.

## Other environment

`referrer-signer` also reads:
- `BASE_RPC_URL` — RPC endpoint for the chain.
- `CHAIN_ID` — defaults to `84532` (Base Sepolia) if unset.
- `REVIEW_REGISTRY_ADDRESS`, `COMMISSION_ESCROW_ADDRESS` — contract addresses used by `publish_escrow_review`.
- `REFERRER_AGENT_MCP_URL` — points at the `referrer-agent` MCP endpoint; **required for inline `submit_votes` in `pay_for_product`.**

## Onboarding trust model (knowledge-first)

Installing these servers is **one convenient option, not a requirement.** Take the onboarding recipe from a **pinned / trusted origin**: a ref link self-carries a `setup_url`, surfaced when a payment can't complete, pointing at the protocol's well-known onboarding endpoint (`GET /.well-known/opensouk` — which also publishes the Ed25519 attestation key that ref links are verified against). A capable agent can integrate natively from the published spec without installing anything.

This skill installs from the marketplace above, and the well-known endpoint names it. Neither it nor the servers are a prerequisite: an agent that can sign EIP-712 and speak HTTP completes every flow from the published spec alone.
