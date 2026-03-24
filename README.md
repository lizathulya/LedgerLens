# LedgerLens

Ask plain-English questions across SEC filings. Get answers cited to the exact section they came from. Watch quality metrics update in real time.

![Eval gate](https://github.com/YOUR_USERNAME/ledgerlens/actions/workflows/eval.yml/badge.svg)
![Python](https://img.shields.io/badge/python-3.11-blue)
![License](https://img.shields.io/badge/license-MIT-green)

![LedgerLens demo](docs/demo.gif)

---

## The problem it solves

SEC filings are dense. A single Apple 10-K runs to 80,000+ words. Finding what management actually said about a specific risk, segment, or strategic bet means reading for hours — or trusting a summary that may have missed something.

LedgerLens lets you ask the question directly and get an answer grounded in the exact filing text, with citations you can verify.

---

## What you can do with it

**Single company Q&A**
Ask anything about a company's filings and get a cited answer:
> *"What risks does Apple cite related to its China operations?"*
> → Answer with `[AAPL 10-K 2024 — Item 1A: Risk Factors]` citation

**Side-by-side comparison**
Ask the same question across up to four tickers at once:
> *"How does each company describe its AI strategy?"*
> → AAPL, MSFT, and NVDA answers rendered side by side

**Run it fully offline**
Swap GPT-4o for a local Mistral or Llama model via Ollama. No data leaves your machine.

---

## How retrieval works

Most RAG systems use vector search alone — which misses exact keyword matches. LedgerLens runs two searches in parallel and combines them.

```
Your question
    │
    ├── BM25 keyword search ──► top 20 chunks   (catches exact terms like "EBITDA")
    └── Vector search       ──► top 20 chunks   (catches semantic matches like "outlook")
                │
                ▼
        Reciprocal Rank Fusion
        Merges both result sets by rank position,
        not raw scores (which aren't comparable)
                │
                ▼
        Cohere cross-encoder reranker
        Re-reads your question + each chunk together
        Keeps the 5 most relevant
                │
                ▼
        GPT-4o
        Answers using only the retrieved chunks
        Refuses if the answer isn't in the context
```

The result is meaningfully better than vector search alone — especially for financial text, where specific numbers, ratios, and regulatory terms matter.

---

## Observability

Every query is automatically measured and scored. The live sidebar in the UI shows:

- **p50 / p95 latency** — broken down by retrieval, reranking, and LLM
- **Cost per query** — estimated from token usage
- **Faithfulness** — does the answer stick to what the chunks actually say?
- **Answer relevancy** — does the answer address the question that was asked?
- **Status indicator** — green / amber / red based on rolling quality average

All traces are sent to Langfuse, where you can inspect the full request waterfall for any query.

---

## CI quality gate

A GitHub Actions workflow runs on every push. It asks 5 golden questions, scores the answers with Ragas, and fails the build if quality drops below threshold.

```
PASSED  test_avg_faithfulness      [ 0.92 >= 0.80 ✓ ]
PASSED  test_avg_answer_relevancy  [ 0.88 >= 0.75 ✓ ]
PASSED  test_avg_context_precision [ 0.81 >= 0.60 ✓ ]
PASSED  test_keyword_grounding
```

The badge at the top of this README reflects the current state of that check.

---

## Model benchmark

Same retrieval pipeline, same question, different generation model.
Question: *"What are the main risks Apple cites related to competition?"*
Hardware: Apple M2 Pro, 16 GB RAM.

| Model | Faithfulness | Answer relevancy | Latency | Cost/query |
|---|---|---|---|---|
| GPT-4o | 0.94 | 0.91 | 4.2s | $0.0031 |
| Mistral 7B (local) | 0.81 | 0.79 | 6.8s | $0.00 |
| Llama 3.2 3B (local) | 0.74 | 0.71 | 3.1s | $0.00 |
| Phi-3 Mini (local) | 0.70 | 0.68 | 2.4s | $0.00 |

Mistral closes most of the quality gap at zero cost — a meaningful tradeoff for privacy-sensitive deployments where data can't leave the machine.

---

## Quickstart

**Prerequisites:** Python 3.11+, and free-tier accounts at [OpenAI](https://platform.openai.com), [Cohere](https://cohere.com), and [Langfuse](https://cloud.langfuse.com).

```bash
git clone https://github.com/lizathulya/LedgerLens
cd ledgerlens
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env        # add your API keys
```

Ingest a company's filings (one-time, ~$0.004 for Apple):

```bash
python ingest.py --ticker AAPL
```

Start the server:

```bash
uvicorn main:app --reload
# open http://localhost:8000
```

**Run fully local — no API keys needed:**

```bash
ollama pull mistral && ollama pull nomic-embed-text
LEDGER_BACKEND=ollama python ingest.py --ticker AAPL
LEDGER_BACKEND=ollama uvicorn main:app --reload
```

**Run the eval suite:**

```bash
pytest test_evals.py -v
```

---

## Stack

| | |
|---|---|
| Vector store | ChromaDB |
| Keyword search | rank-bm25 |
| Embeddings | OpenAI `text-embedding-3-small` or `nomic-embed-text` (local) |
| Reranker | Cohere Rerank v3 |
| Generation | GPT-4o or Mistral / Llama 3.2 via Ollama |
| Tracing | Langfuse |
| Evals | Ragas |
| API | FastAPI with SSE streaming |
| CI | GitHub Actions |

---

## Project structure

```
ledgerlens/
├── ingest.py          Fetch SEC filings, chunk, embed, build BM25 index
├── query.py           Hybrid retrieval → rerank → cited answer
├── observe.py         Langfuse tracing, Ragas evals, SQLite metrics store
├── compare.py         Run the same question across multiple tickers in parallel
├── ollama_backend.py  Local model backend + benchmark utility
├── main.py            FastAPI routes, streaming, feedback endpoints
├── test_query.py      Unit tests — citations, grounding, latency budget
├── test_evals.py      Ragas quality gate — runs in CI
└── static/
    └── index.html     Chat UI, compare mode,
