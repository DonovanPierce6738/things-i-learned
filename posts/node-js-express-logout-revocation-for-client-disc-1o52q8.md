# Node.js Express Logout Revocation for Client Disconnects (Realtime Presence Semantics)

A logout endpoint must revoke the session first, then fan out a disconnect command to every realtime connection bound to that session. Closing only the browser's current socket leaves other tabs, devices, and reconnect races alive. For a customer-support editor with collaborative cursors, the production rule is stricter: remove authorization immediately, mark presence offline only after the server has accounted for every connection, and reject any late cursor frame that carries the revoked session generation.

**Short answer:** keep a small, expiring reverse index from session ID to connection IDs. On logout, atomically advance the session generation, publish a revocation event, close matching connections with an application-defined WebSocket close code, and make every message handler recheck the cached generation. The dominant cost is retention, not the close frame: connection-index entries, revocation markers, and presence leases accumulate with sessions and devices.

## What does forced logout actually cost?

Start with the bill. If there are `S` active sessions, `C` concurrent connections, and `R` recently revoked sessions retained for the reconnect window, storage is proportional to `C + R`, while logout work is proportional to the connections attached to one session. A scan of all live sockets turns that into `O(C)` work per logout. The reverse index changes it to `O(k)`, where `k` is the number of connections for that session. That difference matters more than shaving bytes from a close payload.

For cursor presence, the index needs only a session identifier, connection identifier, user identifier, document identifier, generation, and expiry. Do not retain cursor coordinates in the revocation record. Coordinates change rapidly and are useful only while the presence lease is live; mixing them into a durable audit trail multiplies writes and keeps sensitive interaction data longer than the logout problem requires.

A concrete capacity estimate is more honest than a vague claim. With 40,000 concurrent connections and an estimated 160 bytes of logical index data per connection, the raw index is about 6.4 MB before datastore overhead, replication, allocator slack, and secondary indexes. Measure those multipliers in the chosen store. A 24-hour revocation window for 300,000 daily session terminations at an estimated 80 logical bytes each adds about 24 MB raw. Those figures are arithmetic examples, not benchmarks; substitute serialized record sizes from production samples.

The change that moves the dominant term is bounded retention. Expire connection mappings shortly after their lease, and expire revocation markers after the maximum credential and reconnect lifetime. Keeping markers forever does not make logout safer once no accepted credential can name that generation. It does make deletion requests and incident review harder.

## How should Node.js Express force a client disconnect on logout?

A WebSocket close handshake is connection-scoped. Logout is identity-scoped. One support agent may have the same ticket open in two tabs, a background window may reconnect after losing Wi-Fi, and a mobile handoff may race the HTTP response. The current request does not reliably identify all of those transport objects.

There is another sharp edge: ordering. If the server sends close commands before committing revocation, a disconnected client can reconnect through a different worker while the old session still validates. Commit the authorization change first. Delivery of the close command is then an optimization for fast cleanup, while message-time authorization is the security boundary.

Ordering wins.

This distinction also protects presence accuracy. A cursor should not disappear merely because one of three connections closed, and it must not remain visible because a dead connection missed a broadcast. Track presence as a lease derived from the user's valid connections in that document. After revocation, all matching leases become ineligible; ordinary network loss still waits for the short lease timeout to avoid flicker.

Otherwise, presence lies.

Keep the states separate:

| State | Source of truth | Retention | Effect |
|---|---|---:|---|
| Session generation | Authentication store | Credential lifetime | Accept or reject messages |
| Connection membership | Ephemeral reverse index | Connection lease plus grace | Locate transports to close |
| Cursor presence | Document-scoped lease | A few heartbeat intervals | Render or remove a cursor |
| Security audit | Append-only event | Policy-defined | Explain who revoked what |

**A close event is evidence of transport cleanup, not proof of logout.** Proof comes from a later authorization check rejecting the old generation.

## The revocation path

The HTTP handler, realtime gateway, and shared store need a narrow contract. The example below is Python because the state transition is easier to inspect without framework plumbing; a Node.js implementation should preserve the same ordering and atomicity. The store methods represent conditional, atomic operations rather than a particular database.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Revocation:
    session_id: str
    new_generation: int
    connection_ids: tuple[str, ...]


class SessionStore(Protocol):
    async def revoke_and_list_connections(self, session_id: str) -> Revocation: ...


class RevocationBus(Protocol):
    async def publish(self, event: Revocation) -> None: ...


async def logout(session_id: str, store: SessionStore, bus: RevocationBus) -> None:
    # This operation increments the generation and reads its connection set atomically.
    event = await store.revoke_and_list_connections(session_id)
    await bus.publish(event)
```

Each gateway consumes that event and closes locally owned connection IDs. Use an application-defined WebSocket close code in the 4000-4999 range, as RFC 6455 reserves that range for private use. The client may show a signed-out state, but it must not decide whether the credential remains valid.

At message ingress, compare the connection's captured generation with the current session generation. A small cache is reasonable only if revocation invalidates it and the acceptable stale interval is explicit. Cursor frames are high-volume, so fetching durable session state for each frame can become the dominant load. One workable compromise is a gateway-local generation cache, push invalidation on logout, and a short maximum cache lifetime. If the invalidation bus stalls, expiry limits the authorization gap. This is a deliberate trade-off: tighter accuracy costs more reads or more dependable invalidation.

Publishing can fail after the revocation commits. Return logout success because the credential is already invalid, record the publication failure, and retry through a durable outbox or equivalent commit-coupled mechanism. Never roll the session back merely to make connected screens look tidy. Late messages will fail their generation check.

## Production checks that catch the quiet failures

Test logout as a distributed transition, not as an endpoint status code. Open two connections for one session on different gateway workers, log out through a third process, and assert that both close. Then reconnect with the old credential during publication delay and assert rejection. Repeat after duplicating and reordering the revocation event; consumers must be idempotent.

Presence deserves its own assertions. Two tabs under one valid session should produce one user presence with two contributing connections. Closing one tab must not erase the cursor if the other still renews its lease. Revoking the session must make both contributions ineligible, even when one gateway is partitioned until the lease expires. Test a second valid session for the same user as well: revoking one session should not silently sign out every device unless the product policy explicitly says so.

Observe counts and age, not message contents. Useful signals include revocation-to-last-close latency, rejected frames by reason, orphaned connection-index entries, presence leases surviving past expiry, outbox backlog age, and reconnect attempts using a stale generation. Alert on sustained age or divergence. A single close failure is often repaired by lease expiry; a growing backlog means the promised logout boundary is drifting.

Deployment needs compatibility discipline. Add generation checks before emitting revocation events, so older gateway instances cannot accept a newly invalid session. Roll out event consumers next, then enable logout publication. During rollback, keep the store schema and generation semantics readable by both versions.

Do not log raw session tokens, cursor text, or message bodies. Hash or otherwise pseudonymize correlation identifiers according to the incident-response need, and keep the mapping under separate access controls. Compliance pressure is useful here: it forces the audit record to answer the security question without becoming a shadow transcript of the support conversation.

## Where should retention stop?

Retain the minimum state that can still change a decision. Connection membership ends after disconnect plus a short cleanup grace. Cursor coordinates end with their presence lease. Revocation markers end after every credential derived from that session generation is unusable. Audit events follow the organization's security and regulatory policy, with access and deletion rules stated in advance.

Then stop keeping it.

The cost is diagnostic precision. After ephemeral mappings expire, an incident responder may be able to prove that session generation 12 was revoked and that stale frames were rejected, but not reconstruct the exact cursor position or every transient connection that existed minutes earlier. That is usually the right loss. If exact reconstruction is a requirement, name it explicitly, isolate that data from the hot revocation path, and accept the storage, privacy, and deletion burden rather than retaining everything by accident.

The final acceptance test is plain: after logout commits, no frame authenticated with the old generation can mutate the editor; all discoverable transports close promptly; and cursor presence converges when leases expire even if a close notification is lost. That gives logout a security boundary, presence an honest convergence model, and retention a defensible end.

## Further reading

- RFC 6455, The WebSocket Protocol: https://www.rfc-editor.org/rfc/rfc6455
- OWASP Session Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- W3C WebRTC 1.0: https://www.w3.org/TR/webrtc/
- MDN, WebSocket close event: https://developer.mozilla.org/en-US/docs/Web/API/WebSocket/close_event
