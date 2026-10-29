# Invoice Text Summarization API Quality and Cost: Per-Tenant Controls

A supplier-invoice summarizer should be selected by the evidence it preserves, not by a leaderboard score. In a fintech system, run the same redacted invoice set through OpenAI, Claude, and Gemini behind one adapter, then record quality, latency, and input/output tokens per tenant. A fluent response is still a failed result if it drops currency, tax treatment, or payment terms. The comparison has three adapters, five critical fields, and one release rule: no unreviewed critical omission.

TL;DR: make the summary contract stricter than the prose prompt, test field-level omissions before style, and attach every call to a tenant-scoped usage record. Keep routing reversible until the evaluation set shows a stable quality-cost trade-off.

A supplier invoice can contain a header, line items, tax rows, remittance instructions, and notes that change the meaning of a due date. I start with a typed intermediate record rather than asking a model to produce a paragraph directly. The flow is plain: normalize the file, preserve page references, extract required fields, summarize the verified record, validate it, and write a usage event. The model is one stage, not the accounting system.

Five fields. That is enough to catch a surprising amount of damage before prose quality enters the discussion.

Consider a redacted fixture with a total of 18,420.00 in EUR, a tax amount of 3,070.00, and a due date printed beside a separate early-payment note. The expected summary must retain the invoice identifier, both amounts, the currency, and the due date, while flagging the note for review. A response that says “payment is due soon” is readable but not equivalent. A response that converts EUR to a familiar local currency is worse: it has introduced a value that the source did not provide. This is why the harness stores the structured record beside the generated text. The reviewer can inspect an omission, an unsupported conversion, and a formatting variation as three different failures instead of arguing about whether the paragraph sounds professional.

Here is the boundary I use for an offline harness. It does not assume a vendor SDK, because SDK differences are part of the comparison.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Invoice:
    tenant_id: str
    invoice_id: str
    currency: str
    total: str
    due_date: str
    tax_amount: str
    source_text: str


class TextModel(Protocol):
    name: str

    def summarize(self, prompt: str) -> dict[str, object]:
        ...


def prompt_for(invoice: Invoice) -> str:
    return f"""Summarize this verified supplier invoice in four sentences or fewer.
Preserve invoice_id, currency, total, due_date, tax_amount, and exceptions.
Do not infer missing values.

invoice_id={invoice.invoice_id}
currency={invoice.currency}
total={invoice.total}
due_date={invoice.due_date}
tax_amount={invoice.tax_amount}
source={invoice.source_text}""".strip()


def missing_fields(invoice: Invoice, summary: str) -> list[str]:
    fields = {
        "invoice_id": invoice.invoice_id,
        "currency": invoice.currency,
        "total": invoice.total,
        "due_date": invoice.due_date,
        "tax_amount": invoice.tax_amount,
    }
    return [name for name, value in fields.items() if value not in summary]
```

The adapters should report token counts from the provider response when available. A local tokenizer estimates prompt growth; it is not a universal billing measurement. `tiktoken`, for example, is a BPE tokenizer library for a particular tokenizer family, not a cross-provider accounting oracle.

## What does a good invoice summary need to preserve?

The first rubric is a set of assertions, not “sounds accurate.” For each fixture, check exact preservation of the invoice identifier, currency, total, due date, and tax amount. Add conflicting totals, a missing purchase-order number, a credit note, and a payment instruction that differs from the supplier profile. A reviewer can judge wording later. Missing fields are easier to count and harder to excuse.

This is the notebook-to-prod handoff. Production needs a reason for every accepted summary: source hash, prompt version, provider, returned model identifier, token counts, latency, validation result, and reviewer disposition. Keep raw invoice text out of the cost table by default. Tenant visibility does not require exposing sensitive document contents.

Use two passes. Deterministic validation catches missing values; a small fixed human sample catches semantic errors such as saying “tax included” when the invoice lists tax separately. Do not change fixtures and model at the same time. That makes the question unanswerable.

## How should an invoice text summarization API balance quality and cost?

OpenAI, Anthropic Claude, and Google Gemini all offer hosted APIs for text generation, but their request schemas, authentication setup, model naming, token-counting behavior, and response metadata are not interchangeable. Normalize only the fields the accounting workflow needs. Do not hide provider-specific limits or manufacture a token count. Verify current request and response details in each provider’s documentation before implementing an adapter.

| Axis | Measurement | Invoice consequence |
| --- | --- | --- |
| Field fidelity | Required omissions and incorrect preservation | A fluent summary can create a review gap |
| Cost visibility | Input/output tokens by tenant and prompt version | Finance needs attribution, not one blended number |
| Operations | Latency, timeouts, retries, validation failures | Failures change queue capacity and review load |
| Portability | Adapter changes for one contract | Evaluation evidence stays useful after a route change |

Do not collapse these into one score too early. A weighted score can conceal a dangerous zero in field fidelity. I use a release rule first: no unreviewed critical omissions in the holdout set. Among candidates that pass, compare cost and latency by tenant mix. This is an intentional trade-off: a slightly slower route can be the better production choice when it prevents a human reviewer from chasing a missing tax amount, while a fast route that needs manual repair shifts cost into operations. Keep that decision visible in the evaluation record instead of hiding it inside a single average.

A reranker can select relevant passages after chunking, but retrieval quality and summarization quality are separate measurements. A component such as the one documented by Cohere should be tested for passage selection, not credited for preserving totals or tax semantics.

## What does per-tenant cost visibility look like in production?

Every invocation gets an immutable usage event keyed by tenant ID, document ID, prompt version, provider, model, and request ID. This answers concrete questions: which tenant generated the most output tokens, which prompt revision increased input size, and which retry policy created duplicate work?

Keep the accounting formula explicit: input usage plus output usage, joined to the rate-card version effective at call time. Prices change, so the rate card belongs in timestamped configuration rather than a Python constant. A provider-reported count is `reported`; a tokenizer estimate is `estimated`; missing telemetry is `unknown`, never zero. Otherwise an instrumentation failure looks like free inference.

Retries deserve their own identifiers. If a timeout causes a second request, both attempts belong in the tenant ledger while only one may produce the accepted summary. Keep `attempt_id` separate from `accepted_result_id`, or the system will under-report the cost of the failure mode it needs to fix.

## A release checklist that survives model changes

Freeze the fixture set and prompt contract before changing the default route. Run the same redacted invoices through all three adapters, record raw metadata before normalization, and inspect every critical omission. Then add malformed dates, mixed currencies, long line-item sections, and duplicate invoice IDs.

Ship bounded retries, timeouts, idempotency keys where supported, and a dead-letter path for validation failures. Log identifiers and measurements, not invoice bodies. Keep the route switch separate from evaluation code so one tenant, a cohort, or the whole workload can return to a previously evaluated adapter without a rewrite.

The useful comparison is evidence per tenant: what was spent, what passed, what required review, and why the route met the invoice contract. Provider names start the experiment; field fidelity and tenant-scoped evidence finish it.

## Sources

- OpenAI, `tiktoken` tokenizer library: https://github.com/openai/tiktoken
- OpenAI API documentation: https://platform.openai.com/docs/api-reference
- Anthropic API documentation: https://docs.anthropic.com/en/api/overview
- Google Gemini API documentation: https://ai.google.dev/gemini-api/docs
- Cohere Rerank overview: https://docs.cohere.com/docs/rerank-overview
- NIST AI Risk Management Framework: https://www.nist.gov/itl/ai-risk-management-framework
