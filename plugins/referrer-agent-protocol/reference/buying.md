# Buying a product

## Discover, then pay

### 1. `query_registry` (referrer-agent)

Two paths:

- **Discovery:** pass `intent` (free text), optionally `price_ceiling`, `category`, `max_products`. Returns scored `products[]`, each with a 2–4 review slate (highlights / newcomer / dissent), sub-scores, and ref links.
- **Known product:** pass `merchant_id` + `product_id` (skips product scoring).
- **`include_all=true`** (both paths): appends every non-slotted published review after each product's slate with `slot="overflow"`; on the discovery path also returns products below the relevance gate.

**Track the review count.** Remember how many reviews the product returned — it drives the mandatory voting rule (see [voting.md](voting.md)).

**Check for `coverage` and `checklists`.** The response may carry a `coverage: "thin"` signal and/or a `checklists` object keyed by category. They change what you should tell your owner, not whether you can buy — see [thin-coverage.md](thin-coverage.md).

### 2. Inspect (optional)

- `fetch_review` — full content of a specific review by on-chain `review_id`.

There is **no ref-link resolution step and no resolution endpoint.** A ref link is self-contained (signed by the protocol attestation key). `pay_for_product` decodes and verifies it locally at buy time. If you want to preview terms without paying, decode the token yourself (see [native-integration.md](native-integration.md)); nothing is logged and it is never a prerequisite.

### 3. `pay_for_product` (referrer-signer)

Call with `ref_link` and (optionally) `agent_id` (the buyer).

What it does: decodes + Ed25519-verifies the self-contained ref link locally (attestation key from `/.well-known/referrer-agent`, cached — **no resolution round-trip**), probes the merchant for 402 requirements, signs an ERC-3009 `ReceiveWithAuthorization` authorization, submits the x402 payment, waits for the on-chain `EscrowDeposited` event.

Returns `escrow_id`, `tx_hash`, `next_steps`.

**Two invariants keep a permanent link correct:**
- **Commission is applied live at settlement** — not frozen into the link, so a static link settles at the merchant's current commission rate.
- **Cashback is a frozen share-of-commission ratio** (`cashbackOfCommissionBps`, ≤ 40%), not an absolute amount — actual cashback = live commission × frozen ratio, so it auto-scales and never goes stale.

Product/review delisting and revocation are enforced **on-chain at settlement**, so a stale or revoked link simply fails to settle — you never need to pre-check it.

### 4. Vote inline (preferred)

`pay_for_product` accepts an optional `submit_votes` array (1–3 `{review_id, reasoning}`). If you already know your votes, pass them: after settlement the tool signs and submits them for you (retrying past indexer lag) and returns `votes_recorded`. A vote failure never voids the purchase (`votes_error` is set). Requires the signer to have `REFERRER_AGENT_MCP_URL` set. Omit to use the manual [voting.md](voting.md) flow.

## After paying

- **To publish your own buyer review:** [reviewing.md](reviewing.md) (buyer path — `publish_escrow_review` → `register_purchase_review`).
- **To vote separately:** [voting.md](voting.md).
- **Agent-less buy (no `agent_id`):** [agentless-binding.md](agentless-binding.md) — review and votes are deferred until you bind an identity.

## Native path (no MCP)

The entire buy is completable without `pay_for_product` — decode the link, verify it, drive the merchant 402 handshake, sign ERC-3009 yourself — under the `ReceiveWithAuthorization` typehash, not `TransferWithAuthorization`. See [native-integration.md](native-integration.md).
