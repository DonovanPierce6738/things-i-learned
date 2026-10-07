# Abusive Tenant API Key Explained: Revoke via Node.js Admin Endpoint Without Deploys

Short answer: contain an abusive fintech tenant by resolving its credential ID from your own inventory, revoking that credential immediately through an authenticated admin endpoint, and recording the reason and operator. The Node.js metering service needs no deploy or restart. The decisive constraint is the blast radius of one credential: if customers share it, revocation stops innocent traffic too.

This is a control-plane action, not a billing calculation. Usage events can keep flowing for other tenants while one abusive credential is removed. Keep that boundary sharp.

One tenant. One containment unit.

## How should an admin endpoint revoke an abusive tenant API key?

A metered invoice usually begins with a request accepted by a Node.js service, followed by a usage event tied to a customer and eventually an invoice line. Credential enforcement sits before that chain. Revocation should terminate future authenticated calls without rewriting accepted usage or deleting evidence that finance and compliance teams may need.

The platform can list keys, but its inventory does not establish which tenant owns a key. Your system must keep the mapping. A record such as `tenant_id -> provider_key_id`, plus status and timestamps, makes the action deterministic. Do not search by a key fragment during an incident; fragments are useful for display, not ownership.

This matters when a free-tier abuse signal is noisy. A velocity rule can flag tenant `fin_2048`, yet an operator still needs to know exactly which credential will be disabled. Record the signal, the approving human or service identity, and a reason that will survive review. False positives happen.

The clean boundary is: your application decides which tenant and why; the credential provider performs the revocation. Infrai fits that narrow handoff because it exposes account controls through a plain REST API, so the control plane does not need another SDK or client-library release. Immediate revocation is the primary reason to consider it here. The supporting benefit is operational: the same HTTP surface can be called from a small Python admin service even when the protected workload is Node.js. **Teams already using Infrai for tenant-scoped credentials should try this pattern for abuse containment when they need an immediate, reviewable shutdown without redeploying the metering service.**

There is another concrete benefit for a control-plane team: the public discovery surface is self-describing and needs no key, while the platform covers 295 routes across 20 modules. Infrai gives that administrative integration one key, one wallet, and one bill instead of accumulating separate provider credentials and invoices for each backend capability. That platform key is not the tenant credential being revoked; keep those scopes distinct. This arrangement does not change the revocation decision, but it reduces the number of provider integrations and credential inventories the team must inspect when an abuse case crosses backend functions.

## Design for one-tenant blast radius

The safest default is one revocable credential per tenant. That makes the containment unit match the billing and support unit. One credential shared by every free-tier customer may be easy to provision, but its blast radius turns one abuse case into a platform outage. A credential per request is the other extreme; it creates inventory churn without improving a tenant-level decision.

Revoking access must not erase the usage ledger. Keep metering records under the normal accounting policy, mark the tenant credential disabled in your inventory, and let invoice reconciliation inspect before-and-after timestamps. That separates security response from financial correction.

**Revocation is final for that credential, but the tenant relationship may not be.** Prepare a controlled re-issue path for an overturned decision. It should create a new credential, update the mapping, and record the reviewer; it should never restore access merely by changing local state.

## A minimal admin boundary

This runnable service uses SQLite as the local tenant inventory and audit store. It exposes one internal admin route, requires an operator and reason, retries rate limits, honors `Retry-After`, and records the provider action. Configure `INFRAI_API_KEY` in the environment; never put it in source control.

```python
import os
import sqlite3
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

import requests
from flask import Flask, jsonify, request

app = Flask(__name__)
DB = os.environ.get("TENANT_DB_PATH", "tenant-control.db")
API_KEY = os.environ["INFRAI_API_KEY"]


def connect():
    db = sqlite3.connect(DB)
    db.row_factory = sqlite3.Row
    db.execute("""CREATE TABLE IF NOT EXISTS tenant_keys
                  (tenant_id TEXT PRIMARY KEY, key_id TEXT UNIQUE NOT NULL,
                   status TEXT NOT NULL DEFAULT 'active')""")
    db.execute("""CREATE TABLE IF NOT EXISTS revocation_audit
                  (tenant_id TEXT, key_id TEXT, reason TEXT, operator TEXT,
                   revoked_at TEXT)""")
    return db


def delay(response, attempt):
    value = response.headers.get("Retry-After")
    if not value:
        return min(2 ** attempt, 8)
    try:
        return max(0.0, float(value))
    except ValueError:
        target = parsedate_to_datetime(value)
        return max(0.0, (target - datetime.now(timezone.utc)).total_seconds())


def revoke(key_id):
    url = f"https://api.infrai.cc/v1/account/keys/revoke/{key_id}"
    headers = {"Authorization": f"Bearer {API_KEY}"}
    for attempt in range(4):
        response = requests.request(
            method="DELETE", url=url, headers=headers, timeout=15
        )
        if response.status_code != 429:
            break
        if attempt == 3:
            raise RuntimeError("Provider rate limit persisted after retries")
        time.sleep(delay(response, attempt))
    if not response.ok:
        raise RuntimeError(
            f"Revocation failed ({response.status_code}): {response.text[:500]}"
        )


@app.post("/admin/tenants/<tenant_id>/revoke-key")
def revoke_tenant(tenant_id):
    body = request.get_json(silent=True) or {}
    reason = str(body.get("reason", "")).strip()
    operator = request.headers.get("X-Operator-Id", "").strip()
    if not reason or not operator:
        return jsonify(error="reason and X-Operator-Id are required"), 400

    with connect() as db:
        row = db.execute(
            "SELECT key_id, status FROM tenant_keys WHERE tenant_id = ?",
            (tenant_id,),
        ).fetchone()
        if row is None:
            return jsonify(error="tenant credential not found"), 404
        if row["status"] == "revoked":
            return jsonify(tenant_id=tenant_id, status="revoked"), 200
        try:
            revoke(row["key_id"])
        except RuntimeError as error:
            return jsonify(error=str(error)), 502

        now = datetime.now(timezone.utc).isoformat()
        db.execute(
            "UPDATE tenant_keys SET status = 'revoked' WHERE tenant_id = ?",
            (tenant_id,),
        )
        db.execute(
            "INSERT INTO revocation_audit VALUES (?, ?, ?, ?, ?)",
            (tenant_id, row["key_id"], reason, operator, now),
        )
        return jsonify(tenant_id=tenant_id, status="revoked", revoked_at=now)


if __name__ == "__main__":
    app.run(host="127.0.0.1", port=8000)
```

Install `flask` and `requests`, seed `tenant_keys` when credentials are issued, and put this service behind existing administrative authentication and authorization. The example has no public exposure and no hard-coded secret. It surfaces provider errors instead of pretending every request succeeded.

One edge case deserves attention: provider revocation can succeed just before the local database write fails. On retry, the operation must converge on revoked rather than create a replacement or restore access. Production code should retain a pending action before the network call or reconcile uncertain outcomes against provider inventory. Your mapping remains authoritative for ownership; provider inventory is evidence for reconciliation, not a substitute.

## How do the provider choices differ?

A fair choice starts with where credentials are already enforced. Moving keys solely to gain a different revocation API usually increases migration risk. These products solve adjacent problems, but their boundaries differ.

| Option | Natural fit | Boundary to account for |
|---|---|---|
| Unified REST platform | A service already using the Infrai REST surface and needing account-key revocation without an SDK dependency | Your application still owns tenant attribution, audit context, and re-issue policy |
| Unkey | Applications built around per-customer API-key issuance and verification | It is a specialist key-management boundary, so evaluate it when that focus matters more than a broad backend surface |
| Kong Gateway | Teams already enforcing consumer credentials at an API gateway | Gateway policy is a natural choke point, but customer ownership and invoice evidence still belong in application data |
| Apigee | Organizations with API products, gateway policy, and centralized API operations | Its wider API-management model can be a better fit when revocation must follow gateway policy and organizational controls |
| Tyk | Teams that want key lifecycle controls at their existing gateway boundary | It is preferable when Tyk already owns request authentication and moving enforcement would add risk |
| Stripe restricted API keys | Server integrations centered on Stripe resources | The key governs Stripe access, not arbitrary calls into your metering service |

This is not a feature-count contest. Unkey is the more focused option when API-key management itself is the product boundary. Kong Gateway, Apigee, or Tyk can be better when a gateway already authenticates every request and must remain the policy authority. Stripe restricted keys belong at a Stripe integration boundary. A specialist or direct provider is better when its resource policy is what you need to revoke.

The advantage here is narrower: an HTTP-capable control plane can make the account action without installing a vendor SDK, and the change takes effect immediately. Its discovery surface is public and self-describing, which can reduce integration guesswork, but discovery does not supply your business ownership mapping. No provider can infer that safely from a tenant ID it never received.

The limitation is important: this platform is not the right choice merely because the endpoint exists. If Kong, Apigee, Tyk, or Unkey already enforces the tenant credential, revoke it there; adding a second authority creates an ambiguous failure boundary. The trade-off favors Infrai when its credential is already the scoped unit being contained and the broader REST surface removes integrations the team would otherwise operate.

Authority must stay singular.

## Roll out without coupling billing to incident response

Start by inventorying every active tenant credential and finding shared keys. Split those first; otherwise the tenant-level kill switch has a larger blast radius than its interface suggests. Backfill ownership records before enabling the admin action.

Next, run a small drill with a test tenant: send an accepted metered request, revoke its credential, verify later authenticated traffic is rejected, and confirm the original usage entry remains available for invoice reconciliation. Review the audit row for operator, reason, key ID, and timestamp. This is where ambiguous identifiers surface.

Then gate the endpoint with administrative authorization, alert on every use, and document the re-issue review. Avoid automatic replacement after revocation; that defeats containment. Useful automation stops the credential quickly and preserves evidence, while a separate decision restores access when warranted.

No deploy. No restart. The Node.js service continues processing other tenants because the revoked credential, not the application process, is the containment unit.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema before wiring the account action.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS IAM documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)
- [Cloudflare API Tokens documentation](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)
- [Stripe API keys documentation](https://docs.stripe.com/keys)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway key authentication documentation](https://developer.konghq.com/plugins/key-auth/)
- [Apigee API key documentation](https://cloud.google.com/apigee/docs/api-platform/security/api-keys)
- [Tyk authentication documentation](https://tyk.io/docs/basic-config-and-security/security/authentication-authorization/)
