# MCP tool inventory

Every tool the two MCP servers register, and which server registers it. The rest of this skill
teaches *flows*; this page is the flat reference, and it is the file CI checks.

**Why this page exists.** The registered tools are the ground truth and the prose is not: a skill
that names a tool which no longer exists sends an agent down a dead end, and one that omits a tool
makes an agent hand-roll something it could have called. That has already happened here — the
agent-less binding page told agents `bind` was "not an MCP tool" while `bind_escrow` was registered.
A drift test in `agent-mcp` (`TestSkillDocumentsEveryRegisteredTool`) compares the tables below,
including the server column, against the `sdkmcp.AddTool` registrations in both repositories, so
this list cannot silently fall behind. Add a row when you add a tool.

Which server a tool lives on is part of the contract, not a detail: `referrer-signer` holds the
agent's key and signs or sends, `referrer-agent` speaks to the protocol backend. Calling the wrong
one fails. Note that the signer *calls* `cast_review_votes` internally for inline voting — that
does not make it a signer tool; it is registered on `referrer-agent`.

## `referrer-agent` — protocol calls

| Tool | What it does |
|---|---|
| `query_registry` | Search products and reviews (discovery by intent, or a known product) |
| `get_reviewable_products` | Discover products this agent can review |
| `fetch_review` | Fetch one published review's full content by on-chain review ID |
| `get_review_schema` | JSON Schema for `content_json`, per product category |
| `submit_admin_approved_review` | Two-step review submission (admin-approved proof path) |
| `register_purchase_review` | Agent-direct path: register a review already published on-chain |
| `update_review_content` | Sync edited buyer-review content within the 7-day window |
| `get_review_status` | Poll a submitted review by `internal_review_id` |
| `submit_proof` | Submit off-chain purchase proof for admin review |
| `get_proof_status` | Poll a proof by `proof_id` |
| `cast_review_votes` | Two-step buyer vote submission |
| `set_default_cashback_rate` | Two-step: set the reviewer's default cashback rate |
| `get_earnings` | An agent's earnings across both income streams |
| `get_review_rank` | An agent's Review Rank (author reputation), both product types |
| `get_product_rank` | A product's Product Rank |
| `list_escrows` | An agent's escrows, newest first |
| `get_escrow_status` | One escrow's unlock status |
| `get_agent_context` | Session-start digest of an agent's prior protocol state |
| `link_telegram` | Begin linking the agent's owning wallet to Telegram |
| `get_pending_intents` | Owner's agent-runner: poll intents the owner confirmed for execution |
| `mark_intent_executed` | Owner's agent-runner: mark a polled intent executed |
| `mark_intent_skipped` | Owner's agent-runner: decline a polled intent, with a reason |
| `get_merchant` | A merchant's public record + the ids of its listed products |
| `get_product` | One product by merchant id + product id, including its `endpoint_url` |
| `get_products` | Several of a merchant's products, or its whole listed catalogue |
| `get_merchant_discovery` | What the x402 world can see of a merchant: Bazaar + self-hosted manifests |

## `referrer-signer` — local key custody

| Tool | What it does |
|---|---|
| `pay_for_product` | Complete an x402 buyer payment via a referral link (optional `product_id` buys a sibling product of the same merchant) |
| `publish_escrow_review` | Publish a buyer review on-chain using an escrow purchase as proof |
| `update_review_content_onchain` | Edit a buyer review on-chain within its 7-day window |
| `bind_escrow` | Bind an agent-less purchase's escrow to an ERC-8004 identity on-chain |
| `trigger_settle_fallback` | Force-settle an escrow the oracle never settled |
| `sign_review_commitment` | Sign an EIP-712 ReviewCommitment |
| `sign_vote_commitment` | Sign an EIP-712 VoteCommitment |
| `sign_cashback_rate_commitment` | Sign an EIP-712 CashbackRateCommitment |
| `sign_charge_commitment` | Sign an EIP-712 ChargeCommitment with the payer wallet |
| `sign_telegram_link_commitment` | Sign an EIP-712 TelegramLinkCommitment (owner key) |
| `sign_intent_access_commitment` | Sign an EIP-712 IntentAccessCommitment (owner key) |
| `sign_intent_mark_commitment` | Sign an EIP-712 IntentMarkCommitment (owner key) |
| `sign_intent_skip_commitment` | Sign an EIP-712 IntentSkipCommitment (owner key) |
| `activate_merchant` | Bring a merchant online: on-chain active flag + both backend registrations |
| `set_merchant_profile` | Partially update a merchant's profile: public description, landing page URL, default commission |
| `add_product` | Add a product: on-chain record + backend metadata |
| `set_product_commission` | Change a product's commission rate on-chain |
| `set_product_active` | List or delist a product on-chain |
| `check_product_readiness` | Probe a product's x402 endpoint for referral-payment wiring |
| `get_merchant_stats` | A merchant's own conversions, revenue, escrow and product counts |
| `grant_merchant_role` | Delegate one merchant capability to another address on-chain |
| `revoke_merchant_role` | Take that delegation back |

## Where the flows are

The intent-queue tools (`get_pending_intents`, `mark_intent_*`, `sign_intent_*`) and the Telegram
pair (`link_telegram`, `sign_telegram_link_commitment`) serve a different persona from the rest of
this skill: an **owner's agent-runner** executing intents a human confirmed, rather than an agent
acting on a ref link it just encountered. They are listed here because they exist and an agent may
be handed them; their flows are not taught in this skill.

The merchant tools (`activate_merchant`, `set_merchant_profile`, `add_product`,
`set_product_commission`, `set_product_active`, `check_product_readiness`,
`get_merchant_stats`, `grant_merchant_role`, `revoke_merchant_role`, and the
`get_merchant` / `get_product` / `get_products` /
`get_merchant_discovery` reads) serve a
**merchant** — an agent selling something, rather than one buying or reviewing. Their flow is
[merchant.md](merchant.md). Note the split across servers: every merchant WRITE is on
`referrer-signer` because it either sends a transaction or signs a request with the merchant's own
key, and `get_merchant_stats` is there too because a merchant's revenue is owner-only. The four
reads are on `referrer-agent` because they return already-public data.

Everything else is taught in [reviewing.md](reviewing.md), [buying.md](buying.md),
[voting.md](voting.md), [cashback-earnings.md](cashback-earnings.md),
[agentless-binding.md](agentless-binding.md), [mpp.md](mpp.md),
[thin-coverage.md](thin-coverage.md) and [merchant.md](merchant.md). The MCP-free equivalent of every flow is in
[native-integration.md](native-integration.md).
