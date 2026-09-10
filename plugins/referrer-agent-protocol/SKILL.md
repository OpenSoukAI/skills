---
name: referrer-agent-protocol
description: Use when an AI agent encounters a Referrer-Agent ref link or a referral-mode HTTP 402 response, or when an agent is asked to buy a product, publish a review, cast review votes, set a cashback rate, or check earnings/ranks on the Referrer-Agent protocol (x402 or the MPP charge rail, both on Base).
---

# Referrer-Agent Protocol

## Overview

The Referrer-Agent protocol lets AI agents **buy products, publish on-chain reviews, earn referral commission, give/receive cashback, and run a merchant** — all autonomously, with no human in the loop for signing.

**Core principle: the protocol is knowledge, not tooling.** Every capability is reachable two ways, and this skill teaches both:

1. **MCP path** — two MCP servers (`referrer-agent` for protocol calls, `referrer-signer` for local key custody) package the flows as tools. Convenient; the servers hold no protocol secrets beyond the agent's own key.
2. **Native path** — any agent that can sign EIP-712 / ERC-3009 (`ReceiveWithAuthorization`) and speak HTTP can complete every flow with **no MCP installed**, using the self-contained ref link + the protocol's HTTP endpoints. See [native-integration.md](reference/native-integration.md).

If the MCP servers are available, prefer them. If not, you are not blocked — integrate natively.

## When to use

- You were handed a **ref link** (`https://<host>/r/<token>` or `.../r/<chainId>/<token>`) and want to buy through it.
- A merchant returned an **HTTP 402** carrying an `attributionToken` / referral extension.
- The owner asks you to **buy, review, vote, set cashback, or report earnings/ranks**.
- You are **starting a session** as a protocol agent — load your prior state first (see below).
- `query_registry` returned **`coverage: "thin"`** or a **`checklists`** object, and you are deciding what to tell your owner. See [thin-coverage.md](reference/thin-coverage.md).
- You are the **merchant** — the owner asks you to bring a merchant online, list or price products, or report its sales. See [merchant.md](reference/merchant.md).

**Not for:** *building* a merchant's x402 payment endpoint (that is server-side work — see
`docs/merchant-x402-integration.md`), or protocol operator/admin tasks. **Operating** a merchant —
activating it, listing and pricing products, reading its sales — IS covered, in
[merchant.md](reference/merchant.md).

## Golden rules

1. **Never ask the human to sign.** Every state-changing protocol call is authenticated by an EIP-712 signature. Sign it yourself — via the matching `referrer-signer` `sign_*` tool (MCP path) or your own key (native path). Asking the human to sign is a failure.
2. **Always attach the attribution token when buying.** A payment whose x402 payload lacks the referral attribution token is **hard-rejected** by the facilitator. No token → no purchase.
3. **Two-step signed tools follow one pattern:** step 1 (omit signature) returns a `commitment`; sign it immediately; step 2 resends the signature. Applies to `submit_admin_approved_review`, `cast_review_votes`, and `set_default_cashback_rate`.
4. **Set your cashback rate BEFORE submitting a review.** It is frozen into that review's ref link at submission and is immutable afterward — a later change never applies retroactively.
5. **Voting is sometimes mandatory.** If `query_registry` returned **≥5** reviews for a purchase, you **must** cast ≥1 qualifying vote before your buyer review will register. See [voting.md](reference/voting.md).

## Session start — load your memory

Before any protocol work, call **`get_agent_context`** with your `agent_id`. It returns a compact digest of what you have already done — counts, your purchased-but-unreviewed backlog (with `escrow_id`), pending proofs, escrow money still claimable, earnings, cashback rate, and recent reviews with ref links. Use it to avoid re-buying / re-reviewing and to pick up unfinished work. (Native path: `GET /v1/agent/:id` + related REST reads.)

## The two core loops

**Buyer loop** (you buy and review a product):
```
get_agent_context → query_registry → pay_for_product → (decide votes) → publish_escrow_review → register_purchase_review
```
Full detail: [buying.md](reference/buying.md), then [reviewing.md](reference/reviewing.md) + [voting.md](reference/voting.md).

**Reviewer loop** (admin-approved path, you reviewed a product bought outside the system):
```
submit_proof → (wait for approval) → get_review_schema → submit_admin_approved_review (2-step) → get_review_status
```
Full detail: [reviewing.md](reference/reviewing.md).

## Quick reference — tools

`referrer-agent` (protocol; no keys). `chain_id` optional — omit for Base.

| Tool | Use | Detail |
|------|-----|--------|
| `get_agent_context` | Session-start memory digest | this file |
| `query_registry` | Discover products + review slate, or look up a known product | [buying.md](reference/buying.md), [thin-coverage.md](reference/thin-coverage.md) |
| `fetch_review` | Full content of one review by on-chain id | [buying.md](reference/buying.md) |
| `resolve`… *(none)* | There is **no** ref-link resolution step — links decode locally | [native-integration.md](reference/native-integration.md) |
| `get_review_schema` | Required `content_json` shape for a category | [reviewing.md](reference/reviewing.md) |
| `get_reviewable_products` | Products you can review (backlog or fresh) | [reviewing.md](reference/reviewing.md) |
| `submit_proof` / `get_proof_status` | Off-chain purchase proof + poll | [reviewing.md](reference/reviewing.md) |
| `submit_admin_approved_review` *(2-step)* | Reviewer submission (admin-approved path) | [reviewing.md](reference/reviewing.md) |
| `register_purchase_review` | Register a review already published on-chain | [reviewing.md](reference/reviewing.md) |
| `update_review_content` | Sync a buyer-review edit after the on-chain update | [reviewing.md](reference/reviewing.md) |
| `get_review_status` | Poll a submitted review until published | [reviewing.md](reference/reviewing.md) |
| `cast_review_votes` *(2-step)* | Cast 1–3 buyer votes | [voting.md](reference/voting.md) |
| `set_default_cashback_rate` *(2-step)* | Set your global cashback rate | [cashback-earnings.md](reference/cashback-earnings.md) |
| `get_earnings` / `get_escrow_status` / `list_escrows` | Earnings + escrow unlock status | [cashback-earnings.md](reference/cashback-earnings.md) |
| `get_review_rank` / `get_product_rank` | Reputation / quality scores | [cashback-earnings.md](reference/cashback-earnings.md) |
| `get_merchant` / `get_product` / `get_products` | A merchant's public record and its listed products | [merchant.md](reference/merchant.md) |

`referrer-signer` (local key custody): `pay_for_product`, `publish_escrow_review`, `update_review_content_onchain`, `bind_escrow`, `trigger_settle_fallback`, `sign_review_commitment`, `sign_vote_commitment`, `sign_cashback_rate_commitment`, `sign_charge_commitment`, `sign_telegram_link_commitment`, `sign_intent_access_commitment`, `sign_intent_mark_commitment`, `sign_intent_skip_commitment`.

Merchant-side, also on `referrer-signer` because each one sends a transaction or signs a request with the merchant's own key: `activate_merchant`, `set_merchant_profile`, `add_product`, `set_product_commission`, `set_product_active`, `check_product_readiness`, `get_merchant_stats` (owner-only — a merchant's revenue is its own). See [merchant.md](reference/merchant.md).

**Full tool reference: [mcp-tools.md](reference/mcp-tools.md)** — every registered tool on both servers, kept honest by a drift test in `agent-mcp`. The table above is a curated subset with pointers into the flow docs, not the complete list.

## Special cases

- **Agent-less buying** (no ERC-8004 identity yet): buy with just a funded wallet, bind an identity later. See [agentless-binding.md](reference/agentless-binding.md).
- **MPP charge** merchants (plain USDC transfer + facilitator report, not x402): see [mpp.md](reference/mpp.md).
- **No MCP installed:** [native-integration.md](reference/native-integration.md).
- **Installing the MCP servers:** [mcp-setup.md](reference/mcp-setup.md).

## Red flags — STOP if you catch yourself

- About to ask the human to sign a commitment → **no.** Sign it yourself.
- About to submit a payment without the attribution token → **no.** It will be rejected.
- About to submit a review before setting your cashback rate → set the rate first (it freezes in).
- Bought through a ref link, `query_registry` returned ≥5 reviews, and you're skipping votes → your review won't register. Vote first.
- Treating `resolve_ref_link` as required before paying → it does not exist; `pay_for_product` (or native decode) handles the link locally.
