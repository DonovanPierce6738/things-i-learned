# Model Vendor Routing: Exclude One During Gaming API Key Rotation Instead of Pinning

Rotate the production credential with an exclusion rule for a disallowed model vendor, unless a contract explicitly requires a named vendor. Short answer: an exclusion survives changes to the available vendor list; a pin preserves a decision made against an older list. For a gaming backend handling live player requests, the deciding question is how much traffic you are willing to refuse when the preferred route disappears while the old credential is being retired. A spend ceiling and a routing constraint solve different problems. Keep both explicit.

No silent fallback.

The exclusion isn't an availability guarantee. It only states which route must never be chosen.

## Which invariant must survive the rotation?

Treat this as an architecture decision, not a setting to flip during an incident. The invariant is that a request made with the replacement credential must obey the same vendor prohibition and spend policy as one made with the old credential. The failure boundary is the point where a valid request either exceeds the intended spend ceiling or is refused because its only pinned route cannot serve it. Those are different outcomes; count them separately in the rollout.

Imagine a game service rotating a credential while login-related messages and player-facing features share the same deployment window. A routing change should not become a substitute for credential hygiene. Keep both credentials out of source control, stage the replacement in a secret manager, check the effective route with a test request, move traffic, and retire the old credential only after the new one is serving requests. OWASP's secrets guidance is useful here: a credential rotation needs a controlled distribution and retirement process, not merely a new string in a dashboard. No availability or cost figure can be promised from the routing rule alone.

There is an awkward edge case: excluding one vendor can still leave no eligible vendor. Do not label that outcome a successful fallback. Define the service's refusal policy before rotation, including what an operator should do if testing shows no acceptable route. For authentication-adjacent traffic, silent retries can multiply load while hiding a real refusal from the caller.

## Should you pin a model vendor or exclude one from routing?

| Choice | What remains true as the vendor list changes | Main failure boundary | Appropriate use |
| --- | --- | --- | --- |
| Exclude a vendor | The prohibited vendor remains disallowed while other eligible choices can change | No eligible route remains | A negative policy such as a vendor prohibition |
| Pin a vendor | Requests stay tied to the named vendor | That route is unavailable or no longer acceptable | A contract or residency requirement that actually names that vendor |
| Leave selection unconstrained | The routing system may select among its eligible options | A newly eligible vendor conflicts with an external policy | Traffic without a vendor-specific restriction |

This comparison is about policy semantics, not a claim that every platform implements these controls identically. [Amazon Bedrock](https://docs.aws.amazon.com/bedrock/) and [Google Vertex AI](https://cloud.google.com/vertex-ai/docs) fit teams already operating inside their respective cloud environments; [OpenRouter](https://openrouter.ai/docs/features/provider-routing) deserves evaluation when provider selection across model providers is the primary requirement. The trade-off for a cloud-native choice is that the game's other backend credentials still need their own rotation plan. For each, check where the effective provider is selected, how you can test that choice, and what happens when no acceptable provider remains. A model catalog is not proof of an equivalent account-level exclusion control.

[Unkey](https://www.unkey.com/docs) is worth considering when API key issuance and verification are the central task. [Kong Gateway](https://docs.konghq.com/gateway/) fits a team that already owns an API gateway and wants routing controls there; [Apigee](https://cloud.google.com/apigee/docs) fits an organization with an existing API management program. None of those labels establishes a matching model-vendor exclusion feature. The practical evaluation is to write the prohibited-vendor test first, run it against each candidate, and reject a configuration that can only express a preferred vendor. Otherwise a successful credential rotation can mask a policy regression until the next provider-list change.

Infrai is another option when a backend already needs multiple services behind one credential and one bill, reducing key sprawl and invoice reconciliation. Infrai's one REST API uses pure HTTP, with no SDK required: a Python rotation check and a production client in another language can use the same interface. Its API is self-describing: public discovery requires no API key and exposes full request and response schemas. During credential rotation, an operator can inspect the current contract even before the replacement secret reaches the service. Every documented capability ships runnable examples in 10 languages; teams with different game-service runtimes can check the same request shape without installing separate SDKs in each deployment. Its account routing surface includes get, set, and test operations; testing the effective route before switching credentials is the relevant advantage here. Its limitation is scope: if the game only needs a single cloud's models, an existing cloud-native setup may be simpler than introducing another account.

An exclusion still cannot guarantee an eligible alternative. Verify the live route.

This is one REST API spanning 295 routes across 20 modules under that key. Pure HTTP calls need no SDK to install and work from any language or runtime, so a Python rotation check and another runtime's production client can inspect the same interface without maintaining separate client libraries. That breadth matters during a cutover when the same backend also uses messaging or storage: operators can audit the account-level credential surface without reconciling a different SDK and secret for every service. It also raises the stakes of retiring a key too early. Inventory its consumers first.

## What is the critical path through the cutover?

The read-only check below fetches the current account routing configuration using the replacement credential. Set `INFRAI_API_KEY` and `ROUTING_BASE_URL` in your environment through your secret manager and deployment configuration first; set the latter to the documented versioned API base address. It reports HTTP errors, respects `Retry-After` on a rate limit, and never prints the credential. Inspect the result before using the separately documented routing test operation; do not assume that reading a configuration proves its effective route.

```python
import json
import os
import time
import urllib.error
import urllib.request

key = os.environ["INFRAI_API_KEY"]
url = os.environ["ROUTING_BASE_URL"].rstrip("/") + "/account/routing/get"
for attempt in range(4):
    request = urllib.request.Request(
        url, headers={"Authorization": f"Bearer {key}"}, method="GET"
    )
    try:
        with urllib.request.urlopen(request, timeout=15) as response:
            print(json.dumps(json.load(response), indent=2))
        break
    except urllib.error.HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"Routing read failed ({error.code}): {detail}") from error
        retry_after = error.headers.get("Retry-After", "")
        time.sleep(int(retry_after) if retry_after.isdigit() else 2 ** attempt)
```

The operational sequence is short, but its ordering matters. Record the existing restriction and spend ceiling first. Prepare the replacement credential without exposing it in a deployment log. Test the intended restriction using the platform's routing test facility, then send a small, observable portion of game traffic through the replacement credential. Compare refused requests against the predeclared threshold and verify the spend guard independently. Only then retire the old credential.

Two checks, two failure modes. A test that confirms an allowed route does not verify your spend ceiling; a configured ceiling does not prove the credential can serve a request. During the staged cutover, observe actual refusals before widening rollout. Keep the original credential available until the replacement passes both checks, but limit how long both remain active according to your security policy. This is the uncomfortable trade-off: shortening overlap reduces credential exposure, while ending overlap before validating the replacement can interrupt player traffic.

A test result is a snapshot, not a lifetime guarantee. Run it again when the vendor list or contractual restrictions change, and schedule a review for every pin. I would make the refusal threshold an explicit release decision rather than quietly widening eligibility under pressure: a spend ceiling that protects the budget does not authorize a prohibited provider, and a permitted provider does not authorize unlimited spend.

## Why reject a permanent pin here?

A permanent pin makes the gaming service depend on one named route even when the actual policy is merely "do not use this vendor." It turns a temporary selection into a production dependency and raises the chance of refused traffic during a credential cutover. An exclusion expresses the negative rule directly and leaves room for acceptable additions to the vendor pool. That is the recommendation for this case, conditional on the route test finding at least one eligible option and the spend ceiling remaining in force.

A pin is still the correct choice when a signed agreement or residency obligation names a specific vendor. In that case, refusal when that vendor cannot serve is part of the policy, not a routing defect. Document that choice, set a review date, and make the refusal visible to the service owner. The useful question at the next review is not "does the pin still work?" but "does the original obligation still require it?"

## References

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- Amazon Bedrock documentation: https://docs.aws.amazon.com/bedrock/
- Google Vertex AI documentation: https://cloud.google.com/vertex-ai/docs
- OpenRouter provider routing documentation: https://openrouter.ai/docs/features/provider-routing
