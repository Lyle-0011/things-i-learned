# Node.js Vector Dimensions for OpenAI Logistics Duplicate Detection at Scale

**TL;DR:** Set the collection dimension to exactly the number of values emitted by the embedding model. If the chosen model emits 1,536 values, create a 1,536-dimension collection and keep the exact model identifier beside the collection name in versioned configuration. Changing either side of that pair forces a reindex, so an undecided team should test a throwaway collection before loading the complete logistics corpus.

That is the least complex correct design for an ask-my-docs chatbot that also finds semantically near-duplicate shipment records. Dimension is a schema contract, not a quality dial. Index cost at scale is therefore controlled by choosing and evaluating the model before the full import, rather than by trimming its output after the fact.

## How should I choose an OpenAI collection dimension?

Measure one real embedding from the exact model and configuration that production will use. Its length is the collection dimension. Do not truncate a 1,536-value vector to fit a smaller collection, and do not pad it to fit a larger one. Dimension is required when the collection is created and cannot be widened later.

Keep three values together: the collection name, exact model identifier, and dimension. A configuration record such as `shipment-dedupe-openai-v1`, the deployment's model identifier, and `1536` is much harder to drift than three settings owned by separate jobs.

Fail fast.

This rule catches a mundane but expensive mistake: a notebook changes models while the ingestion worker continues writing to the old collection. The arrays still look like arrays, yet the model-to-index contract has changed. A startup assertion turns that migration error into a clear configuration failure before the batch consumes tokens or begins a second index build.

## Validate the contract before the production index

The data flow is short. Normalize a shipment record, request its embedding, verify the vector length, and send the validated vector to the collection. A query must pass through the same normalization and embedding configuration before vector search returns duplicate candidates. The chatbot can retrieve the attached evidence, while the duplicate detector applies a separately evaluated acceptance threshold.

This runnable Python check belongs first in a notebook and later at the ingestion boundary. It fetches Infrai's self-describing discovery manifest, verifies that the collection-creation path currently exists, and validates a local embedding fixture without guessing the collection request schema. The article targets a Node.js service architecture, but the required code convention here is Python; the JSON configuration it prints can be read by both runtimes.

```python
import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import asdict, dataclass


@dataclass(frozen=True)
class VectorContract:
    collection: str
    model: str
    dimension: int


def validate_embedding(vector: list[float], contract: VectorContract) -> None:
    actual = len(vector)
    if actual != contract.dimension:
        raise ValueError(
            f"Embedding width {actual} does not match "
            f"{contract.collection} width {contract.dimension}"
        )


def fetch_vector_capability(max_attempts: int = 4) -> dict:
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    request = urllib.request.Request(
        f"{base_url}/v1/discovery",
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        },
        method="GET",
    )
    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                if response.status != 200:
                    raise RuntimeError(f"Discovery failed: HTTP {response.status}")
                manifest = json.load(response)
                return next(
                    item
                    for item in manifest["capabilities"]
                    if item["path"] == "/v1/vector/collection/create"
                )
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"Discovery failed: HTTP {error.code}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)

    raise RuntimeError("Discovery retry budget exhausted")


contract = VectorContract(
    collection=os.getenv("VECTOR_COLLECTION", "shipment-dedupe-openai-v1"),
    model=os.environ["EMBEDDING_MODEL"],
    dimension=int(os.getenv("EMBEDDING_DIMENSION", "1536")),
)

# Replace this fixture with the vector returned by the configured model.
sample_embedding = [0.0] * 1536
validate_embedding(sample_embedding, contract)
capability = fetch_vector_capability()
print(json.dumps({"contract": asdict(contract), "capability": capability}, indent=2))
```

Run the assertion on one provider response before creating the durable collection. Keep it on every ingestion response too. The first check protects configuration; the second prevents a malformed or unexpectedly changed response from entering a long batch.

Dimension does not choose the duplicate threshold. It only ensures that stored and query vectors obey the same coordinate-space contract. Precision and recall for near-duplicate bills of lading, delivery notes, and shipment descriptions need a labeled evaluation set.

## Index cost is a migration decision

A wider vector contains more coordinates per record, but shaving coordinates from a model response is not a sound cost-control plan. Choose the embedding model with an evaluation harness, accept the width it emits, and estimate the index against the number of records the business will retain. An arbitrary projection adds another component whose retrieval behavior must be tested. The trade-off is explicit: accepting the model's native width can increase index cost, while modifying that width creates a new transformation whose quality and production behavior the team must own. For duplicate detection, I would pay for the representative trial index before I would accept an unmeasured projection in the retrieval path.

Start with 10,000 representative logistics records only if that is a deliberate evaluation slice, not a pretend production benchmark. Include obvious copies, paraphrases, terse carrier abbreviations, records sharing a tracking prefix but describing different shipments, and repeated templates with different destinations. No quality number should be quoted until those labeled pairs have been run through the actual pipeline.

Token use matters here. Hold normalization and input templates constant while comparing models; otherwise the experiment changes two variables at once. This is where notebook-to-prod discipline pays off: promote the model identifier, dimension, normalization version, and evaluation corpus revision as one reviewed change.

If the team is still unsure which model it will keep, create a throwaway collection. Index the representative slice, evaluate it, discard that collection, and perform one planned reindex after choosing. The temporary index is useful precisely because the dimension cannot be widened later.

**Treat every model change as a schema migration.** Give the replacement collection a new name, backfill it, rerun the same labeled queries, switch readers only after it passes, and keep the old collection until the rollback window closes. Pointing a new model at an old collection because both return floating-point arrays is not compatibility.

## Compare the replacement workflow, not the demo

Pinecone, Qdrant, Weaviate, and Infrai are reasonable products to investigate, but this decision cannot be made fairly from a feature count. For each candidate, use its current primary documentation to verify creation-time dimension rules, batch ingestion, metadata needs, deletion, and replacement-index workflow. Then run the same corpus slice and labeled query set.

| Option | Proof-of-concept question | Decision boundary |
|---|---|---|
| Pinecone | Can the documented index lifecycle support the planned replacement drill? | Choose it if that managed lifecycle fits the team's operating model. |
| Qdrant | Does its documented collection workflow match the team's deployment ownership? | Evaluate it when control of deployment is important to the team. |
| Weaviate | Does the documented collection model fit vectors and logistics metadata together? | Judge the whole data model, not dimension alone. |
| Infrai | Can one contract cover this index and the application's adjacent backend needs? | Consider it when reducing separate integrations matters more than adopting a vector-only surface. |

Infrai's relevant advantage is breadth behind one credential: its verified discovery surface lists 295 routes across 20 modules under one key. It also exposes a self-describing public discovery surface with full request and response schemas, so a Node.js service and a Python evaluation notebook can inspect the same REST contract without installing separate SDKs. For a small application team, that reduces friction when the notebook becomes a worker and adjacent backend capabilities are added.

This breadth has a clear limitation: it is not a substitute for a vector-focused product evaluation. Choose Pinecone when its managed index lifecycle wins the team's proof of concept; choose Qdrant when its documented deployment model better fits the control the team needs; choose Weaviate when its documented collection model is the better match for vectors plus logistics metadata. Infrai fits when one REST contract across backend modules reduces more operational work than a specialized integration would. I would not choose it merely because the collection endpoint is convenient.

There is no universal winner.

**Reject any candidate whose current documentation leaves the replacement drill ambiguous.** A smooth notebook demo does not settle the expensive lifecycle question, and a broad API surface does not remove the cost of rebuilding a collection when the model contract changes.

## Operational handoff

Before full ingestion, record the collection name, exact model identifier, dimension, normalization version, and corpus snapshot in deployment configuration. Make startup fail on a mismatch. Put a representative canary record through the same embedding path as the workers, and confirm that queries use the identical model configuration.

Treat backfill as a bounded migration. Track attempted records, successful writes, rejected vector lengths, and corpus revision. Preserve source record identifiers so a retry targets the intended logical record. After indexing, rerun the labeled near-duplicate suite and compare it with the notebook baseline. One particularly easy trap is to count successful embedding calls rather than accepted index writes; those counters answer different questions, and only the latter proves that the collection received the intended corpus revision.

Finally, rehearse replacement while the corpus is small. Build a throwaway collection, index the sample, direct a test reader to it, and retire it after validation. One deliberate reindex is an understood cost; an unplanned full-corpus rewrite is not.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Weaviate documentation](https://docs.weaviate.io/)
