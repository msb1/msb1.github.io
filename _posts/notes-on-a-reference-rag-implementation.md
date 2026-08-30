---
layout: post
title: "Notes on a Reference RAG Implementation"
permalink: /posts/notes-on-a-reference-rag-implementation/
---

Retrieval-augmented generation is often presented as a short recipe: embed documents, retrieve a few chunks, and ask a model to answer from them. `rag-bench` is deliberately more complete than that. It is a reference implementation for measuring the behavior of an end-to-end RAG system against the Open RAG Benchmark: a corpus of 1,000 arXiv PDFs and 3,045 question-answer pairs, including text, table, and image-oriented questions.

The project is designed to make the entire path observable. It starts with PDFs in S3-compatible object storage, converts them to Markdown, indexes structured chunks in Qdrant, retrieves with dense and sparse signals, reranks locally, generates a grounded answer, and writes every answer together with its retrieved context and ground truth to JSONL. A second pass scores each record and produces an analysis dashboard plus targeted failure sets. The result is not only a score; it is a dataset for understanding why a system succeeded or failed.

## System shape

The implementation is split into small, runnable stages:

1. `convert.py` reads PDFs from an S3-compatible bucket and uses PyMuPDF/PyMuPDF4LLM to write Markdown back to storage.
2. `chunker.py` removes reference sections, preserves document structure, associates tables with nearby text, creates deterministic chunk identifiers, and indexes hybrid vectors in Qdrant.
3. `rag.py` runs RAG Fusion: query expansion, broad hybrid retrieval, deduplication, reranking, context assembly, and answer generation.
4. `eval.py` and `evaluate.py` score the stored records using custom LLM-as-a-judge metrics.
5. `analyze.py` summarizes distributions and exports the records that warrant investigation.

The components use local OpenAI-compatible endpoints (for example, LM Studio), Qdrant, and optional MinIO/RustFS-style storage. Client factories keep the provider-facing configuration in one place. A custom embeddings adapter bypasses client-side pre-tokenization and sends batches directly to the local OpenAI-compatible embeddings endpoint. This is useful when the serving runtime accepts ordinary embedding inputs but does not perfectly match an SDK's assumptions.

### Runtime contracts and configuration

The pipeline is driven by environment variables rather than hard-coded file locations. The storage endpoint, bucket, credentials, source and destination prefixes, Qdrant collection, local and remote OpenAI-compatible endpoints, model names, and JSONL paths are all configured outside the application. The main data contracts are intentionally simple:

- `queries.json` maps a benchmark ID to a query and its type/source metadata.
- `answers.json` maps the same ID to the expected answer.
- `results/results.jsonl` contains one generated record per line: `question`, `answer`, `contexts`, and `ground_truth`.
- `results/output.jsonl` adds the five evaluated metrics to the same record-level view.

This separation matters operationally. The expensive retrieval/generation step emits a durable, self-contained artifact before evaluation begins. A new judge prompt or model can therefore be run against exactly the same answers and contexts, without rerunning embedding, retrieval, or generation. Conversely, a retrieval change can be compared against a prior output dataset without confusing generator behavior with evaluator behavior.

The implementation gives different jobs to different models. A remote, temperature-zero model generates alternative retrieval queries; a local temperature-0.7 model generates answers; a local temperature-zero model judges output. This keeps query expansion deterministic, permits slightly more natural answer composition, and makes evaluator output as stable as the chosen local judge allows. The repository's dependencies cover LangChain composition, Qdrant integration, FastEmbed sparse retrieval, PyMuPDF conversion, local OpenAI APIs, pandas analysis, and plotting.

## Preparing a PDF corpus for retrieval

PDFs are a difficult source format: layout, headers, tables, equations, and page artifacts can all damage retrieval if they are treated as flat text. The conversion stage uses `pymupdf4llm.to_markdown`, producing a representation that is far more useful to a structural splitter than raw PDF extraction.

The chunking stage then makes several intentional quality choices:

- It strips a terminal References, Bibliography, or Works Cited section. Citations are usually high-frequency noise for a benchmark asking about the paper's substantive content.
- It first splits on Markdown headings, retaining header text and header metadata, then applies a recursive splitter with a 512-character chunk size and 51-character overlap. This preserves topical boundaries while preventing oversized retrieval units.
- It filters fragments shorter than 60 characters, isolated image references, and math-heavy scraps with too few ordinary words. Those fragments consume retrieval capacity without providing reliable semantic evidence.
- It extracts Markdown tables before splitting, replaces them with temporary placement markers, and stores the corresponding raw table data in chunk metadata. At query time, tables are restored alongside the selected text instead of being discarded or replaced with an LLM-generated summary.

Each resulting document receives a readable logical ID such as `source__chunk__7__42`, then a UUIDv5 derived from that ID. This makes vector IDs deterministic across re-indexing and gives each chunk an audit-friendly identity. Qdrant stores dense embeddings alongside FastEmbed BM25 sparse vectors in a hybrid collection.

### Conversion and storage details

`convert.py` enumerates PDF objects under the configured PDF prefix with an S3 paginator, streams each object into memory, opens it as a PDF, and calls `pymupdf4llm.to_markdown`. The Markdown is written to the matching storage location with `pdf/` replaced by `markdown/` and the content type set to `text/markdown; charset=utf-8`. The source can be a local MinIO/RustFS deployment or another S3-compatible service because the client uses Signature V4 and path-style addressing.

`chunker.py` pages through the Markdown prefix and processes each document independently. This keeps the unit of indexing clear: a single source key becomes a bounded list of chunks, each retaining heading-derived metadata and a logical chunk ID. It logs the number of chunks and the largest and smallest text chunk for each source, which is a practical first check on extraction and splitter behavior.

### Table attachment, filtering, and IDs

Tables are not embedded as ordinary text before splitting because a long table can distort chunk boundaries and separate its caption from its values. The code temporarily replaces each table with a numbered marker, lets structural and recursive splitting occur, then finds markers in each resulting chunk. The marker is removed from the indexed text and the table is placed in `raw_table_content` metadata, with `has_tables` set to true. At answer time, only tables attached to reranked chunks re-enter the prompt.

The filters are equally deliberate. A references section is likely to contain author names and terms that can create lexical false positives. Tiny fragments tend to be page numbers or layout residue. Image-only Markdown strings do not offer text evidence to the current text embedding and generator path. Math-dense fragments with fewer than ten ordinary words are omitted because their symbols are poor evidence for the kinds of natural-language questions in this run. These choices are parameters to test, not universal rules: a formula-focused benchmark should relax the math filter, and a multimodal answerer should index image content rather than discard it.

For a chunk at index `i` in a document with `n` kept chunks, the logical ID contains the storage key, `i + 1`, and `n`. UUIDv5 turns that same logical ID into the same Qdrant point ID on every repeatable index run. This avoids random identifiers that make it difficult to compare stored points across experiments.

## Retrieval and answer generation

The query path uses RAG Fusion rather than a single nearest-neighbor search. A low-temperature query-generation model creates three alternative phrasings of the original question. Each variation retrieves up to twelve candidates from Qdrant's hybrid dense-plus-BM25 retriever. The combined pool is deduplicated by content so repeated chunks do not dominate the next stage.

The remaining candidates are sent in one request to a local FastAPI service running `jina-reranker-v3.5` on a GPU. The service returns document indices and relevance scores; the client maps those indices back to LangChain `Document` objects and keeps the best five. If that service is unavailable, the implementation degrades safely to the original retrieval order rather than aborting a long benchmark run.

For selected chunks with associated tables, the pipeline cleans common PDF-to-Markdown artifacts—such as repeated headers, continuation banners, duplicate separators, and escaped HTML entities—and appends the restored table material to the chunk before context formatting. The answer model therefore receives numbered source blocks containing both the text that won retrieval and the tabular evidence adjacent to it.

The generator is tightly constrained. Its system prompt requires it to use only facts in the retrieved context, avoid source-reference boilerplate, avoid supplementing from training knowledge, and return a specific abstention when the documents do not contain the answer. This is important because a fluent answer without traceable context is not a successful RAG result.

For every query, `rag.py` stores the question, generated answer, final reranked contexts, and benchmark ground truth in `results/results.jsonl`. It also records progress after every item in `progress.json`, allowing an interrupted multi-thousand-query run to resume without repeating completed work.

### Retrieval control flow

The retrieval chain is composed with LangChain Expression Language. The incoming question is assigned to `user_query`; a runnable invokes query expansion and retrieval; a state object captures the *post-rerank* documents; another runnable formats them into the prompt; and the answer model returns plain text. Capturing contexts after reranking is significant: the evaluator scores the evidence actually offered to the answer model, not the larger candidate set that was available before ranking.

For each generated variation, Qdrant's hybrid retriever returns `k=12` candidates. Hybrid mode combines the configured dense vector with a FastEmbed `Qdrant/bm25` sparse vector. Dense retrieval can bridge paraphrase and terminology differences; BM25 gives a strong lexical signal for exact scientific names, variables, and phrases. The code then concatenates candidates from all variations and removes duplicates by exact `page_content`. It does not apply a score-fusion formula at this point; the cross-encoder reranker is the final relevance decision-maker for the unified candidate set.

The reranker client sends the original question, all unique document texts, and `top_n=5` to `/v1/rerank`. The FastAPI service loads `jinaai/jina-reranker-v3.5` onto CUDA at startup, calls the model's native `rerank` method, and returns the input indices and relevance scores. The client reconstructs `Document` instances with a `rerank_score` metadata field. A request timeout of 15 seconds and a fallback to the first five incoming documents favor batch completion over perfect availability; failures are printed so they can be correlated with degraded output later.

### Context construction and answer guardrails

The final context is formatted as a sequence of explicit blocks: a separator, a numbered source ID, the chunk text, and another separator. This is easier to read in prompt traces than an unstructured concatenation. For table-bearing documents, a cleanup step joins table fragments, removes repeated continuation text and duplicate headers, normalizes line breaks and common escaped entities, and appends the result under an “Associated Table Data” heading.

The answer prompt is intentionally restrictive. It asks for facts only from the “Retrieved Context,” instructs the model not to extrapolate or use training knowledge, disallows document-reference flourishes in the prose answer, and specifies the exact abstention string when context is insufficient. This turns a successful answer into a claim that can be checked against stored evidence. The prompt cannot guarantee compliance—that is why faithfulness is still measured—but it establishes a clear behavior contract.

The main loop reads query and answer files, skips records at or before the last saved checkpoint, times each answer, appends its JSONL result immediately, and then advances the checkpoint. Because output is append-only, an interrupted run can continue safely provided the checkpoint and JSONL are kept together. A practical improvement for future runs would be to record model configuration, collection version, reranker availability, and latency in each line as well.

## A custom evaluator instead of a fixed RAGAS call

`rag-bench` does not invoke RAGAS as a black-box metric. It implements an explicit, embedding-free, LLM-as-a-judge evaluation protocol that covers the same essential dimensions while making each judgment step visible and replaceable. The evaluator runs against the local evaluation model at temperature zero and writes five scores for every record:

| Metric | Method |
| --- | --- |
| Faithfulness | Extract independent claims from the generated answer; verify each claim against retrieved context; score the supported-claim ratio. |
| Answer relevancy | Reverse-engineer the question implied by the answer, then ask the judge to compare that intent with the original question. |
| Context recall | Decompose the ground truth into facts; verify whether each fact is present or clearly deducible from retrieved context. |
| Context precision | Split retrieved context into sentences; judge whether each is useful for the question given the ground truth. |
| Answer correctness | Classify factual overlap as TP, FP, and FN in JSON, then calculate `TP / (TP + 0.5 × (FP + FN))`. |

The explicit decomposition is the important design decision. A low faithfulness score can be traced to unsupported answer claims. A low recall score identifies ground-truth facts the retriever failed to surface. A poor precision score exposes distracting sentences rather than merely reporting a weak aggregate. And answer correctness distinguishes omissions from unsupported additions through its TP/FP/FN breakdown. When the judge produces malformed JSON for correctness, the code falls back to a direct semantic score so a single formatting error does not destroy a batch run.

This is a better fit than a fixed RAGAS invocation when the goal is to evaluate *this* system and evolve the benchmark with it. The project can change the definition of a fact, the relevance rubric, the judge prompt, the evidence granularity, the model, or the failure thresholds without waiting for a library abstraction to expose the necessary control. It also avoids adding a second embedding-based similarity path solely for evaluation; the custom relevancy method uses semantic intent classification instead.

That flexibility should not be confused with claiming that custom evaluation is universally more accurate than RAGAS. RAGAS is valuable for standardized, comparable evaluation. The trade-off here is deliberate: `rag-bench` favors local control, inspectable intermediate judgments, and task-specific criteria. To make those benefits credible, prompts and judge models should be versioned, outputs should be sampled by humans, and scores should be tracked across repeated runs to measure judge variance.

### Metric-by-metric implementation detail

**Faithfulness** begins by asking the judge to turn an answer into a bullet list of independent factual claims. The evaluator normalizes those bullets, then issues one strict YES/NO verification prompt per claim against the formatted retrieved context. The score is `supported_claims / all_claims`; an answer with no extracted claims scores zero. The two-pass design is slower than a single holistic judgment but makes a low score attributable to individual propositions.

**Answer relevancy** asks the judge to infer the concise question that the answer appears to solve. A second prompt scores semantic intent similarity between that reconstructed question and the real question on a 0–1 scale. The code parses the returned float and returns zero if it cannot be parsed. This avoids an evaluator-side embedding model, but it means the score depends on both question reconstruction and similarity judgment; sample reviews are particularly useful here.

**Context recall** applies the same decomposition idea to the ground truth. It extracts core factual statements, asks whether each is directly present or clearly deducible from context, and returns `found_facts / ground_truth_facts`. This distinguishes a generator that ignored available evidence from a retriever that never brought the evidence into the prompt.

**Context precision** treats the formatted context as a set of sentences. For every sentence, the judge receives the question, ground truth, and candidate sentence and returns YES or NO for usefulness. Its ratio is `useful_sentences / all_sentences`. This is intentionally sensitive to context noise; it can surface broad retrieval that appears successful by recall but burdens the generator with irrelevant material.

**Answer correctness** asks for a strict JSON object containing true positives, false positives, and false negatives relative to the ground truth. The calculation `TP / (TP + 0.5 × (FP + FN))` balances omission and extraneous or contradicting claims. If JSON extraction fails—even after stripping common Markdown fences—the evaluator uses a direct 0–1 correctness judgment as a fallback. The fallback preserves throughput but should be logged separately in a future revision because it has a different evidence trail from the structured score.

All judge requests are direct HTTP calls to the local OpenAI-compatible chat-completions endpoint with a 30-second timeout and temperature zero. An API failure currently yields an empty response, which the metric functions generally turn into a zero score. That makes failure visible in aggregates, but production benchmarking should also emit an explicit judge-error field so transport failure is not mistaken for semantic failure.

## Analysis and operational feedback

After scoring, `analyze.py` calculates means, minima, maxima, and standard deviations for all five metrics. It creates a dashboard with both average-score bars and metric-distribution box plots. Aggregates are useful, but the most actionable output is the failure slicing:

- Records below a 0.30 faithfulness threshold are written to `hallucinations.jsonl`.
- Records below a 0.30 context-recall threshold are written to `retrieval_misses.jsonl`.

Those files close the loop. A retrieval miss points back to chunking, embeddings, query expansion, or reranking. A hallucination with good recall points back to the prompt or generator. A low-correctness answer with strong faithfulness can reveal incomplete context, an overly terse response, or a benchmark-label mismatch. The implementation treats evaluation as a diagnostic instrument, not a leaderboard number.

### Reading the outputs

The analysis step reads `output.jsonl` as a pandas DataFrame, keeps the five expected metric columns, drops rows with missing values for plotting, and calculates mean, minimum, maximum, and standard deviation. The dashboard places average performance beside a box plot of per-record score distributions. A mean alone can conceal a system that works well on common cases but fails badly on a meaningful minority; the box plot makes spread and outliers visible.

The failure exports use deliberately severe initial thresholds of 0.30 for faithfulness and context recall. They are triage thresholds, not a declaration that 0.31 is good. The selected records retain question, answer, ground truth, contexts, and all scores, so review begins with concrete evidence rather than a dashboard. A useful review sequence is: inspect a retrieval miss's chunks and query variants; check whether a gold fact was lost during conversion/chunking, absent from hybrid retrieval, or removed by the reranker; then compare the generator answer against the exact context. This is the shortest path from a score to a change in the system.

## Reproducing the run

With Qdrant, the local model endpoints, and optional object storage configured, the normal workflow is:

```bash
python -m src.convert
python -m src.chunker
python -m src.rag
python -m src.eval
python -m src.analyze
```

The Open RAG Benchmark's mix of abstractive and extractive questions—and its text, table, and image-oriented sources—makes it a meaningful stress test for this pipeline. `rag-bench` is therefore useful as a reference not because it eliminates all RAG uncertainty, but because it preserves evidence at every stage and makes the resulting uncertainty measurable.
