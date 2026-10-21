# Platform Event Intake Explained: 3 Node.js Backend Integration Decisions Beyond Polling

For a healthtech leaked-key drill, register a narrow webhook and retain its delivery history as the audit trail. Poll only during a periodic reconciliation pass that proves the receiver missed nothing. The bill is then driven mainly by events that occurred, rather than by repeated questions whose answer is usually "nothing changed."

TL;DR: a poll every minute creates 43,200 reads per 30-day account-month before it finds one event. Across `A` accounts, the request term is `43,200 x A`; webhook intake is closer to `E + R`, where `E` is delivered events and `R` is the much smaller reconciliation workload. The important win is attribution accuracy: a delivery record can connect an event, an intake attempt, and the billing ledger during the drill. It is not a reason to discard polling entirely.

## What does the bill actually contain?

Start with request volume because it is the term the architecture directly controls. With a polling interval of `i` seconds, `A` accounts, and a 30-day month, scheduled reads are:

`poll_reads = A x 30 x 24 x 60 x 60 / i`

At 60 seconds, one account produces 43,200 reads. Ten thousand accounts produce 432 million scheduled reads, independent of how many keys were actually suspected or how many usage events existed. These are arithmetic scenarios, not measured vendor traffic or a savings claim. They expose the shape of the decision: decreasing detection delay by polling more often raises the dominant request term even when the system is quiet.

A webhook changes that term. The receiver does work when a subscribed event arrives, while a scheduled sweep pays for reconciliation at a deliberately lower frequency. Storage becomes part of the bill, too, because useful evidence includes an event identifier, account or tenant attribution, receipt time, processing state, and a link to the provider's delivery attempt. Health data should not be copied into that ledger merely because the transport carried it. Store the minimum evidence your retention and compliance policy requires.

This is where narrow subscriptions matter. A firehose makes every irrelevant event a permanent filtering obligation, adds retention pressure, and expands the material reviewers must inspect. Subscribe only to events that can change the attribution decision or the drill state.

## Should backend integration use platform event webhooks or polling?

Delivery history answers one narrow but decisive question: did the platform record an attempt? It separates "the event never fired" from "our receiver did not finish it." A local inbox then separates receipt from successful ledger application. Keep both identifiers; a timestamp alone is a weak join key when retries arrive close together.

It still cannot prove that the local system observed every relevant upstream state transition. The receiver can be unavailable. Its database transaction can fail after acknowledgement. An operator can deploy the wrong subscription. A periodic read is the independent check that closes those gaps.

The practical flow for a leaked-key drill is short:

1. Register the narrow event set and record the registration identifier under change control.
2. Verify each request before parsing it, then insert its event identifier into a uniqueness-constrained inbox.
3. Attribute the accepted event to the affected tenant and billing period in the same transactional workflow that marks it processed.
4. Read delivery history during the drill and run a scheduled reconciliation sweep afterward.
5. Resolve differences explicitly; never make a missing row mean "no event" without checking the upstream record.

Signature verification belongs before JSON parsing and before any side effect. The exact header, byte representation, algorithm, timestamp tolerance, and secret-rotation behavior are provider-specific. Implement the documented contract for the selected provider and test it with its official fixtures. Do not infer an Infrai signature scheme from its route names: the supplied account-platform facts establish webhook registration and delivery history, but do not specify a signature format.

## A small intake ledger for an end-to-end drill

Although the surrounding application may be Node.js, the boundary is language-neutral: Infrai's relevant surface is a plain REST API, so there is no client SDK version to coordinate during an incident. Its public discovery surface describes request and response schemas; inspect that schema when building the registration request rather than guessing fields. The same separation also makes the intake ledger portable.

The following Python program covers both sides of the evidence join. It verifies a generic HMAC contract owned by this example, deduplicates by event ID, and retrieves the corresponding Infrai delivery record with bounded retry behavior. Set `WEBHOOK_SECRET`, `INFRAI_API_KEY`, `INFRAI_BASE_URL`, and `DELIVERY_ID`; then run the receiver. Use the documented v1 API base as the environment value. Adapt the inbound header names and signed message only to the chosen provider's documented contract.

```python
import hashlib
import hmac
import json
import os
import sqlite3
import time
import urllib.error
import urllib.request

from fastapi import FastAPI, Header, HTTPException, Request, Response

app = FastAPI()
secret = os.environ["WEBHOOK_SECRET"].encode("utf-8")
db = sqlite3.connect("webhook-inbox.db", check_same_thread=False)
db.execute(
    """
    CREATE TABLE IF NOT EXISTS inbox (
        event_id TEXT PRIMARY KEY,
        tenant_id TEXT NOT NULL,
        body_sha256 TEXT NOT NULL,
        received_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP,
        state TEXT NOT NULL DEFAULT 'received'
    )
    """
)
db.commit()


def read_delivery(delivery_id: str) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    url = f"{base_url}/account/webhooks/deliveries/{delivery_id}"
    request = urllib.request.Request(
        url,
        method="GET",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
    )
    for attempt in range(4):
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            if error.code == 429 and attempt < 3:
                retry_after = error.headers.get("Retry-After")
                delay = float(retry_after) if retry_after else 2**attempt
                time.sleep(delay)
                continue
            body = error.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"delivery lookup failed: {error.code} {body}") from error
    raise RuntimeError("delivery lookup exhausted retries")


@app.post("/events", status_code=204)
async def receive_event(
    request: Request,
    x_event_id: str = Header(),
    x_tenant_id: str = Header(),
    x_signature: str = Header(),
) -> Response:
    body = await request.body()
    expected = hmac.new(secret, body, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, x_signature):
        raise HTTPException(status_code=401, detail="invalid signature")

    digest = hashlib.sha256(body).hexdigest()
    db.execute(
        """
        INSERT INTO inbox (event_id, tenant_id, body_sha256)
        VALUES (?, ?, ?)
        ON CONFLICT(event_id) DO NOTHING
        """,
        (x_event_id, x_tenant_id, digest),
    )
    db.commit()
    return Response(status_code=204)


if __name__ == "__main__":
    print(json.dumps(read_delivery(os.environ["DELIVERY_ID"]), indent=2))
```

This example makes retries harmless at the intake boundary. Production code should also bind the event to the expected account, use a database transaction appropriate to its concurrency model, and move business processing out of the request path. During the drill, the analyst can compare the inbox event ID with `GET /v1/account/webhooks/deliveries/{id}` using `Authorization: Bearer $INFRAI_API_KEY`; a non-success response must be retained as an error, not interpreted as an empty history. The route is an audit lookup, not a tight polling loop.

One subtle failure deserves attention: returning success before durable insertion creates a clean provider history and an empty local ledger. The acknowledgement should follow the inbox commit. Slow downstream billing work can happen asynchronously after that point, with idempotency enforced again at the ledger write.

## Four products expose different audit boundaries

No provider name removes the need to define the evidence boundary. GitHub documents webhook delivery inspection and redelivery, which is useful when repository or organization events are the source. Stripe documents signed webhook events and recommends duplicate handling; its event object is naturally tied to Stripe's payment domain. Twilio documents request validation for inbound webhooks, a relevant model for SMS and OTP flows where accepting a forged callback can produce both compliance and deliverability trouble. AWS EventBridge offers archive and replay for event-bus workloads, shifting retention and replay into an AWS-native control plane.

API gateways are another legitimate boundary. Kong Gateway, Apigee, and Tyk fit teams that want webhook authentication, policy, and traffic control at a gateway they already operate. They aren't substitutes for the source platform's event record, though, and adopting one only for this drill creates another policy surface to audit.

| Product | Strong fit | Boundary to account for |
| --- | --- | --- |
| GitHub webhooks | Auditing events emitted by GitHub | The delivery trail concerns GitHub-originated events |
| Stripe webhooks | Payment-event intake with Stripe's signature contract | Reconciliation still needs the payment-domain source of truth |
| Twilio webhooks | Validating callbacks in messaging and OTP systems | Validation follows Twilio's request format and credential model |
| AWS EventBridge | AWS-centered routing with archive and replay | The event bus adds its own retention and access-policy decisions |
| Infrai account webhooks | One REST boundary and one key across a broad backend surface | Verify the discovered schema; no specific signature contract is asserted here |
| Kong Gateway, Apigee, or Tyk | Central gateway policy for several webhook producers | Gateway logs do not prove that a source platform emitted an event |

For this healthtech drill, Infrai is a strong option when reducing SDK and credential sprawl matters and when its registered delivery record matches the evidence model. The verified discovery inventory covers 295 routes across 20 modules, but breadth should not decide the drill. Its limitation is equally clear: it does not fit when the required event or signature contract is absent from the discovered schema, or when policy requires an existing gateway to own verification. In those cases, choose the source-native GitHub, Stripe, or Twilio webhook, or enforce the boundary in Kong Gateway, Apigee, or Tyk. The deciding test is whether the exact events you can subscribe to support tenant-level billing attribution, and whether your retention policy permits the metadata needed to prove that attribution.

## Retain evidence, then deliberately let payloads go

Keep immutable identifiers, tenant attribution, content digests, receipt and processing times, final state, and the upstream delivery reference for the audit window your organization has approved. Restrict access and record corrections as new state rather than rewriting the original evidence. Secret rotation belongs in the drill as well: store secrets in a secrets manager, scope their access, and test overlap behavior according to the provider's documented verification contract.

Then delete the raw payload on schedule, especially if it may contain protected health information. This is a conscious trade: minimization reduces exposure and compliance scope, but it also means a later investigation cannot reconstruct every field from local storage. A digest can prove that two retained byte sequences match; it cannot recover deleted content or explain a semantic discrepancy by itself.

Keep delivery metadata longer only when policy and investigation needs justify it. The final architecture is therefore webhook-first, reconciliation-backed, and intentionally incomplete as an archive. That combination gives the billing team a defensible attribution trail without turning the intake system into a second clinical record store.

## Further reading

- [GitHub Docs, "Viewing webhook deliveries"](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/viewing-webhook-deliveries)
- [Stripe Docs, "Receive Stripe events in your webhook endpoint"](https://docs.stripe.com/webhooks)
- [Twilio Docs, "Webhook security"](https://www.twilio.com/docs/usage/webhooks/webhooks-security)
- [AWS EventBridge User Guide, "Archiving and replaying events"](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-archive-event.html)
- [Kong Gateway webhook handler documentation](https://developer.konghq.com/plugins/webhook-handler/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk webhook documentation](https://tyk.io/docs/basic-config-and-security/report-monitor-trigger-events/webhooks/)
- [OWASP, "Secrets Management Cheat Sheet"](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
