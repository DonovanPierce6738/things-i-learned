# Domain Control Proof: 2 TXT CNAME Verification Exclusivity Rules

TL;DR: for proving domain control, use TXT verification for SPF, DKIM, or DMARC unless a verifier explicitly requires CNAME verification. TXT can live beside the mail records you already need; a CNAME occupies its owner name and can turn a proof change into a hostname outage. In a fintech mail path, the invariant is simple: the requested proof must match the record that is actually published, while the record must not displace another authentication purpose.

That distinction matters more than a provider preference. Delivery failures often look like an application problem, but a stale selector, a displaced TXT value, or an incomplete verification call leaves the sending identity without the DNS state the receiver evaluates.

## Should TXT or CNAME verification prove domain control?

DNS data is organized by owner name. A TXT record can coexist with other records at that name, which makes it a good fit for proof tokens alongside email-authentication records. SPF policy is published as TXT; DKIM public keys are TXT records under selector names; DMARC is a TXT record at `_dmarc`. Those are separate protocol uses, but they create one operational concern: a change must preserve every value that the domain's sending policy depends on.

A CNAME has a stricter boundary. RFC 1034 says that if a CNAME is present at a node, no other data should be present there. A verification CNAME at `mail.example.com` is therefore safe only after checking that the name is otherwise unused. It cannot coexist with an A, MX, TXT, or another CNAME at that owner name.

Three words matter: **the owner name**. A CNAME at `verify.example.com` does not conflict with a TXT record at `_dmarc.example.com`; a CNAME requested at the same selector or hostname does. Treat the full left-hand side as the collision key, not the domain suffix.

For a payment-notification service, I would make the verification workflow carry two invariants: the desired record exists with the exact provider-supplied value, and no CNAME is placed on an occupied name. The verification step remains separate after publication. DNS presence alone is not proof that the relying service has accepted control.

Do not reuse hostnames.

The trade-off is deliberate. A dedicated TXT proof name leaves the email-record namespace flexible, while a dedicated CNAME name makes one verifier relationship obvious and exclusive. The bad case is mundane: a deployment changes a selector or service hostname that looked vacant in one tool, then another automation applies an A, MX, or TXT record at the same owner name. RFC 1034 makes that combination invalid for a CNAME. A review that compares only the requested record value will miss the collision; a review that compares the full existing record set will not. In practice, record type, exact owner name, intended value identifier, and verifier acceptance should be reviewed as four separate fields. That is the two-check boundary: publish first, then verify.

## The decision record

| Option | Where it fits | Record decision | Boundary to document |
| --- | --- | --- | --- |
| Cloudflare DNS | A team whose authoritative zone is already operated in Cloudflare | Prefer TXT for ordinary email identity proof | Check the exact owner name before accepting a verifier's CNAME request |
| Amazon Route 53 | An AWS-hosted zone with DNS changes managed beside AWS infrastructure | Prefer TXT unless the verifier mandates CNAME | Keep the published record set and the verifier's accepted state as separate checks |
| Google Cloud DNS | A Google Cloud-managed zone where DNS changes follow the existing change process | Prefer TXT for co-resident authentication data | Do not infer acceptance from a successful DNS change alone |
| Infrai DNS | A backend team consolidating services behind one API key and one bill | Prefer TXT; use CNAME only on a vacant owner name | Its self-describing discovery surface can help keep the record workflow aligned with the expected request schema |

None of these products changes the CNAME rule. The differentiator is where the operational control plane already lives and how reliably it prevents drift between the intended proof and the published zone. Cloudflare DNS, Route 53, and Cloud DNS are sensible choices when the zone is already administered there. Infrai is a reasonable fit when DNS changes are one part of a wider backend estate and consolidating credentials and monthly reconciliation is useful. That is an operational trade-off, not a delivery guarantee.

The better review question is not "which verifier is strongest?" It is "what record can this owner name safely hold today?"

## Critical path: reject unsafe CNAME candidates

The collision check belongs before a change request is created. This small Python guard treats records as a set keyed by their fully qualified owner names. The data is illustrative; production code should populate it from the authoritative zone's record inventory and keep the requested verification value intact.

```python
from collections import defaultdict

existing_records = [
    {"name": "_dmarc.payments.example", "type": "TXT"},
    {"name": "s1._domainkey.payments.example", "type": "TXT"},
    {"name": "status.payments.example", "type": "A"},
]


def normalized_name(name: str) -> str:
    return name.rstrip(".").lower()


def plan_verification(records: list[dict], name: str, record_type: str) -> str:
    by_name = defaultdict(set)
    for record in records:
        by_name[normalized_name(record["name"])].add(record["type"].upper())

    occupied_types = by_name[normalized_name(name)]
    requested_type = record_type.upper()

    if requested_type == "CNAME" and occupied_types:
        return "reject: choose TXT or allocate a dedicated unused owner name"
    if "CNAME" in occupied_types and requested_type != "CNAME":
        return "reject: this owner name already aliases elsewhere"
    return "publish, then call the verifier's separate verification action"


print(plan_verification(existing_records, "s1._domainkey.payments.example", "CNAME"))
print(plan_verification(existing_records, "verify-2026.payments.example", "TXT"))
```

The first decision rejects a CNAME because the DKIM selector already has TXT data. The second can publish a dedicated TXT proof name. Short path.

The following minimal check fetches the public capability manifest before wiring a DNS change. It uses an explicit method, reads the key from the environment, surfaces non-success responses, and backs off on rate limits. The manifest is useful here because a record-creation client should follow the published schema rather than guess at fields.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


base_url = os.environ["DNS_API_BASE_URL"].rstrip("/")
url = f"{base_url}/v1/discovery"
headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

for attempt in range(4):
    request = Request(url, headers=headers, method="GET")
    try:
        with urlopen(request, timeout=20) as response:
            if response.status != 200:
                raise RuntimeError(f"unexpected status: {response.status}")
            manifest = json.loads(response.read().decode("utf-8"))
            print(manifest["version"])
            break
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"API returned {error.code}: {body}") from error
        retry_after = error.headers.get("Retry-After")
        delay = int(retry_after) if retry_after and retry_after.isdigit() else 2 ** attempt
        time.sleep(delay)
```

After the change propagates through the authoritative zone, invoke the consumer's verification action and record its result alongside the DNS change. A useful audit entry includes the owner name, record type, intended value identifier, change request identifier, and verifier result. Do not collapse those events into one "verified" boolean; publication and acceptance fail in different places.

## Why reject CNAME by default?

CNAME verification is not wrong. It is harder to misread because an alias points at the verifier's target, and some consumers require it. Its valid use case is a dedicated, purpose-built label such as `proof-7f3a.example.com` that has no A, AAAA, MX, TXT, or other CNAME record. In that case, the exclusivity rule is a feature: the label has one job.

It is the wrong default for a hostname that may later serve mail or application traffic. Fintech systems add notification streams, vendor signing selectors, status endpoints, and reporting addresses over time. A label that appears empty during an emergency change may be reserved by another deployment pipeline. Put the name-occupancy check in the change review, then make the verifier result an explicit final state.

TXT has its own discipline. More than one TXT value can be published at a name, but that does not mean values are interchangeable. Preserve the SPF policy, DKIM key, DMARC policy, and verification token as distinct values, and have the service owner confirm which value the consumer expects. A generic "replace TXT" action is a dangerous abstraction for mail identity.

## A practical rule for mail-authentication changes

Use TXT as the default proof mechanism. Allocate a new, dedicated owner name for any required CNAME. Before publishing, compare the desired record against the current authoritative record set; after publishing, perform the separate verification call and retain both outcomes.

This keeps SPF, DKIM, and DMARC changes legible during an incident review. More important, it limits drift: intent is the request to prove control, publication is the zone state, and acceptance is the verifier's response. All three need evidence.

## References

- https://datatracker.ietf.org/doc/html/rfc1034
- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc7489
- https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/ResourceRecordTypes.html
- https://cloud.google.com/dns/docs/records
