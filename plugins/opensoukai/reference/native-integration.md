# Native integration (no MCP)

Every protocol capability is reachable with **no MCP servers installed**, by any agent that can:
- sign **EIP-712** typed data and **ERC-3009** — specifically the `ReceiveWithAuthorization`
  typehash, which is what `SplitRouter.route()` requires (see step 2),
- verify **Ed25519** signatures,
- speak HTTP and read on-chain events.

The MCP servers are a key-custody + ergonomics convenience, not a requirement. This file is the recipe an agent follows when it only has the knowledge, not the tools. The tested reference implementation is the e2e harness (`e2e-script/`) and the `referrer-signer` source (`agent-mcp/cmd/signer/`) — consult them for exact byte layouts.

> Status note: some low-level details below (exact HTTP header names, review-registration REST routes) should be **confirmed against the backend router and `merchant-x402-integration.md` before shipping a native client.** Discrepancies are flagged inline.

## The self-contained ref link

A ref link is `https://<host>/r/<token>` (Base) or `https://<host>/r/<chainId>/<token>` (non-Base rails, e.g. Tempo). The `<token>` is the last path segment: base64url of `payload ‖ Ed25519 signature`.

**There is no resolution round-trip.** (An older `GET /r/:token` JSON resolution existed but was removed in REF-139 — `docs/x402-flow.md` Step 1 is stale.) Instead:

1. Split the path after `/r/` into the optional `chainId` segment and the token.
2. Fetch the attestation keys: `GET <host>/.well-known/opensouk` → read `attestationPubKeys` (a map of key id → hex Ed25519 public key) for your chain. `attestationPubKey` is the current mint key, kept for older readers.
3. Base64url-decode the token, split payload from the trailing Ed25519 signature, read the payload's `kid` (absent = `0`, the launch key), and **verify the signature against `attestationPubKeys[kid]`**. Reject on failure, and reject a `kid` the manifest does not list — refetch the manifest once first, since a rotation may have just happened.
4. Confirm the token's `chainId` matches the chain you are transacting on.

The verified payload gives you everything to pay: `merchantEndpoint`, `splitRouter` (settlement address), `reviewerAgentId`, `reviewId`, `merchantId`, `productId`, `lockedCommissionBps`, `cashbackOfCommissionBps`, `productType`, `issuedAt`, `chainId`, and a `setupUrl` onboarding pointer.

`productId` and `merchantEndpoint` name the product the review was written about. On the x402
rail they are **not** a restriction: point the same token at any endpoint of the same `merchantId`
and the sale settles against the product that merchant names in its 402 (`extra.productId`), at
that product's commission rate, still crediting `reviewId`. A merchant that omits the field
settles the sale as this token's `productId`. The MPP charge rail has no merchant 402, so it
always settles on this token's `productId`.

> Trust caveat: the attestation key is fetched from the ref link's own host. Pinning the well-known to a configured trusted origin (so a link on an attacker host can't self-attest) is a deferred onboarding-trust item — see `docs/decision-edge-reflink-agentless-buy.md`.

## Buyer flow (x402 on Base)

1. **Probe (phase 1):** `POST <merchantEndpoint>` with headers:
   ```
   X-Referrer-Token: <the base64url attribution token>
   X-Referrer-Buyer-Agent-Id: <buyer agent NFT id, decimal>
   ```
   Expect **HTTP 402** with x402 `PaymentRequired` JSON: `asset` (USDC), `amount`, `network`, the pay target, and `extra.name` / `extra.version` (ERC-3009 domain), `extra.authorizationType` (the EIP-712 primary type to sign — `ReceiveWithAuthorization`), plus `extensions.attributionToken` / `extensions.buyerAgentId`.

   `extra.productId`, when present, is the merchant asserting which of its products this endpoint
   sells. That is what the sale settles against, so it may differ from the token's own `productId`
   when you are paying a sibling product with someone else's link. It is the merchant's claim to
   make, not yours: a buyer-supplied product would let a buyer pick whichever product pays the most
   cashback.

2. **Sign ERC-3009 `ReceiveWithAuthorization`:** build an `ExactEIP3009Authorization` — `from` = your wallet, `to` = the token's `splitRouter`, `value` = required amount, `validAfter=0`, `validBefore` = now + a few minutes, random 32-byte `nonce`. Hash it as EIP-712 typed data with primary type **`ReceiveWithAuthorization`** over domain `(name, version, chainId, verifyingContract=tokenAddr)`, and sign; set `v += 27`.

   **Do not sign `TransferWithAuthorization`.** ERC-3009 defines both structs with identical fields and different names, so they produce different EIP-712 digests. `route()` calls `receiveWithAuthorization`, which requires `to == msg.sender`; real Circle USDC rejects a Transfer-typed signature inside the token. The x402 `exact` scheme signs Transfer by default, so a stock SDK client cannot pay this rail — sign the authorization yourself. The 402's `extra.authorizationType` and the ref link's `authorizationType` both name the required type; if the facilitator rejects a payment it returns `requiredAuthorizationType` plus `retryable: true`, and re-signing the same authorization under the right type is free (nothing was sent on-chain and the nonce is unused).

3. **Pay (phase 2):** re-`POST <merchantEndpoint>` with the x402 payment payload JSON, base64url-encoded, in the `PAYMENT-SIGNATURE` header (the x402 v2 name). It **must** carry `Extensions: { attributionToken, buyerAgentId }`. A payload lacking the attribution token is hard-rejected by the facilitator.

4. Expect **200** with `{ transaction: "0x..." }`. Wait for the receipt; extract the `EscrowDeposited` event to get your `escrow_id`.

Agent-less: send the sentinel buyer id (`type(uint256).max`) instead of a real one; the paying wallet is recorded for a later on-chain `bind` ([agentless-binding.md](agentless-binding.md)).

## Review flow (native)

- **Buyer path:** call `ReviewRegistry.publishReviewWithEscrowProof(agentId, contentHash, escrowId)` on-chain (the content hash preimage is `abi.encode(uint256 agentId, uint256 merchantId, bytes32 productId, string contentJSON)` keccak256, with `merchantId`/`productId` read from the escrow). Then register with the backend so it issues your ref link — via `register_purchase_review` (MCP) or the equivalent REST route. **Confirm the REST route in the backend router.**
- **Admin-approved path:** build and sign the EIP-712 **`ReviewCommitment`** yourself:
  ```
  domain:  EIP712Domain(name,version,chainId,verifyingContract=ReviewRegistry)
  struct:  ReviewCommitment(uint256 agentId,uint256 merchantId,bytes32 productId,
                            bytes32 contentHash,uint256 nonce,uint256 expiry)
  ```
  Submit it to the backend (the two-step `submit_admin_approved_review` shape). Requires an approved off-chain proof first.

## Other EIP-712 commitments (type strings)

All are recovered by the backend (only `ReviewCommitment` is verified on-chain). Type strings are byte-identical to `backend/internal/signing` and `agent-mcp/cmd/signer`:

```
VoteCommitment(uint256 agentId,uint256 escrowId,bytes32 votesHash,uint256 nonce,uint256 expiry)
CashbackRateCommitment(uint256 agentId,uint16 cashbackOfCommissionBps,bytes32 nonce,uint256 expiry)
ChargeCommitment(address payer,bytes32 paymentRef,uint256 buyerAgentId,bytes32 attributionTokenHash)
```
`ChargeCommitment` domain name = `ReferrerMPPCharge`, version `1`, with the **Base** `chainId` and `verifyingContract=SplitRouter`. All signatures use `v += 27`.

## Reference implementation

- `agent-mcp/cmd/signer/` — the exact packing/hashing for every commitment and the full x402 buy (`main.go`, `attribution.go`, `charge_commitment.go`).
- `e2e-script/` — the whole flow driven end-to-end against a live stack.
- `docs/x402-flow.md`, `docs/merchant-x402-integration.md`, `docs/merchant-mpp-integration.md` — protocol HTTP contracts.
