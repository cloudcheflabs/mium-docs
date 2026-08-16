# Search: Vector, Full Text, Graph and Images

Some questions have no SQL. "Find products like this one", "search the notes for
the refund policy", "who is two hops from this account", "here is a screenshot -
what is it?" — asking a model to express any of those as analytic SQL produces
either nothing or something confidently wrong.

Ontul answers them with **retrievers**: governed objects, certified like any
metric, that run vector similarity, keyword search and graph traversal over
NeorunBase. Mium calls one with the `search` action, under the asking user's own
Ontul credential.

## What each product does

```
question or pasted image
        │
        ▼
   Mium            picks the retriever, or embeds the image with CLIP
        │          on a worker (512 dimensions, text and image in one space)
        ▼
   Ontul           the retriever: declared parameters, certified,
        │          IAM applied - pushes its statement down
        ▼
   NeorunBase      the engine: HNSW vector index, BM25 full text,
                   GRAPH_NEIGHBORS traversal, over the same rows
```

Mium never talks to NeorunBase directly. It is a connector below Ontul, and the
retriever is the governed surface between them.

## The kinds of retriever

Ontul declares a `kind` on each, and Mium shows it to the model so it can choose
between three of them rather than guessing from names:

| Kind | For |
|---|---|
| `VECTOR` | Similarity by meaning, including across images |
| `FTS` | Full-text keyword search |
| `HYBRID` | Meaning and keywords together, merged |
| `GRAPH` | Walk declared relationships from a starting node |
| `PAGERANK` | Rank nodes by influence in the graph |
| `PATH_EXISTS` | Whether two nodes are connected |

## Asking

In Korean, in any workspace:

```
방수 되는 상품 검색해줘
    → FTS retriever, keyword over Korean text

1번 상품과 비슷한 상품을 벡터로 찾아줘
    → two retrievers, chained by the agent itself: read the reference
      vector, then search by it

1번 상품과 함께 구매한 상품을 두 다리 건너까지 보여줘
    → GRAPH_NEIGHBORS with maxDepth=3; results carry the hop count, so
      "1" is bought together and "2" is one step removed
```

The model maps the phrasing onto the retriever's declared arguments — "두 다리
건너" becomes a depth bound — and the answer says which retriever ran.

## Searching by an image

Paste a screenshot into the prompt (`Ctrl+V`), or use the *이미지로 검색* button.

The worker's CLIP daemon turns the pixels into a 512-dimension vector, and that
vector goes to a retriever with a declared `VECTOR` parameter. The catalogue must
be embedded with **the same model** — CLIP aligns its text and image encoders
during training, which is what lets a photo and a typed phrase land in one space;
two different models produce scores that sort and mean nothing.

Requirements:

- `mium.embedding.enabled` and `mium.embedding.clip.enabled`, or the equivalent
  under **Settings → Embedding**. The image backend is wired from that setting.
- The worker image needs `torch` and `sentence-transformers`; weights download on
  first use into `mium.embedding.hf.home`, which should be a mounted volume so a
  restart does not fetch them again.
- Vectors stored in NeorunBase must be **normalised**, as the daemon normalises
  its queries. Otherwise L2 distance measures magnitude rather than similarity
  and the ranking is noise.

`tests/seed-image-embeddings.py` shows the whole path: fetch the images, embed
with the same model the worker runs, write pgvector literals to NeorunBase.

## How results are shown

Retrieval hits are not table rows and are not rendered as a grid. Each hit shows
its rank, a score bar scaled to the best hit, the text worth reading, and **the
image**, because a cell reading `s3://bucket/img/8891.jpg` communicates nothing.
The image searched with is shown above the results.

Ontul's second-stage re-ranking is reported honestly: *Re-ranked* when its model
ran, *Not re-ranked* with the reason when it did not. Ordering is most of the
answer for a relevance question, and serving first-stage order as if it were
re-ranked overstates what the user is looking at.

## REST API

| Route | Purpose |
|---|---|
| `POST /api/chat` with a relevance-shaped question | The agent picks and calls a retriever |
| `POST /api/search/image` `{imageBase64, connectionId?, retriever?}` | Embed an image and search by it |
| `GET /api/semantic/retrievers` | What is available on a connection |

## Related

- [Grounded Answers](grounded-answers.md) — certification, and how an answer is attributed
- [Tool Integration](tool-integration.md) — the Ontul surfaces Mium reads
