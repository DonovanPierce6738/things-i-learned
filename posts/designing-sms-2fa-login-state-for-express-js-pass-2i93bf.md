# Designing SMS 2FA Login State for Express.js Passwordless Authentication

Short answer: model passwordless sign-in as one server-owned challenge, try SMS first, and offer email only as an explicit fallback on that same challenge. Keep delivery state separate from verification state, hash every one-time code, enforce one shared attempt budget, and let exactly one successful verification consume the challenge.

This is an architecture decision record for an Express.js service, not a provider setup guide. Express should own the HTTP boundary, while a small challenge service owns issuance, fallback, verification, and consumption. That separation matters because a provider accepting a message is not proof that a person received it.

## What should an Express.js 2FA login do when an SMS OTP needs email fallback?

The invariant is simple: one login intent creates one challenge ID. The SMS OTP and any later email code are credentials attached to that challenge; they must not create two unrelated login attempts. A user can request the fallback, but a delayed SMS must remain understandable and safely bounded after email is sent.

For a concrete policy, this example gives a challenge a ten-minute lifetime, permits an email request after 30 seconds, and allows five verification attempts across both channels. Those numbers are application choices, not universal defaults. Tune them against abuse patterns, support load, and observed delivery latency. I'm not sure a single fallback delay is right for every country or carrier; per-route delivery telemetry is what would resolve that decision.

There are four invariants:

1. A challenge has one subject, one expiry, one attempt counter, and one consumed timestamp.
2. Raw codes never enter the database, logs, traces, analytics events, or API responses.
3. Sending and verifying are separate state transitions. A delivery acknowledgment records an accepted send attempt, not successful receipt.
4. Consumption is atomic. Two correct codes arriving together cannot create two sessions.

This also defines the failure boundaries. The SMS adapter may reject a request or rate-limit it; the email adapter has its own outcome. Neither adapter can authenticate a user. The challenge store may decline a stale compare-and-set. The Express route translates those outcomes into deliberately bland client responses so an attacker doesn't learn whether a phone number or email address is registered.

I've found `429` handling especially easy to put in the wrong layer. A provider throttle should become a delivery event for operations and a retry decision for the adapter; it should not reset the user's attempt counter or quietly mint another challenge. Keep those clocks apart.

Clocks aren't credentials.

## Record the state before choosing a provider

A useful record contains `challenge_id`, a stable internal `subject_id`, `expires_at`, `attempts_remaining`, `consumed_at`, and channel-specific send state. Each issued credential needs its own salt, digest, channel, and issuance time. Contact destinations belong behind an internal subject lookup; copying a full phone number and email address into every challenge expands the places where sensitive data can leak.

Make transitions conditional. `request_email_fallback` should succeed only when the challenge is active, the policy delay has elapsed, and email has not already been issued. `verify` should decrement the shared budget on a mismatch and atomically set `consumed_at` on a match. A resend is a transition too, with an idempotency key and a cooldown, not an innocent repeat of the send call.

Short responses help. Return the same accepted shape for an eligible account and an unknown identifier, then perform any real send behind the boundary. Don't put the OTP in a query string, and don't log request bodies on the verification route. The code is a credential. Treat it like one.

No exceptions.

The message body deserves engineering attention as well. SMS encoding and length determine whether a message is split into multiple segments, so keep the authentication copy compact and test the exact characters produced by localization. Email gives more room, but the fallback should still identify the login intent, state the expiration policy, and avoid links that turn a code flow into a second authentication mechanism.

## Compare the fallback models

| Model | State shape | Main benefit | Failure boundary | Best fit |
|---|---|---|---|---|
| One challenge, channel-specific codes | Shared expiry and attempt budget; separate code digests | Delayed messages are attributable without sharing the same secret | Verification and consumption must be atomic | Most SMS-first login flows with manual email fallback |
| One challenge, one code copied to both channels | Shared everything, including the code | Very small state model | Exposure in either inbox compromises the same credential | Low-complexity systems after a deliberate risk review |
| Independent SMS and email challenges | Separate expiry, attempts, and consumption | Channels can evolve independently | Races can create duplicate sessions or confusing lockouts | Separate login methods that are intentionally not fallbacks |
| Automatic email after a timer | Server triggers the second send | Fewer user actions | Normal SMS delay can cause unnecessary duplicate delivery | Controlled environments with measured channel latency |

The first model is the decision here. The catch is the extra state and the need for an atomic store operation. It isn't suitable when email is meant to be a fully independent sign-in method; in that case, keep separate challenges and converge only at session issuance. The single-code model can be valid for a small internal tool whose threat review accepts the shared-secret boundary.

Notice what is absent from the decision: price is not the primary axis. Delivery visibility, data handling, regional reach, throttling semantics, and adapter testability shape the authentication boundary. Cost matters during capacity planning, but it cannot repair ambiguous challenge state.

## Put the critical path behind the Express routes

The Express application needs three thin handlers: start a challenge, request email fallback, and verify a code. Validate input at the edge, resolve the subject without revealing account existence, call the domain service, and map its typed result to a stable response. The following Python models the critical service because the state machine is easier to audit without framework plumbing; the same transitions belong behind the Express handlers.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from enum import Enum
import hashlib
import hmac
import secrets


class VerifyResult(str, Enum):
    ACCEPTED = "accepted"
    REJECTED = "rejected"
    EXPIRED = "expired"
    CONSUMED = "consumed"


@dataclass(frozen=True)
class IssuedCode:
    channel: str
    salt: bytes
    digest: bytes
    issued_at: datetime


@dataclass
class Challenge:
    challenge_id: str
    subject_id: str
    expires_at: datetime
    attempts_remaining: int
    codes: list[IssuedCode]
    consumed_at: datetime | None = None


def digest_code(code: str, salt: bytes) -> bytes:
    return hashlib.pbkdf2_hmac("sha256", code.encode(), salt, 210_000)


def issue_code(channel: str, now: datetime) -> tuple[str, IssuedCode]:
    code = f"{secrets.randbelow(1_000_000):06d}"
    salt = secrets.token_bytes(16)
    issued = IssuedCode(channel, salt, digest_code(code, salt), now)
    return code, issued


def start_challenge(subject_id: str, now: datetime) -> tuple[Challenge, str]:
    sms_code, issued = issue_code("sms", now)
    challenge = Challenge(
        challenge_id=secrets.token_urlsafe(24),
        subject_id=subject_id,
        expires_at=now + timedelta(minutes=10),
        attempts_remaining=5,
        codes=[issued],
    )
    return challenge, sms_code


def add_email_fallback(
    challenge: Challenge, now: datetime
) -> tuple[Challenge, str | None]:
    if challenge.consumed_at or now >= challenge.expires_at:
        return challenge, None
    if any(item.channel == "email" for item in challenge.codes):
        return challenge, None
    if now < challenge.codes[0].issued_at + timedelta(seconds=30):
        return challenge, None

    email_code, issued = issue_code("email", now)
    challenge.codes.append(issued)
    return challenge, email_code


def verify(challenge: Challenge, candidate: str, now: datetime) -> VerifyResult:
    if challenge.consumed_at:
        return VerifyResult.CONSUMED
    if now >= challenge.expires_at or challenge.attempts_remaining <= 0:
        return VerifyResult.EXPIRED

    matched = any(
        hmac.compare_digest(digest_code(candidate, item.salt), item.digest)
        for item in challenge.codes
    )
    if not matched:
        challenge.attempts_remaining -= 1
        return VerifyResult.REJECTED

    # Persist this compare-and-set atomically before issuing a session.
    challenge.consumed_at = now
    return VerifyResult.ACCEPTED


now = datetime.now(timezone.utc)
challenge, sms_code_for_adapter = start_challenge("subject_7f3a", now)
```

The in-memory mutation is illustrative; production persistence must perform the attempt decrement and consumption with conditional writes or a transaction. Code generation and hashing should also live in a reviewed security component. The adapter receives the raw code only long enough to construct the message. Everything persisted is a digest.

At the HTTP layer, `POST /login/challenges` can always return a generic challenge receipt. `POST /login/challenges/:id/email-fallback` records the explicit channel switch. `POST /login/challenges/:id/verify` accepts the challenge ID and candidate code, then creates a session only after the atomic consume succeeds. These are application-owned example routes, not claims about an external API.

Deployment needs a compatibility plan. Add the challenge store and adapters first, deploy handlers with session creation disabled behind an internal flag, exercise synthetic accounts in each supported region, and then enable traffic gradually. Watch send acceptance by channel, fallback requests, verification outcomes, challenge expiry, throttles, and time from issuance to successful verification. Never attach destinations or codes to those metrics.

Test races, not just happy paths: two verify requests with the same correct code; SMS verification after email fallback; fallback requests at 29 and 30 seconds under this example policy; a resend concurrent with verification; expiry between code comparison and persistence; and a provider throttle. The 29/30-second boundary deserves a focused test. Imagine request A reads the challenge one millisecond before fallback becomes eligible, while request B reads it one millisecond after; at the same time, a delayed SMS verification reaches another worker. The expected result is not "whichever process writes last." Request A must receive the policy's not-yet-eligible result, request B may append one email credential exactly once, and the verification may consume the challenge. If consumption wins, the email adapter must not receive a new send instruction. If the email transition wins first, the later correct SMS code may still consume the same challenge under this decision, and the email credential becomes unusable immediately afterward. Put those outcomes into store-level concurrency tests, not only controller mocks. Adapter contract tests should confirm idempotency and normalized error categories without calling a live service in the unit suite.

Race it deliberately.

## Why reject automatic fallback, and when is it still valid?

Automatic fallback looks attractive because the user does nothing. It also makes a claim the application usually cannot support: that elapsed time means the SMS failed. A message may be delayed after the send was accepted, so a timer can create two live delivery paths and train users to act on whichever credential arrives first. For this architecture, the user-visible "use email instead" action is a cleaner state transition and a better consent point.

Stick with automatic fallback when the environment is tightly controlled, duplicate delivery is acceptable, and measured latency supports a documented timer. It can also fit accessibility workflows where requiring another interaction creates a known barrier. Even there, retain one challenge, one shared attempt budget, atomic consumption, and channel-labeled telemetry. Your mileage may vary by geography.

This decision does not make SMS and email equivalent factors. It makes their relationship explicit. If the threat model requires two independent factors, a passwordless SMS-or-email challenge is the wrong architecture; require two distinct proofs instead. If account recovery has stronger identity checks than ordinary login, keep it out of this fallback route rather than letting a convenient email button inherit recovery authority.

The durable choice is the state machine, not the messaging vendor. Once the invariants are encoded behind narrow adapters, an Express.js team can change delivery services, tune policy by region, and investigate gaps without rewriting authentication semantics.

## Further reading

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/glossary/what-sms-character-limit
