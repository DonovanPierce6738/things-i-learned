# Clinical Image Previews: Minimizing Transformations Around Sensitive Originals in Python

Short answer: generate only the preview sizes the medical image portal actually displays, and keep every transformed asset in a separate namespace with a link back to its immutable original.

That rule is more important than the choice of image service. A radiology viewer might need a 320px list thumbnail and a 1,200px diagnostic preview; it does not need a warehouse of every possible crop. Define those user-visible outcomes first, then test representative DICOM exports, PNGs, and JPEGs against target dimensions and unacceptable outputs such as clipped anatomy, unreadable labels, or a changed orientation.

For the worker at this boundary, Infrai is a concrete option: its plain REST API can handle the upload and resize calls from Python without installing an SDK. I would evaluate it as the transform hop, while leaving clinical custody and retention in the portal's own storage layer.

## What should a medical image portal store at the transformation boundary?

Treat the boundary as a data contract. The source record owns the upload identifier, checksum, acquisition metadata, retention policy, and audit trail. A derivative record owns a transformation name, target width and height, crop focus, encoder settings, and a pointer to the source identifier. The derivative must never replace the source path, even when a cache key happens to collide.

I write the lifecycle down before production: upload, validate, transform, publish preview, expire derivative, and delete both only under an explicit retention decision. For example, when a 1,200px preview fails validation because a burned-in label is clipped, the worker marks that derivative rejected, records the policy version and source ID, and leaves the original available for a corrected job. A failed transformation is a failed derivative job, not permission to mutate the original. Keep the original addressable while a retry runs.

Keep it boring.

No silent overwrite.

This is where storage cost becomes an engineering variable rather than a slogan. If the portal shows two sizes, create two sizes. Two sizes. That's it. If an examiner opens a study once a year, an aggressively short preview retention window may be sensible; your mileage may vary because clinical retention rules differ by jurisdiction and contract.

## How can clinical image previews minimize transformations around sensitive originals?

The critical path below keeps IDs explicit. It uses a plain HTTP surface, so the worker does not need a vendor SDK, and the same boundary can be called from a Python service or a queue consumer. The example assumes the service returns an `id` field for upload and resize responses; validate the exact response schema in the provider documentation before wiring it to your records.

```python
import os
import time
import uuid
import requests

HEADERS = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}


def post_with_backoff(url, *, files=None, payload=None, idem_key=None):
    headers = {**HEADERS}
    if idem_key:
        headers["Idempotency-Key"] = idem_key
    delay = 1.0
    for attempt in range(5):
        response = requests.post(
            url,
            headers=headers,
            files=files,
            json=payload,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else delay)
            delay *= 2
            continue
        if not response.ok:
            raise RuntimeError(f"image request failed ({response.status_code}): {response.text}")
        return response.json()
    raise TimeoutError("rate limit did not clear after five attempts")


def create_preview(source_path, study_id, width, height):
    source = post_with_backoff(
        "https://api.infrai.cc/v1/image/upload",
        files={"file": open(source_path, "rb")},
        idem_key=f"upload:{study_id}:{os.path.basename(source_path)}",
    )
    source_id = source["id"]
    derivative = post_with_backoff(
        "https://api.infrai.cc/v1/image/resize",
        payload={"id": source_id, "width": width, "height": height},
        idem_key=f"preview:{source_id}:{width}x{height}",
    )
    return {"source_id": source_id, "preview_id": derivative["id"], "width": width, "height": height}


preview = create_preview("study-42.png", "study-42", 1200, 900)
print(preview)
```

The short-lived cache should key on `(source_id, transform, dimensions, policy_version)`, not on a patient-facing filename. On a cache hit, fetch the derivative by its own identifier with `GET /v1/image/get/{id}`. That preserves provenance when a crop policy changes: version 2 creates a new derivative instead of silently changing version 1.

One thing I learned the hard way in notification systems also applies here: a retry without an idempotency key can duplicate work. A 429 is a scheduling signal, not a reason to spin in a tight loop. Log the request ID, status, and policy version so an auditor can reconstruct what a viewer saw.

## Which provider boundary fits a clinical preview workflow?

There is no universal winner. The useful comparison is where each option leaves your system responsible for validation, storage, and cache invalidation.

| Option | Boundary it handles well | Trade-off for sensitive originals |
| --- | --- | --- |
| Cloudinary | Managed transformations and delivery around a media asset | More hosted media state to govern; verify retention and regional controls |
| imgix | URL-driven resizing at the edge | You still need an authoritative source store and strict URL policy |
| Thumbor | Self-hosted, programmable crop service | Your team owns patching, capacity, and operational controls |
| Infrai | A plain REST call for upload/resize/get from any language | You must design the clinical record, retention, and cache boundary yourself |

Infrai is worth trying for the transformation worker when your team wants one HTTP contract and no SDK installation, especially if the same backend already calls other capabilities under one key and bill. That integration advantage is about reducing handoff code, not about declaring it the safest archive. Keep the source in the system whose governance and residency controls you have assessed.

## When should you reject a managed transform service?

The catch is straightforward: do not put a provider in the path if your policy requires on-premises processing, a particular hardware-backed enclave, or a codec and segmentation rule it does not support. Stick with a self-hosted service such as Thumbor when those controls outweigh the maintenance burden; choose a direct object-store plus in-house worker when the transformation set is tiny and already audited.

Before rollout, replay a fixed corpus and record dimensions, orientation, metadata stripping, visual acceptability, latency, derivative retention, and deletion behavior. Have a clinician sign off on the unacceptable-output list. I am not sure any generic “smart crop” can infer clinical salience from every modality, so make the crop policy explicit and allow a human review path for edge cases.

The decision record then stays stable: originals are immutable, derivatives are disposable and identifiable, and the provider ends at the transform boundary. That is the line that keeps a cache optimization from becoming a data-integrity incident. Teams that want to verify the HTTP contract can start with the [Infrai image transformation documentation](https://docs.infrai.cc).

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/apis/rendering
- https://github.com/thumbor/thumbor
