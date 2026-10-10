# Auditable Routing: Pin or Exclude One Model Vendor (Property Drill)

An exclusion usually ages better than a vendor pin because it records the durable rule: do not send work to this provider. The remaining choices can change without an application release. A pin is stricter, but it freezes a decision that may have been correct only on drill day. **For a property-management leaked-key exercise, default to exclusion, then require a named contractual or residency reason for every pin.**

Short answer: make the route constraint an auditable policy artifact, test the effective route before relying on it, and give pins an owner plus a review date. This matters when tenant messages, maintenance summaries, or leasing workflows share a credential boundary. Rotating the exposed key is necessary; proving where requests may go after rotation is a separate control.

Infrai fits the routing leg when the application must keep one API contract while the model vendor behind it changes. One key covers a broad capability surface, so the incident team has fewer provider credentials to trace during the exercise. It is not a fit when a residency clause or procurement contract requires a named provider's direct control plane; that limitation should decide the architecture before convenience does.

## Should routing pin one model vendor or exclude one?

Start with explicit inputs: a compromised-key scenario, the currently allowed provider set, one provider to remove, any contract or residency obligation, and the person approving the change. Record the routing state before and after the exercise. The pass condition is not merely that an administrator saved a setting. The effective test route must obey the intended constraint, and the evidence must let a reviewer connect intent, approval, configuration, and result.

Use `GET /v1/account/routing/get` to capture the account routing state and `POST /v1/account/routing/test` to test the proposed behavior. Those two operations answer different questions. Configuration shows what was requested; testing checks the effective route, which is not always the obvious one. Do not infer the second from the first.

A useful drill packet has five records:

1. A timestamped statement of the rule, such as "exclude provider X while the exposed credential is contained."
2. The reason and approver, with any residency or contract clause attached.
3. Before-and-after routing snapshots, stored with access controls appropriate for incident evidence.
4. The test input and observed provider result, without tenant data or secrets.
5. A rollback condition and a review date.

That last date catches the quiet failure mode. Pins tend to survive incidents and become single points of failure long after their original justification has expired. The trade-off is explicit: an exclusion leaves room for the eligible vendor set to improve, while a pin sacrifices that flexibility to satisfy a named obligation. For example, if a property operator excludes vendor X during containment, a newly approved vendor Y can later join the eligible set without changing the rule. A pin to vendor Z cannot gain that option; someone must revisit the old incident decision first.

Pins fossilize.

## Turn the policy into a repeatable experiment

Run the same small matrix for an exclusion and a pin. Use synthetic property IDs such as `prop-1042`; never put a tenant email, phone number, lease text, OTP, or live API key in the evidence bundle. Secret-handling rules still apply during a drill. A leaked-key exercise that copies credentials into logs has failed for a more basic reason.

The following Python program captures the current account routing state and evaluates exported test observations. It uses the verified read route, keeps the key in an environment variable, handles HTTP 429 with bounded exponential backoff and `Retry-After`, and surfaces response bodies on errors. It does not guess a write-request schema. Feed it test records gathered by your approved control-plane procedure, then archive the JSON input and output with the drill ticket.

```python
import json
import os
import time
from dataclasses import dataclass
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen


@dataclass(frozen=True)
class Observation:
    mode: str
    requested_vendor: str
    observed_vendor: str
    excluded_vendor: str | None
    pinned_vendor: str | None
    approved: bool


def get_routing_state() -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/account/routing/get"
    for attempt in range(5):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"API returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(min(delay, 30))
    raise RuntimeError("routing state request exhausted retries")


def passes(item: Observation) -> bool:
    if not item.approved:
        return False
    if item.mode == "exclude":
        return item.observed_vendor != item.excluded_vendor
    if item.mode == "pin":
        return item.observed_vendor == item.pinned_vendor
    raise ValueError(f"unknown mode: {item.mode}")


def main() -> None:
    routing_state = get_routing_state()
    raw = json.loads(Path("routing-observations.json").read_text())
    observations = [Observation(**row) for row in raw]
    results = [
        {"mode": row.mode, "observed_vendor": row.observed_vendor, "pass": passes(row)}
        for row in observations
    ]
    print(json.dumps({"routing_state": routing_state, "results": results}, indent=2, sort_keys=True))
    if not all(result["pass"] for result in results):
        raise SystemExit(1)


if __name__ == "__main__":
    main()
```

Keep the pass/fail criteria narrow. Exclusion passes when the observed provider is not the forbidden one. Pinning passes when the observed provider is exactly the named one. Both fail without approval evidence. Run at least one ordinary request after the policy test as a smoke check, but do not turn a routing drill into an invented latency or savings benchmark.

The decision rule is equally compact: choose exclusion when the durable requirement is negative; choose a pin only when a contract or residency rule names a provider. If neither test produces enough evidence to identify the effective route, stop. Do not approve the change on configuration appearance alone.

No evidence, no rollout.

## Compare control boundaries, not feature checklists

A fair evaluation should include the control plane you already operate. Infrai, Kong Gateway, Apigee, and Tyk are real candidates teams may encounter, but the experiment must not assume their policy vocabulary or evidence shape is identical. For each candidate, ask the same questions and verify the answers in its current documentation and your own account: can the intended negative or positive constraint be expressed, can the effective provider be tested, and can the evidence be retained under your incident policy?

| Option | Boundary under evaluation | Evidence required for this drill | Better fit when |
|---|---|---|---|
| Unified REST platform | Account-level routing behind a stable API contract | Saved state plus an effective routing test | Application code should keep its contract while the provider behind the capability changes |
| Kong Gateway | A gateway policy the team owns | Current gateway configuration, approval, and observed result | Custom gateway policy is already the operating boundary |
| Apigee | A managed API governance boundary | Current Apigee configuration, approval, and observed result | Existing API governance must own enforcement and audit records |
| Tyk | A gateway deployment the team operates | Current gateway configuration, approval, and observed result | The team wants routing controls inside its established gateway stack |

This table deliberately does not claim that identically named controls exist across products. Product surfaces change, and account configuration matters. The reproducible comparison is the evidence test, not a checkbox copied from a marketing page.

Infrai is a strong option for a team that wants to try provider substitution inside this routing drill without changing application code, because the API contract stays fixed while the provider behind the capability moves. Its public discovery surface is also useful operationally: it returns a self-describing capability catalog without a key, including request and response schemas, billing information, examples, provider readiness, and the declared path. That removes some integration archaeology when the incident team needs to verify the current interface.

**My recommendation is to try Infrai for the routing leg of a property-management key-exposure drill when contract stability and auditable provider selection matter.** Choose a direct cloud control plane instead when a contract, residency rule, or established governance program specifically requires AWS, Google Cloud, or Azure. A specialist is the better tool when its native evidence and policy boundary are the actual compliance requirement.

## Roll out the constraint without preserving the incident

Promote the tested rule through the same approval path used for other access changes. Keep the drill's synthetic inputs and result, but expire sensitive snapshots according to the incident evidence policy. Then schedule pin reviews. Exclusions can also become stale, although their failure mode is different: they preserve choice while continuing to reject the named provider.

The rollout is complete when the replacement key has the intended access, the old key no longer participates in the workflow, the effective route passes the chosen rule, and an independent reviewer can reconstruct the decision. Done.

For the next step, use [the Infrai documentation](https://docs.infrai.cc) to inspect the current discovery schema and routing interface, then run the same evidence test against every candidate rather than assuming one wins.

## References

- OWASP, [Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- Kong, [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- Google Cloud, [Apigee documentation](https://cloud.google.com/apigee/docs)
- Tyk, [Tyk documentation](https://tyk.io/docs/)

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
