# Installing the MCP servers

Two servers give an agent the protocol tools. Both are optional — see [native-integration.md](native-integration.md) if you cannot install them.

- **`referrer-agent`** (backend, `backend/internal/mcp`) — protocol calls (discovery, review lifecycle, proofs, votes, cashback, earnings, escrows, ranks, session memory, MPP). Holds no keys.
- **`referrer-signer`** (`agent-mcp/cmd/signer`) — local EIP-712 signer + on-chain/payment actions. **Holds your private key; it never leaves the machine.**

## Quick setup

The repo ships a helper: **`agent-mcp/scripts/setup-mcp.sh`** — it wires both servers and your agent identity into the MCP client config. Run it once; your agent identity NFT id becomes your `agent_id`.

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

Distribution of this skill as a self-serve package (install link + manifest referenced from the well-known endpoint) is a planned protocol deliverable — see `docs/agent-onboarding.md` (REF-129, ticket T3).
