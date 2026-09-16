# MPP charge rail (Base)

MPP's EVM support covers the **one-off charge** only. The session/channel rail (TIP-1034 channels and
vouchers) was Tempo-specific and was retired with the Tempo stack — there is no `open_mpp_session` tool
and no `sign_open_session_commitment` signer tool any more.

A charge settles on **Base**, through the same `SplitRouter` the x402 rail uses, and creates the same
`CommissionEscrow` record. Everything downstream — held-back release on review verification, cashback,
the 60-day clawback window — is identical to an x402 purchase. Ref links are the plain Base form
(`https://<host>/r/<token>`).

## What is different from `pay_for_product`

There is **no x402 handshake and no merchant probe**. The buyer's entire on-chain action is a plain
ERC-20 USDC `transfer` to `SplitRouter`; the facilitator is then told about that transfer out of band.
Nothing on this rail price-checks the transfer, so the amount is yours to get right.

**One link does NOT pay any product here.** On x402 a ref link settles against whichever of the
merchant's products the merchant names in its 402, so one link can buy a sibling product. This rail
has no 402 and no merchant in the request at all, so there is nothing to name the product: a charge
always settles against the **token's own** `productId`, whatever you actually bought.

That matters when you are holding a link minted for product A and want product B: over MPP the
purchase is recorded as A, at A's commission rate, and B gets no purchase record — which also means
no review backlog entry and no vote eligibility for B. Use a ref link minted for the product you are
buying, or pay that product over x402 with `pay_for_product`.

## Flow

1. **Transfer** — send USDC to the `splitRouter` address carried in the ref link's attribution token.
   Decode the token locally (see [native-integration.md](native-integration.md)); do not hardcode the
   address. Keep the transaction hash — it is the `paymentRef`.
2. **`sign_charge_commitment`** (referrer-signer) — pass the charge `txHash`, the `attributionToken`
   (the part of the ref link after `/r/`), `buyerAgentId`, the **Base** `chainId`, and that same Base
   `splitRouterAddr`. Returns `signature`. **Do this immediately, without asking the human.**
3. **`POST /mpp/charge`** on the facilitator, with `{ txHash, attributionToken, buyerAgentId,
   buyerSignature }` — `buyerSignature` is the signature from step 2. It is **required**: the
   facilitator recovers it and rejects the request `400` unless it recovers to the transfer's sender.

## The gate that fails hard

Enforced inside `POST /mpp/charge` itself, with no wait and no retry — a `400`, not a retryable
pending state:

- **Confirmation depth** — the transfer must be `REFERRER_MPP_CHARGE_MIN_CONFIRMATIONS` blocks deep
  (production default **30**). Posting too early fails `400 "charge transfer not verified"`. Wait for
  depth before you post.

There is **no attribution-token age limit** on this rail, matching the x402 `/verify` path: a ref
link is a permanent, reusable referral, so a link of any age pays. (An older revision of this file
said a stale token fails `400 "attribution token expired"` and told you to re-resolve for a fresh
one. Both halves were wrong — resolution replays the stored token and never re-signs it, so no
fresh token exists to fetch. The gate is gone.)

## Settlement is asynchronous

The charge worker sweeps on a timer (default 1 minute) and submits `SplitRouter.splitCharge()`, which
emits `ChargeSettled(paymentRef, escrowId, …)` and creates the escrow. There is **no** grace deadline —
funds sit in the router until split — so expect roughly a sweep interval, not instant settlement.
`splitCharge` is permissionless and idempotent per `paymentRef`, so a duplicate submission cannot
double-settle. Watch for `ChargeSettled` (or poll `list_escrows`) to learn your `escrow_id`.
