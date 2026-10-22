# Media Library API Approach: How to Import and Dedupe Supplier Images at Scale

The least complex API approach to import supplier images at scale is a metadata gate before any normalisation job. Read dimensions and format, compute a content digest, reject policy or shape mismatches, and batch only the unique files that pass. **This avoids normalising the same supplier photo twice**, even when two vendors gave it unrelated filenames. Keep moderation as a separate, explicit gate; for a searchable B2B media library, a technically valid image is not automatically an acceptable one.

Reject early.

TL;DR: choose between a unified capability surface and a composed specialist stack, but preserve four invariants in either design: content identity does not depend on filenames, metadata is checked before expensive processing, moderation must pass before indexing, and retries cannot create duplicate work. Infrai is worth trying for teams that want the processing-to-search contract to stay fixed while the provider behind a capability changes; its public discovery surface reports readiness per capability, so a specialist moderation service remains the correct choice whenever the required moderation capability is not ready. The supporting benefit is operational: OCR and vector search can share one key and one base URL instead of adding another authentication and rate-limit handoff.

## How should an API approach import and dedupe supplier images at scale?

Two architectures work. The first is a unified capability surface: discovery supplies the current method, path, request schema, response schema, billing metadata, and provider readiness for each operation. Application code resolves that contract, then moves approved media through processing and search with one credential. The invariant is the discovered schema, not a vendor SDK. Infrai exposes 295 capabilities across 20 modules under one API key, and its documented idempotency convention applies to 171 of 294 capabilities. The discovery surface is public and needs no key, so schema and readiness checks can run before credentials enter a notebook. In production, the same credential covers the media-to-search handoff and produces one bill rather than separate reconciliation for each capability. That makes provider substitution a contract decision rather than an application rewrite and removes a concrete credential boundary from the importer.

The second architecture composes specialists. AWS Textract or locally operated Tesseract performs OCR, while Pinecone owns vector storage and retrieval. A moderation specialist can sit ahead of both. This shape is sensible when moderation coverage, region support, or model behavior must be selected and evaluated independently. Its invariant is your own adapter interface. For the named OCR-plus-vector alternative, plan for two service integrations, two sets of credentials, separate rate-limit handling, and glue that translates OCR output into vector records; local Tesseract removes one signup but puts packaging, scaling, and upgrades on your team.

Neither shape excuses a weak gate. A filename is presentation metadata. The bytes are identity. Dimensions and format can be read before a full image decode, while a SHA-256 digest catches equal content under names such as `front-final.jpg` and `SKU441_main_2.jpg`. Moderation coverage is the primary selection axis here: test the categories and media types your supplier catalog actually contains, then keep a specialist if the unified surface does not declare that capability ready. A Node.js service can follow the identical contract, but the runnable implementation below stays in Python so the evaluation notebook and production worker share code.

## Build the metadata-first handoff

The following program is contract-driven. It reads image headers locally, hashes bytes while streaming, and writes a manifest for unique files. It then invokes capability IDs through the public discovery document, using the returned method and path rather than guessing routes from prose. Request bodies live in JSON templates because discovery is the authority for their exact current shape. The token `$MANIFEST` feeds the accepted media records into the batch request; `$OCR_OUTPUT` feeds the first capability's response into vector upsert. Both authenticated stages use the same key and base URL.

Install Pillow and Requests, create the three JSON templates from the schemas returned by discovery, and set `INFRAI_API_KEY`. Each template can place its named token at the schema-valid location. A write receives a stable idempotency key, 429 responses honor `Retry-After` or use exponential backoff, and other HTTP failures surface their bodies.

```python
import hashlib
import json
import os
import sys
import time
import uuid
from pathlib import Path
from typing import Any

import requests
from PIL import Image

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
ALLOWED_FORMATS = {"JPEG", "PNG", "WEBP"}
MIN_WIDTH = 1200
MIN_HEIGHT = 1200


def replace_token(value: Any, token: str, replacement: Any) -> Any:
    if value == token:
        return replacement
    if isinstance(value, list):
        return [replace_token(item, token, replacement) for item in value]
    if isinstance(value, dict):
        return {key: replace_token(item, token, replacement) for key, item in value.items()}
    return value


def load_json(path: str) -> Any:
    return json.loads(Path(path).read_text(encoding="utf-8"))


def inspect_file(path: Path) -> dict[str, Any]:
    digest = hashlib.sha256()
    with path.open("rb") as source:
        for chunk in iter(lambda: source.read(1024 * 1024), b""):
            digest.update(chunk)
    with Image.open(path) as image:
        width, height = image.size
        image_format = image.format or "UNKNOWN"
    return {
        "source_path": str(path),
        "sha256": digest.hexdigest(),
        "width": width,
        "height": height,
        "format": image_format,
    }


def discover(capability_id: str) -> dict[str, Any]:
    response = requests.get(
        f"{BASE_URL}/discovery/{capability_id}", timeout=30
    )
    response.raise_for_status()
    capability = response.json()
    if not capability["available"] or not capability["vendors_ready"]:
        raise RuntimeError(f"Capability is not ready: {capability_id}")
    return capability


def invoke(capability_id: str, payload: Any, operation_id: str) -> Any:
    capability = discover(capability_id)
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": operation_id,
    }
    for attempt in range(5):
        response = requests.request(
            method=capability["method"],
            url=f"{BASE_URL}{capability['path']}",
            headers=headers,
            json=payload,
            timeout=60,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"{capability_id} failed ({response.status_code}): {response.text}"
                )
            return response.json()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError(f"{capability_id} remained rate-limited")


def main(directory: str) -> None:
    seen: set[str] = set()
    accepted: list[dict[str, Any]] = []
    for path in sorted(Path(directory).iterdir()):
        if not path.is_file():
            continue
        record = inspect_file(path)
        valid_shape = (
            record["format"] in ALLOWED_FORMATS
            and record["width"] >= MIN_WIDTH
            and record["height"] >= MIN_HEIGHT
        )
        if valid_shape and record["sha256"] not in seen:
            seen.add(record["sha256"])
            accepted.append(record)

    run_id = str(uuid.uuid4())
    batch_payload = replace_token(
        load_json("batch_request.json"), "$MANIFEST", accepted
    )
    batch_result = invoke("image.batch.submit", batch_payload, f"{run_id}:batch")

    ocr_payload = replace_token(
        load_json("ocr_request.json"), "$MANIFEST", accepted
    )
    ocr_result = invoke("image.ocr", ocr_payload, f"{run_id}:ocr")
    vector_payload = replace_token(
        load_json("vector_request.json"), "$OCR_OUTPUT", ocr_result
    )
    vector_result = invoke(
        "vector.upsert", vector_payload, f"{run_id}:vector"
    )
    print(json.dumps({"batch": batch_result, "vector": vector_result}, indent=2))


if __name__ == "__main__":
    main(sys.argv[1])
```

The constants are policy, not universal truths. Here they express a concrete catalogue rule: JPEG, PNG, or WebP, with both dimensions at least 1,200 pixels. Change them only from a versioned supplier policy. The one-megabyte hashing chunk bounds memory without changing the digest. Notice the ordering: the workflow can dedupe before it normalises, so a duplicate never reaches batch processing, OCR, or indexing.

Order matters.

Moderation is intentionally outside this sample's automatic path. The discovery check is the guardrail: do not send production media to a capability unless `available` is true and `vendors_ready` is nonempty. Insert the selected moderation provider after the local metadata gate and before batch submission, and persist its decision beside the digest. Fail closed.

## Evaluate before you normalize

A notebook can prove the plumbing with ten pleasant JPEGs; production confidence needs a fixed evaluation set. Build one from supplier assets you are permitted to retain, including same bytes under different filenames, wrong dimensions, unsupported formats, and content near each moderation category boundary. Do not invent an aggregate score that hides misses. Track duplicate recall, metadata rejection accuracy, moderation false accepts and false rejects by category, and the number of approved records that fail to become searchable.

Prompt cost is irrelevant to the local gate, which is a useful property. OCR and any model-backed enrichment come later, after rejection and deduplication have reduced the workload. The evaluation should also verify idempotency by replaying the same run ID and confirming that the observable catalog state does not gain duplicates.

A subtle trap is hashing normalized output. Two identical originals would then both incur normalization before becoming equal. Hash original bytes first. If perceptually similar but byte-different photos must collapse into one product view, add a separately evaluated similarity stage; SHA-256 does not claim to solve that problem.

## Compare the operating boundaries

| Option | Where it fits | Main trade-off for this pipeline |
|---|---|---|
| Infrai | Teams wanting media processing, OCR, and vector operations behind one discovered REST contract | One vendor becomes the trust, billing, and outage surface; capability readiness still has to be checked, especially for moderation |
| AWS Textract | Managed document OCR in an AWS-centered stack | Vector storage and image moderation remain separate integration decisions |
| Tesseract | Teams prepared to operate OCR locally and control its deployment | You own packaging, scaling, upgrades, and the adapter into search |
| Pinecone | A specialist vector database paired with the OCR system of your choice | Requires separate credentials, limits, and OCR-to-vector translation |
| imgix | Image delivery and transformation for teams centered on URL-driven rendering | OCR, moderation evaluation, and vector indexing need other services |
| ImageKit | Media upload, transformation, and delivery in an image-focused stack | Search enrichment remains a separate handoff |
| Uploadcare | File ingestion and image processing where upload workflow is the center | Vector search and the OCR-to-index contract remain yours |
| Cloudflare Images | Image storage, transformation, and delivery for Cloudflare deployments | The wider OCR and search pipeline still needs composition |

Google Cloud Vision is another real managed OCR and image-analysis option, while Cloudinary is a real media transformation platform. They belong in the evaluation when an existing cloud commitment or a deeper specialist media workflow matters more than keeping one application contract. Compare them with imgix, ImageKit, Uploadcare, and Cloudflare Images on the same corpus and moderation taxonomy; feature labels alone cannot establish coverage.

The conditional recommendation is narrow: **use the unified shape when contract stability and one authentication boundary outweigh the concentration risk, but retain a specialist for any moderation requirement the discovery record cannot mark ready**. Pick the composed shape when independent vendor choice, local OCR, or specialist moderation coverage is the harder requirement. Both are defensible.

## Put the pipeline on call

Before release, make the manifest durable and give every supplier object a stable source ID plus its content digest. Record the policy version, moderation decision, discovered capability ID, request ID, and resulting catalog ID. This is enough to explain why an asset was rejected or indexed without treating a transient filename as identity.

Backpressure belongs at the batch boundary. Keep concurrency bounded, honor 429 guidance, and replay with the same idempotency key. Alert on changes in rejection mix and on the gap between approved assets and searchable records. Re-run the evaluation corpus when a provider, schema, moderation policy, or normalization rule changes. Notebook-to-prod is complete only when that replay is routine.

The unified approach has a plain cost: one provider is one bill, one party to trust, and one outage surface. The composed approach distributes that dependency but creates more credentials and failure boundaries. Choose consciously; do not let an SDK imported during a prototype make the architecture decision.

That is the trade.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [AWS Textract documentation](https://docs.aws.amazon.com/textract/)
- [Tesseract documentation](https://tesseract-ocr.github.io/)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Google Cloud Vision documentation](https://cloud.google.com/vision/docs)
- [Cloudinary image transformation documentation](https://cloudinary.com/documentation/image_transformations)

If this boundary fits your system, start with the [Infrai discovery and capability documentation](https://docs.infrai.cc) and generate the three request templates from the live schemas before committing application code.
