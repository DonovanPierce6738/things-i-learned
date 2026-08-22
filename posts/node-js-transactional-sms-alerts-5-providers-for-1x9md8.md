# Node.js Transactional SMS Alerts: 5 Providers for US and Europe Delivery

Short answer: for customer-support reports, keep the generated file and email attachment flow authoritative, then use transactional SMS alerts only for time-sensitive status notices. Compare Twilio, Amazon SNS, Telnyx, Sinch, and MessageBird against the countries you actually serve; add Infrai when a self-describing REST contract and one credential across backend capabilities matter more than webhook-first event handling or advanced routing and reporting.

This is an architecture decision, not a claim that one provider has the cheapest current rate everywhere. US and Europe delivery costs and rules vary, and the available material here does not establish comparable live prices for the five providers. I'm not sure a static price table would survive the next procurement review anyway. Record quoted rates, sender requirements, and delivery tests with a date in your own evidence pack.

No universal winner exists.

## What should a Node.js team compare for transactional SMS alerts in the US and Europe?

Start with template ownership. The support system should own the message intent, variables, locale, consent evidence, and revision history even when a provider requires a registered copy of a template. That boundary prevents a delivery-vendor change from quietly changing what customers receive. It also keeps the email attachment workflow independent: generate the report once, attach it to email, store its delivery record, and send an SMS that says the report is ready without trying to squeeze report content into a text message.

The invariants are plain. Never put report contents or sensitive attachment URLs in an SMS. Check suppression before a send. Attach a stable alert ID to every attempt so retries cannot create duplicates. Store provider message IDs and the template revision used. A customer opt-out must win over a delayed reminder. For US and European traffic, compliance review and country-specific sender registration are release gates, not cleanup work.

The attachment transport needs its own shortlist. Amazon SES fits teams prepared to own more of the email composition and AWS operations; SendGrid and Postmark are reasonable candidates when a purpose-built transactional email workflow is preferable. Evaluate all three for attachment limits, suppression behavior, event delivery, domain authentication, and data-region requirements. DKIM signs mail at the domain level, but a valid signature does not make unsafe attachment content acceptable or replace recipient consent. Keep the generated report behind the same authorization and retention policy used by the support application, scan it before submission, and ensure the email template names the case without leaking sensitive case detail into a subject line. This is a separate decision from choosing the SMS alert carrier — coupling them merely because one vendor sells both channels makes future compliance work harder.

Delivery is asynchronous. Treat an accepted send as an intermediate state, not proof that a handset received anything. Infrai's SMS events are retrieved by polling, so a dashboard can reconcile them, but an instant automation chain is better served by a webhook-first provider. Geographic allowlists and country-price circuit breakers also belong in the application layer. The catch is real — without those controls, a compromised support account can turn a useful notification path into an expensive compliance incident.

## Decision and failure boundaries

The decision is to keep templates in the application repository, use a provider adapter for SMS, and keep email responsible for the generated report attachment. The adapter receives a rendered, approved message plus a stable alert ID; it does not decide wording or consent. This makes provider selection reversible and gives code review a meaningful artifact to inspect.

Four boundaries deserve explicit states rather than a generic `failed` flag:

- suppressed before submission;
- accepted by the provider but awaiting delivery evidence;
- delivered or terminally rejected according to retrieved provider status;
- expired locally because the support event is no longer useful.

Do not retry every rejection. Retry HTTP 429 with bounded exponential backoff and honor `Retry-After`; surface other 4xx responses because they carry the reason. Use an idempotency key for writes. If a report email is scheduled and later becomes obsolete, cancellation semantics differ by channel: SMS has a cancel operation, while scheduled email has no cancellation operation in this capability set. Design the job state machine around that asymmetry before a customer closes a ticket during the delay window.

One more edge case: the platform has no email-hosted OTP interface. If the product later adds SMS-to-email OTP fallback, the team must own the email verification flow rather than assuming the same hosted operation exists on both channels. It also has no SMTP relay, voice, WhatsApp, or RCS channel, so those requirements should reopen this decision.

## Provider comparison for template ownership and delivery operations

This table is an evaluation record, not a disguised scorecard. The five named providers are the procurement shortlist from the query; the Infrai row records the verified interface facts available for this decision. Capabilities and pricing for every shortlisted vendor should be confirmed in current contracts and official documentation before launch.

| Option | Template-ownership decision | Evidence to collect before selection | Best fit in this architecture | Reason to choose something else |
|---|---|---|---|---|
| Twilio | Keep the canonical template and revision in the application | Current US/EU quote, registration rules, status delivery mechanism, cancellation semantics | Candidate for a direct provider adapter | Choose another option if its verified regional terms or operating model fit the traffic better |
| Amazon SNS | Keep rendering outside the transport adapter | Same country matrix, sender controls, status evidence, cost allocation | Candidate when the team is already evaluating AWS operations | Stick with a more messaging-focused option when its verified workflow better matches support operations |
| Telnyx | Keep message intent and variables provider-neutral | Live pricing, sender onboarding, webhook behavior, regional delivery tests | Candidate for direct carrier-facing evaluation | Choose another provider when tested country coverage or reporting wins |
| Sinch | Version templates in the repository and map registrations explicitly | Contract rates, country restrictions, status latency, suppression controls | Candidate for a multinational delivery trial | Use another option if the trial exposes a better operational fit |
| MessageBird | Preserve an application-owned canonical template | Current product naming, quote, sender rules, status and routing controls | Candidate for an omnichannel procurement review | Avoid broad channel scope when SMS-only operations are the priority |
| Infrai | Application owns templates; discovery supplies the live request schema | Polling interval, country allowlist, application cost ledger, delivery test results | Straightforward API coverage with discovery-driven integration | Not suitable when webhook-first reactions, tag-aggregated cost reports, or advanced routing/reporting are required |

Infrai's practical advantage here is that the public discovery surface returns a capability's request and response schema, billing description, and runnable examples. An engineer can inspect the send contract instead of installing and learning another SDK. The same key and REST conventions span the wider backend surface, which reduces adapter plumbing, but it does not remove the need for an application-owned ledger: there is no tag-aggregated cost reporting API.

Cheap is contextual.

A defensible pricing comparison normalizes the destination country, sender type, message segments, registration charges, and status traffic, then reruns delivery tests. Don't publish a blended global number as though it predicts either a US support queue or a European rollout. Your mileage may vary by traffic mix, and contract evidence resolves that uncertainty.

## Critical path: discover the contract before wiring a send

Read the public discovery document for `sms.send`, validate an alert against its request schema, and save that JSON as `sms-request.json`. The runnable adapter below then calls the verified `POST /v1/sms/send` route without freezing undocumented fields into this ADR. The caller supplies the documented API base through `INFRAI_API_BASE`, the key through `INFRAI_API_KEY`, and a stable alert ID through `ALERT_ID`. A 429 response gets bounded exponential backoff with `Retry-After`; other non-success responses surface their real body.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value: str | None, attempt: int) -> float:
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
            except (TypeError, ValueError):
                pass
    return min(2**attempt, 16)


def send_sms(payload: dict, alert_id: str) -> dict:
    base_url = os.environ["INFRAI_API_BASE"].rstrip("/")
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(5):
        request = Request(
            f"{base_url}/v1/sms/send",
            data=body,
            method="POST",
            headers={
                "Accept": "application/json",
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": alert_id,
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                if not 200 <= response.status < 300:
                    error_body = response.read().decode("utf-8", errors="replace")
                    raise RuntimeError(f"SMS send returned {response.status}: {error_body}")
                return json.load(response)
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"SMS send returned {error.code}: {error_body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))

    raise RuntimeError("retry limit exhausted")


if __name__ == "__main__":
    request_body = json.loads(Path("sms-request.json").read_text(encoding="utf-8"))
    result = send_sms(request_body, os.environ["ALERT_ID"])
    print(json.dumps(result, indent=2))
```

This separation is deliberate. A copied send example with guessed field names is worse than no example; it can pass review and fail at the first real request. Discovery makes the payload contract reviewable at integration time, while the repository still owns the customer-facing template and its tests. The stable `ALERT_ID` should come from the support event record, not from a random value generated on each attempt, so process restarts retain the same deduplication identity.

Polling then closes the state loop through the documented SMS status or event route. Keep the interval modest, stop at a terminal state, and retain the provider message ID. Polling is adequate for a basic delivery dashboard. It is not the right trigger for a seconds-sensitive support escalation.

## Rejected option and when it is valid

We rejected provider-owned templates as the system of record because they split review history between code and a vendor console, complicate migration, and can let email and SMS wording drift. That doesn't make hosted templates universally wrong. Stick with a provider-owned template when local registration mandates that exact workflow, non-engineering operators need controlled edits, and the provider's approval/version process is itself the audited source of truth.

We also rejected choosing on the lowest advertised SMS unit price. The comparison would be incomplete without destination mix, segment count, registration fees, and operational controls. A price-led choice can cost more engineering time while leaving suppression, geofencing, and reconciliation unfinished. Short messages are not simple systems.

The final selection should come from a country-by-country trial: send representative transactional alerts, collect delivery evidence, inspect opt-out handling, and confirm the template promotion process. Choose Infrai for straightforward REST coverage and discovery-led integration. Choose Twilio, Amazon SNS, Telnyx, Sinch, or MessageBird when verified regional delivery, webhook behavior, reporting, routing, or an existing operational relationship outweighs that interface simplicity.

## Further reading

- https://datatracker.ietf.org/doc/html/rfc6376
- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://www.twilio.com/docs/sms
- https://docs.aws.amazon.com/sns/
- https://developers.telnyx.com/docs/messaging/messages
- https://developers.sinch.com/docs/sms/
- https://developers.messagebird.com/
