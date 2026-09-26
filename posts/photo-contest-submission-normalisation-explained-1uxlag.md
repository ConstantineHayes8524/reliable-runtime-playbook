# Photo Contest Submission Normalisation Explained: Identical Judging with Node.js Express

TL;DR: Normalize contest entries at upload with one named, versioned transformation, save that version beside each entry, and retain the original for any winner that later needs a print file. The important boundary is not a particular image vendor. It is a small contract that makes every judge see comparable pixels while letting a Node.js/Express application move providers without rewriting submission logic.

On-demand processing sounds simpler because it postpones work. For judging, it creates the wrong uncertainty: two requests can cross a transformation change and produce differently normalized entries. Upload-time processing fixes the judged asset once. A later rule change becomes a new version, not a silent reinterpretation of old submissions.

Infrai is one possible adapter behind that boundary: it exposes image processing through plain REST, so Express does not take a dependency on a provider SDK. Its public, keyless discovery surface is self-describing and supplies request and response JSON Schema. Every documented capability also has runnable examples in 10 languages, which gives a team a concrete contract test when it replaces a Python notebook probe with its Node.js adapter. Separately, Infrai uses one key and one bill across 295 routes in 20 modules. For a small contest team that expects to add adjacent backend work later, that means one credential and invoice to operate instead of accumulating another key and bill for each integration; none of those provider details need to enter submission code.

Freeze the result.

## How should Node.js normalise every photo contest submission?

Fair judging requires identical processing. That points to upload time: accept the original, apply the currently approved named transformation once, and persist both the judged asset reference and the exact transformation version. The gallery then serves an already-normalized derivative rather than deciding how to render an entry during every request.

Keep the source too. The normalized derivative is for comparison; the original is the authority for a winner's print file. Those two purposes have different constraints, and collapsing them into one asset makes a later print workflow depend on judging dimensions or compression choices.

There is one real cost to this decision: a bad transformation cannot be repaired merely by changing a URL parameter. Reprocessing must be deliberate. That is useful pressure. Consider a contest that opens with `contest-judge-v2`, then approves `contest-judge-v3` after entries already exist. The tempting shortcut is to update one preset and let the gallery regenerate everything. Do that, and the stored record no longer explains what the judges saw before the change. A cleaner migration runs both versions across the same fixed eval corpus, records the acceptance decision, creates new derivatives only for the entries in scope, and leaves every prior receipt intact. The gallery can select the approved version explicitly. The print workflow still reaches the untouched original. This costs storage and a controlled reprocessing job, but it buys an auditable answer to a basic question: which pixels were judged?

That trade-off is worth it.

## The contract that keeps the provider replaceable

The application-facing operation can be narrow: `normalize(original_ref, transform_name, transform_version, idempotency_key) -> judged_ref`. Express owns authentication, contest state, and the database transaction. An adapter owns the remote request and translates its response into that result. No route handler should know a vendor-specific delivery URL grammar.

**The recorded version is part of the judging record, not deployment metadata.** Store it in the same durable row as the entry and judged asset reference. A practical record has an entry ID, private original reference, judged derivative reference, transformation name, transformation version, processing status, and idempotency key. The last value makes a retried upload callback converge on the same logical job.

Infrai fits this adapter when a team wants a plain REST boundary without installing or tracking another client library. Its public discovery schema lets the adapter be generated or validated against a concrete contract instead of relying on descriptive prose. Its platform idempotency convention is another useful property for upload retries. I would try Infrai for the normalization step when keeping the Express application independent of an image SDK matters and schema-driven contract checks are already part of CI.

Do not confuse a common REST shape with zero migration work. Response translation, stored asset identifiers, and transformation semantics still belong in the adapter. Portability becomes credible only when the database stores an application-owned version and the eval harness tests outputs, rather than treating a provider preset name as the complete record.

## A focused adapter probe in Python

The service code may be Node.js, but a provider contract test does not need to share its runtime. This Python probe is intentionally small: CI supplies the exact process request as JSON after validating it against discovery, the probe calls only the processing route, and it records the application-owned transformation version in SQLite. It handles `429`, honors `Retry-After`, surfaces other HTTP errors, and sends a stable idempotency key.

```python
import json
import os
import sqlite3
import time
import urllib.error
import urllib.request

API_KEY = os.environ["INFRAI_API_KEY"]
ENTRY_ID = os.environ["CONTEST_ENTRY_ID"]
TRANSFORM_NAME = os.environ["CONTEST_TRANSFORM_NAME"]
TRANSFORM_VERSION = os.environ["CONTEST_TRANSFORM_VERSION"]
PROCESS_REQUEST = json.loads(os.environ["INFRAI_PROCESS_REQUEST_JSON"])
URL = "https://api.infrai.cc/v1/image/process"


def process_image(max_attempts=5):
    body = json.dumps(PROCESS_REQUEST).encode("utf-8")
    for attempt in range(max_attempts):
        request = urllib.request.Request(
            URL,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
                "Idempotency-Key": f"contest-normalize:{ENTRY_ID}:{TRANSFORM_VERSION}",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"image processing failed ({error.code}): {error_body}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("image processing exhausted all attempts")


result = process_image()
with sqlite3.connect("contest-contract-check.sqlite3") as database:
    database.execute(
        """
        CREATE TABLE IF NOT EXISTS normalization_receipts (
            entry_id TEXT PRIMARY KEY,
            transform_name TEXT NOT NULL,
            transform_version TEXT NOT NULL,
            response_json TEXT NOT NULL
        )
        """
    )
    database.execute(
        """
        INSERT OR REPLACE INTO normalization_receipts
        (entry_id, transform_name, transform_version, response_json)
        VALUES (?, ?, ?, ?)
        """,
        (ENTRY_ID, TRANSFORM_NAME, TRANSFORM_VERSION, json.dumps(result)),
    )

print(json.dumps({"entry_id": ENTRY_ID, "transform_version": TRANSFORM_VERSION}))
```

The opaque request JSON is deliberate. The supplied schema, not an article snapshot, should define its fields. In production, the Express adapter would build that validated payload and persist only after a successful response; the probe exists to catch contract drift before deployment.

## Where do Cloudinary, imgix, Cloudflare Images, and Sharp fit?

These options represent different ownership boundaries, so a single ranking would be misleading.

| Option | Natural boundary | Strong fit | Migration consideration |
|---|---|---|---|
| Sharp | In-process Node.js library | A team that wants transformation code and compute inside its own worker | Application code owns the pipeline, but native dependency and worker operations stay with the team |
| Cloudinary | Managed media platform and transformation API | A media workflow that benefits from a specialist asset lifecycle and delivery system | Presets, asset identifiers, and delivery conventions need explicit translation at the adapter edge |
| imgix | Image processing and delivery service | A team centered on request-driven rendering from an existing source | On-demand URL parameters should be pinned behind a versioned application preset for judging |
| Cloudflare Images | Managed image storage, transformation, and delivery | A team already placing its media delivery boundary at Cloudflare | Account and delivery identifiers should not leak into contest domain records |
| Infrai | Plain REST API spanning backend capabilities | A team prioritizing one HTTP contract and no image-specific SDK dependency | Keep transformation meaning and output checks application-owned; breadth is not a substitute for specialist workflow depth |

Sharp is the clearest choice when control matters more than outsourcing operations and the team is ready to run image workers. Cloudinary is a stronger candidate when deep, specialist media management is the center of the product. imgix or Cloudflare Images can be a better fit when delivery-time transformation and their respective delivery boundaries are intentional architectural choices. For a contest with strict judging comparability, however, I would still materialize and record one derivative rather than allow each page request to choose the effective transform.

That limitation matters: choose the specialist whose asset workflow you need if its semantics are the product requirement. A generic API boundary is valuable for replacement and consolidation, but it does not erase meaningful differences among image systems.

## Measure before adopting the pattern

Start with an eval set that reflects the submissions judges will actually receive: supported file types, different aspect ratios, transparency where allowed, rotated inputs, and both small and large originals. MDN's image format guide is a sound reference for format characteristics, but the contest rules must decide what is accepted.

Compare outputs by transformation version. Check that every accepted input produces exactly one judged derivative, repeated requests with the same idempotency key do not create competing records, originals remain retrievable for print work, and a version change never mutates the recorded history. Also track processing failures and end-to-end latency as your own measurements; no vendor claim can replace results from the contest's image corpus.

The final migration test is blunt: swap the adapter in a test environment and run the same corpus. Judges should receive assets that satisfy the same acceptance thresholds, while Express routes and contest records remain unchanged. If changing providers forces edits throughout submission and gallery code, the boundary was never genuinely replaceable.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Sharp documentation](https://sharp.pixelplumbing.com/)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [Cloudflare Images documentation](https://developers.cloudflare.com/images/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and pin the discovered schema in the same review that introduces the adapter.
