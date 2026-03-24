# LedgerLens
> Ask questions across SEC filings. Get cited answers. Watch quality metrics live.
What it does
LedgerLens lets you ask plain-English questions across SEC 10-K and 10-Q filings and get answers grounded strictly in the source documents — with section-level citations, live latency metrics, and automatic quality scoring on every query.
Example questions you can ask:

"What risks does Apple cite related to its China operations?"
"Compare how Apple, Microsoft, and Nvidia describe their AI strategy"
"How has Tesla's gross margin trended across their last four quarters?"


Architecture
SEC EDGAR API
     │
     ▼
┌─────────────────────────────────────────────────────┐
│  Ingestion pipeline                                  │
│  Clone filings → chunk → embed → BM25 + ChromaDB    │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  Query pipeline                                      │
│  Hybrid retrieval (BM25 + vector) → RRF fusion       │
│  → Cohere rerank → GPT-4o cited answer              │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  Observability layer                                 │
│  Langfuse tracing · Ragas evals · SQLite metrics    │
│  CI quality gate (GitHub Actions)                   │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
              FastAPI + streaming UI

Key features
FeatureDetailHybrid retrievalBM25 keyword + vector search fused with Reciprocal Rank FusionCross-encoder rerankingCohere rerank-english-v3.0 selects top 5 from 25 candidatesCitation enforcementEvery claim linked to filing, year, and section headingLive observabilityp50/p95 latency, cost per query, Ragas quality scoresCI quality gateGitHub Actions fails PRs where faithfulness drops below 0.80Multi-ticker compareAsk the same question across up to 4 tickers in parallelHuman feedbackThumbs up/down logged to Langfuse alongside automated evalsLocal modeSwap to Ollama (Mistral/Llama 3.2) — zero API cost, zero data leaving your machine

Model benchmark
Retrieval pipeline is identical across all three runs. Only the generation model changes.
Tested on Apple 10-K 2024, question: "What are the main risks Apple cites related to competition?"
ModelFaithfulnessAnswer relevancyLatencyEst. cost/queryGPT-4o (cloud)0.940.914.2s$0.0031Mistral 7B (local)0.810.796.8s$0.00Llama 3.2 3B (local)0.740.713.1s$0.00Phi-3 Mini (local)0.700.682.4s$0.00

Faithfulness and answer relevancy scored by Ragas.
Local models run on Apple M2 Pro, 16 GB RAM via Ollama.

Takeaway: GPT-4o leads on quality. Mistral 7B gets within ~13 points at zero cost — a strong tradeoff for privacy-sensitive deployments.

Stack
LayerToolEmbeddingsOpenAI text-embedding-3-small / nomic-embed-text (local)Vector storeChromaDBKeyword searchrank-bm25RerankerCohere Rerank v3GenerationGPT-4o / Mistral / Llama 3.2 (via Ollama)TracingLangfuseEvalsRagasAPIFastAPI + SSE streamingCIGitHub Actions

Quickstart
Prerequisites

Python 3.11+
OpenAI API key — platform.openai.com
Cohere API key (free tier) — cohere.com
Langfuse keys (free tier) — cloud.langfuse.com

bashgit clone https://github.com/lizathulya/LedgerLens
cd ledgerlens

python -m venv venv && source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# fill in your API keys

# ingest Apple's last 2 annual reports (~3,600 chunks, ~$0.004)
python ingest.py --ticker AAPL

# start the server
uvicorn main:app --reload

# open http://localhost:8000
Run fully local (no API keys needed)
bashollama pull mistral
ollama pull nomic-embed-text

LEDGER_BACKEND=ollama python ingest.py --ticker AAPL
LEDGER_BACKEND=ollama uvicorn main:app --reload

Run the eval suite
bashpytest test_evals.py -v
PASSED  test_evals.py::TestQualityGate::test_avg_faithfulness       [ 0.92 >= 0.80 ✓ ]
PASSED  test_evals.py::TestQualityGate::test_avg_answer_relevancy    [ 0.88 >= 0.75 ✓ ]
PASSED  test_evals.py::TestQualityGate::test_avg_context_precision   [ 0.81 >= 0.60 ✓ ]
PASSED  test_evals.py::TestQualityGate::test_keyword_grounding

Project structure
ledgerlens/
├── main.py            FastAPI app — routes, streaming, compare, feedback
├── query.py           Hybrid retrieval → rerank → cited generation
├── observe.py         Langfuse tracing, Ragas evals, SQLite metrics
├── ingest.py          SEC EDGAR fetch, chunking, embedding, BM25 index
├── compare.py         Parallel multi-ticker comparison
├── ollama_backend.py  Local model swap + model benchmark utility
├── test_query.py      Unit tests — citations, grounding, latency
├── test_evals.py      Ragas quality gate (runs in CI)
└── static/
    └── index.html     Chat UI with compare mode and feedback buttons

How retrieval works
User question
    │
    ├──► BM25 keyword search  ──► top 20 chunks
    │
    └──► Vector search        ──► top 20 chunks
                │
                ▼
    Reciprocal Rank Fusion (RRF)
    Combines rankings without score normalisation
                │
                ▼
    Cohere cross-encoder reranker
    Reads question + chunk together → top 5
                │
                ▼
    GPT-4o / Mistral
    Cited answer — refuses if answer not in context
Why hybrid? Vector search finds semantically similar chunks. BM25 finds exact keyword matches. Neither alone is sufficient — a question about "EBITDA margin" needs keyword matching; a question about "management's outlook on AI" needs semantic understanding. RRF combines both without requiring score normalisation.

Observability
Every query is automatically traced with:

Langfuse — full request waterfall (retrieval → rerank → LLM) with timing
Ragas — faithfulness, answer relevancy, context precision scored per query
SQLite — local metrics store for the live dashboard sidebar
Human feedback — thumbs up/down synced to Langfuse traces

The CI gate (test_evals.py) runs 5 golden Q&A pairs on every PR and fails if average faithfulness drops below 0.80.

License
MIT
