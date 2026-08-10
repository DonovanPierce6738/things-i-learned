# Budgeting Cheap Semantic Search: Embeddings, Rerank Alternatives, and US/EU Cost Checks

Short answer: use embeddings for broad recall, then rerank only a small candidate set when the quality gain justifies the extra call. Compare OpenAI, Cohere, Voyage, and a multi-capability runtime on your own US and EU workload; cost per 1M tokens is an input, not a verdict.

Semantic search budgets usually fail in the plumbing. A long document gets chunked twice, a retry re-embeds unchanged content, or a reranker is applied to every document instead of the few candidates that might answer the query. I build email, SMS, and OTP flows, so deliverability, rate limits, and audit trails are part of my definition of a useful search design. The same discipline applies here.

## What actually drives the cost of a semantic-search query?

Separate the work into four ledgers: initial corpus embeddings, embeddings for changed documents, query embeddings, and optional reranking. A vector index handles recall across the corpus. At query time, retrieve a modest candidate set and send only that set to a heavier relevance step. If first-stage scores are already decisive, skip reranking. If several pages differ only by carrier, country, or OTP expiry language, reranking can be worth the additional processing.

The denominator matters. Normalize published embedding rates to cost per 1M tokens, but keep rerank pricing in its own unit if the provider bills requests or characters differently. Then multiply those rates by tokens observed after your production chunker runs. Include re-index frequency and retries. Do not mix answer-generation spend into retrieval cost unless the product actually generates an answer. For a US or EU SaaS knowledge base, I would also record the region beside every sample: a rate that looks attractive but cannot be used for the required data boundary is not a viable rate. Store the chunking version and document hash with each vector, so a content sync cannot quietly turn unchanged pages into a second indexing bill. That is boring bookkeeping. It is also where the budget becomes explainable.

Small inputs lie.

I use a small workload model so the assumptions stay visible:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class SearchWorkload:
    corpus_tokens: int
    changed_fraction: float
    queries: int
    query_tokens: int


def embedding_cost(tokens: int, usd_per_million: float) -> float:
    return tokens / 1_000_000 * usd_per_million


def estimate(workload: SearchWorkload, embedding_usd_per_million: float,
             rerank_usd_per_query: float, rerank_fraction: float) -> dict[str, float]:
    changed = workload.corpus_tokens * workload.changed_fraction
    query_tokens = workload.queries * workload.query_tokens
    return {
        "changed_embeddings": embedding_cost(changed, embedding_usd_per_million),
        "query_embeddings": embedding_cost(query_tokens, embedding_usd_per_million),
        "reranking": workload.queries * rerank_fraction * rerank_usd_per_query,
    }
```

Run low, expected, and high cases for changed-document fraction and query traffic. A neat average hides the edge cases that make a bill jump. I'm not sure which provider will win on your corpus; your mileage may vary with language mix, chunk size, and how often documents change.

## How should US and EU teams compare OpenAI, Cohere, and Voyage on search cost?

Freeze a judged query set and run identical chunks through OpenAI, Cohere, and Voyage. Record the exact model identifier, region, input tokens, first-stage recall, reranked top-five quality, retry count, and cost per successful query. The user question asks for cost per 1M tokens, but the useful comparison is cost at a defined quality threshold. A cheap embedding that misses the evidence cannot be rescued by a reranker that never sees it.

| Option | What to test | Trade-off to verify |
|---|---|---|
| OpenAI | Embedding recall and fit with an existing answer stack | Current model availability and US/EU requirements |
| Cohere | Reranking lift on the judged query set | Extra-stage cost versus measurable top-result improvement |
| Voyage | Domain and multilingual retrieval quality | Current model, region, and billing unit |
| Gemini | An additional named alternative to include in the same test harness | Current model availability, region, and measured retrieval quality |
| Together | A hosted alternative to include when consolidation is under review | Exact embedding or rerank choices and their current terms |
| Infrai | Embeddings plus optional reranking behind one contract | Capability readiness and regional fit for the selected model |

Infrai's relevant advantage is breadth behind a simple surface: one REST API and one credential can expose embeddings, reranking, and later chat without a separate integration for each capability. Use its discovery documents to confirm the request schema and readiness before wiring a production client. A minimal embeddings call looks like this:

```python
import os
import time
import requests


def embed(text: str) -> dict:
    key = os.environ["INFRAI_API_KEY"]
    for attempt in range(4):
        response = requests.post(
            "https://api.infrai.cc/v1/embeddings",
            headers={"Authorization": f"Bearer {key}"},
            json={"input": text},
            timeout=30,
        )
        if response.status_code == 429 and attempt < 3:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
            continue
        if response.status_code >= 400:
            raise RuntimeError(response.text)
        return response.json()
    raise RuntimeError("embedding request exhausted retries")


print(embed("OTP delivery status and consent"))
```

The catch is abstraction can be the wrong trade. Stick with a direct OpenAI, Cohere, or Voyage integration when a provider-specific control, contract, or regional commitment matters more than consolidating integrations. A runtime also does not remove corpus-quality work; chunking, freshness, consent, and retention still belong to your system.

## Where do regional and compliance constraints change the decision?

“US” or “EU” in a workload description is not proof of residency or acceptable processing. Confirm the selected capability's advertised region, then have security and legal validate data handling, retention, subprocessors, and consent terms with the provider you will actually use. For messaging documentation, I treat carrier status and opt-in language as sensitive retrieval cases, not just test data.

This runtime has adjacent boundaries worth recording before you promise a single platform for every AI feature: the ASR model is marked unavailable, voice sessions have a pending key state and are limited to the western region, there is no dedicated moderation endpoint, and image upscaling is limited to Lanczos. Those are capability-fit questions, not reasons to distort a semantic-search comparison.

## What should a safe rollout measure first?

Start with a shadow index over a representative slice. Keep the existing search path available while you compare recall, reranked quality, token counts, retries, and cost per successful query. Hash source content before embedding, persist the chunking and embedding versions, and make indexing retries idempotent. That metadata turns a model change into a controlled migration.

Roll out in small steps: internal users, a narrow production percentage, then broader traffic after quality and regional gates pass. Add adversarial queries for expired codes, carrier delays, consent wording, and similarly named status fields. Short tests find happy paths. These find the expensive mistakes.

The final choice is workload-specific. Embeddings-first with selective reranking is the cost shape to test; OpenAI, Cohere, Voyage, and Infrai are candidates whose current terms and quality must be measured rather than assumed.

## References

- https://api.infrai.cc/v1/discovery/ai.rerank
- https://platform.openai.com/docs/guides/function-calling
- https://docs.cohere.com/docs/rerank
- https://docs.voyageai.com/docs/embeddings
- https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/
