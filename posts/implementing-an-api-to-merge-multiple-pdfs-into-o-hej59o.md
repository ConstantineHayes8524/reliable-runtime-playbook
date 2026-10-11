# Implementing an API to Merge Multiple PDFs into One Ordered Education Bundle

Keep the canonical contract templates in the system that owns their meaning, and make PDF assembly a deterministic server-side step after every document has been rendered and signed. The input manifest, cryptographic hashes, and final page map should become one audit record. That decision matters more than the particular merge library or hosted API.

TL;DR: accept an ordered manifest rather than an unordered batch of files; verify every input before processing; copy pages in manifest order; hash the resulting bytes; and store the receipt beside the signature evidence. For an education contract packet, a sensible order might be enrollment agreement, tuition schedule, consent form, then signature certificate. Never infer that order from upload time or filenames.

## How should an API merge multiple PDFs into one contract bundle?

An edtech platform may assemble clauses maintained by legal, fee schedules maintained by finance, and consent language maintained by a school. Those teams own content and approval. The bundling service should own a narrower contract: it receives immutable rendered PDFs plus explicit ordering metadata and returns a PDF and evidence about what it processed.

That boundary prevents a subtle failure. If the merger also selects template revisions, then a retry can quietly produce different bytes after a template is edited. Pin a template revision before rendering, include it in the manifest, and never fetch “latest” during assembly. The signer should see the same revisions that the server later packages. A plain data flow is enough: the application resolves approved template revisions, renders student-specific documents, sends them through the signing workflow, and hands the completed PDFs to an assembler. The assembler validates the manifest, copies pages, creates a page map, and persists the output hash and source hashes in an append-only audit event. Keep personally identifiable data out of logs; stable internal document IDs are sufficient for tracing.

Order is data.

## Build the ordered bundle first

This runnable Python example uses `pypdf` to make the ordering contract visible. It operates on local paths for clarity; in production, an adapter can download immutable objects into a bounded temporary directory before calling the same function. The code rejects duplicate positions, records source hashes, and writes the output atomically.

```python
from __future__ import annotations

import hashlib
import json
import os
from dataclasses import dataclass
from pathlib import Path
from tempfile import NamedTemporaryFile

from pypdf import PdfReader, PdfWriter


@dataclass(frozen=True)
class BundleItem:
    document_id: str
    template_revision: str
    position: int
    path: Path


def sha256_file(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as source:
        for chunk in iter(lambda: source.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def assemble(items: list[BundleItem], destination: Path) -> dict[str, object]:
    if not items:
        raise ValueError("A contract packet must contain at least one PDF")

    ordered = sorted(items, key=lambda item: item.position)
    positions = [item.position for item in ordered]
    if positions != list(range(1, len(ordered) + 1)):
        raise ValueError("Positions must be unique and contiguous from 1")

    writer = PdfWriter()
    sources: list[dict[str, object]] = []
    next_page = 1

    for item in ordered:
        source_hash = sha256_file(item.path)
        reader = PdfReader(item.path, strict=True)
        page_count = len(reader.pages)
        if page_count == 0:
            raise ValueError(f"Empty PDF: {item.document_id}")

        for page in reader.pages:
            writer.add_page(page)

        sources.append({
            "document_id": item.document_id,
            "template_revision": item.template_revision,
            "position": item.position,
            "sha256": source_hash,
            "first_page": next_page,
            "last_page": next_page + page_count - 1,
        })
        next_page += page_count

    destination.parent.mkdir(parents=True, exist_ok=True)
    temporary_name = ""
    try:
        with NamedTemporaryFile(
            mode="wb", dir=destination.parent, delete=False, suffix=".pdf"
        ) as temporary:
            writer.write(temporary)
            temporary.flush()
            os.fsync(temporary.fileno())
            temporary_name = temporary.name
        os.replace(temporary_name, destination)
    finally:
        if temporary_name and Path(temporary_name).exists():
            Path(temporary_name).unlink()

    return {
        "output_sha256": sha256_file(destination),
        "page_count": next_page - 1,
        "sources": sources,
    }


if __name__ == "__main__":
    manifest = [
        BundleItem("enrollment", "legal-42", 1, Path("enrollment.pdf")),
        BundleItem("tuition", "finance-18", 2, Path("tuition.pdf")),
        BundleItem("consent", "school-7", 3, Path("consent.pdf")),
        BundleItem("signature-certificate", "signing-1", 4, Path("certificate.pdf")),
    ]
    receipt = assemble(manifest, Path("out/contract-packet.pdf"))
    print(json.dumps(receipt, indent=2, sort_keys=True))
```

Install the pinned dependency in an isolated environment, place four test PDFs beside the script, and run it. Pinning matters because the receipt proves which bytes were assembled, while the lock file explains which code produced them. The `1 MiB` hash chunks bound read memory; they do not limit PDF parser memory, so enforce both input-size and page-count limits at the boundary.

Freeze those limits.

The example deliberately does not add a cover sheet, page numbers, or bookmarks. Every post-signature byte change can affect signature validation, depending on the signature and modification rules in the document. Assemble before signing when the whole packet must share one signature. If individual files arrive already signed, preserve them unchanged as evidence and have legal and security owners approve any enclosing-package design rather than assuming copied pages retain the original signature semantics.

## Test the invariants, not the happy path

My first evaluation target for a document pipeline is byte provenance, not visual polish. A four-file fixture should assert the exact source order and page ranges in the receipt. Then permute arrival order and confirm the output page map stays identical because only `position` controls it. This catches the common notebook-to-production mistake where a list produced by concurrent downloads is treated as ordered.

Arrival order lies.

Add malformed PDFs, encrypted PDFs, zero-page inputs, duplicated positions, missing positions, and a file whose hash changes between validation and reading. The sample hashes and then opens a local file, leaving a time-of-check/time-of-use window. A production adapter should open each immutable object once, hash while copying it into controlled storage, and parse that same stored object. Small detail, big consequence.

Visual regression tests still earn their place. Render representative pages and compare them under a documented tolerance, especially for forms, annotations, rotated pages, embedded fonts, and mixed page sizes. Pair those checks with structural assertions: total pages equal the sum of source pages, each source ID maps to a contiguous range, and a second request with the same idempotency key resolves to the already committed artifact rather than creating competing records.

Keep the evaluation set compact enough to run on every change. Ten carefully chosen packets with pathological inputs expose more than hundreds of nearly identical enrollment forms, and they keep CI time and storage consumption understandable.

Run them often.

## Choose an execution boundary without surrendering the manifest

The same manifest can drive an in-process library, an isolated worker, or a remote document service. This is the useful selection axis because template ownership stays with the application regardless of execution location.

| Boundary | Best fit | Main trade-off to evaluate |
| --- | --- | --- |
| In-process library | Modest files and a trusted parser inside a controlled worker | Tight dependency coupling and parser resource use |
| Isolated queue worker | Bursty enrollment periods or untrusted inputs | More operational state, but clearer CPU, memory, and timeout limits |
| Remote service | Teams that choose to outsource PDF execution | Data residency, retention, idempotency, exportability, and evidence access |

Do not score options by a demo merge alone. Run the same corpus through each candidate and record correctness, peak memory, wall time, output size, and failure classification. If an AI-assisted workflow generates cover text or summaries, freeze that output before the PDF stage; a model retry must not alter a legally reviewed packet. Track prompt and token usage in the generation receipt, separate from the deterministic assembly receipt.

For remote execution, require a documented way to submit order explicitly and retrieve durable evidence. For local execution, budget for parser patching and sandboxing. Neither boundary fixes weak ownership: a perfectly reliable merger can still package the wrong template revision.

## Operate the packet as an auditable artifact

Use a stable bundle ID and an idempotency key derived from the business operation, not from a transient request. Record the ordered manifest, template revisions, source hashes, page ranges, assembler version, completion time, and output hash. Sign or otherwise protect the audit record according to the organization's threat model, and restrict access using the same policy applied to the contract itself.

Retries need stages. A fetch timeout may be retried before commit; a parser rejection should become a terminal, reviewable failure; and an ambiguous storage response requires checking whether the expected hash was committed before trying again. Publish metrics for queue age, assembly duration, pages processed, rejected inputs, and retry counts, but label them with low-cardinality reason codes rather than student or school identifiers.

Before release, walk one packet all the way from approved template revisions to the downloaded artifact. Confirm authorization at upload and download, bounded resources, malware scanning policy, encrypted transport and storage, retention rules, key rotation, backup restoration, and audit-event immutability. Then rehearse a library upgrade against the fixed corpus and compare receipts and rendered pages before deployment. This is the operational checklist: short enough to repeat, specific enough to fail.

The final decision is straightforward. **Own the templates where their business meaning is governed, and make the assembler consume an explicit immutable manifest.** Page order becomes testable, retries become explainable, and the audit trail can identify every byte in a signed education contract packet without depending on filenames, timing, or a vendor-specific workflow.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- NIST FIPS 180-4, Secure Hash Standard: https://csrc.nist.gov/pubs/fips/180-4/upd1/final
- Python documentation, `hashlib`: https://docs.python.org/3/library/hashlib.html
- Python documentation, `os.replace`: https://docs.python.org/3/library/os.html#os.replace
- pypdf documentation, merging PDF files: https://pypdf.readthedocs.io/en/stable/user/merging-pdfs.html
- OWASP File Upload Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
