# Hosted HTML to PDF API Beats Self Managed Browsers for Monthly Archives

TL;DR: For a Node.js system that turns HTML into PDF invoices or monthly media reports, use a hosted API instead of running Puppeteer, then retain one verified final PDF per report rather than every render attempt. Full render-history retention is justified only when policy requires the exact intermediate outputs. At batch scale, duplicates create storage, replication, inventory, and deletion work without improving the canonical record.

Start with the bill. Suppose an illustrative monthly batch contains 1,000,000 reports, each final PDF averages 2 MiB, and retries create 1.2 successful artifacts per logical report. Keeping every success produces about 2.4 million MiB for that run; keeping only the canonical result produces about 2 million MiB. Across 12 cohorts, before replicas, versions, logs, or indexes, the difference is 4.8 million MiB of duplicate output. Replace these assumptions with measured percentiles before approving capacity.

The recommendation has a boundary: archive the final report for the period specified by legal and records owners, plus the small manifest needed to prove what was rendered. Expire superseded outputs, temporary assets, and retry bodies on a short operational schedule. If an investigation later needs a discarded intermediate PDF, it will not be there. That is the deliberate cost.

## What actually dominates the cost?

One report can produce an HTML package, fetched fonts and images, several PDF responses, object versions, replicas, checksums, indexes, and access logs. Durable output is better modeled as `logical reports × artifact size × retained copies`, with separate terms for metadata and logs. A small increase in copies affects every cohort for its whole retention period.

It affects throughput too. If workers upload every successful retry before choosing a winner, the archive receives more writes than the logical workload. A timeout creates ambiguity: rendering may have completed even though the caller received no response. Retrying without a stable job key can create another valid PDF and another upload. HTTP defines idempotent methods, but a rendering POST needs an application-level identity and reconciliation rule.

Derive the key from report identity, reporting period, template revision, and source-data revision. Never derive it from an attempt number. Keep attempts in telemetry while allowing exactly one object key to become canonical.

**The dominant controllable term is retained copies per report.** Reducing that multiplier changes the storage curve and reduces lifecycle work.

## How should invoice batches control HTML to PDF API throughput?

A direct call from an application server looks simple until the month-end burst. Rendering consumes CPU and memory somewhere, even when your team does not operate a browser. A bounded queue makes that constraint explicit and prevents reports from consuming the request budget used by interactive traffic.

For the workload in this note, the hosted HTML-to-PDF API wins because batch throughput matters more than owning browser processes. The same choice can serve invoices, but those documents may introduce tax, numbering, and delivery controls outside the renderer. Running Puppeteer instead is reasonable when the document depends on browser behavior the hosted boundary cannot reproduce, data cannot cross that boundary, or sustained utilization makes operating a dedicated pool acceptable. **That limitation is real:** outsourcing the runtime also gives up direct control over its release cadence, warm capacity, and diagnostic depth.

Keep the states few: pending, rendering, archived, and failed. A worker claims a leased job, submits self-contained input, validates the response, writes to a temporary key, and promotes it only after validation. If its lease expires, another worker retries with the same logical key. The winner rule must be atomic at the archive boundary.

Short is good. Invisible is not.

Measure completed reports per minute at batch level, not only median request latency. Track p95 render duration, queue age, retry rate, response size, and archive-write duration. Raise concurrency only while completion throughput rises and errors remain inside the agreed envelope. A fixed worker limit is easier to reason about than unbounded promise fan-out in a Node.js caller.

Use explicit timeouts and require an unambiguous result or stable job reference. Handle `429` and retryable `5xx` classes according to the endpoint contract, with exponential backoff and jitter. Malformed HTML, missing required assets, and validation failures should remain terminal until input changes. Retries must stop before the batch deadline.

## Make the archive decision explicit in code

This Python interface sketch separates rendering from retention policy. The same state machine fits a Node.js caller. It checks the file signature, records a digest, and uses one canonical key; production code must also enforce size limits and authenticate both adapters.

```python
from dataclasses import dataclass
from hashlib import sha256


@dataclass(frozen=True)
class ReportJob:
    publication_id: str
    period: str
    template_revision: str
    source_revision: str

    @property
    def canonical_key(self):
        identity = (
            f"{self.publication_id}:{self.period}:"
            f"{self.template_revision}:{self.source_revision}"
        )
        job_id = sha256(identity.encode("utf-8")).hexdigest()[:24]
        return f"reports/{self.period}/{job_id}.pdf"


def render_and_archive(job, html, renderer, archive):
    pdf = renderer.render(html=html, idempotency_key=job.canonical_key)
    if not pdf.startswith(b"%PDF-"):
        raise ValueError("renderer response is not a PDF file")

    digest = sha256(pdf).hexdigest()
    archive.put_if_absent(
        key=job.canonical_key,
        body=pdf,
        metadata={"sha256": digest},
    )
    return {"key": job.canonical_key, "sha256": digest}
```

The prefix check is merely cheap rejection. ISO 32000-2 defines PDF; conformance and archival suitability require more. If long-term preservation is required, decide whether a PDF/A profile applies, then validate that profile rather than assuming every readable PDF is preservation-ready.

Do not put report details, recipient addresses, tokens, or complete HTML in routine logs. The lesson from OTP delivery applies here: retries and observability matter, but payload duplication enlarges the sensitive-data surface. Log job identifiers, attempt counts, timings, byte counts, outcome classes, and digests. Restrict manifest access like artifact access.

## Compare approaches with a batch acceptance test

A checklist will not reveal month-end behavior. Build a representative corpus with long tables, page breaks near totals, slow fonts, large images, Unicode, unavailable assets, and the longest publication names permitted. Freeze source data and template revisions. Run the whole batch through the production queue, timeout, and archive adapters.

| Decision area | Final-only retention | Full render history |
| --- | --- | --- |
| Durable copies | One canonical PDF | Every retained success |
| Retry ambiguity | Stable key and atomic winner | Multiple artifacts preserved |
| Investigation | Manifest, digest, telemetry, final PDF | Intermediate PDFs available |
| Lifecycle | Expire temporary material | Classify and delete every artifact |
| Best fit | Canonical monthly archive | A named rule requires intermediates |

Define success first: every logical report reaches one terminal state; no canonical key has conflicting digests; queue age stays below the deadline; retry amplification fits the capacity model; required fonts and images appear; and a validator accepts the selected conformance level. Review a visual sample, especially totals and table boundaries. A parser can accept a commercially wrong layout.

Test cleanup too. Interrupt workers after submission, delay archive writes, return duplicate successes, and make an asset unavailable. Confirm that leases recover, canonical objects do not fork, and temporary material expires. This is where a convenient endpoint either fits the architecture or sends complexity back to the caller.

One test deserves extra room because it catches several expensive edge cases at once. Submit the same logical report, with the same source and template revisions, from two workers; allow both render calls to finish, delay the first archive write, and let the second reach the canonical key first. The expected result is one archived PDF, one digest in the manifest, two separately visible attempts, and no silent overwrite. Repeat with different source revisions and require different canonical keys. Then repeat after the retention job has removed temporary bodies: the canonical PDF and provenance must remain readable, while retry payloads must be gone. This exercise tests concurrency, identity, cleanup, and auditability without inventing a production incident. It also exposes the trade-off plainly. Final-only retention preserves evidence about the chosen artifact, not every byte produced on the way there.

## The decision rule

Choose final-only retention when the monthly report is the record, source data and template revisions are traceable, and policy does not require every intermediate render. This approach is not suitable when inputs cannot be reproduced or a named legal, audit, or dispute process needs intermediate artifacts; choose full history instead, with an owner for the added storage, access-control, discovery, and deletion burden.

For the hosted boundary, require documented limits, authenticated transport, predictable timeout behavior, a retry contract, support for the corpus's HTML and fonts, and data handling appropriate to the report classification. No endpoint removes queueing, idempotency, validation, or lifecycle rules. It relocates the rendering runtime.

The resulting design is plain: jobs enter a bounded queue, one PDF is validated and archived, a compact manifest preserves provenance, and temporary inputs and superseded outputs expire. Recovery after an intermediate is deleted means rerendering from preserved source data and the exact template revision. If those inputs cannot be reproduced, final-only retention is the wrong choice.

## Further reading

- https://www.iso.org/standard/75839.html
- https://www.iso.org/standard/38920.html
- https://www.rfc-editor.org/rfc/rfc9110
- https://www.w3.org/TR/html52/
