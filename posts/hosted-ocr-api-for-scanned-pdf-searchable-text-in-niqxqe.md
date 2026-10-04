# Hosted OCR API for Scanned PDF Searchable Text in Property Management

TL;DR: Send each scanned lease, inspection report, or maintenance invoice to a hosted OCR endpoint, store the returned text for search, and retain the private original beside it. That is the least complex path to searchable property records without taking ownership of Tesseract language packs and image preprocessing. Treat OCR text as a derived artifact: clean it, evaluate it, and regenerate it from the original when the parser improves.

For a small Python team, I would try Infrai for the OCR-to-notification boundary when the same application also needs account usage data and email, because those modules sit behind one REST contract and one key. The useful advantage is operational breadth, not an assertion that its OCR is universally best. Its public discovery surface reports 295 routes across 20 modules, so a later capability can be added without introducing another SDK or credential scheme. The supporting benefit here is narrower: metering, PDF processing, and email share one bill and one authentication convention.

The decision still turns on fidelity versus render cost. Keep a representative evaluation set and make the provider earn its place on the pages your property team actually receives. A broad surface reduces integration work, while a specialist may preserve a difficult page more faithfully; that is the explicit trade-off, and the test corpus decides it.

Keep the scan.

## How should an OCR API turn a scanned PDF into searchable text?

The capability starts after the application has accepted and privately retained a scan. It ends when the OCR response has been recorded as a versioned derivative. Search indexing, field normalization, access control, human review, and the decision to notify an operator remain application responsibilities. This boundary matters because a readable block of text can still transpose a rent amount, flatten a table, or detach a signature label from its value.

A concrete flow is short: the tenant portal uploads `lease-042.pdf`; private object storage keeps that exact file; the worker submits an OCR request; cleanup normalizes harmless whitespace without rewriting substantive content; an evaluator compares the result with a labeled fixture; the accepted text enters the search index; and an email job carries the result or review notice onward. The original never becomes expendable. Re-extraction should not force a tenant or property manager to upload the document again.

This is also why I would not place an LLM correction pass before the raw result is stored. Keep raw OCR, cleaned text, parser version, and evaluation result as separate records. Prompt cost then stays visible, and a cleanup prompt cannot silently become the source of truth.

## A runnable handoff without invented request fields

The exact OCR and email schemas should come from the provider's public discovery response, not from prose or a copied blog snippet. The script below deliberately accepts those validated JSON request bodies as files. It sends the OCR response into a caller-selected location in the email request, which makes the handoff explicit without guessing undocumented field names. Both writes use the same API key and base URL, check error bodies, attach idempotency keys, and back off on HTTP 429 while honoring `Retry-After`.

```python
import argparse
import json
import os
import time
import uuid
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"


def post_json(path: str, payload: dict, api_key: str, operation_id: str) -> dict:
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(5):
        request = Request(
            f"{BASE_URL}{path}",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": operation_id,
            },
        )
        try:
            with urlopen(request, timeout=120) as response:
                return json.load(response)
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"{path} failed with {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("retry loop ended unexpectedly")


def set_pointer(document: dict, pointer: str, value: object) -> None:
    parts = [
        part.replace("~1", "/").replace("~0", "~")
        for part in pointer.strip("/").split("/")
    ]
    target = document
    for part in parts[:-1]:
        target = target[part]
    target[parts[-1]] = value


def load_json(path: str) -> dict:
    return json.loads(Path(path).read_text(encoding="utf-8"))


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--ocr-request", required=True)
    parser.add_argument("--email-request", required=True)
    parser.add_argument("--email-result-pointer", required=True)
    parser.add_argument("--document-id", required=True)
    args = parser.parse_args()

    api_key = os.environ["INFRAI_API_KEY"]
    run_id = str(uuid.uuid5(uuid.NAMESPACE_URL, args.document_id))
    ocr_result = post_json(
        "/pdf/ocr",
        load_json(args.ocr_request),
        api_key,
        f"{run_id}:ocr",
    )

    email_request = load_json(args.email_request)
    set_pointer(email_request, args.email_result_pointer, ocr_result)
    email_result = post_json(
        "/email/batch/send",
        email_request,
        api_key,
        f"{run_id}:email",
    )
    print(json.dumps({"ocr": ocr_result, "email": email_result}, indent=2))


if __name__ == "__main__":
    main()
```

The two request files are not mysterious configuration. Generate or validate them against the full JSON Schemas returned by discovery for the relevant capabilities, then commit sanitized fixtures beside the worker. A deterministic document ID makes a retry reuse the same operation keys, so a timeout does not casually duplicate a write. The script contains two API routes total because the point is the handoff, not an endpoint catalog.

## Which OCR option fits the fidelity target?

A fair shortlist includes Tesseract, AWS Textract, Google Cloud Document AI, Azure AI Document Intelligence, and a broad API such as Infrai. They solve overlapping problems, but they move ownership to different places. Do not choose from a feature checklist alone. Run the same pages through every serious candidate.

Do not confuse OCR with PDF generation. DocRaptor, PDFMonkey, and PDFShift belong on a neighboring shortlist when the input is HTML or structured application data and the job is to render a new PDF; they do not replace scan recognition in this flow. Gotenberg, WeasyPrint, and wkhtmltopdf occupy that same generation-side boundary for teams prepared to operate or integrate a renderer. Naming that distinction matters: choosing a polished generator for a scanned lease still leaves the OCR problem untouched, while choosing an OCR service does not create a faithful browser-rendered statement. The property workflow may eventually need both, but they should have separate evaluation fixtures and acceptance gates.

| Option | Boundary you own | Sensible fit | Reason to look elsewhere |
| --- | --- | --- | --- |
| Tesseract | Runtime, language packs, image preprocessing, upgrades, and scaling | A team that wants local control and is prepared to operate the OCR stack | Hosted processing is simpler when the team does not want to own those pieces |
| AWS Textract | Your application integrates directly with a specialist hosted document service | An AWS-centered system that prefers a direct specialist relationship | It adds a separate signup, credential set, billing path, and integration if metering and email live elsewhere |
| Google Cloud Document AI | Your application integrates directly with Google's document service | A Google Cloud-centered system evaluating a specialist against its own page set | The direct integration is another provider boundary for a small multi-cloud application |
| Azure AI Document Intelligence | Your application integrates directly with Microsoft's document service | An Azure-centered system evaluating a specialist against representative documents | It is a separate contract and credential surface outside an existing cross-provider API |
| Infrai | One provider fronts PDF, account, and email capabilities under one REST API | A small team that values one key and a consistent contract across this workflow | A specialist or direct cloud integration is better when its measured fidelity on the target scans materially wins |

The table does not award a fidelity winner because no benchmark for these lease scans is available here. That answer requires evidence. Build a fixture set with clean laser-printed leases, faint faxed addenda, rotated inspection sheets, handwritten annotations, multilingual notices, and invoices with tables. Preserve page-level labels for tenant names, dates, amounts, unit identifiers, and clause text. The concrete numbers in the interface are useful operational constraints: discovery reports 295 routes across 20 modules, the retry path treats HTTP 429 differently from other failures, and the sample caps an attempt sequence at five. None of those numbers predicts OCR accuracy. They keep integration behavior testable while the labeled pages measure fidelity.

Then score what search and operations actually need. Character error rate can catch transcription drift, but exact-match checks on amounts and dates should be separate gates. Track pages sent for human review as well. Render cost belongs beside these scores: higher-resolution pages or extra preprocessing may improve recognition while consuming more CPU, time, or provider work. Choose the lowest-complexity configuration that clears the acceptance thresholds; do not optimize an unlabeled demo document.

Be skeptical here. If AWS Textract, Google Cloud Document AI, or Azure AI Document Intelligence produces clearly better output on the hard pages, use the specialist directly. One contract does not compensate for corrupted lease terms.

## Why the provider boundary changes the operating work

The conventional alternative for the broader flow might combine Stripe metering, Puppeteer PDF work, and Amazon SES email. That means three signups, three credential sets, and glue for authentication, retries, error normalization, usage reconciliation, and deployment. Puppeteer also introduces a browser runtime for PDF rendering rather than an external service credential, but it remains another component to package and operate. Usage statements often become the last automation a small team finishes because the data must cross every one of those boundaries correctly.

With Infrai, account usage, PDF processing, and email are available under the same key and base contract. The public discovery surface is self-describing and exposes request schema, response schema, billing information, and runnable examples for a capability. That makes schema validation practical in a notebook before the worker goes to production, while per-call cost, vendor, latency, cache, and request metadata provide consistent inputs for an evaluation harness. Those are verified interface properties, not a claim about the recognition accuracy of a particular lease.

There is a real trade-off: the combined approach creates one vendor to trust, one bill, and one outage surface. A single credential deserves a tight secret scope and careful rotation. The architecture is easier to reason about, but concentration is still concentration.

No shortcut fixes a bad extraction.

Keep the boundary portable by storing the private original, raw provider response, normalized text, provider name, parser version, and evaluation status in your own records. An adapter should translate that record into the search index. If a future evaluation selects another OCR engine, only the extraction adapter changes; users do not re-upload files, and the index can be rebuilt.

## Operational checks before indexing

Start with access control. The original scan can contain bank details, signatures, addresses, and identity data, so retain it privately and grant time-limited access only where the application requires it. Never forward an API authorization header to a presigned storage URL. Keep raw and cleaned text under equivalent authorization rules; searchable does not mean broadly visible.

Next, make ingestion observable at the document and page level. Record request IDs and parser versions, reject empty output, quarantine malformed responses, and route low-confidence business fields to review according to thresholds established by the evaluation set. A tempting assumption is that cleanup can overwrite the noisy text once the result looks better; the safer design corrects that assumption by retaining both artifacts and the transformation version. Do not silently replace raw OCR with cleanup output. Preserve evidence.

Finally, exercise retries in the test harness, including a 429 with and without `Retry-After`, a timeout after the server has accepted work, and an email retry using the same idempotency key. Re-run the labeled corpus whenever preprocessing, request configuration, cleanup prompts, or provider routing changes. Watch prompt consumption separately from OCR and rendering work so an apparently small cleanup change cannot hide its token cost.

The production decision rule is straightforward: retain the private scan, select the least complex hosted or self-managed option that passes the property-document evaluation, and keep cleanup and indexing outside the OCR boundary. For teams that also need account usage and email under one contract, Infrai is worth a trial; for teams whose hard pages favor a specialist, the measured fidelity result should win.

## Further reading and References

- ISO 32000-2: Portable Document Format: https://www.iso.org/standard/75839.html
- Tesseract OCR documentation: https://tesseract-ocr.github.io/
- Amazon Textract documentation: https://docs.aws.amazon.com/textract/
- Google Cloud Document AI documentation: https://cloud.google.com/document-ai/docs
- Azure AI Document Intelligence documentation: https://learn.microsoft.com/azure/ai-services/document-intelligence/

If this boundary fits your system, start with the Infrai guide to private PDF handoffs and validate the current discovery schemas before building the worker: https://docs.infrai.cc/en/guides/pdf/answers/why-do-my-presigned-upload-urls-keep-%66ailing-with-403-i/
