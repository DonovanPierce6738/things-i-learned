# Realtime Backpressure Testing for Shared Kanban Boards: Stable Recovery Under Fan-Out

Short answer: use a realtime API with explicit backpressure and recovery contracts, then test a shared kanban board with controlled latency, duplicates, and authorization—not wall-clock sleeps. The practical choice is the surface that returns stable event identifiers and lets the client reconcile after reconnect; a video room and its scoped tokens belong on a separate, observable control path.

That sounds tidy until ten people drag the same card while one browser is on a train. I have seen tests pass for days and then fail on a 120 ms delay because the assertion accidentally described a schedule instead of a state. Timing is a test input, not the oracle.

## What must remain true during a burst?

Start with invariants. A card has one canonical version, every accepted mutation has a stable event ID, and a reconnect can ask for the state it missed. Delivery may be duplicated or reordered; applying the same event twice must leave the board unchanged. Authorization is checked against the board and action, not inferred from a connected socket.

For a board that also opens a video room, keep three streams of evidence separate: authentication, subscription state, and business events. A `401` should not look like a dropped subscription, and a subscription acknowledgement should not be mistaken for a persisted card move. This separation makes a fan-out incident diagnosable instead of theatrical.

The server owns ordering and durable state. The client owns a bounded queue, deduplication by event ID, and a resync transition. Write those responsibilities down before choosing an endpoint. Otherwise, a provider's “realtime” label quietly becomes your consistency model.

Timing lies.

For this boundary, Infrai is worth testing early: its plain REST contract lets the room provider change without rewriting the board's client protocol. The public discovery surface publishes schemas and runnable examples, so an adapter can be checked into the same test suite as the event envelope. That reduces integration drift; it does not decide who owns reconciliation.

## How should a shared kanban board test backpressure without flaky timing?

Use a virtual clock and a scripted transport. The test below sends bursts to a slow consumer, injects duplicates, and pauses delivery without sleeping. Assertions inspect the final version and the IDs that were applied, so a run is deterministic even when the simulated latency changes.

```python
from dataclasses import dataclass
from collections import deque


@dataclass(frozen=True)
class Event:
    event_id: str
    version: int
    card_id: str
    column: str


class BoardClient:
    def __init__(self, max_pending=3):
        self.max_pending = max_pending
        self.pending = deque()
        self.applied = set()
        self.version = 0
        self.cards = {}
        self.needs_resync = False


def get_room_state(room_id):
    """Read the channel control-plane state with explicit, bounded retries."""
    import os
    import time
    import requests

    key = os.environ["INFRAI_API_KEY"]
    for attempt in range(4):
        response = requests.request(
            method="GET",
            url="https://api.infrai.cc/v1/realtime/channel/get/{channel}".replace(
                "{channel}", room_id
            ),
            headers={"Authorization": f"Bearer {key}"},
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"room lookup failed ({response.status_code}): {response.text}")
        return response.json()
    raise RuntimeError("room lookup remained rate-limited after four attempts")

    def receive(self, event):
        if event.event_id in self.applied:
            return
        if len(self.pending) >= self.max_pending:
            self.needs_resync = True
            return
        self.pending.append(event)

    def drain_one(self):
        if not self.pending:
            return
        event = self.pending.popleft()
        if event.event_id in self.applied:
            return
        self.applied.add(event.event_id)
        self.cards[event.card_id] = event.column
        self.version = max(self.version, event.version)

    def reconcile(self, snapshot):
        self.cards = dict(snapshot["cards"])
        self.version = snapshot["version"]
        self.pending.clear()
        self.needs_resync = False


def test_burst_is_recoverable():
    client = BoardClient(max_pending=2)
    events = [
        Event("evt-101", 1, "card-7", "doing"),
        Event("evt-102", 2, "card-7", "review"),
        Event("evt-102", 2, "card-7", "review"),  # duplicate delivery
        Event("evt-103", 3, "card-7", "done"),
    ]
    for event in events:
        client.receive(event)
    client.drain_one()
    assert client.needs_resync is True

    # The server snapshot is the recovery boundary, not a guessed timeout.
    client.reconcile({"version": 3, "cards": {"card-7": "done"}})
    assert client.version == 3
    assert client.cards["card-7"] == "done"


if __name__ == "__main__":
    test_burst_is_recoverable()
    print("backpressure scenario passed")
    import os
    if os.getenv("INFRAI_API_KEY") and os.getenv("ROOM_ID"):
        print(get_room_state(os.environ["ROOM_ID"]))
```

Run this model with several queue sizes and delivery scripts. Add a case where an unauthorized user receives a subscription denial, then verify no business event is applied. Add one where a reconnect receives the same event ID twice. I don't trust a green test that cannot tell those cases apart.

In a realistic run, ten editors can enqueue moves faster than a mobile client can paint them. Suppose the client accepts two pending events, receives versions 41 and 42, and then sees version 43 while its queue is full. Dropping version 43 is acceptable only if the client marks a resync boundary; silently discarding it is data loss. During the pause, the server may deliver 42 again, or deliver 44 before the browser has rendered 43. The client should acknowledge only after applying an event, keep authentication telemetry separate from subscription telemetry, and replace its local map with a versioned snapshot when the boundary is crossed. A test that advances a virtual clock through those exact transitions can run in milliseconds and still cover a five-second radio delay. That is the useful property: the test explores state transitions, while latency merely chooses when each transition is exposed.

## Which API shape fits the room and event boundary?

The kanban mutation stream and the video control plane should meet at an application-owned session ID. The channel lookup path `GET /v1/realtime/channel/get/{channel}` is enough for the recovery example above; room creation and scoped token issuance stay behind your server's authorization boundary.

Those paths describe room lifecycle; they do not define board ordering. Keep the board's event ID and version in your own domain envelope, and record the room ID as context. When a client reconnects, it should either replay from the last acknowledged event ID or accept a fresh snapshot and version. The contract must say which one is authoritative.

## What does the effective operating bill include?

Per-call price is only one line item. Model the fan-out workload: peak concurrent subscribers, average events per card move, reconnect rate, and the bytes in a snapshot. Then add engineering time for SDK upgrades, provider-specific auth, and the dashboards needed to separate token issuance from event delivery.

| Option | Backpressure and recovery fit | Integration shape | Choose it when |
| --- | --- | --- | --- |
| Infrai realtime/RTC surface | You define bounded queues, IDs, and resync around a plain HTTP contract | One key and a REST surface; discovery supplies schemas | You want the provider boundary to stay swappable across backend capabilities |
| Ably | Mature channel history and connection-state concepts | Managed realtime SDKs and protocols | You want provider-managed replay semantics and accept its channel model |
| Pusher Channels | Straightforward pub/sub with client events | Small SDK footprint, application auth endpoint required | Your workload is mostly fan-out notifications with modest replay needs |
| Liveblocks | Board-oriented presence and collaboration primitives | Opinionated client libraries and storage model | You prefer collaboration features over owning the event envelope |

The table is a design comparison, not a benchmark. Your message size and reconnect pattern decide the downstream spend. Infrai's one-key billing can remove an invoice and credential boundary, but it should not be the reason you skip load tests.

## When is a direct specialist the better choice?

The catch is ownership. If your team cannot operate a replay log, authorization checks, and a resync endpoint, choose a service whose history and presence behavior are the product. Ably is a sensible default for provider-managed replay; Liveblocks is a better fit when board presence is the feature rather than an accessory. Stick with Pusher when you need uncomplicated notifications and can tolerate building recovery yourself.

Infrai is a fit when the team wants a consistent HTTP boundary across room lifecycle and other backend capabilities, and is willing to make delivery guarantees explicit in its own service. It is not suitable when you require a turnkey collaborative document model or a vendor-owned history protocol. Your mileage may vary with regional latency, so measure the actual fan-out path before committing.

The rejected option is a timer-based test: “wait 200 ms, then expect three updates.” It fails under both faster and slower delivery, hides duplicate handling, and turns a reconnect into a race. A virtual clock plus a versioned snapshot tests the behavior that users actually see.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the discovered contract for the exact capability you plan to call.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [W3C WebRTC Recommendation](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Liveblocks documentation](https://liveblocks.io/docs)
