# Changelog

All notable changes to the skill, the tool files and the reference in this repository. The packages have their own versions on npm and PyPI.

## 2026-11-25, the API address is api.agentpay.global

- The base URL is now `https://api.agentpay.global`. The old address `https://api.joomjoo.com` keeps working and reaches the same API, so nothing already installed breaks.
- The pages a cardholder opens (connect a card, approve a purchase, claim a shared card) are now on `https://claim.agentpay.global`.
- Matches @agentpay-global/sdk 0.3.1 and PyPI agentpay-global 0.3.1, whose default base URL is the new address.

## 2026-10-31, AgentPay

- The product is AgentPay. The repository is now `Kato-Official/agentpay-skills` (the old `joomjoo-skills` address redirects), the skill is `agentpay`, the key variable is `AGENTPAY_API_KEY` (`JOOMJOO_API_KEY` is still read by the MCP server), the classes are `AgentPay` and `AgentPayError`.
- New package names: npm `@agentpay-global/sdk` and `@agentpay-global/mcp`, PyPI `agentpay-global` (`import agentpay`). The `get-agentpay` names on npm and PyPI are mirrors that install the same code. The old `@joomjoo/*` and `joomjoo` packages stay published as they were.
- Matches @agentpay-global/sdk 0.3.0, @agentpay-global/mcp 0.3.0, PyPI agentpay-global 0.3.0. The API host stays api.joomjoo.com.

## 2026-09-26, wallet

- The wallet: `GET /v1/wallet`, `POST /v1/wallet/funding-sessions` (a Stripe top-up page), `POST /v1/wallet/bank-transfers` (a declared transfer with the account and a reference code). Webhook event `money.funded`.
- Two more tools everywhere: `get_wallet` and `create_topup_link`. Six tools in the skill, AGENTS.md, the MCP server and the tool files.
- Matches @agentpay-global/sdk 0.2.0, @agentpay-global/mcp 0.2.0, PyPI agentpay 0.2.0.

## 2026-09-26

- First public release: the `agentpay` skill (SKILL.md), AGENTS.md, the MCP config, the Anthropic, OpenAI, Gemini and LangChain tool files, the API reference and the Postman collection.
- Matches @agentpay-global/sdk 0.1.1, @agentpay-global/mcp 0.1.1, PyPI agentpay 0.1.1 and the live API: ten checkout statuses, display-string totals, idempotency_key, Joomjoo-Signature on webhooks.
