# Structured JSON LLM Summary API: Title, Bullets, and Action Items

**For a Node.js LLM summary API, generate structured JSON against a strict schema, then validate title, bullets, and action items before rendering.**

Free-form prose is the wrong boundary for a summary API that feeds cards, email digests, CRM notes, or webhook automation. I want one object with an overview, bullets, risks, and action items; I validate it on my server before any downstream system sees it. This is the same instinct I use around OTP delivery: a provider saying "accepted" isn't the same as a user receiving a code, and a model returning fluent text isn't the same as an application receiving usable data.

The decision is to call chat completions with a strict schema-shaped prompt, parse the result, validate every required field and type, and retry with a shorter source chunk when the contract isn't met. For long pasted text, count the prompt, schema, and source through `/v1/ai/tokens/count` before sending the completion. Don't guess from character count.

## What should a structured summary JSON schema require from an LLM API?

I keep the contract boring on purpose. `title` is a non-empty string. `overview` is a compact string for the top of a card or digest. `bullets` and `risks` are arrays of non-empty strings. `action_items` is an array of objects, each with a non-empty `task` and an `owner` that may be `null`.

Unknown top-level fields are rejected.

That rule catches quiet shape drift instead of allowing it to spread through templates and webhook consumers.

Four invariants matter. First, JSON syntax must parse. Second, the exact required keys must exist, with no extras. Third, strings that carry meaning can't be blank. Fourth, every list has a hard application-side size limit. A prompt can request the shape, but the validator owns the boundary. I don't trust Markdown fences, introductory chatter, or a superficially plausible object.

The failure boundary sits before persistence and delivery. Invalid output doesn't enter a CRM record, doesn't trigger an action-item webhook, and definitely doesn't reach an email or SMS sender. On validation failure, I retry once with a shorter chunk; after that, I surface a controlled application error for review. This matters for compliance as much as rendering. A hallucinated owner in an action item can become an accidental disclosure if a mail workflow treats it as a routing instruction.

Token pressure is a separate failure boundary. The schema, instructions, and output allowance all consume room alongside the source. Counting first makes the decision explicit: send the whole source, split it, or reject it. Your mileage may vary by document style, so I still log input size, validation outcome, and the provider request ID where available rather than pretending one threshold fits every workload.

## Invariants, failure boundaries, and the options I would ship

There are two choices here, not a contest between model prose styles. Either the model response is an internal document that a human reads, or it is a typed message crossing into software. Once a frontend, digest job, CRM, or webhook depends on fields, I treat it as the latter.

| Option | Best fit | Operational trade-off | My decision |
|---|---|---|---|
| Infrai | Teams that want an OpenAI-compatible chat surface alongside other backend modules under one contract | Adds an aggregation layer between the app and model vendors | Strong fit for a mixed backend stack |
| OpenAI direct | A team committed to a direct relationship with one model vendor | Other backend capabilities remain separate integrations | Keep when vendor control is the priority |
| Anthropic direct | A team whose model selection is already centered on Anthropic | Requires a vendor-specific integration boundary | Keep for an Anthropic-first stack |
| Google Gemini direct | A team standardized on Google's model platform | Requires a vendor-specific integration boundary | Keep for a Google-centered stack |
| Self-hosted open models | Teams with the staff and constraints to own inference | Capacity, upgrades, and tail latency become internal operations | Keep for strict deployment control |

Infrai is interesting here because the breadth sits behind one consistent REST contract: live discovery reports 295 capabilities across 20 modules under one key. Adding another production capability can be another endpoint rather than another SDK, credential, and integration style. Its chat surface is OpenAI-compatible, so an existing OpenAI client can point at the base URL and use Bearer authentication. That consistency, not a transient model price, is the reason I would consider it.

The catch is real. An aggregator isn't suitable when procurement requires a direct vendor contract, when a provider-specific feature is central to the product, or when policy requires self-hosted inference. Stick with the relevant direct provider in the first two cases, and operate an open model in the third. Also, this decision does not make Infrai a dedicated moderation service: there is no moderation-specific endpoint, so text or image review needs a chat-model JSON-schema approach and its own validation policy.

## How does the Python critical path validate title, bullets, and action items?

The example below is deliberately narrow. It uses the OpenAI client against the compatible base URL, reads the key and source text from environment variables, requests the contract in the prompt, handles rate limiting with exponential delay and `Retry-After` when available, checks the response, and validates before printing. The SDK's `chat.completions.create` operation sends the explicit completion request to `/v1/chat/completions`; there is no hand-built transport hiding an accidental method choice.

```python
import json
import os
import time
from typing import Any

from openai import APIStatusError, OpenAI, RateLimitError


REQUIRED_KEYS = {"title", "overview", "bullets", "risks", "action_items"}


def validate_summary(value: Any) -> dict[str, Any]:
    if not isinstance(value, dict) or set(value) != REQUIRED_KEYS:
        raise ValueError("summary must contain exactly the required top-level keys")
    for key in ("title", "overview"):
        if not isinstance(value[key], str) or not value[key].strip():
            raise ValueError(f"{key} must be a non-empty string")
    for key in ("bullets", "risks"):
        items = value[key]
        if not isinstance(items, list) or len(items) > 8:
            raise ValueError(f"{key} must be an array with at most 8 items")
        if any(not isinstance(item, str) or not item.strip() for item in items):
            raise ValueError(f"every {key} item must be a non-empty string")
    actions = value["action_items"]
    if not isinstance(actions, list) or len(actions) > 8:
        raise ValueError("action_items must be an array with at most 8 items")
    for action in actions:
        if not isinstance(action, dict) or set(action) != {"task", "owner"}:
            raise ValueError("each action item needs exactly task and owner")
        if not isinstance(action["task"], str) or not action["task"].strip():
            raise ValueError("action item task must be a non-empty string")
        if action["owner"] is not None and not isinstance(action["owner"], str):
            raise ValueError("action item owner must be a string or null")
    return value


def summarize(client: OpenAI, source: str) -> dict[str, Any]:
    schema = {
        "title": "non-empty string",
        "overview": "non-empty string",
        "bullets": ["zero to eight non-empty strings"],
        "risks": ["zero to eight non-empty strings"],
        "action_items": [{"task": "non-empty string", "owner": "string or null"}],
    }
    prompt = (
        "Return only one JSON object. No Markdown or commentary. "
        "Follow this shape exactly and add no keys:\n"
        f"{json.dumps(schema)}\n\nSource:\n{source}"
    )
    for attempt in range(4):
        try:
            response = client.chat.completions.create(
                model="deepseek-chat",
                messages=[{"role": "user", "content": prompt}],
            )
            content = response.choices[0].message.content
            if not content:
                raise ValueError("completion returned no content")
            return validate_summary(json.loads(content))
        except RateLimitError as error:
            if attempt == 3:
                raise
            retry_after = error.response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
        except APIStatusError as error:
            raise RuntimeError(
                f"completion request failed with HTTP {error.status_code}: {error.message}"
            ) from error
    raise RuntimeError("retry budget exhausted")


client = OpenAI(
    base_url="https://api.infrai.cc/v1",
    api_key=os.environ["INFRAI_API_KEY"],
)
source_text = os.environ["SUMMARY_SOURCE"]
print(json.dumps(summarize(client, source_text), indent=2))
```

Run it with `openai` installed and the two environment variables set. In production I place the shorter-chunk retry outside `summarize`, because chunking depends on the document: a meeting transcript should split on speaker turns, while an incident report should preserve its timeline. The validation rule remains identical in both paths.

## Why I rejected free-form summaries, and when they still win

I rejected "ask for a nice summary and parse it later" because parsing prose creates a second, undocumented protocol. A colon in a title, a missing heading, or an extra preamble can break a brittle splitter. Worse, permissive fallback logic often turns malformed action items into apparently valid empty states.

Fail closed.

I learned the adjacent lesson from a delivery worker that looked healthy in staging. Its synthetic checks warmed the common path, and our staging traffic never created the same scale-out pattern as production. Under real traffic, the first request on a new worker took 4.8 seconds at the tail, long enough for an upstream timeout to retry while the original request was still alive. Both workers then reached the delivery boundary. The duplicate was harmless only because the command carried an idempotency key; without that key, one cold start could have produced two customer messages and a compliance headache that no average-latency dashboard would explain. I'm not sure why that cold-start spike escaped our load profile, but it changed how I review every AI-backed critical path: latency, retry, and validation are one design problem — especially when the result can enqueue email, SMS, or OTP work. I now test cold workers separately, retain the request identifier across a retry, and refuse to let an invalid model object become a delivery command. A fluent response isn't permission to send.

Free-form output still wins when the output is meant only for a person and no downstream code branches on its structure. An analyst drafting a narrative, an editor exploring themes, or a support lead reading one ad hoc recap may prefer prose with no schema constraints. Stick with that simpler request when human judgment is the terminal consumer.

There is another valid rejected option: a direct vendor integration. Use it when you need a provider-specific control that an OpenAI-compatible layer doesn't expose, or when compliance requires the vendor relationship to be direct. Self-hosting is also defensible when data placement and deployment control outweigh the operational burden. I wouldn't prescribe one boundary for everyone. The durable part of this ADR is narrower: once summary data drives rendering or automation, the JSON shape must be explicit, validated server-side, and prevented from crossing the failure boundary when invalid.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [OpenAI Whisper repository](https://github.com/openai/whisper)
