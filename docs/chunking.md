# Chunking: The Most Underrated Lever in AI-Ready Content

You can have perfect metadata, a clean taxonomy, and beautifully structured Markdown, and your AI assistant can still return half an answer or the wrong one. The reason is almost always chunking.

## What chunking actually is

Before your content can be retrieved by a RAG pipeline, it is split into pieces — chunks — and each chunk is converted into a vector embedding. When a user asks a question, the system embeds the question, finds the chunks whose embeddings sit closest to it, and feeds those chunks to the model as context.

The model never sees your document. It sees the handful of chunks that were retrieved. So the boundary decisions — where one chunk ends and the next begins — quietly determine answer quality more than almost anything else in the pipeline.

<div class="kl-boundary">
retrieval quality ≈ f(chunk boundaries, chunk size, context enrichment, metadata)
</div>

## Why bad chunking breaks good content

Three failure modes account for most retrieval problems:

=== "Split mid-thought"

    A fixed-size splitter cuts every N tokens regardless of meaning. A procedure gets severed between step 3 and step 4. The model retrieves step 3, answers confidently, and the user never learns steps 4 through 7.

=== "Chunks too large"

    If a chunk holds an entire page, its embedding averages several topics together. It matches weakly to everything and strongly to nothing, so retrieval becomes vague and the model gets diluted context.

=== "Chunks too small"

    A single sentence with no surrounding context ("Click **Save**.") retrieves cleanly but tells the model nothing about *what* is being saved or *where* in the flow it happens.

## The five chunking strategies

Most pipelines use one of these. They are listed roughly from least to most content-aware.

### 1. Fixed-size chunking

Split every N tokens with an overlap window. Simple, fast, content-blind.

```{ .python .annotate }
def fixed_size_chunks(text, size=512, overlap=64):
    tokens = tokenize(text)                      # (1)
    step = size - overlap
    return [
        detokenize(tokens[i : i + size])
        for i in range(0, len(tokens), step)     # (2)
    ]
```

1.  Use the *same* tokenizer your embedding model uses, or your token counts will drift from what the model actually sees.
2.  The overlap (`size - step`) keeps a sentence that lands on a boundary from being orphaned in one chunk.

**Use when:** content is uniform and unstructured (chat logs, transcripts). **Avoid when:** content has real structure — you will cut through it.

### 2. Recursive character/token splitting

Try to split on the largest natural separator first (paragraphs), then fall back to smaller ones (sentences, words) until each piece fits the size budget. This is the common LangChain default and a reasonable baseline.

**Use when:** you want a safe general-purpose default. **Limitation:** it respects *typographic* structure, not *semantic* structure — it does not know a heading introduces the paragraph beneath it.

### 3. Document-structure (heading-anchored) chunking

Split on the document's own hierarchy — headings and subheadings — so each chunk is one coherent section. This is the sweet spot for technical documentation, because docs are already written as modular topics.

```{ .python .annotate }
def heading_anchored_chunks(markdown, max_tokens=500):
    sections = split_on_headings(markdown)       # (1)
    chunks = []
    for section in sections:
        if count_tokens(section.body) <= max_tokens:
            chunks.append(section)               # (2)
        else:
            chunks.extend(
                split_by_paragraph(section, max_tokens)  # (3)
            )
    return chunks
```

1.  Parse `#`, `##`, `###` into a tree; each node carries its heading path.
2.  A section that already fits becomes exactly one chunk — one idea, one embedding.
3.  Only oversized sections get sub-split, and they inherit the parent heading path.

**Use when:** content is structured (docs, KBs, DITA/Markdown). This is what most doc teams should reach for first.

### 4. Semantic chunking

Embed each sentence, then start a new chunk wherever the similarity between consecutive sentences drops below a threshold — i.e. cut where the topic actually shifts.

```{ .python .annotate }
def semantic_chunks(sentences, threshold=0.75):
    embeddings = embed(sentences)
    chunks, current = [], [sentences[0]]
    for i in range(1, len(sentences)):
        sim = cosine(embeddings[i-1], embeddings[i])   # (1)
        if sim < threshold:                            # (2)
            chunks.append(" ".join(current)); current = []
        current.append(sentences[i])
    if current: chunks.append(" ".join(current))
    return chunks
```

1.  Adjacent-sentence cosine similarity is a cheap proxy for "are these about the same thing":

    \[
    \text{sim}(u, v) = \frac{u \cdot v}{\lVert u \rVert \, \lVert v \rVert}
    \]

2.  Below threshold means the topic drifted — a natural boundary.

**Use when:** prose runs long without reliable headings (whitepapers, articles). **Cost:** an extra embedding pass at ingest time.

### 5. Late chunking / contextual retrieval

Newer approaches embed the *whole* document first (or prepend an LLM-generated context blurb to each chunk) so every chunk's embedding carries document-level meaning. Reduces the "orphaned chunk" problem at the cost of more compute.

**Use when:** accuracy matters more than ingest cost and you have the pipeline maturity to support it.

## Token math you should keep in your head

- **Embedding model window.** Most embedding models cap around 512–8192 tokens. A chunk larger than the window is ==silently truncated== — the tail never gets embedded.
- **Rule of thumb.** ~4 characters ≈ 1 token in English; ~750 words ≈ 1000 tokens.
- **Practical target.** For docs, 200–500 tokens per chunk balances context against precision. Below ~100 you lose context; above ~800 you dilute the embedding.
- **Overlap.** 10–20% overlap on fixed/recursive splits; heading-anchored splitting usually needs little or none because boundaries already fall at coherent points.

Quick glossary, for anyone new to the terms above:

Chunk
:   A single retrievable unit of content — the piece an embedding model turns into a vector and a retriever can return on its own.

Overlap
:   Tokens repeated between adjacent chunks so a sentence that lands on a boundary isn't orphaned in only one of them.

Context window
:   The maximum number of tokens a model (embedding or generation) can accept in a single pass; content beyond it is truncated, not summarized.

## Context enrichment: the cheapest win

Whatever strategy you pick, prepend the document title and heading path to each chunk before embedding. "Click Save" is nearly useless in isolation; enriched, it becomes retrievable and answerable.

**Before** — fixed 500-token split:

```text
...to complete authentication. 4. In the next screen,
select your region and click Continue. The system then
provisions your workspace, which can take up to
```

**After** — heading-anchored + context-enriched + metadata:

```yaml
title: API Authentication > Provisioning
product: data-platform
version: "4.2"
topic_type: procedure
chunk_id: api-auth-prov-002
---
## Provisioning your workspace

After authentication completes, select your region and
click Continue. Provisioning takes up to five minutes;
you will receive an email when the workspace is ready.
```

One coherent, self-contained idea, tagged for filtered retrieval.

## Which strategy should you use?

| Content type | Recommended strategy | Target size | Notes |
|---|---|---|---|
| Technical docs / KB | Heading-anchored | 200–500 tok | Docs are already modular; use it |
| Long-form prose | Semantic | 250–500 tok | Cut where the topic shifts |
| Transcripts / chat | Fixed-size + overlap | 300–512 tok | Little structure to preserve |
| Mixed corpus | Recursive baseline | 400 tok | Safe default, then refine |
| Accuracy-critical | Late / contextual | varies | Worth the extra compute |

## How to know it is working

Chunking is testable. Build a small set of real questions with known correct source sections, then measure:

- **Context recall** — did the right chunk get retrieved at all?
- **Chunk coherence** — is each chunk a complete, self-contained idea?
- **Boundary integrity** — are procedures and tables intact, not severed?

If recall is low, your boundaries or enrichment are wrong — not your model.

Before you sign off a chunking pass as done, run it against this checklist:

- [x] Benchmark question set built from real support/search queries
- [x] Every chunk carries its heading path and source metadata
- [ ] Recall measured against the benchmark, not eyeballed
- [ ] Oversized and undersized chunks flagged and re-split
- [ ] Re-run after any change to the splitter or embedding model

!!! tip "Jump to a strategy"
    Most readers land here to check one strategy. Press ++ctrl+f++ (++cmd+f++ on Mac) and search for the strategy name.

## The takeaway

Chunking is not a preprocessing detail you can leave to a default splitter. It is a content-design decision, and it belongs to the people who understand the content: the documentation team. Structure, metadata, and ontology get you to the door. Chunking decides whether the model walks through it.

!!! tip "This is the approach Knowlayer uses"
    Every migration ships heading-anchored, context-enriched, metadata-tagged chunks, validated against a retrieval benchmark before handover. [See how it works](https://bipin-24.github.io/knowlayer/#contact).

*[RAG]: Retrieval-Augmented Generation
*[KB]: Knowledge Base