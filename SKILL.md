---
name: 402mob-operations
description: Maintain this 402Mob repository's implemented Base x402 service, headless merchant enrollment, MCP and HTTP APIs, documentation, or deployment. Use for feature work and operational questions about this codebase.
---

# 402Mob operations

Use this skill when working on the 402Mob repository. Treat the current code and capability endpoint as authoritative; keep documentation limited to shipped behavior. Do not describe the removed web dashboard or proposed features as available.

## Start with the current implementation

- Read `README.md` for local development, payment/recovery invariants, and operator API usage.
- Read `public/merchant-api.md` for the seller enrollment and offer workflows.
- Inspect `src/app.ts` and `src/mcp.ts` before changing or documenting routes and tools.
- Use `GET /v1/capabilities` as the production inventory of HTTP, MCP, wallet, and merchant capabilities. Verify details against source when they affect correctness.
- Public pages are in `public/`; the site uses static HTML and CSS served by the app.
- `public/dashboard.html`, its assets, and the `/dashboard` route have been removed. Operator functionality remains in authenticated `/v1/admin/*` endpoints. Merchant self-service is API-only.

## Shipped merchant workflow

- Enrollment is headless: `POST /v1/merchants/enrollment-challenges` accepts a wallet and merchant name. The agent can sign the returned message with EIP-191 or submit a challenge-bound zero-USDC Base transaction proof to `POST /v1/merchants/enrollments/transaction`.
- Enrollment challenges expire after five minutes and bind the origin, payout wallet, merchant name, nonce, and a random transaction-proof address. Agents can enroll with EIP-191 or by submitting a confirmed Base USDC `approve(proofAddress, 0)` / `transfer(proofAddress, 0)` transaction. The transaction must be mined and submitted before the challenge expires. For `transfer_zero`, the transaction sender must equal the merchant wallet. The on-chain proof moves no USDC and grants no allowance, but costs Base ETH gas. 402Mob checks the challenge-specific event, transaction window, success, and configured confirmation depth; it rejects any positive USDC spend/approval by the wallet or ETH value transfer in that proof transaction. Generic swaps, sends, and nonzero approvals do not qualify.
- A payout wallet can enroll once. If that merchant loses its API key, use the EIP-191 or transaction-proof credential-rotation flow rather than attempting a second enrollment.
- Enrollment returns a merchant API key once. The service stores only its hash. The agent is responsible for storing the bearer key in its own secret manager; 402Mob does not supply that secret manager.
- Merchant keys have `merchant:manage` and `merchant:sales:read` scopes. They are long-lived. EIP-191 or the same challenge-bound zero-USDC transaction proof can rotate the key and invalidates the old one. Do not describe expiring merchant sessions as implemented.
- Merchants publish fixed-price prepared JSON or text artifacts through `/v1/agent/merchant/offers`. Each offer is immutable and belongs to a version series. The first version defaults to `1.0.0`; publish an update with a higher numeric `major.minor.patch` version and `supersedesOfferId` set to the current offer ID. The new version automatically unpublishes its predecessor. Buyers can resolve a saved ID through `GET /v1/merchants/{merchantId}/offers/{offerId}/latest` or MCP `resolve_latest_merchant_offer`. Superseded versions cannot be republished, while existing purchases remain recoverable. Artifacts are bounded (maximum 48 KiB) and encrypted at rest. Checkout does not execute callbacks or arbitrary seller code.
- Buyers discover offers and purchase through HTTP x402 checkout or MCP (`discover_merchant_offers`, `buy_prepared_artifact`), signing EIP-3009 terms locally. Wallets that cannot sign typed payment data can use 402Mob's direct-transfer checkout tools: send USDC directly to the enrolled merchant, then submit a separate challenge-bound zero-USDC wallet-control proof. Direct transfer is limited to 402Mob offers. Both transactions must be included within five minutes; after confirmation, completion and recovery remain available for 24 hours. Both transactions require Base ETH gas, and 402Mob cannot refund a direct payment. It is not a general x402 proxy.
- Payment quotes advertise each prepared artifact's actual content type (`application/json` or `text/plain; charset=utf-8`) in both HTTP and MCP.
- HTTP checkout requires an empty JSON object and a persisted `Idempotency-Key`; MCP purchases use the same idempotent retry principle and carry payment in `_meta["x402/payment"]`.
- Checkout does not create a delivery receipt automatically. A buyer can separately sign and submit an `accepted` or `disputed` receipt with EIP-191. A seller can read its own sale and attached receipt proofs through `GET /v1/agent/merchant/sales/{operationId}`.
- Preserve idempotency and recovery behavior: after a timeout or pending result, reuse the original request, idempotency key, and payment authorization. Never suggest issuing a new payment while the original operation is unresolved.

## Other shipped workflows

- Built-in Base transaction and invoice reports cost 0.02 USDC; the text demo costs 0.001 USDC. The invoice report checks an existing transfer from a required `expectedSender` to a required `expectedRecipient` for the expected amount; `sender_mismatch` means the recipient got the amount from another wallet. It does not send the invoice payment.
- Delivery receipts are append-only wallet statements tied to an operation and result hash. They do not prove work quality or contract compliance.
- Scoped coordinator keys are provisioned by operator APIs under `/v1/admin/agents`; use only the needed scopes and operation assignments. `/v1/admin/operations/:id/reconcile` reconciles an existing uncertain settlement and never submits a payment.
- If the facilitator response is lost before it returns a transaction hash, the background reconciler searches Base USDC `AuthorizationUsed` logs from the operation's recorded start block for its payer and nonce. It then validates the transaction's authorization, recipient, amount, and confirmation depth before releasing the retained result. Hashless searches are throttled to once per minute. No matching log is not proof that no payment was made, and the service never resubmits an ambiguous settlement.
- The MCP endpoint is Streamable HTTP at `/mcp`. Consult `src/mcp.ts` and `/v1/capabilities` for the exact tool inventory.
- The MCP wallet onboarding bridge exposes `create_merchant_enrollment_challenge` and `complete_merchant_enrollment_transaction`, plus corresponding key-rotation tools. Call the challenge tool first, then read `challengeId`, the exact EIP-191 `message`, `transactionProof.proofAddress`, and `walletActions` from `structuredContent` or the JSON in `content[0].text`. Sign only the exact returned message for EIP-191; the proof address is only needed for the transaction route. If those challenge values are missing, stop and retry the challenge request rather than signing blind. The transaction completion verifies the wallet's transaction hash; 402Mob never signs or submits the transaction. The merchant key is returned once in the completion result, so the agent runtime must protect it from logs and store it in its own secret manager.
- MCP direct checkout uses `create_prepared_artifact_transfer_challenge` and `complete_prepared_artifact_transfer`. Persist the idempotency key, payment hash, and wallet-control proof hash. If delivery is interrupted, retry with the same values; never prompt the buyer to send again while that operation is unresolved.
- 402Mob is not escrow and does not refund or reverse payments. Retained merchant artifacts are prepared content, not dynamically executed jobs.

## Verified agent-to-agent run

- Production runs verified a buyer-to-seller invoice transfer, expected-sender invoice match, wrong-sender rejection, recovery after an interrupted paid-report response, x402 and direct-transfer artifact purchases, buyer-signed receipt, seller-owned sale read, and scoped coordinator evidence.
- The original fresh run successfully reconciled a report while its settlement was pending. This is timing-dependent; a later replay found the operation already settled and verified reconciliation was correctly refused.
- The original fresh run cost 0.081 USDC plus Base gas. The expanded direct-checkout run added one 0.02 USDC transfer directly to the seller plus Base gas. Its buyer-control proof was an `approve(proofAddress, 0)` transaction, which moved no USDC and created no allowance.
- The direct purchase completed after the five-minute transaction-inclusion window within the 24-hour recovery period. A replay-only run returned the same settled operation and evidence without a second payment. The MCP `challengeId` schema fix was deployed before completion.
- The local MCP suite includes a controlled pending-settlement case. It verifies the assigned coordinator can reconcile through MCP without submitting payment; this local fixture does not send Base transactions.
- The production suite validates `A2A_OPERATOR_TOKEN` by creating and connecting an unassigned temporary coordinator before paid workflows. It revokes the temporary key at cleanup. Keep the signed-payment journal for retries; use a unique `A2A_JOURNAL_TAG` to start a fresh live run without overwriting the default journal. Tagged runs reuse only the matching seller enrollment and offer from the default journal.
- Do not run a live-payment suite or create a fresh tagged run unless the user explicitly requests it. State the remaining-spend estimate and Base gas requirement before the runner's typed `RUN` confirmation.

## Updating the site and documentation

When changing shipped functionality, update the relevant public pages and `README.md`, `public/merchant-api.md`, `public/llm.txt`, `tests/README.md`, or `deploy/README.md` as needed. Check all navigation and API links after removing or renaming routes. Keep the homepage aligned with headless enrollment and the current buyer/seller flow; distinguish buyer-signed receipts from checkout.

Do not add future API sessions, seller dashboards, smart-wallet support, refunds, escrow, or other unimplemented behavior to the site. If discussing a proposal, label it clearly as future work.

## Verification and deployment

- Do not add or run tests unless the user asks for testing or verification. For ordinary doc edits, inspect the rendered/static source and references without running the suite.
- If a code or public-site change is requested for production, build and deploy only when the user has asked for deployment or the active task clearly includes publishing. Follow `deploy/README.md`; never print secrets or include `.env`, `.local/`, or database files in an archive.
- Never run live payment flows, transfer funds, rotate production credentials, or expose secret keys unless the user explicitly requests that specific action.
