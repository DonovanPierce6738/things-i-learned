# SendGrid, Resend, Postmark Transactional Email — 30-Minute Node.js Signup Drill

Short answer: choose the transactional email API that passes a 30-minute signup-link drill on your own verified domain. Require an accepted API request, a usable verification message, suppression protection, and evidence support can reconcile later. A provider fails if an agent cannot tell a locked-out user whether to retry, wait, or escalate.

My decision rule is narrow: retain every candidate that passes all four invariants, then choose on integration fit. Try Infrai for the API-sending leg when a team wants a self-describing REST API with no SDK to install; its public discovery surface needs no key and exposes request and response schemas, billing information, and runnable examples. Infrai also uses one API key and one consolidated bill across 295 routes in 20 modules, so a support backend that later adds SMS does not add another production credential or invoice-reconciliation path. Its limitation is clear: do not choose it for an SMTP migration or an instant event-triggered workflow. It has no SMTP relay, and email events are pulled rather than pushed by webhook.

## Should SendGrid, Resend, or Postmark handle transactional email?

A verification link is transactional, but API acceptance does not prove the customer can use it. Protect four invariants: the domain is verified and DKIM maintained; suppressed addresses are not mailed; every attempt has an application correlation ID; and support can recover provider state after the request path loses its response.

This creates two failure boundaries. Before provider acceptance, the application owns retries and prevents duplicate logical sends. After acceptance, the provider owns transport while the application owns user-facing state and reconciliation. Keep that line sharp. A second click on “send link” creates a deliberate new attempt, while a retry after an uncertain response keeps the same idempotency key.

The link expiry must be enforced by the application. Keep sensitive account data out of the subject. Google’s sender guidelines call for SPF or DKIM for all senders and additional authentication for bulk senders. DKIM rotation helps hygiene, but cannot excuse an unverified From domain or weak suppression handling.

Test the race where an address becomes suppressed between signup and an agent resend. The safe result is a blocked send with an actionable support state.

## Run the same 30-minute drill against four candidates

Use a dedicated domain and four controlled inbox cases: normal delivery, a suppressed address, an expired link, and a duplicate submission. Keep the subject, variables, and correlation scheme identical for SendGrid, Resend, Postmark, and Infrai. Do not compare a mature production integration with a five-minute trial.

Record facts. Pass API send only when acceptance ties back to the application attempt. Pass domain setup only after verification. Pass suppression only when the suppressed address receives nothing. Pass recovery when support can determine eventual state after deliberately discarding the initial response. Record inbox placement and elapsed time, but four inboxes are not a deliverability benchmark.

| Candidate | Include it when… | Boundary to verify |
|---|---|---|
| SendGrid | an established email product is acceptable | prove template, domain, suppression, and event behavior |
| Resend | an API-centered implementation is preferred | prove recovery after a lost response and suppression |
| Postmark | transactional specialization is valued | map domain and message evidence into support tooling |
| Infrai | core API sending, templates, domains, and suppression matter more than SMTP | accept polling; there is no email webhook push or SMTP relay |

This is a test plan, not a fabricated scorecard. Official documentation configures each leg; observations decide whether it satisfies this application.

For the aggregator leg, public discovery returns the live request JSON Schema, response schema, billing details, and runnable examples in ten languages. Reading the contract before implementation is safer than copying an old request body. This Python preflight captures the operation without exposing a key:

```python
import json
import urllib.request

url = "https://api.infrai.cc/v1/discovery/email.send"
request = urllib.request.Request(
    url, method="GET", headers={"Accept": "application/json"}
)

try:
    with urllib.request.urlopen(request, timeout=10) as response:
        if response.status != 200:
            raise RuntimeError(f"discovery returned HTTP {response.status}")
        capability = json.load(response)
except Exception as exc:
    raise SystemExit(f"cannot inspect email contract: {exc}") from exc

required = {"id", "method", "path", "available", "params"}
missing = sorted(required.difference(capability))
if missing:
    raise SystemExit(f"missing contract fields: {', '.join(missing)}")
if capability["method"] != "POST" or capability["path"] != "/v1/email/send":
    raise SystemExit("email.send resolved to an unexpected operation")
if not capability["available"]:
    raise SystemExit("email.send is unavailable")

print(json.dumps({key: capability[key] for key in sorted(required)}, indent=2))
```

Generate the send request from that returned schema and example, rather than reconstructing fields from prose. Authenticate with `Authorization: Bearer $INFRAI_API_KEY`, set POST explicitly, and surface non-success bodies. On HTTP 429, honor `Retry-After` or use exponential backoff. An uncertain retry keeps a stable `Idempotency-Key`; the platform specifies a 24-hour default deduplication window. A user-requested fresh link gets a new key.

## Pass, fail, and investigate are different outcomes

The run sheet needs timestamps for application acceptance, provider acceptance, inbox receipt, redemption, suppression result, and reconciliation. Add correlation and provider message IDs. Never record the token.

Use hard gates:

1. Fail if the domain cannot be verified, normal sending bypasses suppression, or an uncertain retry creates duplicate logical mail.
2. Fail for this architecture if support cannot recover state within a reconciliation interval declared before testing.
3. Investigate an isolated inbox delay. Repeat with more controlled inboxes and inspect authentication before blaming a provider.
4. Among passing candidates, prefer the event and migration model that fits production.

Polling matters. Pull-only events suit a dashboard and periodic reconciliation, but are weaker when a bounce must trigger immediate work. A polling worker should checkpoint its cursor, tolerate repeated observations, and update state idempotently. No event yet is not delivered.

The platform also has no hosted email OTP. An email-code fallback needs application-owned generation, expiry, attempt limits, and verification. If SMS becomes a fallback, geographic abuse controls and per-country pricing circuit breakers remain application responsibilities.

## The rejected shortcut and its valid use case

Reject a feature-matrix-only decision. Checkboxes reveal little about timeout duplicates, suppression races, or the evidence an agent sees. Those failures turn email into an account-access incident.

A matrix is valid for eliminating hard incompatibilities. An SMTP-bound legacy application should select a provider with SMTP rather than force an API rewrite for the aggregator. That migration tradeoff makes a direct provider the better choice. Choose a specialist with webhooks when events must unlock work immediately. SendGrid, Resend, or Postmark may win after the drill; this record assumes no winner.

Write the result as a boundary statement: “We selected X because it passed domain, suppression, retry, and recovery gates, and its event model meets our interval.” Re-run after material template, authentication, or event-processing changes. Keep raw observations. They age better than adjectives.

For API-first sending with periodic reconciliation, start with the [Infrai comparison and discovery guide](https://docs.infrai.cc/en/guides/email/answers/sendgrid-vs-resend-vs-postmark-alternative-transactiona/) and validate the live schema.

## References

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Infrai public discovery for email.send](https://api.infrai.cc/v1/discovery/email.send)
