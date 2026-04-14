# LedgerLens

Ask plain-English questions across SEC filings and get answers grounded in real financial data with verifiable citations and live quality metrics.

Analyze companies like Apple, Microsoft, or Nvidia in seconds instead of reading 100-page filings.

🔗 Live Demo: [add link]
🎥 Demo: [add GIF]

---

## The problem

SEC filings are dense and time-consuming. A single 10-K can exceed 80,000 words. Extracting insights about risks, strategy, or revenue requires hours of manual reading — or relying on summaries that may miss critical details.

LedgerLens turns these filings into a **queryable system**, returning answers grounded in the original text with precise citations.

---

## What you can do

* Ask questions about a company’s filings
  → *“What risks does Apple cite in China?”*

* Compare companies side-by-side
  → *“How do AAPL vs NVDA approach AI?”*

* Run fully local
  → No data leaves your machine (via Ollama)

---

## Why this matters

Financial decisions depend on unstructured, high-stakes data.
LedgerLens transforms static filings into an interactive system enabling faster, transparent, and verifiable analysis.

This project demonstrates production-grade AI engineering with:

* hybrid retrieval systems
* evaluation pipelines
* real-time observability

---

## Key Features

* Hybrid retrieval (BM25 + vector + reranking)
* Section-level citations (grounded answers, not hallucinations)
* Built-in evaluation (Ragas + CI gate)
* Observability dashboard (latency, cost, quality)
* Multi-company comparison
* Optional fully local inference (Ollama)

---

## How it works

```
User question
    │
    ├── BM25 keyword search ──► top chunks
    └── Vector search       ──► top chunks
                │
                ▼
        Reciprocal Rank Fusion
                │
                ▼
        Cross-encoder reranker
                │
                ▼
        LLM (GPT-4o / local model)
                │
                ▼
        Answer with citations
```

Hybrid retrieval improves accuracy especially for financial text where exact terms and semantics both matter.

---

## Observability

Each query is tracked and evaluated in real time:

* Latency (p50 / p95)
* Cost per query
* Faithfulness (grounding to source)
* Answer relevancy

All traces are logged with Langfuse for debugging and inspection.

---

## CI Quality Gate

A GitHub Actions workflow runs evaluation tests on every push.

Example:

```
PASSED  faithfulness ≥ 0.80
PASSED  answer relevancy ≥ 0.75
PASSED  context precision ≥ 0.60
```

This ensures model quality doesn’t degrade over time.

---

## Model Benchmark

| Model           | Faithfulness | Relevancy | Latency | Cost   |
| --------------- | ------------ | --------- | ------- | ------ |
| GPT-4o          | 0.94         | 0.91      | 4.2s    | $0.003 |
| Mistral (local) | 0.81         | 0.79      | 6.8s    | $0.00  |
| Llama 3.2       | 0.74         | 0.71      | 3.1s    | $0.00  |

Local models trade some quality for zero cost and full privacy.

---

## Quickstart

### Setup

```bash
git clone https://github.com/lizathulya/LedgerLens
cd ledgerlens
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

Add your API keys:

* OpenAI
* Cohere
* Langfuse

---

### Ingest data

```bash
python ingest.py --ticker AAPL
```

(~$0.004 cost)

---

### Run the app

```bash
uvicorn main:app --reload
```

Open: http://localhost:8000

---

### Run locally (no API usage)

```bash
ollama pull mistral
ollama pull nomic-embed-text

LEDGER_BACKEND=ollama python ingest.py --ticker AAPL
LEDGER_BACKEND=ollama uvicorn main:app --reload
```

---

### Run evals

```bash
pytest test_evals.py -v
```

---

## Tech Stack

* Vector DB: ChromaDB
* Keyword search: BM25
* Embeddings: OpenAI / local
* Reranking: Cohere
* LLM: GPT-4o / Ollama
* Backend: FastAPI
* Observability: Langfuse
* Evaluation: Ragas
* CI: GitHub Actions

---

## Project Structure

```
ledgerlens/
├── ingest.py
├── query.py
├── observe.py
├── compare.py
├── ollama_backend.py
├── main.py
├── test_query.py
├── test_evals.py
└── static/
    └── index.html
```

---

## Future Work

* Agent-based query planning
* Multi-step financial reasoning
* Portfolio-level analysis
* User feedback-driven retraining

---

## License

MIT
