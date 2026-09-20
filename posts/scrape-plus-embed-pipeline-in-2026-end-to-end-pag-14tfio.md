# Scrape Plus Embed Pipeline in 2026 — End-to-End Page Change Retrieval Explained

Short answer: a scrape plus embed pipeline looks like a versioned sequence: fetch the page, extract meaningful text, detect changes, chunk and embed the changed version, then retrieve its passages. Store each chunk with a page URL, fetch time, and content version; retrieve against the active version when an analyst asks what changed. For a B2B market-research tool that watches competitor pages, this keeps alerts responsive without letting yesterday's copy appear alongside today's. The hard trade-off is retrieval quality versus latency: smaller, more frequent updates improve freshness but increase processing and indexing work.

The data flow is fetch -> extract -> compare -> chunk -> embed -> index -> retrieve -> explain the diff. A change alert and a retrieval answer share provenance, but they are different outputs. The alert compares two page versions; retrieval finds relevant passages within the version the answer is supposed to describe. Retrieval-augmented generation uses retrieved material as context for generation, not as proof that the underlying source is current.

## What does a scrape plus embed pipeline look like end to end?

Here is a runnable Python sketch. It accepts a URL and a previous snapshot file, extracts visible text, reports changed lines, and builds a tiny in-memory retrieval index. Its token-count vectors are deliberately *not* semantic embeddings: they make the pipeline executable with the standard library, while keeping the `embed` boundary obvious. Replace that function with a real embedding model and persist the vectors before calling this a production search system. Check the site's crawling rules and access terms before fetching it.

```python
import hashlib
import json
import math
import re
import sys
import urllib.request
from collections import Counter
from difflib import unified_diff
from html.parser import HTMLParser
from pathlib import Path


class VisibleText(HTMLParser):
    def __init__(self):
        super().__init__()
        self.hidden = 0
        self.parts = []

    def handle_starttag(self, tag, attrs):
        if tag in {"script", "style", "noscript"}:
            self.hidden += 1
        elif tag in {"p", "h1", "h2", "h3", "li"}:
            self.parts.append("\n")

    def handle_endtag(self, tag):
        if tag in {"script", "style", "noscript"} and self.hidden:
            self.hidden -= 1
        elif tag in {"p", "h1", "h2", "h3", "li"}:
            self.parts.append("\n")

    def handle_data(self, data):
        if not self.hidden:
            self.parts.append(data)


def embed(text):
    return Counter(re.findall(r"[a-z0-9]+", text.lower()))


def similarity(left, right):
    dot = sum(value * right.get(word, 0) for word, value in left.items())
    norm = math.sqrt(sum(v * v for v in left.values()))
    norm *= math.sqrt(sum(v * v for v in right.values()))
    return dot / norm if norm else 0.0


def chunks(lines, size=8, overlap=2):
    step = size - overlap
    return ["\n".join(lines[i:i + size]) for i in range(0, len(lines), step)]


def main(url, snapshot_path, question):
    request = urllib.request.Request(url, headers={"User-Agent": "ResearchChangeMonitor/1.0"})
    with urllib.request.urlopen(request, timeout=15) as response:
        html = response.read().decode(response.headers.get_content_charset() or "utf-8", errors="replace")
    parser = VisibleText()
    parser.feed(html)
    lines = [" ".join(line.split()) for line in "".join(parser.parts).splitlines()]
    lines = [line for line in lines if line]
    text = "\n".join(lines)
    version = hashlib.sha256(text.encode("utf-8")).hexdigest()
    path = Path(snapshot_path)
    previous = json.loads(path.read_text()) if path.exists() else None
    if previous and previous["version"] != version:
        print("\n".join(unified_diff(previous["lines"], lines, fromfile="previous", tofile="current")))
    elif previous:
        print("No extracted-text change")
    else:
        print("Initial snapshot; no prior version to diff")
    records = [{"url": url, "version": version, "text": part, "vector": embed(part)}
               for part in chunks(lines)]
    matches = sorted(records, key=lambda item: similarity(embed(question), item["vector"]), reverse=True)
    for match in matches[:3]:
        print(json.dumps({"url": match["url"], "version": match["version"],
                          "score": similarity(embed(question), match["vector"]),
                          "text": match["text"]}))
    path.write_text(json.dumps({"url": url, "version": version, "lines": lines}))


if __name__ == "__main__":
    if len(sys.argv) != 4:
        raise SystemExit("Usage: python monitor.py URL SNAPSHOT.json QUESTION")
    main(*sys.argv[1:])
```

Run it twice against an authorized page to see the first snapshot and a subsequent comparison. The example writes a local snapshot and prints the top three lexical matches; it does not generate an answer or claim that token counts capture meaning.

Words overlap. Meaning might not.

A notebook demonstration can appear accurate because the question repeats words from the page, then miss a paraphrase entirely. In a deployed monitor, this distinction changes what the analyst sees: a question about a revised cancellation clause might retrieve an unchanged paragraph about account cancellation merely because its words match, while the actual revised clause uses different phrasing. Keep the diff as a separate evidence path, and include known paraphrases in evaluation before relying on the retrieval ranking.

## Why can a correct fetch still produce a wrong alert?

HTML is not the content model. Navigation links, legal footers, timestamps, and rotating banners can change while the underlying offer stays put. This example strips scripts and styles, but its extraction is intentionally crude: it can miss text inserted by client-side rendering, retain repeated navigation, and flatten a comparison table into ambiguous lines. Decide which page regions constitute the watched claim before deciding whether a hash change warrants an alert. Preserve the raw response separately so an extractor revision does not erase the evidence needed to audit an old alert.

For market research, a page version should tie together the source URL, extraction rules, normalized text, and retrieval records. Commit a new version only after all its chunks are indexed; otherwise a query can mix old and new evidence. Imagine the page changing its enterprise contract term in one section while the index still holds the preceding version of another section: an unfiltered answer can combine those fragments into a contract that never existed. Keep the old snapshot long enough to produce a diff, but filter retrieval to the requested version. A useful alert cites both changed passages and their page versions. If the extracted text is empty after a successful HTTP response, treat that as an extraction failure, not a deletion of every product claim. This is a correctness boundary, so a slower committed update is preferable to a faster, contradictory answer.

Conditional HTTP requests can avoid transferring an unchanged representation when a server provides validators such as ETag or Last-Modified. A `304 Not Modified` means the cached representation may be reused under the conditional request, not that an extractor or embedding model change can be skipped.

Keep those checks distinct.

Similarly, robots.txt defines crawler access rules, not the accuracy of extracted content.

## Where should chunking and embeddings enter the pipeline?

Diff before embedding when a stable extraction lets you identify unchanged content. Then chunk on section boundaries where possible, include the heading with each passage, and store a stable identifier derived from the page version and chunk content. The eight-line window and two-line overlap in the sketch make the boundary visible, but lines are not semantic units. A pricing comparison cell separated from its column label will retrieve badly even if the vector itself is excellent.

Choose a real embedding model with a held-out set of actual analyst questions and source passages. Index and query vectors must come from compatible embedding configurations. Measure whether the relevant changed passage appears in the first few results, and record ingestion delay separately from query latency. For an urgent page-change alert, a fast text diff may be preferable to waiting for the entire vector index; the later research answer should still search the committed version. This is a scheduling decision, not a claim that one retrieval method always wins. The sample's eight-line chunk and two-line overlap are illustration parameters, not measured optimums; test section-aware chunks against that baseline, particularly on pages where a heading supplies the meaning of several short rows.

The generation stage should receive the question, retrieved passages, their source URLs and versions, and an instruction to distinguish observed changes from interpretation. If no relevant passage survives the retrieval threshold, say that evidence is insufficient. Avoid passing whole pages by default: duplicated navigation eats context, and a bigger prompt does not repair a missing chunk. Track prompt size alongside retrieval quality when moving from notebook to production.

## How do you know it is ready to run unattended?

Build an evaluation set containing unchanged pages, real claim edits, layout-only changes, deleted sections, and paraphrased analyst questions. Compare expected alerts with emitted diffs, then check whether the changed evidence is retrievable under the correct version. The two metrics answer different questions. A perfect diff with stale search results is still a broken research workflow.

Before deployment, make fetch retries bounded, log status and extraction failures without treating them as content changes, and keep the last known good version available. Instrument fetch duration, extraction emptiness, changed-chunk count, indexing completion, retrieval latency, and question-level evidence hits. Review pages whose layouts shift frequently; extraction rules and evaluation cases need to evolve together. Finally, test a full update while queries are arriving: either the old committed version or the new one should answer, never an accidental blend. That is the operational checklist worth keeping beside the pipeline.

No silent version mixing.

## References

- Retrieval-augmented generation and the distinction between retrieval and generation: https://arxiv.org/abs/2005.11401
- HTTP conditional requests and validators: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Conditional_requests
- Robots Exclusion Protocol: https://www.rfc-editor.org/rfc/rfc9309
- Python HTMLParser behavior: https://docs.python.org/3/library/html.parser.html

## Sources

- https://arxiv.org/abs/2005.11401
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Conditional_requests
- https://www.rfc-editor.org/rfc/rfc9309
- https://docs.python.org/3/library/html.parser.html
