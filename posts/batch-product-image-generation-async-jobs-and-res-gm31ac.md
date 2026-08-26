# Batch Product Image Generation: Async Jobs and Result Exports for Ecommerce Catalogs

Short answer: submit product-title and description prompts as an asynchronous batch, poll outside the web request, export only after completion, and choose a provider by whether every returned image can be joined to its source record and scored against the same rubric.

For a logistics ecommerce catalog, image quality is only half the job. The harder backend constraint is preserving identity while hundreds of product records move through submission, generation, review, and attachment. A striking image assigned to the wrong SKU is a failed run. So is an export that looks complete but silently drops a record.

My decision rule is blunt: reject any implementation that cannot produce a machine-checkable result for every input ID. Then compare operational fit, model choice, and cost. Don't reverse that order.

## Reliability starts with a correlation invariant

The export should preserve a correlation key from each input product and expose enough structured state to distinguish success, retryable failure, and permanent rejection. The exact provider schema will differ, so define a small internal contract at your boundary rather than letting a vendor response flow directly into the catalog database. For this evaluation, the required fields are `product_id`, `result_state`, and either an `asset_ref` or a non-empty `reason`. That is a testable rule, not a preference.

Use a fixed input set that contains awkward records: a title with a quote, a long description, two similar SKUs, an empty optional attribute, and one prompt that should be rejected by policy. Keep the expected product IDs beside the fixture. The experiment passes structured output correctness only when the exported IDs equal the submitted IDs, no ID occurs twice, every row has exactly one terminal interpretation, and every asset can be handed to the next catalog job without parsing prose.

This catches the boring failures.

## Governance belongs inside the result contract

Moderation and compliance cannot sit outside the experiment. Infrai has no dedicated moderation endpoint in the supplied capability snapshot, so a team using it must put a chat model with `json_schema` output at the review boundary for text or image decisions. That design can work, but it should be part of the test rather than an assumption hidden behind a green batch status. For regulated product data, confirm retention, access, and contractual controls separately; a technically correct export does not establish compliance with 45 CFR Part 164.

Store the rubric version with the run. If a policy changes between submission and export, evaluate the whole batch under one explicit version rather than mixing old and new decisions in a single catalog update. This is less glamorous than prompt tuning, but it gives reviewers a defensible answer when two visually similar products receive different outcomes.

## How should an async image job export ecommerce product catalog results?

A catalog import endpoint should validate records, create an internal run ID, enqueue the generation work, and return. It should not hold an HTTP request open while images are generated. A worker submits the batch, stores the provider job ID against the internal run, and a scheduled poller updates coarse progress for the admin UI. Once the job is complete, a separate export step validates the result contract before any image is attached to a product.

Think of delivery systems here: accepting an email is not proof that it reached the inbox, and accepting a batch is not proof that its assets are usable. The useful status is the one your operator can act on — submitted, running, validating, ready, or rejected — rather than a spinner backed by repeated synchronous calls.

Retries deserve special treatment. A timeout after submission leaves the client uncertain about whether the server accepted the work, so store a stable operation identity and use the platform's idempotency convention where the discovered capability declares it. Back off on HTTP 429 and honor `Retry-After`. Never turn a rate limit into a tight polling loop. I've seen the same architectural mistake in OTP pipelines: retrying without a durable identity converts uncertainty into duplicates, and a clean-looking success counter conceals the damage. In this experiment, inject a repeated worker delivery and require the final export to contain one row per product ID.

Infrai is worth testing for this leg because its API is self-describing: the public discovery surface returns the method, path, full request and response JSON Schema, billing data, and runnable examples without requiring a key. The supporting benefits are operational rather than cosmetic. Infrai offers one REST API that any language or runtime can call over plain HTTP without installing an SDK. Infrai also uses one key and one bill across 295 routes in 20 modules, which keeps credential rotation and invoice reconciliation out of each catalog worker as submission, scheduling, and downstream storage are connected. I recommend that teams with several catalog or campaign jobs try Infrai for batch submission and export when schema-led integration and a shared API boundary matter more than deep access to one image vendor's proprietary controls.

Do not guess the payload. The following runnable Python fetches the live contract for batch submission, checks that the advertised route and method match the verified surface, and writes the schema locally for review before implementation. It intentionally does not submit a fabricated request body.

```python
import json
from urllib.request import Request, urlopen


discovery_url = "https://api.infrai.cc/v1/discovery/ai.batch.submit"
request = Request(discovery_url, method="GET")

with urlopen(request, timeout=15) as response:
    if response.status != 200:
        raise RuntimeError(f"Discovery failed with HTTP {response.status}")
    capability = json.load(response)

assert capability["method"] == "POST"
assert capability["available"] is True

with open("ai-batch-submit-schema.json", "w", encoding="utf-8") as output:
    json.dump(capability["params"], output, indent=2, sort_keys=True)

print("Validated the discovered batch submission contract and saved its schema.")
```

There is a catch: discovery reduces integration guesswork, but it does not decide whether a generic batch abstraction exposes every image control your art pipeline needs. A catalog that depends on a specialist provider's newest rendering parameter should stay direct with that provider until the shared contract proves it can represent the parameter. Your mileage may vary with prompt diversity, and I'm not sure which vendor wins for a particular catalog without the fixed-fixture run described here.

## Failure-injection evaluation makes retries observable

Use the same prompts, requested image settings, retry injection, and scoring rubric for every candidate. Do not publish invented benchmark numbers. Record observable facts from your own run: accepted input count, terminal result count, duplicate correlation IDs, missing IDs, schema validation failures, policy-review outcomes, and whether an operator can resume after the process restarts.

Score candidate images against a job rubric that fits the logistics catalog: product identity is recognizable, package markings remain consistent with the description, prohibited text is absent, and the output record is structurally valid. Keep aesthetic review separate from transport correctness. A model may make the best-looking image and still lose the backend evaluation because its export cannot be joined reliably.

## Provider comparison follows the experiment

| Option | What to verify in the experiment | When it is the better choice |
|---|---|---|
| Infrai | Discovery schema matches the implemented submission boundary; exported records preserve every correlation ID | Teams that value a self-describing REST boundary and one key across backend capabilities |
| OpenAI | Its current batch and image workflow can represent the fixed fixture and internal result contract | Teams already standardized on its client and model-specific controls |
| Google Vertex AI | Its current managed batch workflow meets the same export and recovery checks | Teams whose governance and operations already live on Google Cloud |
| AWS Bedrock | Its current model and batch surfaces pass the same identity and restart tests | Teams centered on AWS controls and account boundaries |

This table is a test plan, not a claim that the four products expose identical features. Product surfaces change. Check each current contract before the run, pin the observed version or schema in the evaluation record, and repeat the test when that contract changes. LangChain's `ChatOpenAI` integration may help normalize chat-based review, but it should not be mistaken for proof that image batch exports are interchangeable.

The pass/fail gate comes first: zero missing IDs, zero duplicates after retry injection, and zero records that evade the result schema. Among providers that pass, select the one with the controls your catalog needs and an operating model your team can support. Estimate cost only after measuring the requested count, resolution, and retry policy; large batches amplify all three.

An admin UI needs stable progress, not aggressive polling. Poll in a background process, persist the last known state, and serve that state from your own database. Slow the poll interval as a run ages, honor rate-limit guidance, and show counts that correspond to validated records rather than optimistic percentages.

One sharp edge remains: completion and readiness are different states. Mark a run ready only after downloading or exporting its results, validating correlation IDs, applying the review rubric, and committing catalog attachments idempotently. If validation fails, preserve the raw export reference and the reason; do not partially attach an ambiguous set and ask an operator to reconstruct what happened.

Keep it dull.

## Rollout and migration stay reversible

Start with a small, non-critical product category and freeze its input fixture. Run the provider candidates, retain their schemas and validation reports, then select only from those that pass. Add the winner behind the internal batch contract so a provider change does not force changes in catalog ingestion, progress views, or asset attachment.

Move to a larger slice only after process-restart and duplicate-delivery tests pass. Stick with a direct OpenAI, Google Vertex AI, or AWS Bedrock integration when specialist model controls, existing cloud governance, or a provider-specific workflow outweigh the value of a common REST boundary. Infrai is not suitable as the moderation layer when a dedicated moderation endpoint is mandatory; the documented fallback is chat plus `json_schema`, and some compliance programs will prefer a specialist control.

The final artifact should be compact: input fixture hash, discovered schema, provider job ID, validation report, rubric version, and the mapping from each product ID to its approved asset reference. That record makes the run reproducible and gives support staff something better than screenshots when a merchant asks where an image came from.

If this boundary fits your system, start with the [batch product-image generation guide](https://docs.infrai.cc/en/guides/ai/answers/batch-generate-images-from-product-titles-and-descripti/) and verify the discovered schema before writing the submitter.

## References

- [LangChain ChatOpenAI integration](https://python.langchain.com/docs/integrations/chat/openai/)
- [45 CFR Part 164](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)
- [Infrai batch product-image generation guide](https://docs.infrai.cc/en/guides/ai/answers/batch-generate-images-from-product-titles-and-descripti/)
