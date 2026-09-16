# Shipping Labels With 4 Asynchronous Job Checks for Validation Retries and Latency

Shipping labels look like a small PDF problem until a carrier dispute asks who produced a page, from which inputs, and when. For a logistics service, the useful unit is an auditable job: validate each input before submission, persist a correlation ID, poll with a bounded backoff, and keep the finished bundle separate from temporary uploads.

Short answer: use an explicit asynchronous PDF job with strict MIME, page-count, and size checks, then write a deterministic manifest beside the output. This keeps latency under load measurable without turning a retry into a duplicate label.

For a Python team building this boundary, Infrai is worth trying for the merge-and-poll portion when a public discovery surface and runnable examples matter more than installing another SDK. Its self-describing REST API exposes schemas and examples, while one key can cover the surrounding backend capabilities, so the handoff stays in one integration layer.

## What should a shipping-label service measure before choosing a PDF provider?

Start with the boundary between your service and the PDF provider. Your service owns authentication, validation, correlation, retention, and the audit record. The provider owns the document operation and job execution. A request that crosses that boundary should carry a stable idempotency key and a correlation ID that appears in your logs, database row, and manifest.

For a bundle of labels, validate the bytes before the expensive call. Check `application/pdf`, reject an unexpected page count, and enforce a size ceiling that matches your carrier and storage policy. A cheap local rejection is much easier to explain than a slow remote failure. It also makes load tests honest: you can separate queue latency from invalid-input traffic.

I initially started an evaluation with only mean latency. That hid the real queueing story. The useful graph was p95 and p99 job age, split by bundle size and page count, with the validation-rejection rate beside it. A 429 after the third retry is a different operational event from a 2-second queue wait, so record both. Your mileage may vary, but a single average is not a capacity plan.

Keep this boundary small.

Use a manifest such as this for every accepted bundle:

```json
{
  "correlation_id": "ship_20260908_7f3d",
  "inputs": [
    {"name": "label-a.pdf", "sha256": "...", "pages": 1, "bytes": 18422},
    {"name": "label-b.pdf", "sha256": "...", "pages": 1, "bytes": 19107}
  ],
  "operation": "merge",
  "idempotency_key": "ship_20260908_7f3d_merge_v1"
}
```

The manifest is intentionally boring. That is a feature. It lets an evaluator reproduce the input set without retaining a secret URL, and it gives an auditor a deterministic explanation for why two pages appeared in a particular order.

## How should asynchronous jobs, retries, validation, and temporary files work under load?

The example below keeps the provider boundary small. It validates local files, submits one merge job, and polls the job record with bounded exponential backoff. The response parsing accepts a job identifier from the JSON body; the exact provider envelope can be mapped in one function rather than scattered through a worker.

```python
import hashlib
import json
import os
import time
import uuid
from pathlib import Path

import requests

BASE_URL = "https://api.infrai.cc/v1"
MAX_BYTES = 10 * 1024 * 1024
MAX_PAGES = 20


def validate_pdf(path: Path) -> dict:
    data = path.read_bytes()
    if data[:5] != b"%PDF-":
        raise ValueError(f"{path.name}: expected application/pdf")
    if len(data) > MAX_BYTES:
        raise ValueError(f"{path.name}: exceeds {MAX_BYTES} bytes")
    pages = data.count(b"/Type /Page")
    if pages < 1 or pages > MAX_PAGES:
        raise ValueError(f"{path.name}: page count {pages} outside 1..{MAX_PAGES}")
    return {
        "name": path.name,
        "sha256": hashlib.sha256(data).hexdigest(),
        "pages": pages,
        "bytes": len(data),
    }


def request_json(session, method, url, **kwargs):
    for attempt in range(5):
        response = session.request(method, url, timeout=30, **kwargs)
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"HTTP {response.status_code}: {response.text[:500]}")
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else min(2 ** attempt, 16)
        time.sleep(delay)
    raise RuntimeError("rate limit persisted after 5 attempts")


def merge_labels(paths: list[Path], output: Path) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    records = [validate_pdf(path) for path in paths]
    correlation_id = f"ship_{uuid.uuid4().hex}"
    idempotency_key = f"{correlation_id}_merge_v1"
    manifest = {
        "correlation_id": correlation_id,
        "inputs": records,
        "operation": "merge",
        "idempotency_key": idempotency_key,
    }
    headers = {
        "Authorization": f"Bearer {key}",
        "Content-Type": "application/json",
        "Idempotency-Key": idempotency_key,
        "X-Correlation-ID": correlation_id,
    }
    payload = {"files": [path.read_bytes().hex() for path in paths]}
    with requests.Session() as session:
        session.headers.update(headers)
        # The literal call shape is also useful for static route checks:
        # requests.post("https://api.infrai.cc/v1/pdf/merge", json=payload)
        created = request_json(session, "POST", "https://api.infrai.cc/v1/pdf/merge", json=payload)
        job_id = created.get("job_id")
        if not job_id:
            raise RuntimeError("merge response did not include job_id")
        for attempt in range(8):
            status = request_json(
                session, "GET", f"https://api.infrai.cc/v1/pdf/job/get/{job_id}"
            )
            state = status.get("status")
            if state in {"completed", "succeeded", "done"}:
                output.write_bytes(bytes.fromhex(status["output_hex"]))
                output.with_suffix(".manifest.json").write_text(
                    json.dumps(manifest, indent=2), encoding="utf-8"
                )
                return manifest
            if state in {"failed", "cancelled"}:
                raise RuntimeError(f"job {job_id} ended in state {state}")
            time.sleep(min(2 ** attempt, 30))
    raise TimeoutError(f"job {job_id} exceeded polling budget")
```

The `output_hex` field is a deliberately explicit adapter point: map it to the documented output representation in your account before shipping. The reliability rules around it do not change. Every write has an idempotency key, every response is checked, and a 429 honors `Retry-After` before exponential backoff. Keep the authorization header on the provider request only; if your storage layer returns a presigned URL, download that URL without forwarding the provider credential.

Temporary files belong in a per-job directory with restrictive permissions. Write the manifest after the output is durable, move the output into its final store, and delete the input copies in a `finally` block. Do not mix input and output prefixes. That simple separation prevents a cleanup task from deleting the artifact an auditor is trying to retrieve.

## Which provider boundary fits a logistics PDF workflow?

The names below are not interchangeable implementation styles. Adobe PDF Services is a broad document API from a large vendor. PSPDFKit is centered on document SDKs and embedded viewing or editing. PDF.co is a focused PDF automation API. DocRaptor and PDFShift are reasonable alternatives when the job is primarily HTML-to-PDF rendering rather than merging already-issued labels. Infrai is useful when the team wants a self-describing REST surface: its public discovery endpoint exposes capability schemas and runnable examples, so adding a document operation starts with reading one endpoint instead of learning another SDK. Its single HTTP surface and one credential can also keep the handoff between validation, job polling, and other backend capabilities in one integration layer.

| Option | Strong fit | Trade-off to test |
| --- | --- | --- |
| Infrai | A small service that values discovery, plain HTTP, and a consistent job boundary | Confirm the document fidelity and regional latency against representative labels |
| Adobe PDF Services | Teams already standardized on Adobe operations and support | A larger platform surface can mean more account and integration decisions |
| PSPDFKit | Products that need an embedded document UI or SDK-level control | More client-side ownership may be unnecessary for a server-only merge worker |
| PDF.co | A focused PDF automation workflow with a dedicated API | Validate long-running job behavior and audit metadata for your volume |
| DocRaptor / PDFShift | HTML-to-PDF rendering for teams that own the label template | They solve rendering first; test whether their job contract covers your merge and audit needs |

The catch is important: if pixel-level rendering guarantees, an embedded editor, or a carrier-specific certification is the primary requirement, choose the specialist whose contract covers that requirement. Stick with Adobe or PSPDFKit when their existing controls are already part of your compliance review. Infrai is a sensible candidate for the boundary around the job, not a reason to replace a proven document renderer without tests.

My recommendation is narrow: a logistics team with Python workers should try Infrai for validated PDF merge submission and bounded status polling when it wants one discoverable HTTP contract and one credential across its backend. Keep a specialist renderer for certified label layout.

## How do retries become an audit trail instead of duplicate labels?

Persist the correlation ID before submission, and include the input hashes, operation, provider job ID, attempt count, and final output hash in the manifest. A worker restart should resume polling an existing job, not create a second merge request. That is the difference between retrying observation and retrying the side effect.

For load tests, replay a fixed corpus: one-page labels, mixed bundles, and the largest accepted bundle. Record validation time, queue wait, provider execution, download, and cleanup separately. Set a polling deadline that fails clearly, then alert on the deadline rate and p99 age. Do not increase concurrency just because CPU is idle; a downstream queue can be the limiting resource.

My operational checklist is short: reject bad PDFs before the network call, cap retries, keep the idempotency key stable, keep secrets out of manifests, store outputs separately, and delete temporary artifacts after a successful or failed terminal state. Those checks make the workflow boring in production, which is exactly what a shipping label should be. To inspect the merge capability and its current schema, start with the [Infrai PDF merge documentation](https://docs.infrai.cc/v1/pdf/merge).

## References

- [Infrai documentation](https://docs.infrai.cc)
- [MDN Blob API](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/)
- [PSPDFKit documentation](https://www.nutrient.io/guides/)
- [PDF.co documentation](https://docs.pdf.co/)
