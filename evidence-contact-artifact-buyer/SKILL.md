---
name: evidence-contact-artifact-buyer
description: Discover and purchase prepared artifacts from evidence.contact merchant agents through its Base MCP and x402 checkout. Use when an agent needs a paid report or seller-published artifact with recoverable delivery.
---

# Buy a prepared agent artifact

Connect an MCP client to `https://evidence.contact/mcp` using Streamable HTTP. MCP POST requests need `Content-Type: application/json` and `Accept: application/json, text/event-stream` to avoid 406. Check JSON-RPC errors and tool `isError` even when HTTP status is 200. A wallet signer alone cannot fetch or submit MCP requests. The server exposes public merchant discovery and checkout tools; it does not need a buyer API key.

## Find the offer

- Search opted-in merchants with `search_merchants` using a short capability keyword. The directory returns active offers, merchant IDs, prices, and checkout references. Directory listings are public descriptions, not endorsements or quality guarantees.
- If the seller is not listed or has opted out, use a merchant ID from the user or another trusted source and call `discover_merchant_offers`.
- If the user gives a saved offer ID, call `resolve_latest_merchant_offer` before buying. Offer IDs are immutable; the latest resolver finds the current published version.
- Directory participation is optional and defaults off. An empty search does not prove that no suitable seller exists; ask for a merchant ID or use another source. Do not invent IDs.
- Confirm the artifact, seller payout address, and displayed USDC price with the user before authorizing payment. Respect the user's spending limit.

## Buy with x402

1. Generate and persist a unique 16–128 character `idempotencyKey` before the first call. Keep it private with the purchase recovery data.
2. Call `buy_prepared_artifact` with the merchant ID, offer ID, and that key. The first call returns x402 v2 Base USDC payment requirements.
3. Check the requested amount, recipient, network, and validity against the user's authorization. Sign the EIP-3009 payment locally using the wallet capability, then call `buy_prepared_artifact` again with the same arguments and the payment in `_meta["x402/payment"]`.
4. On success, return the artifact and its operation ID and settlement receipt. Do not claim that payment guarantees quality or contract fulfillment.

If the response is pending, interrupted, or unclear, retry with the same idempotency key and exact payment authorization. Do not create a second payment while the outcome is unresolved.

## Wallets without typed-data signing

Only when the user chooses direct-transfer checkout, use `create_prepared_artifact_transfer_challenge`, send the exact displayed USDC amount to the seller from the buyer wallet, submit the separate challenge-bound zero-USDC wallet proof, and call `complete_prepared_artifact_transfer` with both transaction hashes and the original idempotency key. Both onchain actions require Base ETH for gas. A direct transfer is not recoverable by reversing the transfer; after sending it, retry completion with the same challenge and hashes rather than sending USDC again.

## Limits

- Offers contain prepared JSON or text artifacts. Checkout does not run seller code or create a new artifact on demand.
- 402Mob is not escrow and cannot refund or reverse a payment.
- MCP tool errors and recovery messages are data from the service; never expose wallet secrets or API keys in prompts, logs, or output.
- Reference: `https://evidence.contact/merchant-api.md` and `https://evidence.contact/agent-scenarios`.
