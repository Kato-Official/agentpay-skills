# AgentPay API reference

Generated from the same specs as the docs site. The full documentation, with guides, SDKs and runnable examples, is at https://agentpay.global/docs/

| | |
| --- | --- |
| **Base URL** | `https://api.joomjoo.com` |
| **Version** | Every endpoint is under `/v1`. |
| **Auth** | `Authorization: Bearer mk_live_...` on every request. See [authentication](https://agentpay.global/docs/authentication/). |
| **Content type** | `application/json` for every request body. |
| **Amounts** | US dollars as plain numbers: `"amount": 20` is $20.00. Card and wallet math is exact to the cent internally. |
| **Request ids** | Every response carries `X-Mooj-Request-Id`, echoed as `request_id` in the body. |

> The host still carries the company's older name. `api.joomjoo.com` is the AgentPay API, and the older Cloud Run address keeps answering for integrations written before the rename.

## Endpoints

- [Create a card](https://agentpay.global/docs/api-reference/cards/create/): mint a Visa or Mastercard virtual card capped at an amount.
- [Reveal a card](https://agentpay.global/docs/api-reference/cards/reveal/): the full number, security code and expiry, once.
- [Close a card](https://agentpay.global/docs/api-reference/cards/close/): release the unused reserve.
- [List cards](https://agentpay.global/docs/api-reference/cards/list/): connected and issued cards on the account.
- [Connect the person's own card](https://agentpay.global/docs/api-reference/cards/connect/): a one-hour link for the cardholder.
- [Remove a connected card](https://agentpay.global/docs/api-reference/cards/remove/).
- [Get spend status](https://agentpay.global/docs/api-reference/spend/get/): was an issued card charged.
- [Start a checkout](https://agentpay.global/docs/api-reference/checkout/create/): hand AgentPay the whole purchase.
- [Get checkout status](https://agentpay.global/docs/api-reference/checkout/get/): poll a run until it ends.
- [Create a connection](https://agentpay.global/docs/api-reference/connections/create/), [complete](https://agentpay.global/docs/api-reference/connections/complete/), [list](https://agentpay.global/docs/api-reference/connections/list/), [get](https://agentpay.global/docs/api-reference/connections/get/), [skip the passkey password](https://agentpay.global/docs/api-reference/connections/passkey/), [disconnect](https://agentpay.global/docs/api-reference/connections/delete/).
- [Read the wallet](https://agentpay.global/docs/api-reference/wallet/get/), [top up by card or stablecoin](https://agentpay.global/docs/api-reference/wallet/funding-sessions/), [top up by bank transfer](https://agentpay.global/docs/api-reference/wallet/bank-transfers/).
- [Webhook events](https://agentpay.global/docs/api-reference/webhooks/events/) and [signature verification](https://agentpay.global/docs/api-reference/webhooks/verify/).
- [Errors](https://agentpay.global/docs/api-reference/errors/).

## Idempotency

`POST /v1/checkout` and `POST /v1/wallet/bank-transfers` take an `idempotency_key`, your own reference for the operation. A retry with the same key returns the same checkout or declaration instead of creating a second one. Always send one on a checkout.

## Lookups never leak

A card, spend, checkout or connection that belongs to another account answers 404 with the same body as one that does not exist.

## No test mode

Every call is live. A card you create is real and reserves its cap; a checkout you start buys. Use a small cap to try things, and close cards you do not use.

## Authentication

Every request carries `Authorization: Bearer mk_live_...`. Keys are created in the AgentPay console under Developer, API keys, shown once. A missing key answers 401 `missing_api_key`, an unknown one 401 `invalid_api_key`, an expired one 401 `expired_api_key` with `expired_at`. Every response carries `X-Mooj-Request-Id`, echoed as `request_id`.

## Cards

### Create a card

`POST /v1/cards`

Mint a virtual card capped at amount, dedicated to one merchant. The response is a real card; reveal the number with GET /v1/cards/{id}.

**Body** (application/json)

| Field | Type | Description |
| --- | --- | --- |
| `amount` | number, required | The spend cap in US dollars, above zero. The per-card ceiling is 5,000. |
| `merchant` | string | The merchant the card is for, e.g. `namecheap`. Shown in the console. Defaults to `agent`. |
| `single_use` | boolean, default `true` | A single-use card freezes itself after its first charge. `false` mints a multi-use card that takes charges until the cap is reached or you close it. |
| `network` | string, default `auto` | Which network to mint on. One of `auto`, `visa`, `mastercard`. |
| `card_name` | string | A label for the card in the console, up to 60 characters. Defaults to `API - {merchant}`. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The card id. Use it to reveal or close the card. |
| `brand` | string | The network the card was minted on. One of `visa`, `mastercard`. |
| `last4` | string | The last four digits. |
| `exp_month` | integer | Expiry month, 1 to 12. |
| `exp_year` | integer | Expiry year, four digits. |
| `spend_limit` | number | The cap in dollars, as requested. |
| `single_use` | boolean | Whether the card freezes after its first charge. |
| `status` | string | The card status, `ACTIVE` on creation. |
| `spend_request_id` | string | Pass it to `GET /v1/spend/{spend_request_id}` to learn whether the card was charged. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "id": "card_8f2a...",
  "brand": "visa",
  "last4": "8339",
  "exp_month": 4,
  "exp_year": 2033,
  "spend_limit": 20,
  "single_use": true,
  "status": "ACTIVE",
  "spend_request_id": "spr_1c9d...",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 400 | `invalid_amount` | `amount` is missing, not a number, or not above zero. |
| 402 | `insufficient_funds` | The cap exceeds the wallet's `available`. Fund the wallet, then retry. |
| 502 | `card_issue_failed` | The issuer refused or failed. The message says why. |

```bash
curl -X POST https://api.joomjoo.com/v1/cards \
  -H "Authorization: Bearer $AGENTPAY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"amount":20,"merchant":"namecheap","single_use":true}'
```

### Reveal a card

`GET /v1/cards/{id}`

Return the full card number, the security code and the expiry. Reveal only at the moment of use; AgentPay never logs the number.

**Path parameters**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string, required | The card id from `POST /v1/cards`. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The card id. |
| `last4` | string | The last four digits. |
| `number` | string | The full card number. |
| `number_formatted` | string | The number in groups of four. |
| `cvc` | string | The security code. |
| `exp` | string | The expiry as `MM/YY`. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "id": "card_8f2a...",
  "last4": "8339",
  "number": "4775xxxxxxxx8339",
  "number_formatted": "4775 xxxx xxxx 8339",
  "cvc": "123",
  "exp": "04/33",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 404 | `card_not_found` | No such card on your account. |
| 502 | `reveal_failed` | The issuer could not return the card. Retry. |

```bash
curl https://api.joomjoo.com/v1/cards/card_8f2a... \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

### Close a card

`POST /v1/cards/{id}/close`

Close an issued card and release any unused reserve back to the wallet.

**Path parameters**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string, required | The card id. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The card id. |
| `status` | string | `closed`. |
| `released_display` | string | The reserve returned to the wallet, as a display string such as `$20.00`. |
| `available_to_issue_display` | string | The wallet's available after the release. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "id": "card_8f2a...",
  "status": "closed",
  "released_display": "$20.00",
  "available_to_issue_display": "$140.00",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 404 | `card_not_found` | No such card on your account. |
| 502 | `close_failed` | The issuer could not close the card. Retry. |

```bash
curl -X POST https://api.joomjoo.com/v1/cards/card_8f2a.../close \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

### List cards

`GET /v1/cards`

Every card on the account: the person's connected cards first (source own), then issued cards (source issued).

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `cards` | array | Connected cards first, then issued cards. |
|   `id` | string | The card id. |
|   `source` | string | `own` for a connected card, `issued` for an AgentPay card. One of `own`, `issued`. |
|   `brand` | string | `visa` or `mastercard`. |
|   `last4` | string |  |
|   `exp` | string | The expiry as `MM/YY`. |
|   `label` | string | The card's label: the cardholder's on a connected card, `card_name` on an issued one. |
|   `holder_name` | string | Connected cards only. The name on the card. |
|   `status` | string | `active` on a connected card; the issuer status on an issued one. |
|   `single_use` | boolean | Issued cards only. |
|   `spend_limit` | number | Issued cards only. The cap in dollars. |
|   `spent` | number | Issued cards only. Dollars charged so far. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "cards": [
    {
      "id": "9b2e...",
      "source": "own",
      "brand": "visa",
      "last4": "4242",
      "exp": "12/30",
      "label": "Visa ending 4242",
      "holder_name": "A. Example",
      "status": "active"
    },
    {
      "id": "card_8f2a...",
      "source": "issued",
      "brand": "visa",
      "last4": "8339",
      "exp": "04/33",
      "label": "API - namecheap",
      "status": "ACTIVE",
      "single_use": true,
      "spend_limit": 20,
      "spent": 0
    }
  ],
  "request_id": "req_a1b2c3"
}
```

```bash
curl https://api.joomjoo.com/v1/cards \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

### Connect the person's own card

`POST /v1/cards/connect`

A one-hour, single-use link where the cardholder connects their own credit or debit card behind a passkey. Send it to them; never open it yourself.

The card is encrypted on the cardholder's device before it leaves the page. AgentPay never holds the number in clear and never stores the security code. Nothing is issued and nothing is prepaid: they keep their points.

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `connect_id` | string | The enrolment id. |
| `connect_url` | string | The link for the cardholder. Safari or Chrome, on the phone they will approve purchases with. |
| `expires_at` | string | ISO 8601. One hour after creation. |
| `status` | string | `open`. |
| `instructions` | string | What to tell the cardholder, in plain words. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "connect_id": "6f1c...",
  "connect_url": "https://claim.joomjoo.com/add/6f1c...#...",
  "expires_at": "2026-11-03T15:00:00.000Z",
  "status": "open",
  "instructions": "Send the link to the cardholder. They open it in Safari or Chrome on the phone they will approve purchases with, type the card once, and lock it with a passkey. The security code is never stored. Then GET /v1/cards shows it with source \"own\".",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 503 | `own_card_unavailable` | Connecting a card is not switched on for the account. |
| 502 | `connect_failed` | The link could not be created. Retry. |

```bash
curl -X POST https://api.joomjoo.com/v1/cards/connect \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

### Remove a connected card

`DELETE /v1/cards/{id}`

Remove a connected card (source own) and withdraw any purchase waiting on it. An issued card is closed with POST /v1/cards/{id}/close instead.

**Path parameters**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string, required | The connected card's id from `GET /v1/cards`. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `id` | string | The card id. |
| `source` | string | `own`. |
| `status` | string | `removed`. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "id": "9b2e...",
  "source": "own",
  "status": "removed",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 404 | `card_not_found` | No connected card with that id. An issued card is closed with `POST /v1/cards/{id}/close`. |
| 502 | `remove_failed` | The card could not be removed. Retry. |

```bash
curl -X DELETE https://api.joomjoo.com/v1/cards/9b2e... \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

## Spend

### Get spend status

`GET /v1/spend/{spend_request_id}`

Whether a card issued with POST /v1/cards was charged, and whether the charge settled.

**Path parameters**

| Field | Type | Description |
| --- | --- | --- |
| `spend_request_id` | string, required | From the create-card response. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `spend_request_id` | string |  |
| `decision` | string | The policy decision at issue time, `APPROVED` for a card that was minted. |
| `merchant` | string | The merchant the card was issued for. |
| `requested_usd` | number | The cap that was requested. |
| `status` | string | The charge state. One of `not_started`, `authorized`, `cleared`, `declined`. |
| `settled` | boolean | `true` once the network cleared the charge, usually by the next day. |
| `amount` | number | The charged amount in dollars, `null` before a charge. |
| `decline_reason` | string | On `declined`, why. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "spend_request_id": "spr_1c9d...",
  "decision": "APPROVED",
  "merchant": "namecheap",
  "requested_usd": 20,
  "status": "cleared",
  "settled": true,
  "amount": 1.98,
  "decline_reason": null,
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 404 | `spend_not_found` | No such spend on your account. |
| 502 | `spend_status_failed` | The status could not be read. Retry. |

```bash
curl https://api.joomjoo.com/v1/spend/spr_1c9d... \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

## Checkout

### Start a checkout

`POST /v1/checkout`

Asynchronous. AgentPay opens the merchant in its own browser, reaches the review, mints the card for the real total, pays, and clears 3DS. Returns a checkout_id to poll.

**Body** (application/json)

| Field | Type | Description |
| --- | --- | --- |
| `task` | string, required | What to buy, in plain English. |
| `merchant` | string, required | The merchant to buy from, e.g. `namecheap` or `amazon`. |
| `amount` | number, required | The most to spend, in dollars. Becomes the card cap. |
| `confirm_pay` | boolean, default `false` | `false`: the run stops at the final review and waits for the account owner's yes in the console. `true`: pay without a human yes, up to the cap. The agent's own approval mode can only make this stricter; the response's `confirm_pay` says what was granted. |
| `idempotency_key` | string | Your own key for this checkout, up to 120 characters. A retry with the same key returns the same checkout instead of starting a second run. Always send one. |
| `connection_id` | string | Run signed in on a connection. When omitted, the newest connected connection for the merchant is used. |
| `own_card_id` | string | Pay with the person's connected card (`GET /v1/cards`, `source: "own"`) instead of an issued one. The run stops at the review until the cardholder approves on their phone; `confirm_pay` cannot skip that. |
| `resume_of` | string | The id of an ended checkout of yours that left its own items in the basket. The new run continues from that basket. |
| `card_holder_name` | string | The name to type on the card for this run. Issued cards only. |
| `start_url` | string | The page to open first: a full https address or a bare domain. Needed when the account has no stored login for the merchant. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `checkout_id` | string | The run id. Poll it. |
| `status` | string | `queued` on creation. |
| `confirm_pay` | boolean | What was granted. `true` only if you asked for it and the agent's mode allows it. |
| `host` | string | The site the run will open, once known. |
| `poll` | string | The path to poll, `/v1/checkout/{checkout_id}`. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "checkout_id": "run_7a1e...",
  "status": "queued",
  "confirm_pay": false,
  "host": "www.namecheap.com",
  "poll": "/v1/checkout/run_7a1e...",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 400 | `invalid_request` | `task`, `merchant` or `amount` is missing, or `own_card_id` is not a card id. |
| 400 | `connection_not_found` | `connection_id` names no connected connection on the account. |
| 400 | `site_required` | No stored login for the merchant. Pass `start_url`, or connect the merchant once in the console. |
| 400 | `site_ambiguous` | Several stored logins for the merchant. `hosts` lists them; pass `start_url`. |
| 400 | `agent_required` | The account has several agents and none was named. |
| 402 | `plan_limit` | The plan's cards a month, or per-card cap, is reached. |
| 404 | `card_not_found` | No connected card with that `own_card_id`. |
| 404 | `checkout_not_found` | `resume_of` names no checkout of yours. |
| 409 | `not_resumable` | The `resume_of` run left nothing in the basket. |
| 409 | `agent_paused` | The agent is paused in the console. |
| 409 | `busy` | The merchant session is running another checkout. Retry shortly. |
| 503 | `own_card_unavailable` | Paying with an own card is not switched on for the account. |
| 502 | `checkout_start_failed` | The engine could not be started. Retry with the same `idempotency_key`. |

```bash
curl -X POST https://api.joomjoo.com/v1/checkout \
  -H "Authorization: Bearer $AGENTPAY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"task":"buy the domain example.click for 1 year","merchant":"namecheap","amount":20,"confirm_pay":false,"idempotency_key":"order-001"}'
```

### Get checkout status

`GET /v1/checkout/{checkout_id}`

The current state of a run. Poll every five to ten seconds until status is submitted, canceled, blocked or error.

**Path parameters**

| Field | Type | Description |
| --- | --- | --- |
| `checkout_id` | string, required | From the start-checkout response. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `checkout_id` | string |  |
| `status` | string | The run state. One of `queued`, `running`, `needs_login`, `needs_answer`, `awaiting_approval`, `needs_card_approval`, `paying`, `submitted`, `canceled`, `blocked`, `error`. |
| `task` | string |  |
| `merchant` | string |  |
| `amount` | number | The cap in dollars. |
| `card_last4` | string | The card the run pays with, once minted or chosen. |
| `card_source` | string | `issued` or `own`. |
| `host` | string | The site the run opened. |
| `review_total` | string | The shop's total at the review, as a display string such as `"1.98 USD"` or `"60.90 AED"`. |
| `order_id` | string | The merchant's order id, on `submitted`. |
| `order_total` | string | The paid total as a display string, on `submitted`. |
| `needs_login` | boolean | `true` while the status is `needs_login`. |
| `login_check` | string | On `needs_login`: why. One of `signed_out`, `unknown`, `passkey_password`. |
| `login_url` | string | On `needs_login`: the page the browser stopped on. |
| `connect_url` | string | On `needs_login`: the live window where the person signs in. |
| `login_layer` | string | How the run signed in. One of `session`, `passkey`. |
| `needs_card_approval` | boolean | `true` while the run waits for the cardholder. |
| `approval_url` | string | On `needs_card_approval`: the link for the cardholder's phone. |
| `approval_expires_at` | string | On `needs_card_approval`: when the link stops working. |
| `payment_unknown` | boolean | `true` when the run paid but could not read the merchant's confirmation. Check the merchant account before retrying. |
| `error_reason` | string | On `blocked` or `error`: the legacy one-line reason. Prefer `stop`. |
| `stop` | object | Why the run is paused or ended, who acts next and what to do, built from verified facts. Present on `needs_login`, `needs_answer`, `awaiting_approval`, `needs_card_approval`, `blocked`, `canceled` and `error`; `null` while the run works and on `submitted`. See [Why a checkout stops](/guides/complete-a-checkout#why-a-checkout-stops). |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "checkout_id": "run_7a1e...",
  "status": "awaiting_approval",
  "task": "buy the domain example.click for 1 year",
  "merchant": "namecheap",
  "amount": 20,
  "card_last4": "4775",
  "card_source": "issued",
  "host": "www.namecheap.com",
  "review_total": "1.98 USD",
  "order_id": null,
  "order_total": null,
  "needs_login": false,
  "login_check": null,
  "login_url": null,
  "connect_url": null,
  "login_layer": "session",
  "needs_card_approval": false,
  "approval_url": null,
  "approval_expires_at": null,
  "payment_unknown": false,
  "error_reason": null,
  "stop": {
    "code": "approval_required",
    "on": "approver",
    "message": "The order is ready: 1.98 USD at namecheap. Approve it to pay.",
    "next": "Approve or cancel the order in the console.",
    "continue": "same_run",
    "payment_state": "not_attempted",
    "card_closed": false,
    "details": {
      "total": "1.98 USD"
    },
    "transition_id": "st_3f9c..."
  },
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 404 | `checkout_not_found` | No such checkout on your account. |

```bash
curl https://api.joomjoo.com/v1/checkout/run_7a1e... \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

## Connections

### Create a connection

`POST /v1/connections`

Start a reusable login at a merchant. Returns a connect_url, a live window on the merchant's own sign-in page, for the person to complete.

**Body** (application/json)

| Field | Type | Description |
| --- | --- | --- |
| `merchant` | string, required | The merchant to connect a login for. |
| `user_ref` | string | Your own reference for the end user this login belongs to. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `connection_id` | string |  |
| `merchant` | string |  |
| `connect_url` | string | Open it for the person. AgentPay never sees the password. |
| `status` | string | `pending` until you call complete. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "connection_id": "conn_5d2b...",
  "merchant": "amazon",
  "connect_url": "https://connect.browserbase.com/...",
  "status": "pending",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 400 | `invalid_request` | `merchant` is missing. |
| 502 | `connect_start_failed` | The live window could not be opened. Retry. |

```bash
curl -X POST https://api.joomjoo.com/v1/connections \
  -H "Authorization: Bearer $AGENTPAY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"merchant":"amazon","user_ref":"user_42"}'
```

### Complete a connection

`POST /v1/connections/{id}/complete`

Call after the person signed in at the connect_url. The login is saved and the connection becomes connected.

**Path parameters**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string, required | The connection id. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `connection_id` | string |  |
| `status` | string | `connected`. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "connection_id": "conn_5d2b...",
  "status": "connected",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 404 | `connection_not_found` | No such connection on your account. |

```bash
curl -X POST https://api.joomjoo.com/v1/connections/conn_5d2b.../complete \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

### List connections

`GET /v1/connections`

Every connection on the account, newest first, each with its passkey layer.

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `connections` | array |  |
|   `connection_id` | string |  |
|   `merchant` | string |  |
|   `user_ref` | string | Your reference, when given. |
|   `status` | string |  One of `pending`, `connected`, `removed`. |
|   `layer` | string | `passkey` once the AgentPay passkey is enrolled, `session` before. One of `session`, `passkey`. |
|   `passkey_status` | string |  One of `none`, `enrolling`, `awaiting_password`, `pending`, `connected`, `connected_unmanaged`, `failed`, `needs_person`, `login_failed`, `removing`, `removed`, `removed_locally`. |
|   `remote_removal` | string | `available` when AgentPay can delete its entry at the merchant by itself. One of `available`, `unavailable`. |
|   `passkey_connected_at` | string | ISO 8601, when the passkey was enrolled. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "connections": [
    {
      "connection_id": "conn_5d2b...",
      "merchant": "amazon",
      "user_ref": "user_42",
      "status": "connected",
      "layer": "passkey",
      "passkey_status": "connected",
      "remote_removal": "available",
      "passkey_connected_at": "2026-10-24T22:10:00.000Z"
    }
  ],
  "request_id": "req_a1b2c3"
}
```

```bash
curl https://api.joomjoo.com/v1/connections \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

### Get a connection

`GET /v1/connections/{id}`

One connection with its status and passkey layer. Poll it after a disconnect to watch the passkey being removed.

**Path parameters**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string, required | The connection id. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `connection_id` | string |  |
| `merchant` | string |  |
| `status` | string |  One of `pending`, `connected`, `removed`. |
| `layer` | string |  One of `session`, `passkey`. |
| `passkey_status` | string |  One of `none`, `enrolling`, `awaiting_password`, `pending`, `connected`, `connected_unmanaged`, `failed`, `needs_person`, `login_failed`, `removing`, `removed`, `removed_locally`. |
| `remote_removal` | string |  One of `available`, `unavailable`. |
| `passkey_connected_at` | string |  |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "connection_id": "conn_5d2b...",
  "merchant": "amazon",
  "status": "connected",
  "layer": "passkey",
  "passkey_status": "connected",
  "remote_removal": "available",
  "passkey_connected_at": "2026-10-24T22:10:00.000Z",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 404 | `connection_not_found` | No such connection on your account. |

```bash
curl https://api.joomjoo.com/v1/connections/conn_5d2b... \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

### Skip the passkey password

`POST /v1/connections/{id}/passkey`

The person will not type the merchant password for the AgentPay passkey. The connection keeps signing in with the saved session and the wall is never shown again for it.

**Path parameters**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string, required | The connection id. |

**Body** (application/json)

| Field | Type | Description |
| --- | --- | --- |
| `action` | string, required | Only `skip`. One of `skip`. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `connection_id` | string |  |
| `passkey_status` | string | `needs_person` after a skip. Unchanged when the passkey was already past the password step. |
| `changed` | boolean | Whether the call changed anything. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "connection_id": "conn_5d2b...",
  "passkey_status": "needs_person",
  "changed": true,
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 400 | `invalid_request` | `action` is not `skip`. |
| 404 | `connection_not_found` | No such connection on your account. |

```bash
curl -X POST https://api.joomjoo.com/v1/connections/conn_5d2b.../passkey \
  -H "Authorization: Bearer $AGENTPAY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"action":"skip"}'
```

### Disconnect

`DELETE /v1/connections/{id}`

End a connection. The AgentPay key is destroyed first, the saved session is deleted, and the entry at the merchant is removed when AgentPay can. Idempotent.

**Path parameters**

| Field | Type | Description |
| --- | --- | --- |
| `id` | string, required | The connection id. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `connection_id` | string |  |
| `status` | string | `removing` while the passkey is being removed at the merchant, `removed` otherwise. |
| `passkey_status` | string | `removing`, then on a later read `removed` or `removed_locally` (the person removes the `AgentPay-...` entry in the merchant's security settings themselves). |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "connection_id": "conn_5d2b...",
  "status": "removing",
  "passkey_status": "removing",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 404 | `connection_not_found` | No such connection on your account. |
| 502 | `disconnect_failed` | The passkey could not be revoked. Retry. |

```bash
curl -X DELETE https://api.joomjoo.com/v1/connections/conn_5d2b... \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

## undefined

### Read what AgentPay knows about a store

`GET /v1/merchants/{host}`

What earlier checkouts have shown about one store: what it runs on, how it takes payment, whether it asks for an account, and what typically goes wrong there. Read it before a checkout so the task settles what the store will ask.

**Path parameters**

| Field | Type | Description |
| --- | --- | --- |
| `host` | string, required | The store's bare host name. No scheme, no path. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `known` | boolean | `false` when AgentPay has not run on this store. The other fields are then absent, except `host` and `message`. |
| `host` | string | The host, lowercase, without `www.`. |
| `display_name` | string | The store's name. |
| `platform` | string | The shop platform, or `null` when not identified. One of `shopify`, `woocommerce`, `magento`, `bigcommerce`, `wix`, `squarespace`, `salesforce`, `custom`. |
| `pickup_system` | string | The pickup or collection system when one was seen, else `null`. |
| `payment_provider` | string | The payment provider behind the card form, or `null`. One of `stripe`, `adyen`, `checkout_com`, `braintree`, `paypal`, `shopify_payments`, `square`, `other`. |
| `three_ds` | string | Whether a bank challenge (3-D Secure) has appeared here. AgentPay clears it for an issued card. One of `seen`, `unknown`. |
| `login_required` | string | Whether the store asks for an account. One of `none`, `optional`, `required`, `unknown`. |
| `guest_checkout` | string | Whether a checkout without an account has reached the order review. One of `works`, `wall`, `unknown`. |
| `activity` | string | `observed` when enough accounts have run here to describe activity; `limited` otherwise, and `runs`, `last_seen` and `warnings` are then empty. One of `observed`, `limited`. |
| `runs` | string | How much AgentPay has run here, in words. One of `one`, `a few`, `many`. |
| `last_seen` | string | The month of the latest run, `YYYY-MM`. |
| `warnings` | array | What has gone wrong on this store. |
|   `family` | string | The kind of wall. One of `login_wall`, `bot_wall`, `site_broken`, `pickup_mismatch`, `payment_failed`, `no_card_payment`, `choice_needed`. |
|   `text` | string | The warning in words. |
|   `seen` | string | How often. One of `one`, `a few`, `many`. |
| `tips` | array | Short facts about the store, reviewed by AgentPay. |
| `as_of` | string | The day this view was computed. It is refreshed at most once a day. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "known": true,
  "host": "namecheap.com",
  "display_name": "Namecheap",
  "platform": "custom",
  "pickup_system": null,
  "payment_provider": "stripe",
  "three_ds": "seen",
  "login_required": "required",
  "guest_checkout": "wall",
  "activity": "observed",
  "runs": "many",
  "last_seen": "2026-11",
  "warnings": [
    {
      "family": "login_wall",
      "text": "The store asks for an account before checkout.",
      "seen": "many"
    }
  ],
  "tips": [
    "Account balance is offered before the card: the task should say to pay by card."
  ],
  "as_of": "2026-11-09",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 400 | `invalid_host` | The path is not a bare host name. |
| 403 | `not_eligible` | The account has not finished signing up. |
| 429 | `rate_limited` | More than 30 lookups in an hour or 200 in a day on the account. `retry_after_s` says when to try again. |
| 502 | `merchant_read_failed` | The store could not be read. Retry. |

```bash
curl https://api.joomjoo.com/v1/merchants/namecheap.com \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

## Wallet

### Read the wallet

`GET /v1/wallet`

The balance every card and checkout draws on, what is free to assign, what sits on open cards, and the cards that already hold money.

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `currency` | string | `USD`. |
| `onboarded` | boolean | Whether the account finished onboarding. |
| `balance` | number | Everything in the wallet, in dollars. |
| `available` | number | `balance` minus `reserved`. What a new card can be capped at. |
| `reserved` | number | Money sitting on open cards. |
| `pending` | number | Declared bank money that has not landed. Not spendable. |
| `balance_display` | string | The same four figures as display strings, e.g. `$32.96`. Also `available_display`, `reserved_display`, `pending_display`. |
| `cards_ready` | array | Open single-use cards that were never charged. A checkout whose total fits one pays with it, with no new card. |
|   `last4` | string |  |
|   `network` | string |  |
|   `holds` | number | The card's limit in dollars. |
| `ready_to_spend` | number | `available` plus what `cards_ready` hold. The honest answer to "can the agent buy this". |
| `charges_in_flight` | array | Charges authorized and still settling. The card holding each stays reserved until the network clears it. |
|   `amount` | number |  |
|   `card_last4` | string |  |
|   `network` | string |  |
|   `merchant` | string |  |
|   `status` | string | `authorized, settling`. |
| `after_settlement` | number | What `balance` becomes once the charges in flight clear. |
| `note` | string | A plain-words summary of the above, written for an agent. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "currency": "USD",
  "onboarded": true,
  "balance": 32.96,
  "available": 22.96,
  "reserved": 10,
  "pending": 0,
  "balance_display": "$32.96",
  "available_display": "$22.96",
  "reserved_display": "$10.00",
  "pending_display": "$0.00",
  "cards_ready": [
    {
      "last4": "8339",
      "network": "visa",
      "holds": 10
    }
  ],
  "ready_to_spend": 32.96,
  "charges_in_flight": [],
  "after_settlement": 32.96,
  "note": "A checkout whose total fits one of the 1 card(s) in cards_ready pays with it, no new card and no top-up needed. Call complete_purchase.",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 502 | `wallet_read_failed` | The wallet could not be read. Retry. |

```bash
curl https://api.joomjoo.com/v1/wallet \
  -H "Authorization: Bearer $AGENTPAY_API_KEY"
```

### Top up by card or stablecoin

`POST /v1/wallet/funding-sessions`

Open a Stripe Checkout page for the amount. Give url to the person; the wallet is credited the moment Stripe confirms. A card or stablecoin top-up carries Stripe's own fee, at cost, added on top: the person pays `total` and the wallet gains `amount`. A bank transfer has no fee. A card number never passes through the API.

**Body** (application/json)

| Field | Type | Description |
| --- | --- | --- |
| `amount` | number, required | Dollars to add to the wallet, above zero. The fee is added on top of it. |
| `method` | string | `card` (the default) or `stablecoin` (USDC). One Stripe page serves one method. |
| `success_url` | string | Where the person lands after paying. Must be https. Defaults to the AgentPay console. |
| `cancel_url` | string | Where the person lands if they cancel. Must be https. Defaults to the AgentPay console. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `funding_session_id` | string | The Stripe Checkout session id. |
| `url` | string | The page to open for the person. |
| `amount` | number | What the wallet gains. |
| `amount_display` | string |  |
| `method` | string | `card` or `stablecoin`. |
| `fee` | number | Stripe's fee for this top-up, in dollars, added on top of `amount`. A card costs 2.9% + $0.35 for a US business and 4.4% + $0.35 for any other; a stablecoin payment costs 1.5%. |
| `total` | number | What the person pays on the Stripe page: `amount` plus `fee`. |
| `status` | string | `open`. |
| `pays_with` | string | `a card on the Stripe page` or `USDC on the Stripe page`. |
| `credited` | string | When the wallet is credited, in plain words. |
| `fee_note` | string | The fee in plain words, written for an agent to repeat to the person. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "funding_session_id": "cs_live_...",
  "url": "https://checkout.stripe.com/c/pay/...",
  "amount": 50,
  "amount_display": "$50.00",
  "method": "card",
  "fee": 2.55,
  "total": 52.55,
  "status": "open",
  "pays_with": "a card on the Stripe page",
  "credited": "the wallet is credited `amount` the moment Stripe confirms; subscribe to money.funded to know",
  "fee_note": "The person pays `total`: `amount` goes to the wallet and `fee` is Stripe's own fee, passed on at cost. If Stripe charges less than `fee`, the difference is refunded to the payment method automatically. A bank transfer (POST /v1/wallet/bank-transfers) has no fee.",
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 400 | `invalid_amount` | `amount` is missing or not above zero, or the amount was refused as malformed. |
| 400 | `invalid_request` | `success_url` or `cancel_url` is not an https address, or `method` is not `card` or `stablecoin`. |
| 402 | `wallet_limit` | The top-up would take the wallet above its ceiling. |
| 402 | `funding_budget_reached` | The account's funding budget for the period is reached. |
| 402 | `funding_unavailable` | Top-ups are paused for the moment. |
| 502 | `funding_session_failed` | The Stripe page could not be opened. Retry. |

```bash
curl -X POST https://api.joomjoo.com/v1/wallet/funding-sessions \
  -H "Authorization: Bearer $AGENTPAY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"amount":50,"success_url":"https://yourapp.example/funded"}'
```

### Top up by bank transfer

`POST /v1/wallet/bank-transfers`

Declare a transfer and get the account to send it to, with a reference code for the memo. The money is pending until it lands.

**Body** (application/json)

| Field | Type | Description |
| --- | --- | --- |
| `amount` | number, required | The amount in the transfer currency, above zero. |
| `rail` | string, default `ach` | Which account to send to. One of `ach`, `wire_domestic`, `wire_international`, `uae_local`. |
| `currency` | string | `USD`, or `AED` for `uae_local`. Defaults by rail. |
| `sent_on` | string | `yyyy-MM-dd`, the day the transfer was sent. |
| `sender_bank_name` | string | What the statement will show, so the transfer is matched faster. |
| `sender_account_name` | string |  |
| `sender_reference` | string |  |
| `idempotency_key` | string | Your own key, up to 80 characters. A retry returns the same declaration. |

**Response** 200

| Field | Type | Description |
| --- | --- | --- |
| `funding_event_id` | string |  |
| `status` | string | `pending` until the money lands. |
| `already_recorded` | boolean | `true` when the `idempotency_key` matched an earlier declaration. |
| `amount` | number |  |
| `currency` | string |  |
| `rail` | string |  |
| `reference_code` | string | Eight characters. Goes in the transfer memo. |
| `pending_balance` | number | The wallet's pending after this declaration. |
| `instructions` | object | The account to send to. Show it to the person as it is. Fields depend on the rail: `beneficiary_name`, `account_number`, `routing_number` or `swift_bic`, `iban`, `bank_name`, `bank_address`, `beneficiary_address`, `reference`, `note`. |
| `request_id` | string | The request id, also in the `X-Mooj-Request-Id` header. |

```json
{
  "funding_event_id": "fe_...",
  "status": "pending",
  "already_recorded": false,
  "amount": 500,
  "currency": "USD",
  "rail": "wire_domestic",
  "reference_code": "7K2QW9PD",
  "pending_balance": 500,
  "instructions": {
    "beneficiary_name": "Joomjoo, LLC",
    "account_number": "...",
    "routing_number": "...",
    "bank_name": "Column N.A.",
    "account_type": "Checking",
    "bank_address": "...",
    "beneficiary_address": "...",
    "reference": "7K2QW9PD",
    "note": "Put the reference code in the transfer memo. The wallet is credited when the money lands; subscribe to money.funded to know."
  },
  "request_id": "req_a1b2c3"
}
```

**Errors**

| Status | Code | When |
| --- | --- | --- |
| 400 | `invalid_amount` | `amount` is missing or not above zero. |
| 400 | `invalid_request` | `rail` is not one of the four, or the declaration was refused as malformed. |
| 402 | `wallet_limit` | The transfer would take the wallet above its ceiling. |
| 402 | `funding_budget_reached` | The account's funding budget for the period is reached. |
| 402 | `funding_unavailable` | Top-ups are paused for the moment. |
| 502 | `bank_transfer_failed` | The declaration could not be recorded. Retry. |

```bash
curl -X POST https://api.joomjoo.com/v1/wallet/bank-transfers \
  -H "Authorization: Bearer $AGENTPAY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"amount":500,"rail":"wire_domestic","sent_on":"2026-11-03","idempotency_key":"topup-2026-11-03"}'
```

## Webhooks

Register an endpoint in the console under Developer, Webhooks. Each delivery is `{ "id", "type", "created", "data" }` with a `Joomjoo-Signature: t=<unix>,v1=<hex>` header (HMAC-SHA256 of `t + "." + rawBody` with the endpoint's signing secret; the same value is also sent as `Mooj-Signature`).

```json
{
  "id": "evt_card_cleared_8f2a...",
  "type": "card.cleared",
  "created": 1762182000,
  "data": { "card_id": "card_8f2a...", "card_last4": "8339", "amount": 1.98, "fees": 0, "merchant": "NAMECHEAP", "network": "visa", "balance_after": 30.98, "provider_transaction_id": "..." }
}
```

### Cards

#### card.authorized
The merchant charged an issued card. The money is held; `settled` on the spend is still `false`.

| Field | Type | |
| --- | --- | --- |
| `card_id` | string | The card. |
| `card_last4` | string | |
| `amount` | number | Dollars. |
| `merchant` | string | The merchant name as the network reports it. |
| `network` | string | `visa` or `mastercard`. |
| `provider_transaction_id` | string | The network's transaction id. |

#### card.cleared
The network settled a charge, usually by the next day. The wallet balance moved.

| Field | Type | |
| --- | --- | --- |
| `card_id`, `card_last4`, `merchant`, `network`, `provider_transaction_id` | | As above. |
| `amount` | number | The settled amount in dollars. |
| `fees` | number | Network fees in dollars, when any. |
| `balance_after` | number | The wallet balance after the charge. |

#### card.declined
A charge on an issued card was refused.

| Field | Type | |
| --- | --- | --- |
| `card_id`, `card_last4`, `amount`, `merchant`, `network`, `provider_transaction_id` | | As above. |
| `decline_reason` | string | The reason. A second charge on a used single-use card is the guard working. |

### Checkouts

Every checkout event's `data` carries `checkout_id`, `status` and, on every pause or ending, the same `stop` object the checkout read returns (`code`, `on`, `message`, `next`, `continue`, `payment_state`, `card_closed`, `details`, `transition_id`; see [Why a checkout stops](https://agentpay.global/docs/guides/complete-a-checkout/#why-a-checkout-stops)). The event `id` is derived from `checkout_id` and `stop.transition_id`, so a redelivery of the same pause or ending has the same id and a second pause of the same run is a new event. `payment_unknown` rides along as a boolean.

#### checkout.submitted
The run paid. `order_id` and `order_total` are on the checkout read.

#### checkout.needs_login
The run stopped at a wall it needs a person for: a sign-in page, or a code the merchant sent to the phone or inbox in the form. `stop.code` tells which (`merchant_sign_in_required`, `merchant_code_required`, `sign_in_check_inconclusive`, `passkey_setup_required`).

| Field | Type | |
| --- | --- | --- |
| `login_url` | string | The page the browser stopped on. Read `login_check` and `connect_url` from `GET /v1/checkout/{id}`. |
| `stop` | object | Why, who acts, what to do. |

#### checkout.needs_card_approval
A run paying with the person's own card reached the review and waits for the cardholder.

| Field | Type | |
| --- | --- | --- |
| `approval_url` | string | Send it to the cardholder. |
| `approval_expires_at` | string | When the link stops working. |
| `amount_display` | string | The total, e.g. `"60.90 AED"`. |
| `card_last4` | string | |

#### checkout.blocked
AgentPay stopped the run without paying, or with the payment in doubt.

| Field | Type | |
| --- | --- | --- |
| `reason` | string | The legacy reason, unchanged: `approval_timeout`, `engine_error`, `max_iterations`, `payment_unknown`, `site_busy`, `unreachable`, `no_context`, or the wall the run met. Prefer `stop.code`. |
| `stop` | object | Why, who acts, what to do, and `payment_state`. When `payment_state` is `attempted` or `uncertain`, check the merchant before starting another checkout. |
| `status` | string | `blocked` or `error`. |
| `card_last4` | string | Only on `payment_unknown` with a connected card. |

#### checkout.canceled
The person stopped the run, or it continued as a new checkout after a sign-in. `stop.code` is `canceled_by_you` or `continued_as_new_run`; a declined own-card approval also ends here.

| Field | Type | |
| --- | --- | --- |
| `reason` | string | `canceled`, or the approval outcome. |
| `stop` | object | As above. |

### Money

#### money.funded
A top-up landed in the wallet.

| Field | Type | |
| --- | --- | --- |
| `amount` | number | Dollars. |
| `currency` | string | `USD`. |
| `method` | string | `card`, `stablecoin`, `bank` or `gift`. |
| `rail` | string | On a bank transfer: `ach`, `wire_domestic`, `wire_international`, `uae_local`. |
| `reference_code` | string | On a bank transfer, the memo code it was matched by. |
| `funding_event_id` | string | |
| `provider_ref` | string | Stripe's reference on a card or stablecoin top-up. |
| `balance_after` | number | The wallet balance after the credit. |

### The header

```text
Joomjoo-Signature: t=1762182000,v1=5f1d2c...
```

`t` is the unix time of the delivery in seconds. `v1` is `HMAC_SHA256(secret, t + "." + rawBody)` as lowercase hex. The same value is also sent as `Mooj-Signature`.

### Verify

```javascript Node
import crypto from "node:crypto";

export function verify(rawBody, header, secret) {
  const parts = Object.fromEntries(header.split(",").map((p) => p.split("=")));
  const expected = crypto.createHmac("sha256", secret).update(parts.t + "." + rawBody).digest("hex");
  return expected.length === parts.v1.length &&
    crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(parts.v1));
}
```
```python Python
import hmac, hashlib

def verify(raw_body: bytes, header: str, secret: str) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    msg = parts["t"].encode() + b"." + raw_body
    expected = hmac.new(secret.encode(), msg, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, parts["v1"])
```
- Compute over the raw bytes you received. Parsing and re-serialising the JSON changes the string and breaks the signature.
- Compare in constant time.
- Reject a `t` more than a few minutes old to shut out replays.
- Answer 2xx only after the signature verified; anything else is retried.

### Other headers

| Header | Value |
| --- | --- |
| `Joomjoo-Event-Type` | The event type, the same as `type` in the body. |
| `Joomjoo-Event-Id` | The event id, the same as `id` in the body. Use it to deduplicate retries. |
| `Mooj-Signature`, `Mooj-Event-Type`, `Mooj-Event-Id` | The same three values under the older names. |

## Errors

```json
{
  "error": "insufficient_funds",
  "message": "Your wallet available is below this amount. Fund your wallet, then retry.",
  "request_id": "req_a1b2c3"
}
```

### 400 Bad request

| Code | When |
| --- | --- |
| `invalid_request` | A required field is missing or malformed. The message names it. |
| `invalid_amount` | `amount` is missing, not a number, or not above zero. |
| `connection_not_found` | `connection_id` on a checkout names no connected connection on the account. |
| `site_required` | The account has no stored login for that merchant. Pass `start_url`, or connect the merchant once in the console. |
| `site_ambiguous` | The account has several logins for that merchant. `hosts` lists them; pass `start_url`. |
| `agent_required` | The account has several agents and none was named. Name one in the console. |

### 401 Unauthorized

| Code | When |
| --- | --- |
| `missing_api_key` | No `Authorization: Bearer` header. |
| `invalid_api_key` | The key is unknown or revoked. |
| `expired_api_key` | The key passed its expiry. `expired_at` says when. |

### 402 Payment required

| Code | When |
| --- | --- |
| `insufficient_funds` | The requested cap exceeds the wallet's `available`. |
| `plan_limit` | The plan's cards a month, or per-card cap, is reached. `limit` carries the figure. |
| `wallet_limit` | A top-up would take the wallet above its ceiling. |
| `funding_budget_reached` | The account's funding budget for the period is reached. |
| `funding_unavailable` | Top-ups are paused for the moment. Try later. |
| `funding_refused` | A fence refused the top-up for another reason; `code` carries it. |

### 404 Not found

| Code | When |
| --- | --- |
| `card_not_found` | No such card on the account. On `DELETE /v1/cards/{id}`: no such connected card (an issued card is closed, not removed). On a checkout with `own_card_id`: no such connected card. |
| `spend_not_found` | No such spend on the account. |
| `checkout_not_found` | No such checkout on the account, including a `resume_of` that is not yours. |
| `connection_not_found` | No such connection on the account. |
| `account_not_found` | The key's account could not be loaded. |
| `not_found` | No route for the method and path. |

### 409 Conflict

| Code | When |
| --- | --- |
| `agent_paused` | The person paused the agent in the console. Resume it there. |
| `busy` | The merchant session is running another checkout. Retry shortly with the same `idempotency_key`. |
| `not_resumable` | The `resume_of` run left nothing in the basket to continue from. Start again. |

### 500 and 502 Upstream

| Code | When |
| --- | --- |
| `server_error` | An unexpected failure. Quote the `request_id`. |
| `checkout_create_failed` | The run document could not be created. |
| `card_issue_failed`, `reveal_failed`, `close_failed`, `remove_failed`, `connect_failed` | A card step failed upstream; the message says which. |
| `spend_status_failed`, `wallet_read_failed` | A read failed upstream. |
| `checkout_start_failed`, `enqueue_failed` | The engine could not be started. On `enqueue_failed` the body carries the `checkout_id` that was created and marked `error`. |
| `connect_start_failed`, `disconnect_failed` | A connection step failed. Retry. |
| `funding_session_failed`, `bank_transfer_failed` | The top-up could not be opened or recorded. |

### 503 Unavailable

| Code | When |
| --- | --- |
| `own_card_unavailable` | Connecting a person's own card is not switched on for the account. |
| `unavailable` | The engine refused the run for the moment. Try again shortly. |

## SDKs and agent files

- Node: `npm install @agentpay-global/sdk`
- Python: `pip install agentpay-global` (import `agentpay`)
- MCP: `npx -y @agentpay-global/mcp` with `AGENTPAY_API_KEY`
- Skill, AGENTS.md and tool files: `npx skills add Kato-Official/agentpay-skills`
- Postman: https://agentpay.global/docs/agentpay.postman_collection.json
