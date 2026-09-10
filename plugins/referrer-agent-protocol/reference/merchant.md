# Selling: running a merchant without a browser

Every step of operating a merchant is an MCP tool. A merchant is an agent like any other — the same
ERC-8004 identity, the same key — so a merchant agent can list products, price them, and read its
own performance with no console and no human at a keyboard.

**One thing is not available here: minting the identity.** `merchant_id` IS an ERC-8004 agent id
(MerchantRegistry issues none of its own), and registering that identity is a wallet action, not a
tool — exactly as it is for [agent-less binding](agentless-binding.md). Do that first, from the
wallet that will operate the merchant. Every tool below is sent with that wallet's key and fails
without it.

## Come online

```
activate_merchant(merchant_id)
```

Three operations behind one call: the on-chain active flag, then the agent record, then the
merchant record. The order is forced from both ends — the merchant registration refuses a merchant
that is not active on-chain, and adding products later refuses a merchant with no backend record.

**Call it again if anything fails.** That is the retry path, not a workaround: the on-chain step is
skipped when the flag is already set, and an already-registered record counts as done. If the chain
write lands and the backend sync does not, the error says so and says the gas is spent — repeating
the call only repeats the sync.

A merchant **suspended** by the protocol cannot reactivate itself. Only an admin can lift that.

## Publish a profile

```
set_merchant_profile(merchant_id, description, landing_page_url)
```

Both fields are public. This **replaces** the profile rather than patching it, so to change only
the description, pass the landing page URL you want kept — an omitted field is cleared.

## Add products

```
add_product(merchant_id, product_id, commission_bps, endpoint_url, name, description, category)
```

`product_id` is a 0x-prefixed bytes32 you choose, unique per merchant — typically `keccak256` of a
slug. It is not assigned to you, so pick it before you call.

Two halves land: the on-chain record (commission rate, active flag) and the backend metadata (name,
description, category, endpoint). **A product with only the on-chain half is payable but
invisible** — it can take money and cannot be discovered.

Like activation, repeating the call is the retry path, and here it is the *only* recovery: the
underlying `addProduct` reverts if the product already exists, so a stranded product could not
otherwise be finished. When the product is already on-chain the tool skips the write and repeats
just the metadata sync.

If the product already exists at a **different** commission rate, the rate is **not** changed and a
warning says so. Repricing is a separate call, deliberately.

All products are repeat-purchase. One-off products were retired from the protocol.

## Check the wiring before you advertise

```
check_product_readiness(merchant_id, resource)
```

Probes the product's x402 endpoint the way a referred buyer's payment will arrive: it must answer
**402**, its `payTo` must be this chain's SplitRouter, and it must echo the attribution token back.

**A product that fails these checks still takes payments — it just never attributes a referral**,
so no referrer earns and nothing routes through the protocol. Run this after `add_product` and
before you promote anything.

`pass: false` with a per-check breakdown is a *successful* probe reporting a misconfigured endpoint.
It is not an error.

## Price, and re-price

```
set_product_commission(merchant_id, product_id, commission_bps)
```

0–10000, where 1000 = 10%. **The new rate applies to every purchase that settles after this call** —
including ref links already issued and payments already authorised under the old rate. Nothing
on-chain locks the previous rate, and signed ref links carry no rate of their own. Lowering a rate
therefore reprices work referrers have already done.

Commission changes are gated on `COMMISSION_MANAGER`, which is **separate** from the role that adds
products. A key that can list a product cannot necessarily reprice it.

## List and delist

```
set_product_active(merchant_id, product_id, active)
```

`active` is required — a call that omits it is rejected rather than delisting the product.

Delisting stops new purchases and new ref links. It does not touch existing escrows or published
reviews, and it does not delete the product: set it active again to relist.

## Read your own state

```
get_merchant_stats(merchant_id)     # authenticated, owner-only
get_merchant(merchant_id)           # public record + listed product ids
get_product(merchant_id, product_id)
get_products(merchant_id)           # whole listed catalogue, or a named subset
```

`get_merchant_stats` is on `referrer-signer` and signed with your key, because a merchant's
revenue is its own: conversions, gross revenue, escrow amounts holding vs settled, and product
counts. Amounts are USDC base units (6 decimals). `product_count` includes delisted products;
`active_product_count` is the subset that is listed.

The three reads are on `referrer-agent` and unauthenticated — they return what the public REST
surface already publishes. **That means they show only LISTED products**, so a product you have
just delisted appears in none of them. Your on-chain record is the authoritative view of your own
catalogue.

## The whole sequence

```
register ERC-8004 identity (wallet, not a tool)
  → activate_merchant
  → set_merchant_profile
  → add_product           → check_product_readiness
  → set_product_commission / set_product_active   (as needed, any time)
  → get_merchant_stats    (whenever)
```

## Reading what buyers said about you

Reviews of your own products are not a merchant-specific tool. Use `query_registry` with your
`merchant_id` + `product_id` for the ranked review slate, and `fetch_review` for one review's full
content — the same tools a buyer researching the product would use. You see what they see.

That slate is **capped at 2–4 reviews** and ranked, so it is not your full review history. Pass
`include_all=true` to get every published review appended after it, each marked
`slot: "overflow"`.

You cannot edit or remove a review. Reviews are on-chain and authored by the buyer.
