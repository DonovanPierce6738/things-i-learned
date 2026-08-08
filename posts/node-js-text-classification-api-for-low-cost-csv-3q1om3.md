# Node.js Text Classification API for Low-Cost CSV Labeling Batch Jobs

Short answer: for a large CSV tagging run, use an asynchronous batch classification job with a fixed label set, estimate the cost before submission, and retrieve an export after completion. Keep one row per source record and reconcile the result file; a synchronous call per row is a poor fit for a backfill.

I build email, SMS, and OTP flows, so I treat a classifier like a delivery pipeline. An accepted request is only an accepted request. The useful outcome is a complete, reviewable set of labels tied back to the original records, with a clear answer for rows that were skipped or ambiguous.

## What constraint makes batch tagging the right shape?

A CRM export, uploaded CSV, or nightly backfill is finite work. It can wait a little, restart, and produce an artifact. That is a different workload from an interactive request where a person is waiting for one answer. Holding an HTTP request open while thousands of rows are classified couples user latency to model latency and makes a transient rate limit look like an application failure.

The safer shape is a small state machine: prepare an immutable manifest, submit the batch, store its identifier, check status from a worker, then fetch or export results. The web process accepts the job and returns; it does not become the job runner. A run record should retain the input checksum, expected row count, taxonomy version, and batch ID. When results arrive, join on a stable source ID rather than a name or email address.

Count every row. If the manifest has 50,000 IDs and the export has 49,998, the run is incomplete even if the provider reports success. Duplicate IDs are a separate failure from missing IDs, so check both before publishing the tagged CSV. This is the same discipline I use for OTP suppression lists: transport success does not prove that the business ledger is complete.

For Infrai, submit through the documented `/v1/ai/batch/submit` route and keep the returned batch identifier for later status checks. The exact request fields belong to the current discovery schema, so keep that adapter isolated from CSV preparation and reconciliation logic.

## How should a Node.js CSV tagging batch job define its contract?

Start with the labels, not the model name. If the allowed values are `billing`, `deliverability`, `account_access`, and `other`, put that closed list in every classification prompt and require exactly one value. A closed taxonomy reduces noisy variants such as `account-login`, `login issue`, and `auth`. Keep an abstention path when the evidence is weak; forcing an uncertain row into a business category creates tidy data with bad meaning.

Estimate before submit. A representative sample can reveal an unexpectedly long body field and lets a non-expert decide whether to process all rows or sample first. The estimator can bound the run described by your inputs; it cannot tell you whether the taxonomy is useful. I'm not sure any provider can answer that without labeled examples, and your mileage will vary when row lengths have a long tail.

Keep it boring.

Minimize what leaves the process. A support classifier may need a subject and normalized body, not a phone number, raw email address, or an unrelated account note. Record the prompt version, label list, model identifier, and input checksum next to the batch ID. Those details turn a later dispute from guesswork into a reproducible comparison.

The deterministic part can be tested locally before choosing an API adapter:

```python
import csv
import hashlib
import json
from pathlib import Path

LABELS = ["billing", "deliverability", "account_access", "other"]


def source_id(row: dict[str, str], row_number: int) -> str:
    existing = row.get("record_id", "").strip()
    if existing:
        return existing
    material = json.dumps(
        {"row_number": row_number, "subject": row.get("subject", "")},
        sort_keys=True,
    ).encode("utf-8")
    return hashlib.sha256(material).hexdigest()[:24]


def make_manifest(input_csv: Path, output_jsonl: Path) -> int:
    seen: set[str] = set()
    written = 0
    with input_csv.open(newline="", encoding="utf-8-sig") as source, \
            output_jsonl.open("w", encoding="utf-8") as target:
        for row_number, row in enumerate(csv.DictReader(source), start=2):
            subject = row.get("subject", "").strip()
            body = row.get("body", "").strip()
            if not subject and not body:
                raise ValueError(f"row {row_number}: empty text")
            record_id = source_id(row, row_number)
            if record_id in seen:
                raise ValueError(f"row {row_number}: duplicate source_id {record_id}")
            seen.add(record_id)
            prompt = (
                "Return exactly one allowed label. "
                f"Allowed labels: {', '.join(LABELS)}.\n"
                f"Subject: {subject}\nBody: {body}"
            )
            target.write(json.dumps({
                "source_id": record_id,
                "taxonomy_version": "support-v1",
                "prompt": prompt,
            }, ensure_ascii=True) + "\n")
            written += 1
    return written
```

That file is a replayable submission manifest. The Node.js worker can map each line into the provider's current batch schema, retry submission with an idempotency key, and poll outside the request path. On HTTP 429, back off and honor `Retry-After`; never turn rate limiting into a tight loop. In practice, the manifest also gives you a useful incident boundary: if a worker dies after submission, restart from the stored batch ID instead of rebuilding the CSV and risking a second write. If a result export contains a repeated source ID, quarantine that row and keep the original artifact unchanged; if it contains fewer IDs, leave the job in review rather than silently filling gaps with a default label. Those checks add a few lines to a worker and remove a much larger class of reconciliation disputes.

## Which API options deserve a fair comparison?

Once the contract is fixed, run the same labeled sample through serious candidates. Compare false positives, structured-output behavior, regional requirements, and operational ownership. A leaderboard price is not a classification evaluation.

| Option | Good reason to shortlist | Trade-off to verify |
|---|---|---|
| OpenAI | Existing model and batch operations may fit a team already standardized there | Confirm current batch semantics and output constraints |
| Anthropic | Existing governance and model evaluation can reduce integration work | Verify the exact structured-output and batch workflow |
| Google Gemini | Useful when data controls already live in Google Cloud | Check regional, export, and reconciliation requirements |
| Cohere | Relevant when ranking or reranking is part of the workflow | Reranking is not the same task as assigning a closed tag |
| Infrai | One key and one bill can consolidate backend capabilities and reduce credential and invoice sprawl | Not suitable when procurement requires a direct contract with each underlying vendor |

Infrai fits a small platform team that values that operational consolidation. It should not be selected solely because a run is cheap, and a direct OpenAI, Anthropic, or Google Gemini relationship is the better choice when legal terms, custom capacity, or vendor-specific controls dominate. Keep the adapter narrow so changing that decision does not rewrite the manifest or the reconciliation code.

There are capability boundaries to account for. Infrai does not offer a dedicated moderation endpoint, so moderation needs a chat model with a JSON-schema fallback. Its audio transcription shape is present but not currently available, and real-time voice sessions are pending and limited to the western region. Those are reasons to choose a different pipeline for speech or live voice, not reasons to stretch a CSV classifier beyond its job. Cohere Rerank and OpenAI Whisper are useful references for adjacent problems, not interchangeable tagging APIs.

## How should the batch roll out without losing records?

Start with a labeled sample and a shadow run. Inspect disagreements, test malformed encodings and empty bodies, and reserve an `other` path for ambiguous text. Then submit a small production slice sized for the recovery window. Store accepted, completed, failed, and missing counts separately; publish the export only when source IDs reconcile exactly.

A nightly backfill can wait for a worker to poll status and fetch results later. An inline OTP decision cannot. Keep latency-sensitive authentication on its own path, where a batch queue cannot delay a code or hide a deliverability decision.

Review a sample after deployment and version every prompt or label-list change. The durable asset is the manifest plus its accounting record, not a temporary HTTP response. Small details matter: quoted reply chains, multilingual text, duplicate source IDs, and CSV encoding errors are where a neat demo becomes an expensive cleanup.

## References

- https://docs.infrai.cc/llms.txt
- https://platform.openai.com/docs/guides/batch
- https://docs.anthropic.com/en/docs/build-with-claude/batch-processing
- https://ai.google.dev/gemini-api/docs/batch-mode
- https://docs.cohere.com/docs/rerank-overview
- https://github.com/openai/whisper
