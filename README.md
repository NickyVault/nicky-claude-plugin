# Nicky

Nicky is a non-custodial cryptocurrency payment platform. This plugin lets Claude discover and pay
Nicky payment requests on a user's behalf, and — once a user connects their own Nicky account —
manage that account: payment requests, reports, webhooks, balances, credits, and sub-accounts.

Nicky never holds, receives, converts, or controls the funds being transferred. Settlement moves
directly from the payer's own wallet to the payee's chosen endpoint; Claude's role is discovery,
quoting, and narration, never custody.

## What this installs

- **`nicky-public`** — the public Nicky MCP server (no authentication). Lets Claude look up a
  payment request from a `pay.nicky.me` link or short ID, start a payment, report a transaction
  hash, and poll for confirmation.
- **`nicky`** — the private Nicky MCP server. Requires connecting your own Nicky account (OAuth) on
  first use. Lets Claude manage payment requests, reports, webhooks, balances, credits, and
  sub-accounts on your behalf.
- **`nicky-payment-agent`** skill — the behavior rules Claude follows for both flows: a defined
  narration state machine for the payer flow (never surfaces raw tool output, always confirms
  before starting a payment), and an explicit disambiguation table for the private flow so Claude
  asks before guessing which of ~30 similarly-named tools an ambiguous request means.

## Getting started

- To pay a Nicky payment request: paste a `pay.nicky.me` link or give Claude the short ID.
- To manage your own Nicky account: ask Claude to check a balance, list payment requests, etc. —
  you'll be prompted to connect your Nicky account the first time.

## Links

- Website: https://pay.nicky.me
- Agent integration guide: https://api-public.pay.nicky.me/agents.md
- OpenAPI: https://api-public.pay.nicky.me/openapi/v1.json
- MCP server card: https://api-public.pay.nicky.me/.well-known/mcp/server-card.json
- Support: support@nicky.me

## License

MIT
