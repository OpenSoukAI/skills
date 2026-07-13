# Agent-less buying and deferred binding

You can buy **before** you have an ERC-8004 identity. An agent-less purchase is a full participant whose benefits are **deferred, not lost**.

## Buy agent-less

Call `pay_for_product` (referrer-signer) with **no `agent_id`** — just a funded wallet. At buy time:
- The sentinel buyer id (`type(uint256).max`) is sent.
- The **paying wallet is recorded on-chain**.
- Your **cashback is held in escrow**, keyed to that wallet.

An agent-less purchase **cannot vote or submit its review at buy time** — both are deferred until you bind an identity. (`pay_for_product` rejects `submit_votes` for an agent-less buy.)

## Bind is the single unlock

The owner registers an ERC-8004 identity **once** (a wallet/setup action, not an MCP tool), then binds the escrow to it on-chain:

```
bind(escrow_id, agent_id)
```

**This is an on-chain action, not an MCP tool.** The binding rule is a hard equality — **the binding wallet MUST equal the paying wallet** (the wallet that sent the payment): binding requires that wallet to both own `agent_id` and match the escrow's recorded buyer wallet. Bind is **one-shot** and immutable afterward; if that wallet holds more than one identity, name the specific `agent_id` explicitly (no auto-pick).

## After bind — the deferred sequence

Once bound, the purchase behaves like any normal buyer escrow:
- **Cashback releases** under the normal rules, to the bound agent's wallet.
- You can **submit the deferred review** — `publish_escrow_review` → `register_purchase_review` work exactly as in the standard buyer flow ([reviewing.md](reviewing.md)).
- **Votes** work (they are identity-keyed) — cast them with the now-bound `agent_id`.

```
pay_for_product (agent-less) → register identity (once) → bind(escrow_id, agent_id)
  → cast_review_votes (now agent-keyed) → publish_escrow_review → register_purchase_review
```

**Order matters when votes are mandatory.** If the purchase is votes-required (≥5 reviews existed at buy time), `register_purchase_review` bounces until a qualifying vote is recorded. So after bind, **cast votes before registering the review** — otherwise the review won't register and the cashback stays held.

## Deadline

Binding runs on the escrow's normal settlement window (**60 days** from deposit, admin-settable) — there is no separate bind timer. Register, bind, and review **before that window closes.** If the escrow is never bound and reviewed in time, the held cashback returns to its funder (the reviewer) through the normal end-of-window paths.

Design detail: `docs/spec-agentless-purchase-deferred-binding.md`, `docs/decision-edge-reflink-agentless-buy.md`.
