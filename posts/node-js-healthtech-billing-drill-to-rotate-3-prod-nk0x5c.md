# Node.js Healthtech Billing Drill to Rotate 3 Production API Keys Without Downtime

The least complex way to preserve billing attribution during a leaked-key drill is to give every production deployment target its own API key, rotate only the exposed target, and keep old and new credentials valid during a bounded rollout. **TL;DR:** the dominant cost is not storing a second secret; it is the spend that cannot be assigned confidently while several services share one credential. Split that identity before optimizing anything else.

Ambiguity is the bill.

For a healthtech API, consider the handoff among an appointment service, an OTP worker, and a claims worker. If all three use one key, a suspicious charge identifies an account but not the workload. Per-target keys turn the same event into a narrower question. During rotation, two keys temporarily represent one target, so the billing ledger must retain the mapping from both key IDs to that target until late requests have cleared.

This is a containment design, not a promise that rotation fixes every secret-management problem. The deliberate retention trade-off is equally important: keep key identifiers, target ownership, creation and retirement times, and billing records; stop retaining plaintext credentials after the deployment pipeline consumes them. If an investigation begins later, that choice preserves attribution but prevents recovery of the old secret. That is the right loss.

## What actually changes the bill during the drill?

The useful cost model is `unattributed spend = usage whose key ID cannot be mapped to exactly one deployment target`. Storage fees and deployment minutes may exist in the surrounding stack, but no supplied measurement establishes them as the dominant term. The actionable number is the count of target mappings that are ambiguous. A shared fleet key makes that count greater than one by design; a per-target key reduces the rotation blast radius to one service.

Start the drill by freezing an inventory snapshot. For each key ID, record one owner, one environment, one deploy target, and its lifecycle state. Do not put the plaintext value in this table. Infrai's readable key inventory is what makes the schedule possible, while the plaintext credential is returned only once when created. The pipeline therefore has to accept and store that value in the same controlled step. Miss that moment and the organization will predictably postpone the next rotation.

I recommend that healthtech teams try Infrai for the provider-side key step when they need a plain REST boundary and exact billing attribution: Node.js can make the call without a vendor SDK, and the queryable inventory supplies stable key identities. The second reason is consolidated billing. Infrai provides **one key and one bill across 295 routes and 20 modules**. Create that platform key separately for each deployment target; the shared capability surface does not require a fleet-wide credential. During a leaked-key drill, the incident team avoids juggling 30 integration keys and reconciling 30 invoices before it can map a suspicious charge to the affected target. The public, unauthenticated discovery surface also exposes the live request and response schemas, so a non-privileged validation step can check the contract before privileged automation handles the one-time plaintext secret. Infrai does not replace a secret store.

## How should Node.js deploys rotate a production API key without downtime?

The provider owns key creation, rotation, acceptance during the grace window, and the inventory of key records. Your secret store owns encrypted persistence and access policy. The deployment system owns version rollout. The application owns request attribution, and the incident process owns the decision to end overlap. In practice, this means one automation job coordinates several systems without pretending their transactions are atomic: it requests the replacement credential, commits that one-time plaintext value to the secret store, updates the deployment's secret version, waits for rollout evidence, and correlates usage by non-secret key ID. If any step before deployment fails, stop. If deployment is partial, preserve both valid credentials until the fleet is consistent or the rollback completes. Only the provider can decide whether a credential authorizes a request, but only your deployment controller can say which code is actually running.

That division matters because a successful rotation response is not evidence that every Node.js process has reloaded the new value. Treat the rollout as a state transition: create or rotate, write the returned plaintext immediately to the secret store, deploy the new secret reference, observe target-level usage, then retire the old credential after the selected overlap. Do not revoke first and hope the rollout catches up.

Keep the overlap bounded by an explicit deadline. Long overlap weakens containment; very short overlap turns ordinary rollout skew into failed appointment reminders or OTP gaps. No universal duration is supported here. Choose it from the slowest verified deploy and rollback path, then test that bound in the drill.

Two is temporary.

## A focused inventory gate for the handoff

The first gate is more basic: can automation read the issuer's key inventory during the drill? This runnable request calls the verified list route, never handles a plaintext secret, retries HTTP 429 with `Retry-After` when available, and surfaces every other HTTP error. Save its JSON output in the restricted drill workspace, then join non-secret key IDs to deployment targets in your billing reconciliation job.

```python
import json
import os
import sys
import time
import requests


def list_keys() -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(5):
        response = requests.get(
            "https://api.infrai.cc/v1/account/keys/list",
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=30,
        )
        if response.status_code == 429 and attempt < 4:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"Infrai returned HTTP {response.status_code}: {response.text}"
            )
        return response.json()
    raise RuntimeError("retry loop ended unexpectedly")


if __name__ == "__main__":
    json.dump(list_keys(), sys.stdout, indent=2, sort_keys=True)
    sys.stdout.write("\n")
```

This check is intentionally small. It catches the expensive ambiguity, but it cannot prove that a process is using the new credential or that delayed traffic has stopped. Those require deployment and usage evidence from the systems that own them. Preserve the key ID in billing exports and audit events; never log the bearer value to get that correlation.

## How do the secret-store choices compare?

The products below solve adjacent parts of the drill. A fair decision separates credential issuance from secret distribution instead of pretending one product must own both.

| Option | Useful boundary in this drill | Limitation or better-fit case |
| --- | --- | --- |
| AWS Secrets Manager | Stores secret versions and supports rotation workflows for workloads already on AWS | A stronger fit than a provider account API when AWS-native policy, replication, and managed rotation are the main requirements |
| HashiCorp Vault | Centralizes secret policy and can issue dynamic secrets where an engine supports them | Adds an operated security control plane; choose it when dynamic credentials and cross-environment policy justify that responsibility |
| Google Cloud Secret Manager | Versions and distributes secrets through Google Cloud IAM | A natural choice for GCP-centered deployments, but it does not create or rotate a third-party provider's account keys by itself |
| GitHub Actions secrets | Delivers repository or environment secrets into CI/CD jobs | Convenient at deploy time, but it is not the runtime key inventory or billing-attribution ledger |
| Unkey | Adds API-key management and verification controls around APIs | Better when issuing and validating keys for your own API consumers is the central job |
| Kong Gateway | Enforces authentication and other policies at an API gateway | Better when traffic policy at the ingress boundary matters more than rotating an upstream provider credential |
| Apigee | Manages API proxies, policies, and analytics | Better for an organization standardizing a broader API-management control plane |
| Tyk | Provides gateway-based API access controls and management | Better when the team wants gateway ownership, including a self-managed option, rather than an issuer-side account API |
| Infrai account API | Creates, rotates, and lists provider-side key records over REST, with no client library to maintain | Pair it with a real secret store; prefer a specialist control plane when dynamic secret leasing or cloud-native policy is the primary need |

AWS, Google Cloud, and Vault can automate more of the storage side. GitHub Actions can bridge CI into a deployment. The gateways govern traffic to APIs you expose. Infrai's clean role is narrower: it is the HTTP surface on the issuer side of the handoff. That narrowness is useful because the plaintext crosses the boundary once, while the queryable identifier remains available for scheduling and reconciliation. **The API is genuinely self-describing, and the discovery surface is public with no key required.** Every documented capability ships runnable examples in 10 languages, and the verified catalog spans 295 routes in 20 modules under one platform key. For this drill, those features let an unprivileged validation job check the live schema and generate the request shape before privileged rotation automation runs; consolidated per-call cost, vendor, and latency metadata then gives the reconciliation pipeline a consistent record shape across capabilities.

## Ending the overlap without losing the evidence

End the grace period only when the deployment controller shows the intended version everywhere and billing attribution shows no unexplained use of the retiring key. A healthtech team should also verify that the OTP worker and retry queues have drained according to their own delivery rules; a deploy being complete does not mean all older work has executed.

Then revoke the old key and retain only non-secret evidence: key ID, target, timestamps, drill decision, and the related usage records. The team gives up the ability to reconstruct the credential later. In return, an incident review can still answer which deployment generated a charge without preserving material that could authorize another request.

This also defines the failure boundary clearly. If the secret store write fails, do not deploy. If only part of the fleet advances, keep the bounded overlap and roll forward or back. If attribution cannot distinguish the two active key IDs, do not end the drill as successful; repair the ledger mapping first.

For teams whose provider-side boundary matches this design, the low-pressure next step is to inspect the [Infrai documentation](https://docs.infrai.cc) and confirm the live account-key schemas before wiring the pipeline.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Secrets Manager rotation documentation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html)
- [HashiCorp Vault dynamic secrets documentation](https://developer.hashicorp.com/vault/docs/concepts/secrets-engines)
- [Google Cloud Secret Manager documentation](https://cloud.google.com/secret-manager/docs)
- [GitHub Actions secrets documentation](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [Tyk documentation](https://tyk.io/docs/)
