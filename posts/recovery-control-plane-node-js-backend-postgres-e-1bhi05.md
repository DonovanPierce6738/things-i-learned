# Recovery Control Plane: Node.js Backend, Postgres Email Send, and Enumeration Defense

Short answer: build the forgot-password feature as a database-backed control plane that always returns the same success response, applies cooldown and retry policy before sending email, and records the resulting message ID for later audit.

The mail API is downstream of that decision. It cannot know that an address is unknown, that the same IP has requested five resets, or that a support agent will need to trace one message next week. Keep those facts in PostgreSQL and let the sender do the narrow job of accepting a transactional message and reporting its status.

This split matters in a Node.js backend even though the focused sender example below is Python. The HTTP boundary stays the same in either language, while the application owns the security invariants. Don't let an SDK's convenience methods become the architecture.

## How should a Node.js backend prevent user enumeration during email retry?

Every public branch should return one status and one body, such as: “If an account matches that address, a reset message will be sent.” Use it for an existing account, an unknown address, and a request rejected by the cooldown. A generic body closes the obvious enumeration path; applying the same outward behavior to throttled requests closes the less obvious one.

The invariants are small enough to review directly:

1. Normalize the email address, open a transaction, and record the reset request before deciding whether a message should be sent.
2. Enforce the cooldown and retry counter in PostgreSQL. Abuse prevention is application-managed, including any source-IP or account policy.
3. Generate a single-use reset token only for an eligible account, store its hash, and pass the clear token only into the message being prepared.
4. Return the generic response without waiting for delivery. The worker records the provider result and message ID against the request.
5. Poll message or event status when support needs to investigate “the email never arrived.”

There is an awkward edge here. If an unknown address leaves no request row but a known address does, internal metrics and cooldown behavior diverge even though the HTTP bodies match. Recording all attempts gives the rate limiter one consistent input and gives the audit trail an honest account of suppressed work. Access to that audit data must be restricted because the data can itself reveal membership.

No branch gets a special 404. No known-account-only 429, either.

The database model needs a request identifier, normalized address, token hash where applicable, creation and expiry times, cooldown boundary, retry count, status, provider message ID, and a compact provider response. The audit stream should distinguish requested, suppressed, accepted, and status-checked events without putting the clear reset token into logs. I'm not sure what retention period fits your jurisdiction; settle that with your security and privacy owners, then enforce it as a deletion policy rather than an aspiration in a runbook.

## Put the failure boundaries in the database

Treat the API handler, worker, and mail provider as three separate failure domains. The handler commits the policy decision and returns the generic response. The worker claims an eligible request and sends it. The provider accepts the message and exposes status that can be polled later. That ordering means a process restart cannot erase the fact that a user asked for recovery, and a retry counter survives worker replacement.

The critical transaction should lock the applicable cooldown state, increment the attempt record, and create at most one sendable job. A client-supplied idempotency key derived from the immutable request ID then follows that job to the provider. If the provider responds with HTTP 429, honor `Retry-After` when it is present; otherwise use exponential backoff. Keep the same idempotency key across those attempts so uncertainty about a response does not become a duplicate reset email.

Be strict here — a retry is a continuation of one request, not a new request.

The audit log is useful only if it joins the layers. Store the reset request ID before enqueueing, the provider message ID after acceptance, and the later status observation with its timestamp. When someone reports a missing email, support can start from the request, find the accepted message, and poll its send or event status. Normal password recovery is a single-send flow; batch sending belongs to genuine fan-out notices, not one person's reset request.

## Compare the delivery choices against the recovery path

Provider selection comes after the invariants because the providers solve different parts of the path. This is not a price contest.

| Option | Verified role in this decision | Integration consequence | Prefer it when | Do not choose it for |
| --- | --- | --- | --- | --- |
| Infrai | Email send plus pull-based message and event status | Plain REST with Bearer authentication; no SDK or client-library version to maintain | The service should make one direct HTTP integration and polling is acceptable | SMTP relay or webhook-driven delivery events |
| Amazon SES | Email delivery service | Evaluate its official integration and operating model against the same request-ID and audit requirements | Your architecture has already selected SES and its documented email workflow | An SMS recovery channel |
| Twilio | SMS delivery service | It is a separate transport rather than the email sender in this design | A product decision calls for SMS as a distinct recovery or notification channel | The password-reset email itself |

Infrai's meaningful advantage in this narrow flow is mechanical: anything that can make an HTTP request can call one REST API, so the worker needs no vendor SDK to install or update. That keeps the provider adapter small. The trade-off is equally concrete: email events are pulled rather than delivered through webhooks, so it is a fit for audit and support polling, not for a UI that promises immediate event updates.

Several boundaries should stop a casual expansion of this design. There is no SMTP relay. Email has no hosted OTP endpoint, so an emailed numeric code remains application-owned, and scheduled email has no cancellation interface. Voice, WhatsApp, and RCS are outside the available channels. There is no cost report grouped by tag, while SMS template listing is also unavailable. A pending domestic Tencent email vendor cannot serve as evidence of China compliance. For SMS, geographic fencing and country-price circuit breakers remain application controls. Those aren't footnotes; they determine when Amazon SES, Twilio, or another architecture deserves a fresh evaluation.

## Run one idempotent send on the critical path

The example intentionally accepts the provider's JSON payload from a file. The verified facts establish the route, retry rules, and authentication scheme, but not the message schema; inventing fields would make a copyable example dangerous. Export the key, place a payload that conforms to the current email-send schema in `payload.json`, and run the script. It makes one logical send, even if rate limiting requires several HTTP attempts.

```python
import json
import os
import sys
import time
from pathlib import Path

import requests


BASE_URL = os.environ["EMAIL_API_BASE_URL"].rstrip("/")
SEND_URL = f"{BASE_URL}/v1/email/send"
API_KEY = os.environ["INFRAI_API_KEY"]


def retry_delay(response: requests.Response, attempt: int) -> float:
    value = response.headers.get("Retry-After")
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError as error:
        raise RuntimeError("Retry-After must be a number of seconds") from error


def send_email(payload: dict, request_id: str, attempts: int = 4) -> dict:
    for attempt in range(attempts):
        response = requests.request(
            method="POST",
            url=SEND_URL,
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Idempotency-Key": request_id,
                "Content-Type": "application/json",
            },
            json=payload,
            timeout=15,
        )
        if response.status_code == 429:
            time.sleep(retry_delay(response, attempt))
            continue
        if response.status_code >= 400:
            raise RuntimeError(
                f"email send rejected with {response.status_code}: "
                f"{response.text[:300]}"
            )
        return response.json()
    raise RuntimeError("email send remained rate-limited after four attempts")


def main() -> None:
    if len(sys.argv) != 3:
        raise SystemExit("usage: python send_reset.py payload.json REQUEST_ID")
    payload = json.loads(Path(sys.argv[1]).read_text(encoding="utf-8"))
    result = send_email(payload, sys.argv[2])
    print(json.dumps(result, indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
```

Persist the returned message ID and response beside the request rather than treating printed JSON as the audit system. A separate polling job can retrieve message and event status for unresolved support cases. Choose its interval for the actual consumer: a support dashboard can tolerate delay; an interaction claiming live delivery cannot.

The rejected architecture is direct SMTP from the application. It remains valid when an organizational requirement mandates SMTP relay, but it is not supported by this REST path and it discards the small-adapter advantage described above. The other rejected shortcut is delegating cooldowns to the provider. Provider rate limits protect provider capacity; they do not implement your account, IP, retry, or enumeration policy.

## References

- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Amazon SES documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Twilio SMS documentation: https://www.twilio.com/docs/sms
