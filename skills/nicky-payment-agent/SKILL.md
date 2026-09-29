---
name: nicky-payment-agent
description: >-
  Discover and pay Nicky cryptocurrency payment requests on behalf of a user (public,
  no-auth flow), and manage an authenticated Nicky account — payment requests, balances,
  credits, sub-accounts, webhooks — via the private Nicky MCP server. Use this whenever the
  user shares a pay.nicky.me / nicky.me link or short ID, asks to pay a Nicky invoice, or
  asks about their own Nicky account, balance, payment requests, or reports.
metadata:
  short-description: Pay and manage Nicky crypto payments
---

# Nicky Payment Agent Skill

## What is Nicky?

Nicky is a modern cryptocurrency payment platform. Users create payment requests and share a link
to a payment page, for example `https://pay.nicky.me/payment-report/ABCDE?paymentId=VWXYZ` (where
`ABCDE` is the receiver's profile short ID and `VWXYZ` is the payment request / bill short ID) or
`https://pay.nicky.me/s/ABCDE` for a profile. Anyone — including AI agents — can discover the
request, receive a wallet address, and pay it in cryptocurrency.

## URL Formats

Use these canonical link formats when referencing Nicky:

- **Profile (by short ID):** `https://pay.nicky.me/s/ABCDE` where `ABCDE` is the user's short ID
- **Profile (by email):** `https://pay.nicky.me/e/EMAIL`
- **Profile (by domain):** `https://pay.nicky.me/d/DOMAIN`
- **Payment request:** `https://pay.nicky.me/payment-report/ABCDE?paymentId=VWXYZ` where `ABCDE` is
  the receiver's short ID and `VWXYZ` is the payment request / bill short ID

## What Can Agents Do on This Website?

### 1. Discover Payment Requests

Agents can look up a payment request by its short ID (minimum 6 characters) to learn:

- Who receives the payment (`receiverUser` — public profile: id, email, name, public name, website, bio, country, has-profile-picture)
- `description` and `invoiceReference` from the underlying bill
- `totalAmount` and `openAmount` (remaining unpaid) in the native asset
- `nativeAssetId` — the asset the request is denominated in
- `status` — `PaymentPending` means the request is open and can be paid
- `availableAssets` — every asset the receiver accepts, with:
  - `assetId` — identifier to use in `start-payment`
  - `assetName` — human-readable name
  - `isFiat` / `decimalPrecisionUI` — formatting hints
  - For the native asset, the open amount is included on the entry; for non-native assets, the exact
    converted amount is returned by `start-payment` only.

**Endpoint:**
```
GET https://api-public.pay.nicky.me/api/agents/payment-request?shortId=ABCDEF
```

### 2. Start a Payment

Agents can initiate a payment on behalf of a user. Nicky fetches a live conversion quote
automatically when a non-native asset is chosen, and returns the exact amount to send in the chosen
asset. The response includes:

- `paymentAttemptId` — short ID for tracking
- `walletAddress` — send funds here
- `memo` — destination tag for networks that need one (null for most networks)
- `amountToSend` — exact amount in `assetId`, already converted if needed
- `assetId` / `assetName`
- `paymentRequestShortId` — the request this attempt belongs to
- `expiresAt` — 30-minute payment window
- `createdAt` — when the attempt was created

**Endpoint:**
```
POST https://api-public.pay.nicky.me/api/agents/payment/start
```

**Body:**
```json
{
  "paymentRequestShortId": "ABCDEF",
  "assetId": "USDT_TRC20",
  "payerName": "John Doe",
  "payerEmail": "john@example.com"
}
```

Idempotency: calling `start-payment` again with the same `paymentRequestShortId`, `assetId`, and
`payerEmail` returns the same `paymentAttemptId` and `walletAddress`.

### 3. Report Transaction Hashes

After the user (or the agent's own wallet) broadcasts the blockchain transaction, agents can report
the hash to skip the deposit-polling stage and jump directly to blockchain validation. The response
is the updated payment progress.

**Endpoint:**
```
POST https://api-public.pay.nicky.me/api/agents/payment/report-transaction
```

**Body:**
```json
{
  "paymentAttemptId": "X7K2P1",
  "transactionHash": "abc123..."
}
```

### 4. Poll for Payment Confirmation

Poll this endpoint until the payment is confirmed.

**Endpoint:**
```
GET https://api-public.pay.nicky.me/api/agents/payment/progress?paymentAttemptId=X7K2P1
```

**Poll every 15–30 seconds.** The payment is complete when `status == "Confirmed"` OR
`isPaid == true`.

**Response fields:**
- `paymentAttemptId`, `walletAddress`, `expectedAmount`, `assetId`, `assetName`
- `status` — one of the values below
- `isPaid` — true once the request is fully settled
- `createdAt`, `expiresAt`, `finishedAt`
- `paymentRequest` — the underlying `PaymentRequest` object (id, amount, status, success/cancel URLs, etc.)
- `transactions` — list of `AgentTransactionSummary` items: `transactionHash`, `assetId`, `receiverAddress`, `createdDate`, `blockchainStatus`, `paymentReportStatus`
- `suggestedTransactionHash` — the hash reported by the agent (if any)

**Status values:**
| Status      | Meaning                                                            |
|-------------|--------------------------------------------------------------------|
| `Pending`   | Waiting for the transaction to appear on-chain                     |
| `Received`  | Transaction detected, waiting for confirmations                    |
| `Confirmed` | Payment confirmed and matched to the request                       |
| `Expired`   | Payment window closed without a detected transaction               |

---

## MCP Server Support

Nicky exposes a Model Context Protocol (MCP) server for agents that support it. The same four
public tools are exposed alongside `SearchNicks` (a public user-search tool).

- **Public MCP** (no auth): `https://api-public.pay.nicky.me/mcp-public/`
  Tools: `GetPaymentRequest`, `StartPayment`, `ReportTransaction`, `GetPaymentProgress`, `SearchNicks`
- **Private MCP** (API key via `X-Api-Key`): `https://api-public.pay.nicky.me/mcp/`
  Tools: `GetProfile`, `AgreePrivacyPolicy`, `UpdateProfile`, `GetAllNicks`, `AddNicks`, `GetBalances`,
  `GetAcceptedAssets`, `GetUserAssetConnections`, `GetDepositAddress`, `GetConversionQuote`,
  `GetUnusedConversionQuote`, `CreatePaymentRequest`, `GetPaymentRequests`, `GetPaymentRequestById`,
  `GetPaymentRequestByShortId`, `CancelPaymentRequest`, `FinishPaymentRequest`,
  `ResendPaymentRequestEmail`, `GetPaymentOrders`,
  `GetPaymentOrderById`, `GetPaymentOrdersByBillShortId`, `GetCreditBalance`,
  `GetCreditBalanceForSubAccount`, `ListCreditTransfers`, `TransferCredits`, `GetAllBillingGroups`,
  `ListSubAccounts`, `UnlinkSubAccount`, `CreateSubAccount`, `CreateWebHook`, `ListWebHooks`,
  `DeleteWebHook`, `Echo`, plus all of the public tools above. `GetPaymentReports` also exists as a
  deprecated alias of `GetPaymentOrders` — kept for backward compatibility only; use
  `GetPaymentOrders` for new integrations.

All private payment-request tools (read, create, cancel, finish, resend) only operate on payment
requests owned by the authenticated caller — calling one with another user's short ID returns an
authorization error, never that other user's data.
- **Server card:** https://api-public.pay.nicky.me/.well-known/mcp/server-card.json

**Available MCP tools (public):**
- `GetPaymentRequest` – discover payment details by short ID
- `StartPayment` – initiate a payment and get a wallet address
- `ReportTransaction` – report a blockchain transaction hash (returns the updated progress)
- `GetPaymentProgress` – poll payment confirmation status
- `SearchNicks` – search for users by nick, email, or domain

---

## Conversational Flow for the Public/Payer Tools (State Machine)

The payer flow is **non-custodial**: Nicky never holds or moves the payer's funds. The payer's own
wallet sends the transfer; the agent's job is discovery, quoting, and narration — not settlement.
Treat every payer conversation as moving through these states, in order, and never skip or
silently combine them:

| State | Trigger | What the agent must say |
|---|---|---|
| `Discovering` | Agent calls `GetPaymentRequest`/`get-payment-request` | Narrate who is being paid, what for, and the open amount — never paste the raw response |
| `Quoted` | Payer has chosen an asset (native or converted) | State the exact amount and asset in plain language, e.g. "That's 42.10 USDT (TRC-20)." |
| `AwaitingConfirmation` | Immediately before calling `StartPayment`/`start-payment` | Ask an explicit yes/no: "Want me to start this payment? You'll need to send the funds from your own wallet." **Do not call `StartPayment` until the payer explicitly confirms in this turn.** |
| `Submitted` | `StartPayment` has returned | Give the wallet address, exact amount, memo (if any), and expiry — this is the moment funds must move, from the payer's wallet, outside the chat |
| `Pending` (poll) | `GetPaymentProgress` returns `Pending`/`Received` | Narrate plainly, e.g. "Still waiting for the transaction to confirm on-chain — this can take a few minutes depending on the network." **Never repeat an identical status line on consecutive polls** — vary the wording and, past ~3 polls, mention that some networks (e.g. Bitcoin) can take up to two hours |
| `Confirmed` / `Failed` | `status == "Confirmed"` or `isPaid == true`, or `status == "Expired"` | State the outcome plainly. Only ever declare a payment complete when the tool response says so — never infer or predict completion ahead of the actual status |

**Hard rules:**
- Never call a state-changing tool (`StartPayment`/`start-payment`, `ReportTransaction`/`report-transaction`) without the confirmation turn in `AwaitingConfirmation` immediately preceding it in the same conversation.
- Never surface raw JSON tool output to the payer — always narrate through the state's template above.
- Never state or imply a payment is complete unless the polled `status`/`isPaid` field says so. If the payer asks "is it done yet?" while still `Pending`, say so plainly rather than reassuring them it's probably fine.
- If `GetPaymentRequest` or `StartPayment` errors (expired, wrong asset, already paid), narrate the specific reason from the error — never invent a wallet address or amount to keep the conversation moving.

---

## Ambiguous Request Handling for the Private/Authenticated Tools

The private MCP surface has ~30 tools, several of which sound like plausible matches for the same
everyday phrase. When a request is ambiguous, **ask which tool/resource is meant — never guess** on
any tool that writes, deletes, or moves money or permissions. Read-only lookups can tolerate a
best-effort guess if you state which one you picked.

| What the user might say | Candidate tools (do not silently pick one) | Required behavior |
|---|---|---|
| "What's my balance?" | `GetBalances` (asset/wallet balances) vs. `GetCreditBalance` / `GetCreditBalanceForSubAccount` (Nicky account credit) | Ask which balance they mean, or state you're showing both |
| "Cancel"/"clear"/"close out" a payment request | `CancelPaymentRequest` (voids it, unpaid) vs. `FinishPaymentRequest` (marks it fulfilled/closed) | Ask whether the request was paid — these have opposite meanings and neither is reversible |
| "Remove"/"unlink" an account | `UnlinkSubAccount` (detaches a sub-account) vs. `DeleteWebHook` (removes a webhook) vs. no tool exists for removing a Nick (only `AddNicks` exists) | Confirm which resource (sub-account, webhook, nick) before calling anything; say plainly if no removal tool exists for nicks |
| "Transfer credits"/"send credits" | `TransferCredits` | Confirm exact amount and destination sub-account/user before calling — this moves balance between accounts and is a one-way action |
| "Show my accounts" | `ListSubAccounts` vs. `GetAllBillingGroups` (billing groups, not accounts) | Ask which they mean if unclear from context |
| "Set up a webhook"/"add a webhook" | `CreateWebHook` | Confirm the target URL and event type explicitly before calling; never infer a URL from earlier context |
| "Update my profile" | `UpdateProfile` vs. `AddNicks` (adding a nick is not the same as editing profile fields) | Confirm which fields are being changed vs. which nick is being added |
| "Resend the payment request" / "email them again" | `ResendPaymentRequestEmail` (resends the existing request notification) vs. `CreatePaymentRequest` (makes a new one) | Confirm the request already exists and just needs re-notifying, rather than creating a duplicate |

**Hard rule:** any tool that creates, cancels, finishes, deletes, unlinks, or transfers something
requires an explicit confirmation turn stating exactly what will happen, in plain language, before
the call — the same pattern as `StartPayment` in the public flow above. Read-only tools
(`Get*`/`List*`) do not require confirmation but should state which resource was queried if the
request was ambiguous.

---

## OpenAPI & Documentation

- **OpenAPI spec (agents):** https://api-public.pay.nicky.me/openapi/agents.json
- **OpenAPI spec (full public API):** https://api-public.pay.nicky.me/openapi/v1.json
- **Interactive docs:** https://api-public.pay.nicky.me/scalar
- **Agent integration guide:** https://api-public.pay.nicky.me/agents.md

---

## Payment Processing Pipeline (for setting expectations)

After a payer sends funds, Nicky's backend runs a sequence of background workers to discover, validate, match, and finalize the payment.

| Stage | What happens | Polling interval |
|---|---|---|
| Deposit discovery | Exchange connector polled for new deposits | every 30 s |
| Blockchain validation | On-chain tx verified; **10+ confirmations required** | every 60 s |
| Transaction matching | Deposit matched to the payer's 30-minute address lease (±2% amount tolerance) | every 35 s |
| Payment report creation | Report linked to the payment request | every 45 s |
| Finalization | Fraud/blacklist checks, quote validation, confirmation email sent | every 61 s |

**Typical end-to-end confirmation time:**
- TRC-20 / Polygon: **3–8 minutes**
- Ethereum: **5–15 minutes**
- Bitcoin: **60–120 minutes**

**Speed up confirmation:** submit the tx hash via `report-transaction` immediately after
broadcasting. This skips deposit polling and jumps straight to blockchain validation.

---

## Important Notes

- **No API key required** for public agent endpoints.
- **Payment window:** 30 minutes from calling `start-payment`. Call again to renew (idempotent).
- **Conversion:** Nicky handles asset conversion transparently. The exact converted amount is
  returned by `start-payment`. Indicative amounts are NOT included in `get-payment-request` for
  non-native assets.
- **Amount tolerance:** Deposits are matched with **±2%** tolerance. Sending the exact quoted
  amount is strongly recommended.
- **Memo/destination tag:** for networks that require a memo or destination tag, the
  `start-payment` response includes a `memo` field — pass it to the wallet.
- **URLs to share with the user:** profile `https://pay.nicky.me/s/ABCDE`, payment request
  `https://pay.nicky.me/payment-report/ABCDE?paymentId=VWXYZ`.
- **Support:** support@nicky.me
