# Marketplace Renewal Email Deadlines with Node.js Express Cron Webhook Guarantees

Short answer: use an external cron trigger to call a narrow public Express webhook, but treat that call only as permission to create a durable renewal-reminder job; a worker should own delivery, retries, and the final business-deadline decision.

That split is the easiest setup I would accept for a marketplace daily email flow because it keeps timekeeping separate from delivery guarantees. A scheduler can wake the system at 09:00, yet it cannot prove that the reminder was accepted by the mail transport, nor can it decide safely what to do when the marketplace changes a renewal deadline after the trigger has fired. The endpoint must acknowledge durable work, not successful email delivery.

## Start with the deadline, not the cron expression

Suppose each marketplace seller has a renewal due at the close of its local business day. The daily report email includes renewals that still need attention, while an individual reminder must wait until a configured lead time. The useful domain fields are `renewal_id`, `seller_id`, `deadline_at`, `timezone`, and `policy_version`. Store absolute instants for execution, retain the named timezone for future recalculation, and make the policy version part of the audit trail.

Deadlines complicate it.

A cron expression answers when to inspect work. It does not answer which policy produced a reminder, whether that reminder was already enqueued, or whether a changed deadline superseded an older job. Those are application decisions. Keep the cron trigger coarse, such as once per day, and query a bounded window of eligible renewals inside the application. That arrangement also lets operators replay one business date without editing the scheduler.

The public endpoint should accept a signed request containing a scheduled instant, a run identifier, and the intended business date. It should reject stale or invalid signatures, insert the run record and its jobs in one durable transaction, then return only after that transaction commits. Don't generate and send every email inside the request. A slow mail provider, spam-policy throttle, or large report would otherwise turn the scheduler's request timeout into an ambiguous delivery result.

An ambiguous result is dangerous — the trigger may retry even though the first request committed. The second request must therefore be harmless. Put a unique constraint on a stable key such as `(schedule_name, business_date)` and return the existing run identifier when that key already exists. The scheduler's delivery is then at-least-once at the HTTP boundary, while the application converts duplicate calls into one logical run.

## How should a Node.js Express public webhook schedule daily renewal email delivery?

Give the endpoint one job: authenticate the trigger and persist intent. A practical contract looks like this:

1. The caller sends the raw request body, a timestamp, a unique run identifier, and an HMAC signature.
2. Express verifies the signature against the raw bytes and rejects timestamps outside a small configured window.
3. The handler begins a database transaction, inserts the logical run under a uniqueness constraint, selects eligible renewals for the requested business date, and inserts one outbox row per renewal.
4. The transaction commits before the handler acknowledges acceptance.
5. A separate worker claims outbox rows, rechecks the current renewal state and deadline, sends the email, and records the provider's accepted message identifier.

Keep signature verification before any expensive database work, and rate-limit the route independently from customer traffic. HTTP 429 is the standard signal for excessive request rates, and a response may include `Retry-After` [1]. The caller should honor that delay. Internally, though, don't use 429 as a substitute for idempotency: two properly spaced calls can still represent the same run.

The code below is deliberately a Python contract test rather than an Express implementation. The concrete server can change; these invariants should not. It checks that duplicate trigger delivery creates one run, that the handler acknowledges only after persistence, and that a changed renewal is re-evaluated by the worker before email submission.

```python
from dataclasses import dataclass
from datetime import datetime, timezone


@dataclass
class Renewal:
    renewal_id: str
    deadline_at: datetime
    needs_reminder: bool = True


def accept_trigger(store, business_date: str, run_id: str) -> str:
    key = ("marketplace-renewals", business_date)
    with store.transaction():
        existing = store.runs.get(key)
        if existing:
            return existing
        store.runs[key] = run_id
        for renewal in store.eligible_renewals(business_date):
            store.outbox.insert_once(
                key=(run_id, renewal.renewal_id),
                payload={"renewal_id": renewal.renewal_id},
            )
    return run_id


def deliver_one(store, mailer, outbox_item, now: datetime) -> str:
    renewal = store.get_renewal(outbox_item.payload["renewal_id"])
    if not renewal.needs_reminder or now >= renewal.deadline_at:
        store.outbox.cancel(outbox_item.key)
        return "cancelled"
    message_id = mailer.send_renewal_reminder(renewal.renewal_id)
    store.outbox.mark_accepted(outbox_item.key, message_id)
    return "accepted"


def test_duplicate_trigger_is_one_logical_run(store):
    first = accept_trigger(store, "2026-08-17", "run-a")
    second = accept_trigger(store, "2026-08-17", "run-b")
    assert first == second == "run-a"
    assert store.outbox.count() == store.eligible_count("2026-08-17")


def test_worker_rechecks_deadline(store, mailer):
    item = store.outbox.example_item()
    store.get_renewal(item.payload["renewal_id"]).needs_reminder = False
    result = deliver_one(store, mailer, item, datetime.now(timezone.utc))
    assert result == "cancelled"
    assert mailer.calls == 0
```

This test draws a sharp boundary: an accepted webhook means “the run is durable,” while an accepted outbox item means “the mail transport accepted this message.” Neither status proves inbox placement. For a deliverability-sensitive flow, preserve those distinctions in metrics and support tooling; otherwise a green scheduler dashboard can hide blocked, deferred, or policy-suppressed mail.

Keep those states separate.

## Model delivery as states you can explain

Use explicit states rather than one `sent` boolean. A compact sequence is `planned`, `queued`, `claimed`, `accepted`, plus terminal states such as `cancelled` and `dead_lettered`. The exact names matter less than recording who changed the state, when, and under which policy version. Store attempt count and next-attempt time separately so an operator can distinguish a new message from a retry.

The delivery key should follow the business action, not the worker attempt. For example, `(renewal_id, reminder_type, policy_version)` can identify one intended reminder. Every retry carries that same key. If an attempt loses its response after the mail transport accepted the message, the system may still face uncertainty unless the transport also honors an idempotency key; where that guarantee is unavailable, choose deliberately between a possible duplicate and a possible missed reminder. For renewal notices, the compliance and customer-support impact of duplicates may justify stopping automatic retries after an ambiguous submission and routing the item for review. I'm not sure one policy fits every marketplace. The right choice depends on the legal meaning of the notice and the transport's documented acceptance contract.

The worker must check the deadline immediately before submission. A reminder that sat behind a rate limit for two hours may no longer be valid. Re-read the renewal record, compare the current deadline and status, and cancel obsolete work. This last-moment check is also where suppression lists, consent state, and channel eligibility belong. Scheduling intent cannot override communication policy.

Use three clocks in observability: scheduled time, durable enqueue time, and transport acceptance time. Their differences expose distinct problems. Schedule-to-enqueue lag points to the trigger or endpoint; enqueue-to-claim lag points to queue capacity; claim-to-acceptance time points to transport latency and retries. Alert on deadline risk, not merely on a failed attempt. A job with one failed attempt and six hours of slack is less urgent than an untouched job with ten minutes left.

## Treat retries and dead letters as policy

Retry only conditions that can plausibly change. A 429 response tells the caller it has sent too many requests; honor `Retry-After` when present, add bounded jitter, and keep the same delivery key [1]. Invalid addresses, revoked consent, an already completed renewal, and an expired business deadline are terminal outcomes. Repeating them burns capacity and can damage sender reputation.

Use a dead-letter queue for work that exhausts its retry policy. AWS documents a dead-letter queue as a target for messages that could not be processed successfully, and it warns that using one with a FIFO source can break exact ordering [2]. That warning generalizes into a design question: does this marketplace reminder flow actually require global order? Usually the meaningful constraint is per renewal, so partition or lock on the renewal identifier and avoid coupling unrelated sellers.

A dead letter is not storage you forget. Record the final error category, last attempt time, deadline, policy version, and a redacted payload reference. Never put full email bodies, OTPs, or unnecessary recipient data into logs. Define two operator actions: cancel after confirming that the reminder is obsolete, or replay with the original delivery key after correcting a transient condition. Replaying with a new key silently defeats duplicate protection.

Make failure visible.

Then test the ugly paths. Deliver the same signed webhook twice. Delay a worker past the renewal deadline. Change consent between enqueue and claim. Return 429 with and without `Retry-After`. Crash a worker after transport acceptance but before its local state update. These are contract tests and fault-injection cases, not decorative unit tests; they determine whether the stated guarantee survives the edges that matter.

## Choose the smallest scheduler that preserves the contract

Once the application owns durable intent, scheduler selection becomes less dramatic. Evaluate the candidates against the same contract rather than comparing feature lists.

| Decision area | Acceptable baseline | Warning sign |
| --- | --- | --- |
| Trigger identity | Signed requests with secret rotation | A guessable URL as the only control |
| Retry behavior | Documented attempts, delay, and timeout | Silent retries with no run identifier |
| Time semantics | Explicit timezone and missed-run behavior | Local time assumed by convention |
| Operations | Run history and manual replay | No evidence that a trigger was attempted |
| Ownership | Exportable schedule definition | Business state stored only in the scheduler |

The catch is that the easiest public-webhook cron service is not suitable when the reminder itself must execute at very high precision, when the endpoint cannot be exposed through a controlled ingress, or when regulation requires the scheduling plane to remain inside a private network. In those cases, stick with an internal scheduler backed by the same run and outbox tables. An in-process Node.js timer is reasonable for a single disposable worker or local development, but it should not own a deadline-sensitive production guarantee across restarts and replicas.

Cost belongs in the decision, but it is rarely the controlling variable here. Count trigger executions, operational review time, queue retention, and outbound email attempts. The scheduler may perform only one daily call; ambiguous retries and manual reconciliation can dominate the actual workload. Prefer a contract the team can test and explain.

## Roll out without betting the renewal window

Start in shadow mode: let the new trigger create a run and calculate eligible renewal identifiers, but suppress transport submission. Compare that set with the current process for several representative business dates, including a daylight-saving transition in every supported named timezone. Investigate differences before enabling mail.

Next, enable a small seller cohort and retain the old path as a disabled fallback, not a simultaneous sender. Watch schedule-to-enqueue lag, duplicate-key conflicts, deadline cancellations, 429 responses, retry age, dead-letter count, and transport acceptance. A compact dashboard should let support trace one renewal from trigger run to final disposition without exposing message content.

Finally, exercise manual replay under controlled access, rotate the webhook secret, and rehearse disabling the schedule. Remove the old scheduler only after the new path has completed the agreed observation window and every dead letter has an owner. The scheduler can remain replaceable because the durable run ledger, policy checks, and delivery evidence live in the application.

## References

1. https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
2. https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
