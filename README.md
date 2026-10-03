# SigWise API SDK for Python

Send events about your objects, get typed answers back.

Covers version 1.0.0 of the API. Full documentation, guides and the API reference:
<https://sigwise.ai/docs>.

## Install

```bash
pip install sigwise-sdk
```

## Quick start

```python
from sigwise import SigWise

# Reads SIGWISE_API_KEY and SIGWISE_SECRET when called without arguments.
sigwise = SigWise(api_key="your_key_id", secret="your_secret")

# Configure what you want to know about your objects.
sigwise.signals.upsert(
    "is_scammer",
    type="noul",
    instructions="Decide if this user is likely a scammer.",
    criteria={"true": "clear scam signals", "false": "legitimate behaviour"},
)

# Send events. Analysis runs in the background.
sigwise.events.ingest(
    "user-42",
    object_type="user",
    events=[{"type": "message", "content": "is this still available? can I pay by wire?"}],
)

# Read the latest answers.
obj = sigwise.objects.get("user-42")
print(obj["analysis"])

# Or score inline and gate the content before publishing it.
verdict = sigwise.events.ingest(
    "user-42",
    wait=True,
    signals=["is_scammer"],
    events=[{"type": "message", "content": "pay me by wire and I double it"}],
)
if verdict.get("answers") and verdict["answers"][0].get("noul", 0) > 0.9:
    ...  # block it
```

## Authentication

Create an API key in the console. It is a pair: a public key ID and a
signing secret (`your_secret`, shown once). The client sends the key ID with every
request and signs a short-lived HS256 token with the secret, bound to the
request's method and path. The secret itself is never sent, so keep it on your
server. Without explicit options the client reads `SIGWISE_API_KEY`,
`SIGWISE_SECRET` and `SIGWISE_BASE_URL` from the environment.

## Configuration

```python
sigwise = SigWise(
    api_key="your_key_id",
    secret="your_secret",
    base_url="http://localhost:8080",  # default: SIGWISE_BASE_URL or the production API
    timeout=10.0,                      # seconds, per attempt
    max_retries=3,                     # idempotent requests only
)
```

Idempotent requests (`GET`, `PUT`, `DELETE`) are retried with exponential
backoff after a network error, a `429` or a `5xx`. Other requests are never
retried, so an event is never ingested twice.

## Errors

An error response raises an error carrying the HTTP status, the stable
machine-readable `code` (`not_found`, `payment_required`, …) and the message.

```python
from sigwise import SigWiseConnectionError, SigWiseError

try:
    sigwise.objects.get("user-42")
except SigWiseError as err:
    print(err.status, err.code, err.message)  # 404 not_found no data for this object
except SigWiseConnectionError:
    ...  # network error or timeout
```

## Webhooks

Verify every delivery before trusting it. The helper checks the
`X-Webhook-Signature` HMAC against the raw body and rejects timestamps older
than five minutes.

```python
import os
from sigwise import construct_webhook_event

# Pass the raw request body, exactly as received.
event = construct_webhook_event(
    request.body,
    request.headers.get("X-Webhook-Signature"),
    request.headers.get("X-Webhook-Timestamp"),
    os.environ["SIGWISE_WEBHOOK_SECRET"],
)
if event["event"] == "analysis.completed":
    print(event["object_id"], event["answers"])
```

## Reference

### me

The authenticated principal and tenant settings.

- `sigwise.me.get(*, timeout: Optional[float] = None)`  
  `GET /v1/me`: Get the current principal

### overview

An object is anything you want answers about: a user, a listing, an order.

- `sigwise.overview.get(*, timeout: Optional[float] = None)`  
  `GET /v1/overview`: Get tenant overview

### objects

An object is anything you want answers about: a user, a listing, an order.

- `sigwise.objects.analyze_all(*, timeout: Optional[float] = None)`  
  `POST /v1/analyze`: Re-analyze every object
- `sigwise.objects.list(*, q: Optional[str] = None, sort: Optional[Literal["recent", "flagged", "trust"]] = None, limit: Optional[int] = None, offset: Optional[int] = None, cursor: Optional[str] = None, timeout: Optional[float] = None)`  
  `GET /v1/objects`: List objects
- `sigwise.objects.get(object_id: str, *, timeout: Optional[float] = None)`  
  `GET /v1/objects/{object_id}`: Get an object's analysis
- `sigwise.objects.get_state(object_id: str, *, timeout: Optional[float] = None)`  
  `GET /v1/objects/{object_id}/state`: Get an object's compacted history
- `sigwise.objects.analyze(object_id: str, *, timeout: Optional[float] = None)`  
  `POST /v1/objects/{object_id}/analyze`: Re-analyze an object

### playground

Events and messages are the evidence an object's answers are computed from.

- `sigwise.playground.run(*, events: List[EventInput], object_type: Optional[str] = None, signals: Optional[List[str]] = None, timeout: Optional[float] = None)`  
  `POST /v1/playground`: Try signals on sample events

### events

Events and messages are the evidence an object's answers are computed from.

- `sigwise.events.ingest(object_id: str, *, events: List[EventInput], object_type: Optional[str] = None, wait: Optional[bool] = None, signals: Optional[List[str]] = None, include_history: Optional[bool] = None, timeout: Optional[float] = None)`  
  `POST /v1/objects/{object_id}/events`: Ingest events
- `sigwise.events.list(object_id: str, *, timeout: Optional[float] = None)`  
  `GET /v1/objects/{object_id}/events`: List an object's events

### signals

A signal is a question you ask about every object, such as "is this a scammer?" (`noul`), "how trustworthy is this user?" (`score`) or "what is their buyer intent?" (`choice`).

- `sigwise.signals.list(*, timeout: Optional[float] = None)`  
  `GET /v1/signals`: List signals
- `sigwise.signals.upsert(key: str, *, type: SignalType, instructions: Any, criteria: Any = None, enabled: Optional[bool] = None, timeout: Optional[float] = None)`  
  `PUT /v1/signals/{key}`: Create or update a signal
- `sigwise.signals.delete(key: str, *, timeout: Optional[float] = None)`  
  `DELETE /v1/signals/{key}`: Delete a signal
- `sigwise.signals.get_backfill(key: str, *, timeout: Optional[float] = None)`  
  `GET /v1/signals/{key}/backfill`: Get backfill estimate and progress
- `sigwise.signals.start_backfill(key: str, *, timeout: Optional[float] = None)`  
  `POST /v1/signals/{key}/backfill`: Start a backfill

### settings

The authenticated principal and tenant settings.

- `sigwise.settings.get(*, timeout: Optional[float] = None)`  
  `GET /v1/settings`: Get tenant settings
- `sigwise.settings.update(*, auto_backfill_signals: Optional[bool] = None, timeout: Optional[float] = None)`  
  `PATCH /v1/settings`: Update tenant settings

### webhooks

Webhook endpoints receive a signed `POST` for the event types they subscribe to: `analysis.completed` each time an analysis completes, and `rule.triggered` when a rule with a webhook action fires.

- `sigwise.webhooks.list(*, timeout: Optional[float] = None)`  
  `GET /v1/webhooks`: List webhook endpoints
- `sigwise.webhooks.create(*, url: str, description: Optional[str] = None, signal_keys: Optional[List[str]] = None, events: Optional[List[WebhookEventType]] = None, enabled: Optional[bool] = None, timeout: Optional[float] = None)`  
  `POST /v1/webhooks`: Register a webhook endpoint
- `sigwise.webhooks.get(endpoint_id: str, *, timeout: Optional[float] = None)`  
  `GET /v1/webhooks/{endpoint_id}`: Get a webhook endpoint
- `sigwise.webhooks.update(endpoint_id: str, *, url: Optional[str] = None, description: Optional[str] = None, signal_keys: Optional[List[str]] = None, events: Optional[List[WebhookEventType]] = None, enabled: Optional[bool] = None, timeout: Optional[float] = None)`  
  `PATCH /v1/webhooks/{endpoint_id}`: Update a webhook endpoint
- `sigwise.webhooks.delete(endpoint_id: str, *, timeout: Optional[float] = None)`  
  `DELETE /v1/webhooks/{endpoint_id}`: Delete a webhook endpoint

### webhookDeliveries

Webhook endpoints receive a signed `POST` for the event types they subscribe to: `analysis.completed` each time an analysis completes, and `rule.triggered` when a rule with a webhook action fires.

- `sigwise.webhook_deliveries.list(*, status: Optional[DeliveryStatus] = None, endpoint_id: Optional[str] = None, limit: Optional[int] = None, timeout: Optional[float] = None)`  
  `GET /v1/webhook_deliveries`: List webhook deliveries
- `sigwise.webhook_deliveries.replay(delivery_id: str, *, timeout: Optional[float] = None)`  
  `POST /v1/webhook_deliveries/{delivery_id}/replay`: Replay a delivery

### rules

A rule fires an action (email, Slack or webhook) when an object's signal values cross a line you care about, such as `is_scammer >= 90`.

- `sigwise.rules.list(*, timeout: Optional[float] = None)`  
  `GET /v1/rules`: List rules
- `sigwise.rules.create(*, when: str, action_type: ActionType, action_config: ActionConfig, name: Optional[str] = None, enabled: Optional[bool] = None, cooldown_seconds: Optional[int] = None, timeout: Optional[float] = None)`  
  `POST /v1/rules`: Create a rule
- `sigwise.rules.get(rule_id: str, *, timeout: Optional[float] = None)`  
  `GET /v1/rules/{rule_id}`: Get a rule
- `sigwise.rules.update(rule_id: str, *, name: Optional[str] = None, when: Optional[str] = None, action_type: Optional[ActionType] = None, action_config: Optional[ActionConfig] = None, enabled: Optional[bool] = None, cooldown_seconds: Optional[int] = None, timeout: Optional[float] = None)`  
  `PATCH /v1/rules/{rule_id}`: Update a rule
- `sigwise.rules.delete(rule_id: str, *, timeout: Optional[float] = None)`  
  `DELETE /v1/rules/{rule_id}`: Delete a rule

### ruleFirings

A rule fires an action (email, Slack or webhook) when an object's signal values cross a line you care about, such as `is_scammer >= 90`.

- `sigwise.rule_firings.list(*, rule_id: Optional[str] = None, object_id: Optional[str] = None, limit: Optional[int] = None, timeout: Optional[float] = None)`  
  `GET /v1/rule_firings`: List rule firings

### apiKeys

Create, rotate and revoke the keys your integrations sign requests with.

- `sigwise.api_keys.list(*, timeout: Optional[float] = None)`  
  `GET /v1/api_keys`: List API keys
- `sigwise.api_keys.create(*, name: Optional[str] = None, timeout: Optional[float] = None)`  
  `POST /v1/api_keys`: Create an API key
- `sigwise.api_keys.update(key_id: str, *, name: Optional[str] = None, status: Optional[Literal["active", "inactive"]] = None, timeout: Optional[float] = None)`  
  `PATCH /v1/api_keys/{key_id}`: Update an API key
- `sigwise.api_keys.revoke(key_id: str, *, timeout: Optional[float] = None)`  
  `DELETE /v1/api_keys/{key_id}`: Revoke an API key
- `sigwise.api_keys.rotate(key_id: str, *, timeout: Optional[float] = None)`  
  `POST /v1/api_keys/{key_id}/rotate`: Rotate an API key's secret

### billing

Prepaid balance, the billing ledger, and card top-ups.

- `sigwise.billing.get_balance(*, timeout: Optional[float] = None)`  
  `GET /v1/billing`: Get the prepaid balance
- `sigwise.billing.list_ledger(*, type: Optional[Literal["credit", "charge"]] = None, limit: Optional[int] = None, group: Optional[Literal["batch"]] = None, batch_id: Optional[str] = None, cursor: Optional[str] = None, timeout: Optional[float] = None)`  
  `GET /v1/billing/ledger`: List ledger entries
- `sigwise.billing.create_checkout(*, amount_cents: int, timeout: Optional[float] = None)`  
  `POST /v1/billing/checkout`: Start a card top-up
- `sigwise.billing.get_checkout(session_id: str, *, timeout: Optional[float] = None)`  
  `GET /v1/billing/checkout/{session_id}`: Get a top-up's status

### usage

Analysis volume, spend and token usage over a date range.

- `sigwise.usage.get(*, from_: Optional[str] = None, to: Optional[str] = None, timeout: Optional[float] = None)`  
  `GET /v1/usage`: Get usage

