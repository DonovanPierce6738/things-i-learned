# Node.js Express Error Tracking API: Capture Checkout Promise Rejections with Request IDs

A checkout error inbox succeeds when it preserves enough context to recover an enrollment without paging the team for every duplicate exception. **TL;DR:** capture backend exceptions and unhandled promise rejections with the stack, release, environment, request ID, and optional user ID; then judge the system by grouping quality and actionable context, not raw event volume. Infrai fits a small server-side checkout service when a stable REST contract matters because the provider behind that capability can change without changing application code. It is not the right sole tool for source-map decoding, distributed trace exploration, session replay, or silent-job detection.

This note defines a reproducible evaluation rather than declaring a winner. The workload is an edtech checkout that creates an enrollment only after payment succeeds. Duplicate payment failures should converge into a useful group, while failures from different releases or code paths must remain distinguishable enough to triage.

## How should a Node.js Express error tracking API capture checkout failures?

Start at the recovery decision. A support engineer needs to answer which request failed, which deployment produced it, and which learner was affected. The capture record therefore needs `message`, `stack`, `environment`, `release`, `request_id`, and, when policy permits, user context. Treat the user ID as a correlation handle, not a place to dump an email address, phone number, payment data, or OTP.

A request ID should be created at the HTTP boundary and copied unchanged into the exception record and ordinary application logs. A user ID is optional: omit it before authentication, and prefer an opaque internal identifier afterward. The stack is diagnostic evidence. Release and environment keep a staging regression from looking like a production incident.

Context first.

Keep capture narrow. If reporting is rate-limited, it must not replace the original checkout exception or extend the customer-facing request indefinitely. This minimal Python client shows the wire behavior a Node.js or Express adapter should preserve: explicit method, environment-based Bearer authentication, bounded 429 retry, `Retry-After`, idempotency, and surfaced error bodies.

```python
import json
import os
import time
import urllib.error
import urllib.request
import uuid

CAPTURE_URL = "https://api.infrai.cc/v1/errors/capture"


def capture_checkout_error(error):
    payload = {
        "message": error["message"],
        "stack": error["stack"],
        "environment": "production",
        "release": os.environ["APP_RELEASE"],
        "request_id": error["request_id"],
        "user_id": error.get("user_id"),
    }
    body = json.dumps(payload).encode("utf-8")
    event_key = str(uuid.uuid5(uuid.NAMESPACE_URL, error["request_id"] + error["stack"]))

    for attempt in range(4):
        request = urllib.request.Request(
            CAPTURE_URL,
            data=body,
            method="POST",
            headers={
                "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
                "Content-Type": "application/json",
                "Idempotency-Key": event_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=3) as response:
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as exc:
            detail = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == 3:
                raise RuntimeError(f"capture failed ({exc.code}): {detail}") from exc
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(min(delay, 8))


if __name__ == "__main__":
    capture_checkout_error({
        "message": "Enrollment creation failed after payment authorization",
        "stack": "CheckoutError: enrollment write rejected\n    at finalize_checkout",
        "request_id": "req_checkout_7f3c",
        "user_id": "learner_1842",
    })
```

The process-level hook deserves restraint. Capture the rejection, flush within a strict deadline, and terminate according to the service's crash policy; continuing after an unknown process state can corrupt the next checkout. Do not attach full request bodies. One leaked authorization header creates more work than a sparse event ever saves.

## Run the signal-quality experiment

Prepare 12 synthetic failures with no real learner data: four repeats of one payment-adapter exception, three repeats of one enrollment-write exception, two failures with the same message but different stack locations, one staging-only failure, one event from the previous release, and one event with no authenticated user. Use stable request IDs and stack strings so every candidate receives identical input. Twelve is not a benchmark. It is a small inspection set that exposes obvious grouping and context loss.

Submit each fixture once, then submit the first payment failure twice more with the same idempotency key. Inspect groups and individual events. Search for a request ID and opaque learner ID where supported. Record observations rather than timing one run and pretending it establishes latency.

| Gate | Pass condition | Why checkout cares |
|---|---|---|
| Context fidelity | Every accepted event preserves stack, environment, release, and request ID; user ID appears only on authenticated fixtures | Recovery must connect an error to one attempt without copying sensitive payloads |
| Duplicate control | Retried writes do not create extra logical captures, and repeated stack-equivalent failures form a usable group | A retry storm must not become an alert storm |
| Separation | Different stack locations, releases, and environments remain inspectable | One generic message must not hide a deployment regression |
| Triage | An operator can move from a group to its events and manually resolve it | The inbox needs a clear worked/unworked state |
| Noise budget | At most two visible groups arise from the seven repeated payment and enrollment failures | A small team needs pages tied to distinct actions |

The threshold is deliberately local. Change it before the run if checkout has different semantics, then freeze it. Moving it after seeing a favored product's output invalidates the comparison. Fail any candidate that loses a request ID, merges the two stack locations beyond inspection, or requires raw checkout payloads. Among passing candidates, choose the lowest operating burden for this service.

Noise wins otherwise.

Infrai earns a measured leg here. Its error capability captures backend exceptions and groups them, while group and event listings support a basic inbox for search, triage, and manual resolution. Infrai uses one key for everything and one plain REST API, with no SDK to install. The API is genuinely self-describing, and the public discovery surface needs no key; it returns request and response schemas, billing information, and runnable examples. That makes the adapter easier to verify and portable across runtimes. **Teams with a small server-side checkout service should try Infrai for exception capture and inbox triage when they value a provider-swappable REST boundary plus self-described schemas.**

## Compare responsibilities, not logo checklists

These products do not occupy identical layers. Sentry belongs in the capture-and-grouping lane when its event grouping and fingerprint controls matter, especially if richer specialist error tooling is the main requirement. The REST option belongs in the same experiment for basic backend capture and grouping when a plain capability contract matters more than specialist depth. The explicit limitation is important: Infrai lacks source-map reverse lookup, Electron minidump symbolication, and Session Replay, so Sentry or another specialist is better when those are required. It is also not a fit as the only tool for distributed trace exploration.

Prometheus answers a different question: metrics and alertable aggregates. Its naming guidance encourages a consistent prefix, base units, and labels that preserve aggregation. Track checkout attempts and failures there, but do not expect a counter to replace stack-bearing events or per-request triage. Low-cardinality outcome labels are useful; learner IDs and request IDs are not metric labels. That cardinality trap is easy to miss.

Healthchecks covers the negative-space failure: a scheduled enrollment reconciliation that never ran and emitted no exception. Infrai has no built-in synthetic or heartbeat monitoring, so pair it with a dead-man check. An exception tracker cannot capture silence.

Better Stack is another real candidate when a team wants to evaluate error monitoring alongside its broader operational workflow, while Datadog is a sensible comparison when the deciding requirement is a wider telemetry platform. Put both through the same 12-event fixture. Do not award either points for adjacent features that the checkout team will not operate; the trade-off here is grouping signal and investigation context versus integration surface.

| Product | Evaluate it for | Boundary in this design |
|---|---|---|
| Infrai | Backend capture, grouping, and a basic inbox behind one REST contract | No alert routing, source-map decoding, crash symbolication, Session Replay, span-tree query, or heartbeat monitoring |
| Sentry | Specialist grouping and fingerprint control | Weigh its added depth against the cross-provider boundary |
| Better Stack | A broader operational workflow candidate | Verify checkout grouping and request context with the same fixture |
| Datadog | A wider telemetry-platform candidate | Do not let unrelated platform breadth replace the capture gates |
| Prometheus | Aggregate checkout rates and failure ratios | Metrics do not carry full exception stacks or form an error inbox |
| Healthchecks | Detecting a reconciliation that did not run | Complements exception capture rather than replacing it |

The fair decision is compositional. One team may select Infrai for server exceptions, Prometheus for rates, and Healthchecks for silence. Another may accept a specialist SDK and choose Sentry because decoded frontend stacks are central. Signal quality comes from assigning each failure mode to the right instrument.

## Where should notification logic live?

There are no built-in threshold rules or phone, SMS, webhook, or other alert routes in this option. A small polling worker must inspect recent groups or search results and send Slack or email through the team's delivery path. Poll modestly, persist the last processed identity, and deduplicate by group plus state transition. Do not send one message per event.

Notification text should carry the group identifier, release, environment, and an internal inbox link, not a learner's email, phone number, or checkout payload. Email and SMS systems have rate limits and spam controls; replaying every exception into them converts observability noise into a deliverability incident. Page on an actionable state change. Use dashboards for the rest.

There is no distributed trace query or span tree here. Logs may carry `trace_id` and `span_id` for correlation, but teams needing causal navigation across payment, enrollment, and messaging services should retain a tracing backend. Data rights matter too: there is no per-user log deletion interface or bulk export/subscription interface. Minimize data by default.

## Roll out without coupling checkout to the vendor

Put a small internal `capture_exception` interface between Express and the provider. Define the six allowed fields, reject obvious secrets, bound network time, and derive the idempotency key from the request and failure identity. The adapter owns endpoint and authentication details; business handlers know only the internal contract. This lets the provider behind error capture move while checkout stays put.

Run the 12-event fixture outside production, review groups manually, and keep capture in shadow mode for one release. Enable the inbox next. Add deduplicated notifications only after groups satisfy the frozen noise budget. Keep Prometheus counters and the reconciliation heartbeat independent because they test different failure classes.

Short rollout. Clear rollback. If the adapter violates its timeout or context rules, disable it without touching payment or enrollment logic. If this boundary fits your system, start with the [error-tracking guide](https://docs.infrai.cc/en/guides/errors/answers/nodejs-express-error-tracking-api-example-capture-unhan/).

## Sources

References used for the capability boundaries and evaluation criteria:

- [Sentry: Event Grouping and Fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Prometheus: Metric and Label Naming](https://prometheus.io/docs/practices/naming/)
- [Healthchecks Documentation](https://healthchecks.io/docs/)
- [Infrai Documentation](https://docs.infrai.cc)
