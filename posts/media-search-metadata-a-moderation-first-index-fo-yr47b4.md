# Media Search Metadata: A Moderation-First Index for Captions and Asset Filters

**Short answer:** index approved image captions and metadata behind an exact asset ID filter; keep pending and rejected records out of the student search path.

The hard trade-off in an edtech media library is recall versus moderation coverage. Index every caption and you get useful search results quickly, but one unsafe or misleading label can surface in a classroom. I would make moderation state a first-class field, then index captions and metadata only after a deterministic policy check. That keeps the search API fast without pretending that an image label is trustworthy by default.

The example below describes an Express-style service, but the boundaries are ordinary HTTP and JSON. The implementation detail that matters is the data contract: an asset has an immutable ID, normalized metadata, provenance for each label, and a moderation decision that search can filter.

## What should a Node.js index guarantee for image captions, metadata, and search?

A search index should answer three separate questions: does this asset match the text, is it allowed for this audience, and can the caller identify the exact source object? Combining those questions into one free-form caption field creates awkward failure modes. A caption can match perfectly while the asset is pending review. An asset ID can be valid while its metadata is stale.

I use an explicit record shape:

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class MediaRecord:
    asset_id: str
    caption: str
    alt_text: str
    labels: list[str]
    moderation: str       # approved, rejected, or pending
    audience: str         # student, instructor, or internal
    metadata_version: int
    source_uri: str
    reviewed_at: Optional[str]
```

The `asset_id` is opaque and immutable. Do not derive it from a filename: renames, Unicode normalization, and duplicate uploads make that contract brittle. `metadata_version` gives re-indexing a safe checkpoint, while `reviewed_at` makes an audit query possible without parsing prose.

## How do captions become searchable without bypassing review?

Treat labeling as an asynchronous pipeline. Upload writes the immutable asset row and emits a job. A worker extracts dimensions and format, generates candidate captions, normalizes text, and submits the candidate to moderation. Only an approved candidate is copied into the public search document. Pending records can remain in a staff index, but the student query path must filter them before scoring.

The filter belongs in the query object, not in a controller comment. This is the shape I want the HTTP layer to produce:

Keep it explicit.

```python
def build_search_request(text: str, asset_id: str | None, audience: str) -> dict:
    filters = {
        "moderation": "approved",
        "audience": audience,
    }
    if asset_id:
        filters["asset_id"] = asset_id
    return {
        "query": text.strip(),
        "filters": filters,
        "limit": 25,
    }
```

An `asset_id` filter should be exact, never a substring match. Search terms can use tokenization and ranking; identifiers cannot. Validate length and character policy at the edge, then pass the value to a parameterized query. That small distinction prevents a request such as `abc` from returning `abc-lesson-1`, `abc-old`, and an unrelated imported object.

Caption text also needs a provenance flag. If a teacher edits a caption, store the human revision separately from the generated suggestion and increment the metadata version. Otherwise a late worker result can overwrite a careful correction. I initially assumed a last-write-wins update would be harmless; it was the opposite for moderation-sensitive content.

The most expensive bugs are state transitions, not tokenizer choices. Test these as contract tests around the worker and the search adapter:

- A rejected asset never appears in a student result, even when its caption is an exact match.
- A pending revision cannot replace an approved revision with a newer `metadata_version` unless the moderation decision is also approved.
- An exact asset ID returns one record or none; it does not broaden into text search.
- Replaying the same job is idempotent.
- A deleted source leaves a tombstone or removal event so stale index documents are removed.

Keep a fixture with a real-looking classroom edge case: an anatomy diagram, a scanned worksheet, and an image containing a student name. The last item tests both metadata redaction and moderation routing. Small fixtures beat a large synthetic corpus when the policy boundary is the thing under review.

For observability, record job ID, asset ID, metadata version, moderation result, and index response time. Do not log raw student names or full captions by default. A counter for `pending_age_seconds` is more actionable than a dashboard that only reports total indexing throughput.

## Rollout without corrupting search quality

Start with a shadow index. Populate it from the same events, but keep reads on the existing index while you compare document counts, approved-only counts, and exact-ID hit rates. A mismatch should identify the asset and metadata version so an engineer can replay one event instead of rebuilding everything.

Then enable the new filter for one audience, such as instructors, where a pending state can be visible with a clear badge. Students should receive only approved records from the first request. Keep a rollback switch at the index-alias level; changing an alias is safer than deleting documents during an incident.

The decision rule is uncomplicated: optimize caption recall inside the approved set, and optimize moderation coverage before expanding that set. Search is a consumer of policy, not a replacement for it.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://www.rfc-editor.org/rfc/rfc9110
- https://www.w3.org/TR/WCAG22/
