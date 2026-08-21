---
title: "From static CAG to the RAG flow: the four stages and why retrieval dominates"
author: Antonio Perez
lang: en
translated_from: es
doc_type: reference-article
series: servicio-ia
session: 9
series_part: 1
scope: evergreen
source: user-supplied article
reading_time: 25 min
added: 2026-08-14
summary: >
  The canonical four stages (Query, Retrieval, Augmentation, Generation), five
  operational differences between CAG and RAG, an honest account of when CAG is
  still the right answer (Chan et al. 2024), and the operating rule that no
  amount of prompt engineering fixes bad retrieval — so retrieval quality is the
  ceiling on the whole system.
keywords: [RAG, CAG, query reformulation, retrieval, augmentation, generation,
           grounding, hallucination, traceability, lost in the middle, hybrid]
---

# From static CAG to the RAG flow: the four stages and why retrieval dominates

*Antonio Perez* · 🔴 25 min

The system you closed in Session 05 works reasonably well up to a specific point. That point has a name: the budgets that were in your system prompt when you set the CAG up. As long as a new transcript describes projects similar to the ones you saw when you selected the static context, the model produces decent estimates leaning on those examples. Quality falls sharply the moment the client mentions something your prompt does not cover.

Imagine you receive a transcript where the client says *"I need something like Stripe Connect but adapted to healthcare marketplaces in Germany, with KYC reinforced by BaFin and accounting reconciliation against SAP"*. Your CAG, with the 30 budgets you put in the system prompt when you built Module 2, probably has no similar project. The model will do one of two things: either invent an estimate leaning on its generic parametric knowledge, or produce a reasonable number with no real grounding in your company's data. Both results are unacceptable for a system whose reason to exist is estimating from real historical budgets.

The problem is not the model's. The problem is that the system's knowledge is frozen at the moment you built the prompt. If your company closes five new projects this week in sectors that were not represented, the system does not find out until you redo the system prompt and redeploy. If you have 800 historical budgets and only 30 fit reasonably in context, the other 770 are out of the model's reach forever.

The way past that ceiling is changing the architectural question. Instead of *"which budgets do I put in the prompt so the model always has references"*, the question becomes *"how do I find, at request time, the budgets most relevant to the transcript I just received, and give them to the model"*. That is the RAG flow. And like any non-trivial architectural change, it comes with a cost: four stages, four failure points, four places where quality can degrade. This article takes those four stages apart and explains why the module's first operational lesson is that **retrieval dominates everything else**.

## The anatomy of the RAG flow

The RAG pattern has four canonical stages: **Query, Retrieval, Augmentation and Generation**. The nomenclature is the one established by Lewis et al.'s original paper in 2020, and it has held without significant change since, despite the many variants that have appeared (Agentic RAG, GraphRAG, Self-RAG and company). Every modern variant is a sophistication of one of these four stages, not a reorganisation of the flow.

**The Query stage** converts the user's input into something the search system can use. This is less trivial than it sounds. In a textbook RAG system, "the query" is the user's question as-is, and the conversion is just embedding that question into the same vector space where the chunks live. In your project the input is a transcript of several thousand tokens with conversational noise, digressions, anaphora and a mix of topics. **Embedding that directly and comparing against 300-token chunks produces vectors that sit "somewhere averaged" in semantic space and barely discriminate** between the historical budgets. The Query stage, in serious RAG systems, does real work: it extracts the key requirements from the transcript, decomposes if there are multiple sub-topics, generates queries optimised for semantic search. We come back to this in Article 2.

**The Retrieval stage** is where your vector database comes in. It receives one or more optimised queries and returns the most relevant chunks from the corpus. The system you left built at the close of Session 08 already does this, but in its most naive version: top-K by cosine similarity, no filters, no relevance threshold. In production, real retrieval combines vector similarity with structural filters (sector, year, cost range), thresholds that discard weakly relevant results, and eventually reranking with cross-encoders to refine the order — that last one arrives in Session 10. **The key detail is that this stage has the greatest impact on the system's final quality.** We go deeper in Article 3.

**The Augmentation stage** assembles the retrieved chunks into a context block the LLM can use well. The naive reflex is to concatenate the chunks with a separator and put them in the prompt. And that is exactly what produces hallucinations, invented citations, or answers that ignore the context. The Augmentation stage, done well, decides in what order the chunks go (mitigating the "lost in the middle" phenomenon), what delimiters it uses so the model can distinguish sources, how it truncates when the context exceeds the token budget, and what metadata accompanies each chunk to allow citation. Article 4 goes into detail.

**The Generation stage** is the LLM call with the context already assembled and a prompt that instructs the model on how to use it. Here too the difference between a functional RAG and a serious one is in the details: the prompt must force grounding ("use only the provided context, not your general knowledge"), must define the behaviour when context is insufficient ("if you do not have enough evidence, say so, do not invent"), must specify the output format (JSON with the estimate, the cited sources, the assumptions made, the estimated confidence), and must allow subsequent validation of citations to detect hallucinations. Article 4 closes it together with Augmentation.

The four stages chain into a pipeline. In code, the flow's orchestrator looks roughly like this:

```python
async def estimate_from_transcript(transcript: str) -> EstimateResponse:
    structured_query = await query_reformulator.reformulate(transcript)
    chunks = await retriever.search(
        query=structured_query,
        top_k=10,
        threshold=0.65,
        filters=structured_query.metadata_filters,
    )
    context = context_assembler.assemble(chunks, max_tokens=4000)
    estimate = await generator.generate(
        transcript=transcript,
        context=context,
        schema=EstimateSchema,
    )
    return estimate
```

Five operational lines. Each of them is a failure point, a design decision, a potential source of quality degradation. The complexity of the whole module is, essentially, in making each of those five calls robust, observable and independently improvable.

> *(Figure in the original: `art_1_figura-1-anatomia-flujo-rag.jpg` — image not included in this repo.)*

On the AI service, this pipeline introduces two new modules relative to the state at the close of Session 08. The `retrieval/` folder holds `query_reformulator.py` and `retriever.py`. The `generation/` folder holds `context_assembler.py`, `prompt_builder.py` and `estimator.py` (this last one being the orchestrator from the fragment above). The existing modules — `ingest/` from Session 06, `embedding_pipeline/` from Session 07, `storage/` from Session 08 — are not touched; they remain stable dependencies the new pieces consume. That separation is not aesthetic: it means the retriever can evolve in Session 10 by introducing reranking without touching the generation module, and vice versa.

> *(Figure in the original: `art_1_figura-2-modulos-servicio-ia.jpg` — image not included in this repo.)*

## The operational difference from Module 2's CAG

When a student first sees the contrast between CAG and RAG, the reflex is to think the difference is the "size" of the context: CAG puts the budgets in the system prompt once, RAG selects them dynamically per request. That is true but it is the tip of the iceberg. The operational differences that matter are five, and each has measurable implications for the system you are going to build.

**The first is knowledge freshness.** In CAG, adding a new historical budget requires editing the system prompt, validating that it does not break the provider's prompt cache, and redeploying. In RAG, adding a budget is a `POST /v1/retrieval/insert` the business backend can call when a new sale closes. The system stays synchronised in minutes without touching code or service. This difference alone justifies RAG in any company that closes projects regularly: the system stops ageing.

**The second is the corpus ceiling.** CAG is limited by the model's context size. With gpt-5 and its 400,000 tokens you can fit a good many budgets, but if your company has 800 historical projects averaging 3,000 tokens each, the full corpus is 2.4 million tokens; not even the most generous models let you fit everything. RAG has no such ceiling: the corpus grows as much as you want and you only send the relevant chunks to the model. The system scales with your company, not with the model.

**The third is latency and cost per request.** Here RAG loses, and it is important to acknowledge that openly. CAG makes a single LLM call with a static prompt that benefits from aggressive prompt caching at OpenAI or Anthropic; latency is a second or two and cost is low. RAG makes, in the minimum flow, an LLM call for reformulation, one to the embeddings service, a query to pgvector, and a final LLM call for generation; latency accumulates and cost per request can be three or four times CAG's. If your request volume is high and your corpus is stable, that is not trivial. If your volume is low or moderate and quality matters more, it is a cost worth paying. **The decision depends on the case, not on the pattern's prestige.**

**The fourth is traceability and audit.** If a client questions an estimate your system produced three months ago, in CAG you have no way to explain where that number came from: the model leaned on some diffuse subset of the system prompt mixed with its parametric knowledge, and nobody knows in what proportion. In RAG you can answer precisely: these are the historical budgets that were retrieved, this is the context that was assembled, this is the estimate that was generated. If your system has to answer audits, demanding technical clients, or regulation, this difference is decisive.

**The fifth is resistance to hallucination.** When you ask an LLM for an estimate grounded in data that is not in its context, the model fills the gaps. CAG, even though its data *is* in context, does not force the model to lean on it: nothing in the prompt stops the model ignoring the system prompt's budgets and producing a number of its own invention. RAG, with a well-built generation prompt (explicit instruction to "use only the provided context, cite the sources, declare when evidence is insufficient"), drastically reduces the hallucination rate. It does not eliminate it; it reduces it measurably and reproducibly.

These five differences boil down to one idea: **CAG is the system that assumes the correct context is always available because you chose it in advance; RAG is the system that assumes the correct context must be fetched each time and that this search is the heart of the system.** RAG's assumption is more robust to reality but puts more complexity in the architecture. There is no free lunch.

> *(Figure in the original: `art_1_figura-3-comparativa-cag-rag.jpg` — image not included in this repo.)*

## When CAG is still the right answer

Before going on, it is worth qualifying a claim that beats under everything above and that is badly told in the industry literature: that RAG is "better" than CAG. It is not. It is different and applies to different problems.

The paper worth reading here is **"Don't Do RAG: When Cache-Augmented Generation is All You Need for Knowledge Tasks"** by Chan et al. (2024), which demonstrated empirically that **when the corpus is small and stable, CAG is not only simpler but produces better answers** on standard QA metrics. The reason is intuitive: in CAG the model "sees" the whole corpus in a single full attention pass; in RAG the model sees a subset selected by a retriever that can be wrong. When the cost of retriever error is high and the corpus fits in context, CAG wins.

For your project in particular, RAG is justified because a serious software company's historical budget corpus grows over time and does not fit whole in context. But in other architecture decisions you will make outside the programme, it is worth being clear about this nuance: **CAG is perfectly respectable for corpora under about 100,000 effective tokens, stable, where cost per request matters and traceability is not critical.** Internal support systems based on a company FAQ, technical assistants over product documentation that changes little, legal tools over a fixed body of regulation. There are serious products in production that are CAG and should be.

There is also a third way worth mentioning: **hybrid CAG + RAG systems**, where the stable base context (policies, terminology, domain structure) goes via CAG and the variable context (case-specific data) goes via RAG. Chan et al.'s paper mentions it as a future direction and it is reasonable; in your project, the JSON output templates, the coherence rules between budget components, and the domain's technical terminology could go as CAG in the system prompt while similar historical budgets are retrieved by RAG. It is not what we are going to build in this module — the added complexity is not justified for the programme's scope — but it is the natural next step if you want to optimise later.

## Retrieval dominates

There is a mantra in the industrial RAG literature of 2025 and 2026 worth internalising before going on: **"no amount of prompt engineering fixes bad retrieval"**. It appears in technical blogs, in post-mortems of failed projects, in conference talks. The reason it has become a slogan is not that it sounds good but that it describes the most common failure mode in production RAG systems.

The intuition is this. If your retrieval returns the correct chunks, even a mediocre generation prompt produces decent answers: the model has evidence in front of it and uses it. If your retrieval returns irrelevant chunks, no generation prompt, however sophisticated, is going to save you: the model cannot invent evidence it does not have, and if you force it to lean only on the context, it will answer "I do not have enough information". **The quality ceiling of the entire system is fixed by the quality of retrieval.** The other three stages can push you toward that ceiling, but not above it.

The operational consequence is that the priority order when a RAG system fails is always the same. First, **am I retrieving the correct chunks?** If the answer is no, everything else is noise. Second, **am I assembling the context well?** If the correct chunks reach the LLM but it is ignoring them, look at the Augmentation stage. Third, **does the generation prompt force grounding adequately?** And only last, **is the model capable enough?** The temptation to start with the model is strong and almost always wrong.

This does not mean the other stages do not matter — they do, and Articles 4 and 5 go deep into Augmentation and Generation. It means **retrieval engineering is where the leverage is.** And that is why this module devotes the rest of the sessions (10 on retrieval techniques, 11 on evaluation) to making that stage better and better.

## Connection with the live session

In the session's first block we are going to run the same ambiguous transcript through two systems: the CAG you closed in Session 05 and a skeleton of the RAG you will build during the module. Both will produce an estimate. We will measure four things: the answer produced (is it coherent with the available data?), total latency, cost in tokens, and traceability (can you explain where the numbers come from?). Each student's results will diverge quite a bit because each will have closed the CAG with different budgets in the prompt; that divergence is exactly what opens the conversation about when the RAG pattern starts to justify itself.

If you have already done the pre-session exercise, you will arrive with the trace of the ambiguous transcript over your current system and with the five failures identified. Some of those failures will map directly onto one of the four stages you have just read: the raw query does not retrieve well (Query stage), the returned chunks are irrelevant or mixed (Retrieval stage), naive concatenation of the context produces poor answers (Augmentation stage), the model invents or does not cite (Generation stage). Arriving with that mental correspondence made is worth more than any explanation I can give you live. The rest of the module is, essentially, attacking those four stages one by one until the system does what the project has asked of it since the first session: estimate based on the company's real budgets, not on the model's intuition.
