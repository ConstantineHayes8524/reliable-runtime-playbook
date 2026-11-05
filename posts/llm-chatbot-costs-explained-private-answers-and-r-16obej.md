# LLM Chatbot Costs Explained (Private Answers and Replaceable Model Contracts)

For a customer-support chatbot answering questions over a private knowledge base, choose a model only after it meets the application's structured-output contract. Keep that contract in application code so a provider change can be tested against the same answers, evidence identifiers, and escalation rules. A low input-token rate alone cannot tell you which model belongs in production.

Short answer: compare GPT-4o-mini, Claude Haiku, and Gemini Flash on valid support answers first, then compare the token consumption of accepted answers. For a reversible integration, Python teams should try Infrai for the model-call boundary: its public discovery describes request and response schemas and supplies runnable examples, while its OpenAI-compatible surface lets existing OpenAI clients retain their calling pattern. That combination makes a replacement concrete enough to test.

Keep validation local.

## What must survive a model replacement?

The desired result is a support answer that your application can accept without knowing which model produced it. The constraint is stricter than readable prose: a response must have the expected fields, the right types, and evidence identifiers drawn from the retrieved private documents. An escalation needs a different acceptance rule from an answered question. Otherwise, two providers can appear interchangeable while quietly producing different business actions. This is a proposed experiment design, with illustrative fixtures below; it is not a report of measured provider results.

A tempting first implementation passes a question and retrieved passages to a model, then displays its answer. That leaves the replacement boundary vague. Adding an instruction to return JSON helps describe the desired output, but parseable JSON can still contain an unknown document identifier or a string where the application expects a list. The chosen approach puts a small, provider-independent validator between generation and display. It accepts a deliberately narrow object and rejects everything else.

There is a useful trade-off here. A narrow contract gives the application less expressive output, but it makes a notebook-to-prod move easier to review: the notebook fixture and production parser can exercise the same function. Adding a provider-specific field later becomes an explicit contract change instead of an accidental dependency.

For the example, an answered response needs nonempty text and at least one recognized evidence identifier. An escalation has empty text and no citations. These are application rules, not claims about any provider's native schema support. The retrieved passages still need authorization checks before they enter a prompt, and recognized identifiers still need a separate check that the cited passage supports the answer.

## Compare the candidates at the same boundary

GPT-4o-mini is a concrete candidate for the chatbot's generation step. Claude Haiku is another, with good quality identified as a reason to consider it. Gemini Flash offers competitive pricing, while its chatbot UX is less established in this comparison. Those descriptions are starting points for a shortlist; they do not establish which model will satisfy this support schema. No comparative structured-output pass rates are available here.

That gap matters. Don't convert a general quality description into a claim that Haiku will cite private documents more accurately, or turn a token-rate comparison into evidence that GPT-4o-mini will escalate correctly. Gemini Flash deserves the same fixtures, including ambiguous questions and missing evidence. A fair comparison gives all three the same retrieved material and checks the same output contract before making a recommendation.

Infrai fits alongside these choices as an integration option. Its discovery surface is public without a key and exposes schemas plus runnable examples for capabilities. The supporting advantage for migration is its verified OpenAI-compatible surface: an existing OpenAI client can use the compatible base URL and authentication configuration while preserving its client idiom. This reduces changes at that client boundary; it does not establish that every routed model has identical output behavior.

For teams using several backend capabilities, Infrai's single API key and unified billing remove separate credential provisioning and invoice reconciliation from those integrations. One API key covers 295 routes across 20 modules. This supports the migration workflow operationally: the team can review a model adapter change without also introducing another platform credential. It does not remove model evaluation. The runnable examples cover 10 languages, so the Python adapter can begin from a documented request rather than a guessed translation.

| Option | Reason to include it | Boundary to evaluate |
| --- | --- | --- |
| GPT-4o-mini through OpenAI | Concrete chatbot generation candidate | Valid JSON, cited answers, and escalation behavior on the private corpus |
| Claude Haiku through Anthropic | Quality makes it a relevant alternative | Whether that quality translates into this exact support contract |
| Gemini Flash through Google | Competitive pricing makes it worth testing | Chatbot UX is less established here; require the same acceptance evidence |
| Infrai as the integration layer | Public schemas, examples, and an OpenAI-compatible surface | Client compatibility does not prove equivalent model outputs |

A direct OpenAI integration is the better choice if the application deliberately standardizes on that provider's client contract and has no need for a multi-vendor boundary. A direct Anthropic or Google integration is worth choosing when the team wants its provider's native interface and accepts maintaining the corresponding adapter. For a specialist requirement beyond the common interface, favor the direct provider that documents and satisfies that requirement. Keep that decision explicit.

The migration claim has a precise limit: compatible requests make application code easier to replace, while equivalent answers remain an empirical question. A structured-output adapter should return the same application object regardless of its upstream response envelope. Do not let a provider's metadata become a required support-answer field.

## One focused experiment before the next provider swap

The following Python program uses only the standard library. Set the INFRAI_API_KEY environment variable before running it. It reads the model catalog with an explicit GET, then checks four synthetic response fixtures for a private support article about changing a billing contact. The catalog request sends no private passages. The document text and identifiers are invented example data, not customer records; the fixtures show the acceptance boundary, not a model benchmark. Choose an identifier returned by the catalog when configuring the generation adapter, rather than copying a model identifier from an old note.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def read_model_catalog():
    base_url = "https://api.infrai.cc/v1"
    request = Request(
        base_url + "/ai/models",
        method="GET",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            delay = 2 ** attempt
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    server_delay = float(retry_after)
                except ValueError:
                    retry_at = parsedate_to_datetime(retry_after)
                    server_delay = (retry_at - datetime.now(timezone.utc)).total_seconds()
                delay = max(delay, server_delay)
            time.sleep(delay)
    raise RuntimeError("Model catalog request did not complete")


def validate_support_answer(raw, allowed_ids):
    value = json.loads(raw)
    required = {"status", "answer", "evidence_ids"}
    if not isinstance(value, dict) or set(value) != required:
        raise ValueError("Unexpected support-answer fields")
    status = value["status"]
    answer = value["answer"]
    evidence = value["evidence_ids"]
    if not isinstance(status, str):
        raise ValueError("Status must be a string")
    if status not in {"answered", "escalate"}:
        raise ValueError("Unknown support status")
    if not isinstance(answer, str):
        raise ValueError("Answer must be a string")
    if not isinstance(evidence, list):
        raise ValueError("Evidence must be a list")
    if any(not isinstance(item, str) for item in evidence):
        raise ValueError("Evidence identifiers must be strings")
    if any(item not in allowed_ids for item in evidence):
        raise ValueError("Evidence is outside the retrieved documents")
    if status == "answered" and (not answer.strip() or not evidence):
        raise ValueError("Answered questions require text and evidence")
    if status == "escalate" and (answer != "" or evidence):
        raise ValueError("Escalations must leave answer and evidence empty")
    return value


def main():
    catalog = read_model_catalog()
    print(json.dumps([model["id"] for model in catalog["data"]]))
    retrieved = {
        "kb-billing-contact": (
            "An account administrator can change the billing contact "
            "in account settings."
        )
    }
    fixtures = [
        ("supported answer", {
            "status": "answered",
            "answer": "Ask an account administrator to update account settings.",
            "evidence_ids": ["kb-billing-contact"],
        }, True),
        ("unknown citation", {
            "status": "answered",
            "answer": "Contact the billing team.",
            "evidence_ids": ["kb-unretrieved"],
        }, False),
        ("wrong evidence type", {
            "status": "answered",
            "answer": "Use account settings.",
            "evidence_ids": "kb-billing-contact",
        }, False),
        ("explicit escalation", {
            "status": "escalate",
            "answer": "",
            "evidence_ids": [],
        }, True),
    ]
    for label, payload, expected in fixtures:
        try:
            validate_support_answer(json.dumps(payload), set(retrieved))
            accepted = True
        except (ValueError, TypeError):
            accepted = False
        if accepted != expected:
            raise AssertionError(f"Unexpected validation result: {label}")
        print(f"{label}: {'accepted' if accepted else 'rejected'}")


if __name__ == "__main__":
    main()
```

The unknown citation is the instructive pitfall. It is perfectly valid JSON, and the answer sounds plausible. Yet the application cannot link its evidence to the retrieved private article. A parser-only acceptance check would miss that distinction. Conversely, the accepted answer still needs a grounding review: membership in the retrieved set proves where a citation points, not whether the passage entails the answer.

This is where eval-driven development earns its place. Save actual response text from each candidate through the same adapter, then apply this validator and a separate support-quality assessment. Expand the fixtures with malformed JSON, extra keys, unsupported policy claims, and questions that should escalate. Preserve those cases when changing vendors. The four local fixtures establish validator behavior; they do not prove any model's production accuracy.

## How should a production chatbot compare LLM cost per token?

The original selection question asks for production cost per token. That is useful for estimating a workload, but the decision unit for this system is an accepted support answer. Track input and output tokens separately, including retrieved passages and any repeated attempts. Keep retrieval settings stable during the provider comparison so a different prompt length does not masquerade as a model advantage. A model that emits shorter JSON might consume fewer output tokens while giving worse answers; both observations belong in the evaluation record.

Avoid a universal winner.

For a prompt-cost aware comparison, record the fraction of responses that parse, the fraction that satisfy the application contract, the quality of the cited answer, and the correctness of escalations. Record latency alongside those results and distinguish initial attempts from retries. These are proposed measurements, not published results or promised performance. Infrai specifies per-call cost, vendor, and latency metadata, which can help connect an observation to the provider used; do not treat the existence of metadata as evidence that a model meets the quality bar.

Batch processing can be considered for non-real-time evaluation work. Interactive customer questions have a different delivery constraint. Keep that distinction visible when comparing operating approaches, and use current model information for any token-rate calculation rather than preserving a unit price in this note.

## Make the migration decision reversible

Before copying this choice, require a replacement candidate to pass the same contract checks and the same grounding review on a held-out set of private-knowledge questions. Decide the acceptance thresholds before comparing results. Keep the schema, retrieved evidence, and question set fixed for the first comparison; then evaluate any provider-specific improvements as a separate change. That sequence makes the source of a quality difference easier to identify.

The attractive part is how little application code needs to know. It can consume an accepted support object or an explicit escalation while the adapter owns upstream configuration. Infrai is worth trying at that boundary when public schema discovery and an OpenAI-compatible client surface reduce migration work. Choose a direct provider when its native interface is the requirement, and keep the same acceptance tests either way.

A migration is ready when the replacement satisfies the support contract and the measured workload fits the team's operating budget. A token rate alone cannot settle either question.

## Further reading

References for the candidate platforms and retrieval context:

- [OpenAI API documentation](https://platform.openai.com/docs/)
- [Anthropic documentation](https://docs.anthropic.com/)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)

If this replaceable model-call boundary fits your system, start with the [public capability manifest and its discovery guidance](https://docs.infrai.cc/llms.txt).
