---
layout: single
title: "FinCanon"
collection: portfolio
date: 2025-10-27
author_profile: false
excerpt: "FinCanon is a portfolio analysis tool that combines quantitative metrics with RAG-powered insights from classic finance literature."
external_url: https://fincanon.vercel.app/
# Optional thumbnail on the /portfolio/ list:

---

<p><a class="btn btn--primary" href="{{ page.external_url }}" target="_blank" rel="noopener">
  https://fincanon.vercel.app/
</a></p>


FinCanon is a full-stack web application that analyzes investment portfolios using advanced financial metrics including Sharpe ratio, maximum drawdown, rolling performance, and efficient frontier detection. The tool integrates a Retrieval-Augmented Generation (RAG) system powered by 10 seminal finance papers (Markowitz, Sharpe, Fama-French, Black-Scholes, etc.) to provide portfolio-aware insights grounded in academic research. Built with React, FastAPI, LangChain, and Qdrant vector database, the application is deployed on Vercel and Railway with real-time data visualization and personalized investment analysis.

---

## FinCanon: Building a RAG-Powered Portfolio Analysis Tool

### What It Does

Upload a CSV of daily asset returns with portfolio weights. FinCanon does two things:

**Quantitative analysis** — instantly computes a full suite of portfolio metrics: Sharpe ratio, Sortino ratio, maximum drawdown, annualized return and volatility, per-asset return contributions, correlation matrix, diversification ratio, and efficient frontier detection. It also runs a portfolio optimization to find the minimum-variance and maximum-Sharpe portfolios for your specific asset universe, and traces out the full efficient frontier across 20 target-return points.

**RAG-powered Q&A** — ask any question about your portfolio or finance concepts in general. The system retrieves relevant passages from 10 embedded seminal finance papers and generates a cited, portfolio-aware answer using GPT-4o-mini. Ask "how does my Sharpe ratio compare to theory?" and get an answer that references Sharpe's 1964 paper and relates it to your specific numbers.

### Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React, Recharts |
| Backend | FastAPI, Python |
| Data processing | pandas, NumPy |
| Optimization | SciPy (SLSQP) |
| RAG orchestration | LangChain |
| Embeddings | OpenAI text-embedding-3-large |
| LLM | OpenAI GPT-4o-mini |
| Vector database | Qdrant Cloud |
| Deployment | Vercel (frontend), Railway (backend) |

### The Quantitative Engine

The backend is built with FastAPI. When a user uploads a CSV, pandas parses it into a DataFrame indexed by date with asset tickers as columns. From there, NumPy handles the vectorized math.

Portfolio return and variance are computed using the standard matrix formulation:

- **Portfolio return:** w^T · μ (dot product of weights and mean returns)
- **Portfolio variance:** w^T · Σ · w (quadratic form of the covariance matrix)

All metrics are annualized assuming 252 trading days — returns multiply by 252, volatilities multiply by √252.

The efficient frontier is computed using SciPy's `minimize` with the SLSQP (Sequential Least Squares Programming) method. Three separate optimizations run:

- **Minimum variance** — minimize portfolio volatility subject to weights summing to 1 and long-only constraints
- **Maximum Sharpe** — minimize the negative Sharpe ratio with the same constraints
- **Efficient frontier** — loop over 20 linearly-spaced target returns and solve a minimum-variance problem with an additional equality constraint fixing portfolio return at each target, tracing the full frontier curve

The frontend renders these using Recharts' `ScatterChart` — the frontier as a connected scatter line, with the current portfolio, min-variance, and max-Sharpe points overlaid as distinct shapes.

### The RAG Pipeline

This is the most technically interesting part of the project. The goal: when a user asks a question, retrieve the most relevant passages from the finance papers and use them to ground the LLM's answer.

**Ingestion (offline, runs once)**

- **Load** — `UnstructuredPDFLoader` reads each PDF element-by-element, preserving page numbers as metadata
- **Chunk** — `RecursiveCharacterTextSplitter` splits text into 2,000-character chunks with 250-character overlap, using a `["\n\n", "\n", " ", ""]` separator hierarchy to prefer natural paragraph breaks
- **Embed** — `OpenAIEmbeddings` with `text-embedding-3-large` converts each chunk into a 3,072-dimensional vector
- **Store** — chunks are written to a Qdrant Cloud collection with their vectors and metadata (paper title, page number)

The 10 papers span the canon of quantitative finance: Markowitz (1952), Sharpe (1964), Fama (1970), Black-Scholes (1973), Merton (1973), Ross (1976), Kahneman & Tversky (1979), Engle (1982), Fama-French (1993), and Jegadeesh & Titman (1993).

**Query (live, on every request)**

When a user submits a question, the pipeline runs:

1. **Query expansion** — the query is expanded into up to 3 variations using a static term-replacement map. This addresses a real vocabulary mismatch problem: users say "efficient frontier," but Markowitz's 1952 paper says "efficient set" and "E-V combinations." Without expansion, the right chunks would never be retrieved.

2. **MMR retrieval** — each query variation is embedded and searched against Qdrant using Maximal Marginal Relevance (MMR) with `fetch_k=40`, returning the top 10 chunks per variation. MMR optimizes for both relevance and diversity — it penalizes redundant chunks so you don't get the same concept repeated 10 times from adjacent paragraphs.

3. **Deduplication** — results across the 3 query variations are deduplicated by hashing the first 200 characters of each chunk's content, returning up to 15 unique documents.

4. **Prompt assembly** — a LangChain `PromptTemplate` injects the retrieved chunks alongside the user's actual portfolio metrics (Sharpe, drawdown, asset weights, quarterly trends, efficient frontier position) as structured context.

5. **Generation** — GPT-4o-mini generates a cited answer at `temperature=0` for determinism.
