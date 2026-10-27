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

- Enrollment is headless: `POST /v1/merchants/enrollment-challenges` accepts a wallet and merchant name. The agent signs the returned message with EIP-191 using its own wallet signer, then submits the signature to `POST /v1/merchants/enrollments`.
- Enrollment challenges expire after five minutes and bind the origin, payout wallet, merchant name, and nonce. The server verifies EOA signatures; smart-contract wallet signatures are outside the supported scope.
- A payout wallet can enroll once. If that merchant loses its API key, use the wallet-signed rotation flow rather than attempting a second enrollment.
- Enrollment returns a merchant API key once. The service stores only its hash. The agent is responsible for storing the bearer key in its own secret manager; 402Mob does not supply that secret manager.
- Merchant keys have `merchant:manage` and `merchant:sales:read` scopes. They are long-lived. Wallet-signed rotation replaces the key and invalidates the old one. Do not describe expiring merchant sessions as implemented.
- Merchants publish fixed-price prepared JSON or text artifacts through `/v1/agent/merchant/offers`. Content and price are immutable after creation; offers can be activated or deactivated. Artifacts are bounded (maximum 48 KiB) and encrypted at rest. Checkout does not execute callbacks or arbitrary seller code.
- Buyers discover offers and purchase through HTTP x402 checkout or MCP (`discover_merchant_offers`, `buy_prepared_artifact`). The buyer signs EIP-3009 payment terms locally. Settlement goes to the merchant's enrolled Base wallet; the service advertises no platform fee.
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
- 402Mob is not escrow and does not refund or reverse payments. Retained merchant artifacts are prepared content, not dynamically executed jobs.

## Verified agent-to-agent run

- The fresh live merchant suite has completed a real Base run covering a buyer-to-seller invoice transfer, expected-sender invoice match, wrong-sender rejection, recovery after an interrupted paid-report response, prepared-artifact purchase, buyer-signed receipt, seller-owned sale read, and scoped coordinator access.
- The coordinator successfully reconciled a report while its settlement was pending. This is timing-dependent; later production runs may find the operation already settled. A settled operation must reject reconciliation.
- The fresh run cost 0.081 USDC plus Base gas. The service transfers were 0.02 USDC per invoice check, 0.001 USDC for the text report, and 0.02 USDC for the artifact; the buyer separately sent 0.02 USDC to the seller as the invoice.
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
