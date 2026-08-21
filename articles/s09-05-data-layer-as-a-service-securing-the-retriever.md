---
title: "The data layer as a service: isolating and securing the retriever"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 9
series_part: 5
scope: evergreen
source: user-supplied article
reading_time: 32 min
added: 2026-08-21
summary: >
  The retriever and the generator are two logical services that happen to share
  a codebase, and three tensions prove it: blast radius, rate limits that must
  differ by an order of magnitude because one call costs milliseconds and the
  other costs euros, and credentials that should not hand a script the power to
  spend the LLM budget. Two FastAPI routers with deliberately asymmetric
  contracts, two API keys compared with secrets.compare_digest, per-key rate
  limiting, idempotency keys so a client retry does not buy a second estimate,
  structured per-stage logging tied by request_id, and a Ruby client whose
  timeouts and retry policy make the whole pattern work.
keywords: [APIRouter, FastAPI, service isolation, blast radius, API keys,
           secrets.compare_digest, timing attack, key rotation, slowapi, rate
           limiting, Retry-After, idempotency key, structlog, request_id,
           Faraday, timeouts, retry policy, JWT, mTLS, OWASP API Security]
---

# The data layer as a service: isolating and securing the retriever

*Antonio Perez* · 🔴 32 min

At the close of Article 4 you have the complete RAG flow inside the AI service: reformulator, retriever, assembler, generator, all orchestrated by an `estimate_from_transcript` function. The AI service's public endpoint is a single one — `POST /v1/estimate` — and behind it lives all the logic. The Rails business backend calls that endpoint, receives the structured estimate, persists it, shows it to the client. It works. It is what any RAG system MVP offers.

The problem starts when the system leaves the MVP phase. Imagine the realistic scenario of the second or third month in production. The sales team wants a new feature: inside the CRM, when reviewing an old project, being able to search for other similar projects in the historical base to support a negotiation. The search generates no estimate: it just returns the similar budgets and lets the salesperson inspect them. It is exactly what retrieval already does internally, but now it has to be exposed.

The minimum-effort option is to add a parameter to the existing endpoint: `POST /v1/estimate?retrieval_only=true`. The medium-effort option is to duplicate the endpoint: `POST /v1/estimate` for the full flow and `POST /v1/retrieve` for retrieval only. Both are tempting and both create operational problems that do not show up until three months later.

Consider what those two options imply. When Session 10 introduces reranking and hybrid search over the retriever, any change touches the estimation endpoint, even though the generation logic does not vary; the change's blast radius is unnecessarily wide. When the time comes to apply rate limiting, the estimation endpoint needs a severe regime — every call costs euros in tokens and seconds in latency — while the retrieval one can be far more permissive — every call costs milliseconds and almost no money. Applying the same limit to both means either strangling the cheap consumer or leaving the expensive one unprotected. When a colleague asks for access to retrieval for an ad-hoc analysis script, giving them the same API key that controls the generation endpoint also gives them permission to spend the LLM budget without control.

The three tensions — **blast radius, differentiated rate limiting, credential granularity** — point in the same direction. **The retriever and the generator are two distinct logical services that happen to share a codebase.** Treating them as one is a decision that pays a toll in the long run. This article applies the inverse pattern: two separate routers in FastAPI, two distinct public contracts, two differentiated security regimes, and a Ruby client that invokes the AI service from the business backend. The separation is not an aesthetic refactor; it is what allows Session 10 to evolve the retriever without touching the generator, and what lets the system's operator reason about each layer separately.

## Two routers, two contracts

FastAPI organises endpoints in `APIRouter`, a composition mechanism that lets you separate the service's logic into independent modules that are then mounted onto the main application. The AI service's final structure at the close of S09 looks like this:

```
src/estimator/
├── api/
│   ├── main.py
│   ├── security.py
│   └── routers/
│       ├── retrieval.py
│       └── estimate.py
├── retrieval/
│   ├── query_reformulator.py
│   └── retriever.py
└── generation/
    ├── context_assembler.py
    ├── prompt_builder.py
    └── estimator.py
```

`retrieval.py` exposes the endpoints that consume the retrieval module directly, without touching the generation layer. `estimate.py` exposes the endpoints that orchestrate the complete flow (reformulator → retriever → assembler → generator). `main.py` mounts both routers on the application with different URL prefixes:

```python
from fastapi import FastAPI
from estimator.api.routers import retrieval, estimate

app = FastAPI(title="Estimator AI Service", version="0.9.0")

app.include_router(retrieval.router, prefix="/v1/retrieval", tags=["retrieval"])
app.include_router(estimate.router, prefix="/v1/estimate", tags=["estimate"])
```

The public contracts are completely different. The retrieval router exposes two endpoints, both with strict Pydantic schemas:

```python
from fastapi import APIRouter, Depends
from pydantic import BaseModel, Field
from estimator.retrieval.retriever import search_chunks

router = APIRouter()


class SearchRequest(BaseModel):
    query_text: str = Field(min_length=10, max_length=2000)
    top_k: int = Field(default=10, ge=1, le=30)
    distance_threshold: float = Field(default=0.6, ge=0.0, le=2.0)
    sectors: list[str] | None = None
    project_year_min: int | None = Field(default=None, ge=2010, le=2100)
    chunk_types: list[str] | None = None


class SearchResponseChunk(BaseModel):
    id: int
    content: str
    sector: str
    project_year: int
    chunk_type: str
    distance: float


class SearchResponse(BaseModel):
    chunks: list[SearchResponseChunk]
    low_confidence: bool
    total_candidates_considered: int


@router.post("/search", response_model=SearchResponse)
def search(req: SearchRequest, _: str = Depends(require_retrieval_key)):
    result = search_chunks(
        query_text=req.query_text,
        top_k=req.top_k,
        distance_threshold=req.distance_threshold,
        sectors=req.sectors,
        project_year_min=req.project_year_min,
        chunk_types=req.chunk_types,
    )
    return SearchResponse(
        chunks=[SearchResponseChunk(**c.model_dump()) for c in result.chunks],
        low_confidence=len(result.chunks) == 0,
        total_candidates_considered=result.candidates_evaluated,
    )
```

Two deliberate details in this endpoint. First, **the contract is exhaustive**: the response includes `low_confidence` and `total_candidates_considered` as top-level fields, neither nested nor optional. The consumer always knows, without parsing anything odd, whether the retriever found relevant material. Second, `Depends(require_retrieval_key)` applies retrieval-specific authentication, distinct from estimate's — the detail of how that dependency resolves we will see in the security section.

The estimate router has a simpler contract because it encapsulates more:

```python
from estimator.generation.estimator import estimate_from_transcript

router = APIRouter()


class EstimateRequest(BaseModel):
    transcript: str = Field(min_length=100, max_length=50000)
    idempotency_key: str | None = Field(default=None, max_length=128)


@router.post("/from-transcript", response_model=Estimate)
def estimate(req: EstimateRequest, _: str = Depends(require_estimate_key)):
    return estimate_from_transcript(
        transcript=req.transcript,
        idempotency_key=req.idempotency_key,
    )
```

**The asymmetry between the two contracts is deliberate.** The retrieval endpoint exposes operational levers — `top_k`, `distance_threshold`, filters — because its consumers may be internal teams who want to tune the behaviour for their case. The estimate endpoint exposes only the minimum input (the transcript) because all the internal complexity must be managed by the service, not by the client. The Rails business backend should not know what `top_k` is used internally; what it needs to know is "I hand it a transcript, it hands me back a validated estimate".

> *(Figure in the original: `art_5_figura-13-topologia-routers.jpg` — image not included in this repo.)*

## API keys and constant-time comparison

The AI service's authentication is done with API keys. The justification is practical: the service's consumer is always another internal service (the business backend or, occasionally, the team's internal scripts), not an end user with a personal identity. For that pattern, API keys are the right thing: simple, stateless, with no need for an OAuth flow, with no additional identity server. What changes relative to a single global API key is two things: there are **two separate keys** (one for retrieval, another for estimate) and the comparison is done with `secrets.compare_digest` instead of `==`.

```python
import os
import secrets
from fastapi import Header, HTTPException, status

RETRIEVAL_API_KEY = os.environ["RETRIEVAL_API_KEY"]
ESTIMATE_API_KEY = os.environ["ESTIMATE_API_KEY"]


def require_retrieval_key(x_api_key: str = Header(...)) -> str:
    if not secrets.compare_digest(x_api_key, RETRIEVAL_API_KEY):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid API key",
        )
    return x_api_key


def require_estimate_key(x_api_key: str = Header(...)) -> str:
    if not secrets.compare_digest(x_api_key, ESTIMATE_API_KEY):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid API key",
        )
    return x_api_key
```

The use of `secrets.compare_digest` instead of the `==` operator is a cryptography detail that deserves a line of explanation. Python's native comparison with `==` over strings is **non-constant-time**: it terminates as soon as it finds the first differing character. This creates a measurable side channel — an attacker who measures how long the server takes to respond with a 401 can infer, byte by byte, how close their key is to the real one. It is a known attack (**timing attack**) and although the differential latency is on the order of microseconds, over a local network with many requests it is exploitable. `compare_digest` is designed specifically for comparing secrets: it takes the same time regardless of how "close" the supplied key is to the real one. The cost of using the safe version is zero — they are the same line of code — so there is no justification for using `==` with secrets.

The operational detail of **key rotation** is worth naming even though the implementation falls outside S09's scope. API keys should be rotated periodically and the rotation must be graceful: during the rotation window, two valid keys at once (the old and the new), so the consumer can update its configuration with no downtime. The typical pattern is to load `RETRIEVAL_API_KEY` and `RETRIEVAL_API_KEY_PREVIOUS` and accept both; once all consumers have migrated, the old one is withdrawn. Rotation is covered briefly in S15 (going to production) but the current code is already prepared for it with no additional effort.

## Differentiated rate limiting with slowapi

`slowapi` is the rate-limiting library the programme adopts for its natural integration with FastAPI: it uses Starlette directly and is mounted as middleware with no structural changes. The minimum installation and configuration:

```python
from slowapi import Limiter
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from slowapi.middleware import SlowAPIMiddleware


def get_api_key(request) -> str:
    return request.headers.get("x-api-key", get_remote_address(request))


limiter = Limiter(key_func=get_api_key)
app.state.limiter = limiter
app.add_middleware(SlowAPIMiddleware)
app.add_exception_handler(RateLimitExceeded, custom_rate_limit_handler)
```

The `key_func` is what makes the rate limiting **per API key instead of per IP**. The `get_api_key` function takes the `X-API-Key` header when present and falls back to the client's IP as a last resort for unauthenticated requests (typically the ones that are going to fail anyway). This choice matters: if the Rails business backend shares a single IP — because it lives on a server behind NAT, or because it sits behind a shared proxy — per-IP rate limiting would be trivial to saturate and would block legitimate users. Per API key, each consumer has its own token bucket.

The two regimes are applied as specific decorators on each endpoint:

```python
@router.post("/search", response_model=SearchResponse)
@limiter.limit("120/minute")
def search(request, req: SearchRequest, _: str = Depends(require_retrieval_key)):
    ...


@router.post("/from-transcript", response_model=Estimate)
@limiter.limit("10/minute")
def estimate(request, req: EstimateRequest, _: str = Depends(require_estimate_key)):
    ...
```

The numbers — 120/minute for retrieval, 10/minute for estimate — are a first approximation based on expected costs. A retrieval request costs on the order of a millisecond of latency and nothing significant in infrastructure; allowing 120 per minute per consumer is generous but reasonable. An estimate request costs between five and fifteen seconds of latency and between twenty cents and one euro in tokens; ten per minute per consumer is already 600 per hour, which for a typical sales team is more than enough and for everyone else creates protection against runaway costs. **These numbers are not universal**; the system's operator calibrates them by observing the real usage pattern.

When the limit is exceeded, the response should be informative so the client knows how to recover:

```python
from fastapi import Request
from fastapi.responses import JSONResponse


def custom_rate_limit_handler(request: Request, exc: RateLimitExceeded):
    return JSONResponse(
        status_code=status.HTTP_429_TOO_MANY_REQUESTS,
        content={
            "error": "rate_limit_exceeded",
            "limit": str(exc.detail),
            "retry_after_seconds": 60,
        },
        headers={"Retry-After": "60"},
    )
```

The `Retry-After` header is the HTTP standard that well-built clients consult to decide when to retry; the `retry_after_seconds` field in the body is the friendly version for frontends that prefer JSON. Both communicate the same thing in two ways.

## Idempotency: duplicate requests, one single estimate

The estimate endpoint has a characteristic the retrieval one does not: **every call costs significantly**. If the Rails business backend retries a request because its HTTP client cut the socket on a timeout, you do not want the AI service to generate a second estimate, duplicate the cost, and produce a result different from the first one through the LLM's inherent variability. The standard pattern for avoiding this is **idempotency keys**.

The contract is: the client sends an `idempotency_key` field (a UUID it generates) on every estimate request. The AI service stores in a temporary cache — Redis, or memory in an MVP version — the association `idempotency_key → estimate`. If a request arrives with an already-known `idempotency_key`, the service returns the cached estimate without calling the LLM again. If not, it processes the request normally and saves the result.

```python
import json
from estimator.cache import idempotency_store  # Redis wrapper, TTL = 24h


def estimate_from_transcript(transcript: str, idempotency_key: str | None = None) -> Estimate:
    if idempotency_key:
        cached = idempotency_store.get(idempotency_key)
        if cached:
            return Estimate.model_validate_json(cached)

    structured_query = reformulate_query(transcript)
    retrieved = search_chunks(...)
    context_block = build_context_block(retrieved.chunks)
    estimate = generate_estimate(context_block, structured_query)

    if idempotency_key:
        idempotency_store.set(
            idempotency_key,
            estimate.model_dump_json(),
            ttl_seconds=86400,
        )
    return estimate
```

The 24-hour TTL is a deliberate decision. Too short and the client's legitimate retries fall outside the window; too long and the cache becomes an implicit repository of historical estimates — something that should live in the business backend's database, not in the AI service's cache. Twenty-four hours covers the realistic scenario (a retry typically happens within minutes of the original failure) and keeps the cache manageable.

There is a subtlety with idempotency worth mentioning. The client may send the same `idempotency_key` with a slightly different `transcript` — for example, if they edit the text and retry. Without protection, the service will return the cached estimate from the first transcript, confusing the user. The standard protection is to hash the transcript and store the hash alongside the estimate; if a later request with the same key brings a different hash, a **409 Conflict** is returned with a message explaining the problem. The implementation is left as an optional improvement outside S09's scope but the pattern is worth knowing.

> *(Figure in the original: `articulo-05-figura-02-halfvec.jpg` — image not included in this repo.)*

> *(Editor's note: that filename and the one below — `articulo-05-figura-03-senales-migracion.jpg` — are almost certainly wrong in the source. `halfvec` quantisation and the three migration signals are the subject matter of s08-05, which is likewise "article 5", and neither figure has anything to do with idempotency or logging. Treat both placements as copy-paste artefacts rather than as clues to what the figures showed.)*

## Structured logging by stage

The AI service has five internal stages that can fail in different ways (reformulation, retrieval, assembly, generation, validation) and efficient debugging demands distinguishing them. The programme adopts `structlog` with JSON output so every log line is parseable by observability tooling — the detail of which specific tool (Logfire, Langfuse, Helicone) is covered in S15. S09's configuration is the minimum base:

```python
import structlog
import time
import uuid
from contextlib import contextmanager

logger = structlog.get_logger()


@contextmanager
def log_stage(stage: str, request_id: str, **context):
    start = time.perf_counter()
    log = logger.bind(stage=stage, request_id=request_id, **context)
    log.info("stage.started")
    try:
        yield log
        duration_ms = (time.perf_counter() - start) * 1000
        log.info("stage.completed", duration_ms=round(duration_ms, 2))
    except Exception as exc:
        duration_ms = (time.perf_counter() - start) * 1000
        log.exception("stage.failed", duration_ms=round(duration_ms, 2), error=str(exc))
        raise


def estimate_from_transcript(transcript: str, idempotency_key: str | None = None) -> Estimate:
    request_id = str(uuid.uuid4())
    with log_stage("reformulation", request_id):
        structured_query = reformulate_query(transcript)
    with log_stage("retrieval", request_id, sectors=structured_query.sector):
        retrieved = search_chunks(...)
    with log_stage("context_assembly", request_id, chunks=len(retrieved.chunks)):
        context_block = build_context_block(retrieved.chunks)
    with log_stage("generation", request_id, confidence_target="adaptive"):
        estimate = generate_estimate(context_block, structured_query)
    with log_stage("validation", request_id):
        validate_estimate(estimate, retrieved.chunks)
    return estimate
```

The `request_id` is what ties all of a request's log lines into a coherent trace; without it, when you inspect the logs you are going to see entries from five different stages intermingled with entries from other concurrent requests and it will be impossible to reconstruct what happened in which. The `request_id` is also included as an `X-Request-ID` header in the AI service's response, so the business backend can correlate its own logs with the service's when something breaks.

Two additional per-stage attributes the programme always includes: `duration_ms` to detect latency regressions, and a specific field that helps debugging — `sectors` in retrieval (which filters were applied?), `chunks` in assembly (how many chunks ended up in the context?), `confidence` in validation (what level did the model return?). These fields are the ones that, when in three months a client reports a strange estimate, will let you reconstruct the chain of decisions the service made without having to reproduce the request.

> *(Figure in the original: `articulo-05-figura-03-senales-migracion.jpg` — image not included in this repo; see the note above on this filename.)*

## The Ruby client from the business backend

The AI service is assembled. The way the Rails business backend invokes it is what closes the pattern. The programme shows the client in Ruby for alignment with the reference implementation, but **the pattern is independent of the stack** — any HTTP client in any language serves.

```ruby
require "faraday"
require "faraday/retry"
require "securerandom"

class EstimatorClient
  ESTIMATE_TIMEOUT = 30  # seconds
  RETRY_OPTIONS = {
    max: 2,
    interval: 1.5,
    backoff_factor: 2,
    retry_statuses: [502, 503, 504],
    methods: [:post],
  }.freeze

  def initialize(base_url:, api_key:)
    @conn = Faraday.new(url: base_url) do |f|
      f.request :json
      f.request :retry, RETRY_OPTIONS
      f.response :json, content_type: /\bjson$/
      f.options.timeout = ESTIMATE_TIMEOUT
      f.options.open_timeout = 5
      f.headers["X-API-Key"] = api_key
    end
  end

  def estimate_from_transcript(transcript:, idempotency_key: SecureRandom.uuid)
    response = @conn.post("/v1/estimate/from-transcript") do |req|
      req.body = {
        transcript: transcript,
        idempotency_key: idempotency_key,
      }
    end
    raise EstimationError, response.body["detail"] if response.status >= 400
    response.body
  end
end
```

Three decisions in this client deserve comment. First, **the differentiated timeouts**: an `open_timeout` of 5 seconds to detect quickly that the AI service is down, and a total `timeout` of 30 seconds to cover the worst case of an LLM call with high `reasoning.effort`. Without these explicit timeouts, the client falls back to Faraday's defaults (60 seconds for everything) and a user who opens an estimation window is left staring at a spinner with no idea whether the system is broken or thinking. Second, **the retry policy is restricted to 5xx** and specifically to codes that indicate a transient failure (502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout); retrying a 400 or a 401 makes no sense — the server is saying the request is malformed — and retrying a pure 500 is ambiguous. Third, **the `idempotency_key` is generated by default in the client** with `SecureRandom.uuid`: every call carries a key even if the calling code does not think about the matter. If Faraday's retry policy fires a retry, the same key travels to the AI service and the idempotency pattern activates automatically without the programmer having to think about it.

## Honest trade-offs

**The choice of API key vs JWT vs mTLS** is the classic internal-service security debate and it is worth naming precisely. API keys have two real limitations: they carry no identity information beyond "someone who has this key", and if the key leaks (in a log, in a badly configured repository, in an exposed environment variable) anyone can use it until it is rotated. JWT mitigates the first (tokens carry claims) but not the second; mTLS mitigates both at the cost of significant operational complexity (certificate management, CA infrastructure, more complex rotation). For an internal service whose only consumer is the Rails business backend and whose client is deployed on controlled infrastructure, API keys are the best cost/benefit option. If the AI service were exposed to multiple external consumers with distinct identities, JWT would become the correct option. If you lived in an infrastructure with a service mesh (Istio, Linkerd), mTLS would be almost free and would be the default option. **The decision depends on the operational context, not on a universal preference.**

**The OWASP API Security Top 10** is the reference the programme cites as complementary reading for students who want to go deeper. The list covers the standard categories of API failures (broken authentication, broken object level authorization, security misconfiguration, etc.) and is worth knowing even though the project's AI service only directly applies two or three of the items. What matters is internalising the reflex of reviewing the list every time a new endpoint is added.

**Rate limiting in-memory vs distributed** is a decision the MVP dodges for simplicity but the operator must know about. `slowapi` by default uses process memory to keep the request count; if the AI service is deployed with multiple workers (gunicorn with `-w 4`) or multiple instances behind a load balancer, each one keeps its own count and the effective limit is multiplied by the number of workers. For the MVP this is acceptable — a single worker in a single container — but S15 will introduce Redis as the rate-limiting backend when the system scales horizontally. The change is configuration, not code: `slowapi` supports Redis natively.

## Connection with the live session

The session's sixth block is the closing of the end-to-end flow and the didactic scenario is deliberately provocative. We are going to set the estimate endpoint's rate limit to an absurdly low number (two per minute), generate three consecutive requests from the Ruby client, and observe Faraday's behaviour when the third comes back with a 429. We will see the `Retry-After` header, we will see how the client respects it, and we will see the effect of the `idempotency_key` when one of the retries gets through to the service: the cached response comes back in milliseconds instead of the LLM's fifteen seconds.

The second scenario is about security. We are going to deliberately leak the retrieval API key in a commit (a real use case we see every month at some client) and discuss the response procedure: immediate rotation, deploy of the new key, withdrawal of the old one, and why having the two keys separated — retrieval and estimate — makes the incident far more manageable than if they were one. If the leaked key had been the service's only key, the attacker would have had access to the endpoint that costs money per request; having them separated limits the damage to data access with no cost, which is serious but not catastrophic.

And the conceptual close of the whole of Session 09: what you have built is no longer a script with an LLM behind it, it is **an operable service**. It has clear contracts, differentiated authentication, reasonable rate limits, idempotency, structured logging and a robust client that invokes it. Session 10 is going to evolve the retrieval layer with reranking and hybrid search, and that evolution is going to touch exactly one module — the retriever — without the estimate endpoint, the rate limit, the credentials or the Ruby client having to change. The isolation you have built here is what makes that kind of evolution possible. It is what distinguishes a toy RAG system from one the team can operate without fear for years.

> *(Editor's note: session 10 largely bore out that prediction — [reranking](s10-01-reranking.md) and [hybrid search](s10-03-hybrid-search.md) both land inside the retrieval layer and leave the estimate contract untouched. Two later articles push past it, though. [s10-05](s10-05-multi-index-and-routing.md) argues the collection to search belongs in the API contract rather than being guessed by the service — which grows the retrieval router's request schema, exactly the kind of change this article's separation is designed to keep cheap. And [s10-06](s10-06-contextual-and-temporal-filtering.md) adds metadata filters and per-stage toggles that surface as more of the operational levers `SearchRequest` already exposes. The isolation held; the retrieval contract still moved.)*
