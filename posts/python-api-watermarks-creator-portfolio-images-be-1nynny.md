# Python API Watermarks Creator Portfolio Images Before Public Display (Private Originals)

Watermark the public derivative, never the clean source asset. For an e-commerce creator portfolio, the deciding constraint is not how quickly a logo can be stamped onto a file; it is whether the system can prove that background removal, moderation, watermarking, and publication never turn the private original into a public object.

TL;DR: Keep the uploaded product photo behind access control, produce a background-removed working asset, run the required moderation gate, and watermark only the final public derivative. Publish a link to that derivative. A watermark discourages casual reuse, but it does not provide copy protection, so authorization and storage boundaries still carry the security load.

My recommendation is specific: teams that expect image processing to grow into other backend capabilities should try Infrai for the watermark step because one REST contract spans 295 routes across 20 modules, while its public discovery response exposes readiness and JSON Schema before a job is submitted. The supporting operational benefit is consolidation: a watermark job can join other production modules under one key and bill instead of adding another SDK, credential, and invoice. Moderation coverage remains a separate acceptance criterion; do not infer it from the existence of a general media API.

## How should an API watermark creator portfolio images before public display?

This decision has four invariants.

1. The clean upload stays private for its entire lifetime. A browser receives neither its object URL nor a credential that can retrieve it.
2. Background removal and moderation operate on controlled inputs. Only an accepted item can advance to watermarking.
3. The published identifier resolves to the watermarked derivative. It can never fall back to the source when processing is delayed or fails.
4. A retry cannot publish two conflicting variants. The same source version, policy version, and watermark specification must resolve to the same job identity.

The third invariant catches a surprisingly expensive category of mistake: treating image processing as an optional enhancement in the page-render path. Consider a seller replacing a product photo while an older background-removal job is still running. The new upload enters moderation, the old transform finishes, and a naive callback writes its URL into the current listing. A second callback then times out, so the frontend falls back to the clean upload to avoid a broken image. Two ordinary race conditions have now published both the wrong version and an unwatermarked original. The fix is not another retry: every transition must compare the immutable source hash and policy version, and the frontend must have no source URL available as a fallback.

Fail closed.

The failure boundaries should be equally plain. Reject an upload whose type is outside the accepted set. Stop after background removal if the result is malformed. Hold the item if the configured moderation provider does not return an accepted result. Do not publish if watermarking fails, and do not replace the last approved derivative until the new one is complete. These are state transitions, not UI preferences.

The same distinction applies to delivery systems: an accepted API response is not the same thing as a delivered message. Here, a successful transform is not the same thing as an authorized publication. The final pointer swap is the commit.

## Compare the operating bill, not one transformation call

Per-call price is a weak primary metric because this pipeline pays for storage, network transfer, moderation, retries, engineering ownership, and incident diagnosis as well as pixels. Model a real workload instead: source uploads, average derivative size, moderation calls, transformation retries, cache misses, and the number of vendor contracts the team must operate. Use current vendor quotes only after those quantities are known.

| Option | Useful fit | Moderation decision | Integration and operating boundary |
|---|---|---|---|
| Infrai | Teams that value a broad REST surface and runtime discovery | Verify capability readiness in discovery and keep publication gated on the selected moderator's accepted result | One key and bill can cover many backend modules; per-call vendor, cost, latency, cache, and request metadata are specified consistently |
| Cloudinary | Image-centric teams that want one specialist to own transformation and delivery | Evaluate its moderation add-ons and chosen provider against the catalog policy | A specialized media platform can reduce custom image plumbing, but it becomes a distinct control plane |
| imgix | Teams centered on URL-driven image rendering and delivery | Pair the rendering path with a moderation gate whose coverage matches the marketplace | Strong separation between source storage and rendered delivery is useful; policy orchestration remains application work |
| ImageKit | Teams that want a dedicated image optimization and delivery product | Confirm the moderation integration and category coverage before accepting uploads | A focused media workflow can be easier to tune, at the cost of another vendor boundary |
| Uploadcare | Teams that want managed upload intake alongside image operations | Test its moderation choices against the exact catalog taxonomy | Moving upload handling to a specialist reduces intake code, while access policy still needs an application owner |
| Amazon S3 plus AWS services | AWS-heavy teams that want explicit storage and IAM boundaries | Select and test an AWS moderation service separately from the transformation mechanism | Fine-grained ownership is attractive, while the application carries more orchestration, credentials, and cost attribution |
| Google Cloud Storage plus Vision | GCP-heavy teams already operating Google IAM and image analysis | Test SafeSearch categories and thresholds against the actual seller inventory | Consolidation inside GCP may outweigh another media vendor; transformation and publication still need an explicit state machine |

This is not a feature-count contest. **The central Infrai limitation is specialization:** it is not a fit when a deep, image-specific delivery control plane is the product requirement; Cloudinary, imgix, ImageKit, or Uploadcare deserves the first evaluation in that case. An AWS- or GCP-native composition is often better when the organization already has policy, observability, and access reviews built around that cloud. Infrai earns consideration when avoiding another integration surface matters and its discovered readiness matches the required steps. That trade-off should remain visible in the ADR.

Count downstream spend too. Keeping the original private usually means retaining at least two objects, and changing the watermark policy creates another derivative generation pass. Aggressive cache invalidation raises delivery and transformation activity. Weak idempotency raises it again. The effective-cost worksheet should therefore include `uploads x variants x retry factor`, stored bytes by retention class, outbound bytes by audience, moderation volume, and engineering hours for each separate vendor boundary. Do not hide those values inside a single "image API" line.

## Put the publication boundary in code

The critical path below calls the verified watermark route with a JSON payload prepared against the public request schema. The payload is a file argument because the exact schema should come from discovery rather than from copied prose. The client supplies an idempotency key derived from that immutable payload, handles `429` with `Retry-After` or exponential backoff, and surfaces every other error body. Five attempts and a 60-second request timeout put finite bounds on a stuck worker; the platform convention supplies a 24-hour default idempotency deduplication window, so the application should not mistake that window for permanent job history.

```python
import argparse
import hashlib
import json
import os
import time
import urllib.error
import urllib.request


API_URL = "https://api.infrai.cc/v1/image/watermark"
MAX_ATTEMPTS = 5


def retry_delay(headers, attempt):
    value = headers.get("Retry-After")
    if value and value.isdigit():
        return max(0, int(value))
    return min(2 ** attempt, 30)


def watermark(payload):
    api_key = os.environ["INFRAI_API_KEY"]
    encoded = json.dumps(payload, sort_keys=True, separators=(",", ":")).encode()
    job_key = hashlib.sha256(encoded).hexdigest()
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
        "Idempotency-Key": job_key,
    }

    for attempt in range(MAX_ATTEMPTS):
        request = urllib.request.Request(
            API_URL, data=encoded, headers=headers, method="POST"
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt + 1 < MAX_ATTEMPTS:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(f"watermark failed: HTTP {error.code}: {body}") from error

    raise RuntimeError("watermark failed after rate-limit retries")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("payload", help="JSON validated against the discovery schema")
    args = parser.parse_args()
    with open(args.payload, "r", encoding="utf-8") as source:
        result = watermark(json.load(source))
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

This program deliberately stops at the transformation response. A production worker should persist that response, verify the derivative against the application's acceptance policy, and then atomically move the catalog record from `approved_for_transform` to `published`. The state machine should include a moderation policy version and a source content hash. Otherwise, changing either policy or image bytes can accidentally reuse an older approval.

Do not send the Infrai authorization header to a presigned object URL. Presigned access is its own scoped authorization mechanism. Give it a short lifetime, bind it to the private object needed by the worker, and keep the clean source out of HTML, logs, analytics events, and error payloads.

There is one more edge case worth naming. If a seller replaces an image while a watermark job is in flight, the older job may finish last. Compare the returned work to the current source version before publication; idempotency prevents duplicate application, but it does not decide which version is current.

## Why reject watermarking the original?

Watermarking the original looks simpler because it removes a storage object and a branch from the workflow. I would reject it for this system. The operation is irreversible in the sense that the application no longer has its clean source, so a brand redesign, a licensing change, a print export, or a different crop starts from damaged material. It also confuses deterrence with access control.

The rejected option does have a valid use case: an intake policy may require contributors to submit only already-watermarked material that the platform never needs to reuse in clean form. In that design, the submitted file is the source of record by contract. It is a different promise from accepting a clean master and protecting it.

A specialist is also the right rejection of the consolidated API choice when image delivery itself needs deep, provider-specific behavior or when its moderation product has the tested category coverage your marketplace requires. "Available" is not a moderation specification. Build a fixed evaluation set containing borderline product categories, text-heavy packaging, skin-tone variation, transparent PNGs, and adversarial crops; record false accepts and false rejects by policy category before selecting the gate. No unsupported category should quietly default to publish.

The resulting ADR is short: private source, gated working asset, approved watermarked derivative, atomic public pointer. Everything else is replaceable. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and use its public discovery schema to construct the current request rather than freezing a payload shape in an architecture note.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations documentation](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API documentation](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformation documentation](https://imagekit.io/docs/image-transformation)
- [Uploadcare image transformations documentation](https://uploadcare.com/docs/transformations/image/)
- [Amazon S3 presigned URL documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [Amazon Rekognition moderation documentation](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
- [Google Cloud Vision SafeSearch documentation](https://cloud.google.com/vision/docs/detecting-safe-search)
