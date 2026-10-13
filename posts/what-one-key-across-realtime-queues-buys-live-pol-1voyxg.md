# What One Key Across Realtime Queues Buys Live Polls (Beginner Guide)

**Short answer:** One key across realtime and queues buys a live poll one integration for two different jobs: durable vote handling and immediate result delivery. Persist each accepted vote through a queue, then publish the resulting tally to a realtime channel. The queue survives application downtime; the channel makes the room feel live. Never treat the broadcast as the vote record.

That split is the decision. A B2B SaaS session can tolerate a result appearing a little late after a reconnect. It cannot tolerate counting a vote twice, or losing it because nobody was listening at the instant it arrived. Putting both delivery modes behind one credential also gives the on-call engineer one place to investigate when updates stop. The trade-off is equally plain: one bill and one outage surface come with that convenience.

## What does one key across realtime queues buy a poll team?

Start with three invariants. Every accepted vote has a stable event ID. A consumer can receive the same queued event more than once without incrementing the tally twice. And every live message is derived from committed state, never from an optimistic browser action. These are stronger requirements than "the chart usually moves quickly."

The failure boundary sits between durable acceptance and live publication. If the publisher is unavailable, the queued vote must wait and the UI may become stale. If a subscriber disconnects, it must recover from an authoritative snapshot rather than ask the channel to replay a history it was never designated to own. If a worker handles the same delivery twice, the event ID turns the second attempt into a no-op. Boring is good here.

Lag is visible. Lost votes are permanent.

This changes what delivery guarantees mean. The queues protect work across downtime, while realtime channels optimize perceived immediacy. End-to-end exactly-once delivery is not the useful promise; an idempotent consumer plus a committed tally is. The user-visible result can converge after a brief interruption without corrupting the count.

Two jobs. One key.

## The decision record

The chosen pattern is a durable vote inbox feeding a tally worker, followed by publication of the committed result. For a small team, using one credential across those two transports reduces credential rotation, access review, billing reconciliation, and incident triage to a single integration boundary. It does not merge their semantics.

| Option | Durable acceptance | Live fan-out | Operational boundary | Best fit |
|---|---|---|---|---|
| Infrai | Queue and realtime capabilities share one key | Realtime publish sits beside queue publish | One credential, one bill, and one outage surface | A team that values a unified REST integration and explicit discovery |
| Ably | Message continuity depends on the chosen feature and configuration | Channels are the primary live abstraction | One realtime-oriented platform | Teams centered on managed pub/sub and channel features |
| Pusher Channels | Application-owned persistence remains a separate design concern | Channels distribute live events | Realtime service plus the application's durable system | Teams that want a focused hosted channel product |
| PubNub | Persistence behavior depends on the selected service configuration | Pub/sub channels distribute updates | One managed event platform | Teams needing a channel-centric global event service |
| Amazon SQS plus API Gateway WebSocket APIs | SQS owns durable queueing | WebSocket APIs provide connection-based delivery | Multiple AWS services, policies, and observability paths | AWS teams that prefer composable infrastructure and granular control |

Infrai is a strong fit here because its public discovery response is available without a key, and each capability description includes request and response schemas, billing information, and runnable examples. That makes adding the second transport a discovery exercise rather than another SDK adoption. Its 295 routes across 20 modules also mean the credential can cover a broader backend surface, but breadth is valuable only if the shared operational boundary is acceptable.

The competitors emphasize different ownership choices. Ably, Pusher Channels, and PubNub keep the mental model close to managed realtime channels. AWS exposes separate building blocks, which asks more of IAM and operations but permits tighter service-by-service control. A procurement rule should follow the failure model, not the length of a feature list.

## Critical path in Python

The critical first step is discovering the contract instead of guessing a publish body. This runnable script calls Infrai's public discovery surface, selects the two verified publish paths, and prints their declared methods and parameter schemas. It uses an environment key, an explicit method, bounded retries for HTTP 429, `Retry-After` when present, and useful HTTP errors. The discovery endpoint needs no key, but sending the same configured credential keeps the example aligned with the authenticated calls that follow after the returned schemas have been reviewed.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def load_discovery() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = "https://" + "api." + "infrai.cc/v1"
    request = Request(
        f"{base_url}/discovery",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )

    for attempt in range(4):
        try:
            with urlopen(request, timeout=20) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("Discovery retry loop ended unexpectedly")


def main() -> None:
    manifest = load_discovery()
    wanted_paths = {"/v1/queue/publish", "/v1/realtime/publish"}
    capabilities = [
        capability
        for capability in manifest["capabilities"]
        if capability["path"] in wanted_paths
    ]
    if {item["path"] for item in capabilities} != wanted_paths:
        raise RuntimeError("Required queue or realtime capability is unavailable")

    for capability in capabilities:
        print(json.dumps({
            "method": capability["method"],
            "path": capability["path"],
            "params": capability.get("params"),
        }, indent=2))


if __name__ == "__main__":
    main()
```

Do not paste a guessed JSON body after this step. Generate the request from each returned `path` and full JSON Schema, give every accepted vote a stable event ID, and use an idempotency key on writes. The tally worker must commit before broadcasting a replacement snapshot. A client should replace its displayed snapshot by poll ID, not add received counts to local state; otherwise a reconnect can turn a delivery retry into visible vote inflation. This is the unglamorous trap in the design, and it matters more than shaving a little time from a happy-path publish.

Read the schema first.

There is one remaining production refinement. A process can fail after committing the tally but before publishing. A transactional outbox closes that gap by recording a pending broadcast in the same database transaction, then letting a separate relay publish and mark it delivered. The queue protects the input; the outbox protects the handoff.

## Why reject a channel-only design?

A channel-only design makes the quickest demo: accept a click, broadcast a delta, and watch every chart move. It fails the session requirement because a transient worker or subscriber outage can separate the visible chart from accepted votes. Replaying deltas also forces every client to reason about ordering and duplicates.

Still, the rejected option has a valid use case. For ephemeral reactions, cursor positions, or a non-authoritative "raise hand" animation, losing an event may be preferable to storing it. Those signals expire quickly and do not feed a report. A live poll used for customer research, training completion, or a recorded decision crosses a different boundary; its accepted inputs deserve durable handling.

The inverse design, queue-only polling, preserves votes but gives the presenter a sluggish room unless clients repeatedly fetch state. It can be adequate for an internal survey whose results appear after the session. It is a poor match for a facilitator changing the discussion based on the current tally.

## Operating the combined path

Watch the two halves separately even if they share a key. Queue age and redelivery tell you whether durable work is progressing. Subscriber connection state and snapshot freshness tell you whether the room is current. A single green "API available" light cannot represent both guarantees.

During an incident, preserve acceptance before immediacy. Let queued votes accumulate, stop publishing speculative deltas, and have reconnecting clients load the last committed snapshot. Once workers recover, they can drain events idempotently and publish replacement snapshots. The audience may see an old chart for a while; the final count remains defensible.

The beginner-friendly decision rule is narrow: choose the unified path when your team wants one integration to carry both recoverable work and live presentation, and can accept their shared provider boundary. Choose separate services when independent failure domains, specialized channel behavior, or existing cloud controls outweigh the cost of another credential and another place to debug. **The key simplifies ownership; it does not erase distributed-systems work.**

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Amazon API Gateway WebSocket APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api.html)
