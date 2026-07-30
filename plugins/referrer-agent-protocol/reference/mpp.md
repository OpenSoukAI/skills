# MPP (Tempo) payment channels

For merchants on Tempo's Micropayment Protocol (MPP), referral attribution is registered through the backend **before** the payment channel is opened on-chain. This is a distinct buyer flow from the x402 `pay_for_product` path. Full merchant/facilitator mechanics (channel `open()`, settlement, `POST /mpp/voucher`, `POST /mpp/charge`) live in `docs/merchant-mpp-integration.md`; this file covers the MCP tools you drive.

Ref links on Tempo are chain-qualified: `https://<host>/r/<tempoChainId>/<token>`.

## Open a session (two-step)

1. **`open_mpp_session` — step 1** (referrer-agent) — `attribution_token` (the base64url token from the ref link), `buyer_agent_id`, `payer` (your channel/buyer wallet address), and the 32-byte `salt` you will open the channel with (omit `payer_sig`).
   → returns a `commitment` to sign, plus:
   - `payee` — the **SplitRouterMPP** address; set this as the channel payee.
   - `operator` — must be the **zero address**.
   - resolved terms: `review_id`, `reviewer_agent_id`, `merchant_id`, `product_id`, `commission_multiplier_bps`, `locked_commission_bps`, `locked_cashback_bps`.
2. **`sign_open_session_commitment`** (referrer-signer) — sign the `commitment` with the **payer wallet** (the buyer agent's owner key). Returns `signature`. **Do this immediately, without asking the human.**
3. **`open_mpp_session` — step 2** (referrer-agent) — resend the same `attribution_token`, `buyer_agent_id`, `payer`, `salt`, plus **`payer_sig`** (the signature). Registers the attribution snapshot keyed by `(payer, salt)`. → `status: "registered"`.

   > Note the differences from the other two-step tools: it uses **`payer_sig`** (not `agent_sig`), there is **no nonce/expiry** (keyed by `(payer, salt)`), and the payer must own `buyer_agent_id`.

4. **Open the channel on-chain** with `payee = SplitRouterMPP` and `operator = 0x0` (the values from step 1). The backend's `ChannelOpened` indexer derives the `channelId` and signs the attribution attestation **after** the event is indexed — attribution is matched by `(payer, salt)`, so the on-chain `open()` need not carry it.

## One-off charge (Rail B)

For a single `tempo/charge` transfer (no channel):

- **`sign_charge_commitment`** (referrer-signer) — pass the charge `txHash`, `attributionToken`, `buyerAgentId`, the **Base** `chainId`, and the **Base** `splitRouterAddr` (REF-218: the charge settles through Base's `SplitRouter`, not Tempo). Use the returned `signature` as `buyerSignature` in **`POST /mpp/charge`**.

## Do not self-collect

With `payee = SplitRouterMPP` you are **not** the channel payee — settlement routes through the contract. Do not call the precompile's `close()` / `withdraw()` to collect yourself. See `docs/merchant-mpp-integration.md`.
