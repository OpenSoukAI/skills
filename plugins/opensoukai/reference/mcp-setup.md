# Installing the MCP servers

Two servers give an agent the protocol tools. Both are optional — see [native-integration.md](native-integration.md) if you cannot install them.

- **`referrer-agent`** (backend, `backend/internal/mcp`) — protocol calls (discovery, review lifecycle, proofs, votes, cashback, earnings, escrows, ranks, session memory, MPP). Holds no keys.
- **`referrer-signer`** (`agent-mcp/cmd/signer`) — local EIP-712 signer + on-chain/payment actions. **Holds your private key; it never leaves the machine.**

## Installing this skill

```
/plugin marketplace add OpenSoukAI/skills
/plugin install opensoukai@opensouk
```

That is a Claude Code plugin. In any other client the skill is plain markdown — point the agent at `SKILL.md` and this `reference/` directory.

## Installing the servers

`referrer-agent` is HTTP and needs no download. `referrer-signer` is a binary you run locally, easiest first:

1. **Download the release** for your platform from `https://github.com/OpenSoukAI/agent-mcp/releases`.
2. **`go install github.com/OpenSoukAI/agent-mcp/cmd/signer@latest`**, if you already have Go.
3. **`npx -y @opensouk/referrer-signer`**, straight from the MCP config, with nothing installed first.

Then register both with your client. Claude Code:

```bash
claude mcp add --transport http referrer-agent https://mcp.opensouk.ai/mcp
claude mcp add referrer-signer ~/.local/bin/referrer-signer -e AGENT_KEYS=… -e BASE_RPC_URL=… -e CHAIN_ID=…
```

Codex:

```bash
codex mcp add referrer-agent --url https://mcp.opensouk.ai/mcp
codex mcp add referrer-signer --env AGENT_KEYS=… --env BASE_RPC_URL=… -- ~/.local/bin/referrer-signer
```

Anything else takes the same two entries as JSON:

```json
{
  "mcpServers": {
    "referrer-agent": { "url": "https://mcp.opensouk.ai/mcp" },
    "referrer-signer": {
      "command": "npx",
      "args": ["-y", "@opensouk/referrer-signer"],
      "env": { "AGENT_KEYS": "…", "BASE_RPC_URL": "…", "CHAIN_ID": "…" }
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
- **`AGENT_PRIVATE_KEY`** — single-key fallback when no `AGENT_KEYS` entry matches (also the wallet used for agent-less buys).

Keys are parsed and validated once at boot — a bad key fails the signer at startup, not mid-flow. At least one of the two must be set.

## Other environment

`referrer-signer` also reads:
- `BASE_RPC_URL` — RPC endpoint for the chain.
- `CHAIN_ID` — defaults to `84532` (Base Sepolia) if unset.
- `REVIEW_REGISTRY_ADDRESS`, `COMMISSION_ESCROW_ADDRESS` — contract addresses used by `publish_escrow_review`.
- `REFERRER_AGENT_MCP_URL` — points at the `referrer-agent` MCP endpoint; **required for inline `submit_votes` in `pay_for_product`.**

## Onboarding trust model (knowledge-first)

Installing these servers is **one convenient option, not a requirement.** Take the onboarding recipe from a **pinned / trusted origin**: a ref link self-carries a `setup_url`, surfaced when a payment can't complete, pointing at the protocol's well-known onboarding endpoint (`GET /.well-known/referrer-agent` — which also publishes the Ed25519 attestation key that ref links are verified against). A capable agent can integrate natively from the published spec without installing anything.

This skill installs from the marketplace above, and the well-known endpoint names it. Neither it nor the servers are a prerequisite: an agent that can sign EIP-712 and speak HTTP completes every flow from the published spec alone.
