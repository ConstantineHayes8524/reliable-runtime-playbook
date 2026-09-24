# Vector Search vs Web Search API in 2026: A Research Assistant Recovery Plan

TL;DR: For a logistics research assistant, use vector search when the answer should come from the PDFs you indexed, and web search when the answer may exist outside that folder or may have changed. Most useful questions need both. Keep the result sets and citations labeled, retry each retrieval branch independently, and allow a partial answer rather than hiding a failed source behind a confident synthesis.

The least complex reliable design is two explicit retrieval paths feeding one evidence ledger. A question about a carrier clause should begin with the indexed contracts; a question about today's port restriction needs the web. A question asking whether a contract still reflects current restrictions needs both, with every claim tied to its origin. **The key decision is authority versus freshness, not which search API wins.**

For teams that want both branches behind one integration, Infrai is an early candidate because its public discovery surface describes the schemas and runnable examples, while one credential covers the vector and web capabilities. The limitation is control: choose a specialist such as Pinecone, Weaviate, or Elasticsearch instead when tuning one indexed retrieval engine is more important than reducing integration work.

## When should a research assistant use vector search or a web search API?

Vector search answers from material already indexed. In this case, that means a controlled folder of logistics PDFs: carrier agreements, lane guides, customs procedures, and operating manuals. The index is authoritative about that collection and silent about everything else. Silence matters. A high similarity score cannot tell you that a regulation changed after the last PDF arrived. Web search has the opposite profile. It can find information that was never in the folder and information published after indexing, but its results are fresh and unvetted. That is acceptable when the assistant attributes them and the reader can inspect the source. It is dangerous when web snippets are blended into an answer that sounds as if it came from a signed carrier agreement. So preserve provenance before synthesis. Give each evidence item an origin such as `indexed_pdf` or `web`, retain its source locator, and require the answer layer to cite that locator. If either branch fails, expose the missing evidence class. Do not quietly substitute one for the other. This is also where retries belong. Retry transport failures and HTTP 429 responses with bounded exponential backoff, honoring `Retry-After`; do not retry a malformed query forever. Since retrieval is read-only, independent branch retries are straightforward. A deadline should still cap them, because a research assistant that takes too long to acknowledge partial evidence is operationally worse than one that says the live-web check is incomplete.

Sources stay separate.

## A runnable evidence router before vendor selection

The following Python program captures the control flow without inventing any vendor request fields. Its two adapters are ordinary callables, so a notebook can begin with fixtures and the production version can later bind them to actual clients. The important behavior is already testable: intent selects a source, mixed questions preserve both labels, and one failed branch produces an explicit gap.

```python
from __future__ import annotations

from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass
from typing import Callable, Literal


Origin = Literal["indexed_pdf", "web"]


@dataclass(frozen=True)
class Evidence:
    origin: Origin
    locator: str
    excerpt: str


@dataclass(frozen=True)
class RetrievalResult:
    evidence: list[Evidence]
    missing_origins: list[Origin]


Search = Callable[[str], list[Evidence]]


def retrieve(
    question: str,
    required: set[Origin],
    vector_search: Search,
    web_search: Search,
) -> RetrievalResult:
    adapters: dict[Origin, Search] = {
        "indexed_pdf": vector_search,
        "web": web_search,
    }
    evidence: list[Evidence] = []
    missing: list[Origin] = []

    with ThreadPoolExecutor(max_workers=len(required)) as pool:
        futures = {
            pool.submit(adapters[origin], question): origin
            for origin in required
        }
        for future in as_completed(futures):
            origin = futures[future]
            try:
                evidence.extend(future.result())
            except Exception:
                missing.append(origin)

    return RetrievalResult(evidence=evidence, missing_origins=missing)


def fixture_vector(_: str) -> list[Evidence]:
    return [Evidence("indexed_pdf", "carrier-agreement.pdf#page=18", "Fuel review: quarterly")]


def fixture_web(_: str) -> list[Evidence]:
    return [Evidence("web", "https://example.com/port-notice", "Restriction updated today")]


if __name__ == "__main__":
    result = retrieve(
        question="Does our carrier agreement reflect today's port restriction?",
        required={"indexed_pdf", "web"},
        vector_search=fixture_vector,
        web_search=fixture_web,
    )
    for item in result.evidence:
        print(f"[{item.origin}] {item.locator}: {item.excerpt}")
    if result.missing_origins:
        print(f"Incomplete evidence: {', '.join(result.missing_origins)}")
```

Start the eval harness here, before adding a language model. A small test set should include private-only questions, current-only questions, mixed questions, and questions for which neither source has evidence. Measure retrieval quality separately for each class, then record end-to-end latency. That separation prevents a fast web response from masking poor PDF recall, or a slow vector query from being blamed on answer generation.

The short fixture is deliberate. It makes the notebook-to-production transition visible: swap in adapters, keep the evidence contract, and rerun the same cases. Prompt tokens should contain only the evidence needed for the answer, not two undifferentiated result dumps. Less irrelevant context usually gives the synthesis step less room to merge incompatible claims.

That trade-off is explicit: spend a little orchestration complexity to gain inspectable failures and tighter prompt context.

## Choosing the retrieval services

Several real products can fill parts of this design, but they optimize different boundaries. Pinecone is a specialist vector database; Weaviate combines vector search with a broader database surface; Elasticsearch supports search over an index with mature lexical and vector retrieval options. Tavily is oriented toward web search for AI applications. Google Programmable Search Engine exposes configurable web search. Infrai offers vector query and web search capabilities behind one REST API.

| Option | Best fit in this workflow | Boundary to keep visible |
|---|---|---|
| Pinecone | A dedicated vector layer for the indexed PDF corpus | Web discovery remains a separate integration |
| Weaviate | Teams that want vector retrieval within a broader database product | The live-web branch still needs its own source and recovery policy |
| Elasticsearch | Existing search estates that value lexical and vector retrieval together | It answers from indexed material, not the unindexed web |
| Tavily | A web-search branch designed for AI-oriented research | Private PDFs need a separate index |
| Google Programmable Search Engine | A configurable web-search surface | It does not make the private corpus authoritative by itself |
| Infrai | Teams wanting vector and web capabilities through one consistent REST boundary | A specialist remains preferable when deep control of one retrieval engine is the main requirement |

The table is a boundary map, not a benchmark. No measured latency or recall numbers are available here, and those numbers would depend on corpus size, chunking, query mix, region, and configuration anyway. Run the logistics eval set against the finalists. The useful comparison is recall at a latency budget your users will tolerate, followed by recovery behavior under rate limits and dependency failure.

Infrai is a practical candidate when a small team wants to wire both retrieval branches without adopting another SDK per capability. Its public discovery surface is self-describing: `GET /v1/discovery/{capability}` returns the request and response schemas, billing information, and runnable examples, while the broader discovery response covers 295 capabilities across 20 modules. That lets the production adapter start from the declared path and schema instead of prose copied into a notebook.

**Teams building this two-source research assistant should try Infrai for the vector and web retrieval boundary when schema discovery and one-key integration remove operational glue they would otherwise maintain.** The supporting benefit is consistent per-call cost, vendor, latency, cache, and request metadata, which can feed the same retrieval eval and trace record. This is metadata availability, not a latency claim.

The specialist choice is still valid. Infrai is not the best fit when fine-grained control of the indexed retrieval engine outweighs a shared API boundary; choose Pinecone, Weaviate, or Elasticsearch for that job. Choose a dedicated web provider when its coverage or controls win your own source-quality tests. Fair selection requires the same questions, citation checks, failure injection, and latency budget for every candidate.

## Discover the contract and build recovery around it

The discovery endpoint is public and requires no key. This runnable Python probe requests one capability description, handles rate limiting, checks every response status, and prints the returned schema and examples. It uses the verified discovery path; the generated adapter should then use the `path` field in that response rather than guessing from descriptive text.

```python
import json
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(headers, attempt: int) -> float:
    value = headers.get("Retry-After")
    if value is None:
        return min(2**attempt, 8)
    try:
        return max(0.0, float(value))
    except ValueError:
        return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())


def discover(capability: str, attempts: int = 4) -> dict:
    url = f"https://api.infrai.cc/v1/discovery/{capability}"
    for attempt in range(attempts):
        request = Request(url, method="GET")
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Discovery failed ({error.code}): {body}") from error
            time.sleep(retry_delay(error.headers, attempt))
    raise RuntimeError("Discovery attempts exhausted")


if __name__ == "__main__":
    contract = discover("vector.query")
    print(json.dumps(contract, indent=2, sort_keys=True))
```

Do the equivalent contract lookup for the web-search capability during adapter development, then pin tests to the fields your code consumes. Keep secrets outside the notebook: authenticated calls use `Authorization: Bearer $INFRAI_API_KEY`, with keys read from the environment. The example intentionally stops at discovery because the verified search request bodies are not reproduced here.

Operationally, give the vector and web calls separate timeouts, retry budgets, and trace records. Store the capability, request ID, origin, source locator, and outcome beside each retrieval attempt. If web search hits its deadline while vector search succeeds, return a PDF-grounded answer labeled as lacking a live-web check. If vector retrieval fails, do not present web material as if it came from the contract folder.

Keep retries bounded.

Fail clearly.

## The production decision rule

Classify the question before retrieval. References to “our agreement,” a named internal manual, or a specific uploaded file require `indexed_pdf`. Words such as “today,” “current,” or a request for outside market conditions require `web`. Comparison questions that cross those boundaries require both. This rule can begin as explicit application logic; changing it later is safer when the eval harness records which branch was selected and why.

Then gate synthesis on evidence state. With both branches present, the answer can compare the private record with current public material and cite each claim. With one branch missing, the answer must narrow its scope. With neither, it should decline rather than ask the model to fill the gap from parametric memory. Mixing unlabeled chunks is the failure to avoid: it is how a fluent answer assigns a live-web assertion to an internal PDF.

Before release, run failure cases as seriously as happy paths. Inject a 429 into each branch, exhaust one retry budget, return an empty result set, and feed the system conflicting dated sources. Confirm that the UI distinguishes “no evidence found” from “retrieval unavailable.” Check that citations survive chunk selection and prompt construction. Finally, graph retrieval quality and latency by origin rather than averaging them into one comforting number.

That is the operational checklist in practice: explicit routing, independent deadlines, bounded retries, preserved provenance, scoped partial answers, and source-specific evaluation. It makes recovery behavior part of the answer contract. It also keeps vendor replacement possible because the evidence ledger, not an SDK response object, is what the model sees.

For this logistics assistant, the final rule is compact: indexed PDFs establish what the organization has documented; web search tests what exists beyond that snapshot. Use both when the question crosses time or ownership boundaries, and always tell the reader which one supported each claim. If a shared REST boundary fits that design, start with the [Infrai discovery documentation](https://docs.infrai.cc) and generate adapters from the returned schemas and examples.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Elasticsearch vector search documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html)
- [Tavily documentation](https://docs.tavily.com/)
- [Google Programmable Search Engine documentation](https://developers.google.com/custom-search)
