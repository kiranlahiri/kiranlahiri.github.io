---
layout: single
title: "Arbiter"
collection: portfolio
date: 2026-04-22
author_profile: false
excerpt: "Arbiter is a distributed streaming market-data system for detecting cross-exchange arbitrage opportunities in near real time."
external_url: https://github.com/kiranlahiri/arbiter
---

<p><a class="btn btn--primary" href="{{ page.external_url }}" target="_blank" rel="noopener">
  GitHub: kiranlahiri/arbiter
</a></p>

Arbiter is a distributed streaming market-data system for detecting cross-exchange arbitrage opportunities in near real time. The project ingests live BTC-USD ticker data from Coinbase and Kraken, publishes exchange-specific events into Kafka using Protobuf, normalizes them into a common schema, and runs a detector service that compares quotes across venues to emit arbitrage signals. Built in Go with Kafka, Docker Compose, and WebSocket feeds from public exchanges, the system is designed as an event-driven pipeline that prioritizes service boundaries, message contracts, and low-latency data flow over direct trade execution.

---

## Arbiter: Building A Distributed Market-Data Arbitrage Pipeline

### What It Does

Arbiter watches live market data from multiple crypto exchanges and looks for moments where the best bid on one venue is higher than the best ask on another.

The pipeline currently does four things:

1. **Live ingestion** — separate ingestor services connect to the Coinbase and Kraken public WebSocket feeds, subscribe to ticker updates for BTC-USD, and publish raw market-data events into Kafka. Each raw message includes exchange metadata, sequence IDs, raw payload bytes, and the timestamp when the ingestor received the tick.

2. **Normalization** — a normalizer service consumes those raw events and maps them into a shared `NormalizedTick` schema so downstream services do not need exchange-specific parsing logic. This stage also canonicalizes symbol formats, for example mapping Kraken's `BTC/USD` into `BTC-USD`.

3. **Detection** — a detector service consumes normalized ticks, maintains the latest quote per symbol per exchange in memory, filters out stale quotes, and computes simple cross-exchange spreads of the form:

   ```
   sell bid - buy ask
   ```

   If the spread beats the configured fee and profit thresholds, the detector emits a `Signal` message into Kafka.

4. **Signal consumption** — a lightweight signal consumer subscribes to the signals topic and prints a clean real-time stream of detected opportunities, including symbol, buy exchange, sell exchange, spread, profit, and measured latency.

The result is a working event-driven system that ingests live market data, transforms it through multiple services, and produces arbitrage-like signals in real time.

### Tech Stack

| Layer | Technology |
|---|---|
| Language | Go |
| Messaging | Kafka |
| Serialization | Protobuf |
| Exchange feeds | Coinbase WebSocket API, Kraken WebSocket API |
| Kafka client | segmentio/kafka-go |
| WebSocket client | gorilla/websocket |
| Local orchestration | Docker Compose |
| Local Kafka tooling | Schema Registry, Kafdrop |

### The Architecture

The long-term architecture is:

```
exchange feeds -> ingestors -> Kafka -> normalizer -> detector -> signals -> WebSocket/REST API
```

The current implementation already follows that structure in a simplified form.

**Ingestors**

There is one ingestor per exchange. This matters because exchange connections fail independently. If Coinbase disconnects, Kraken can continue publishing.

Each ingestor:

- opens a persistent WebSocket connection
- subscribes to a ticker channel
- parses exchange-specific JSON messages
- wraps them in a `RawTick` protobuf message
- publishes them to Kafka

This stage is intentionally thin. The ingestors understand exchange-specific wire formats, but they do not make downstream assumptions about analysis or arbitrage logic.

**Normalizer**

The normalizer is a separate Kafka consumer/producer stage. It reads raw exchange messages, converts them into a shared `NormalizedTick` schema, and publishes into a common normalized topic.

This separation turned out to be a good design choice. It keeps exchange-specific parsing isolated at the edges while letting every downstream service work from the same contract.

**Detector**

The detector is the core analytical service.

For each symbol, it keeps the latest quote from each exchange in memory:

- exchange
- bid
- ask
- ingestor receive timestamp
- sequence ID
- source metadata

On every new normalized tick, it recomputes eligible exchange pairs and emits a signal if a profitable spread is found.

This is also where a lot of the most interesting distributed-systems behavior showed up: stale-quote filtering, quote freshness windows, repeated-signal throttling, and the effect of backlog/history on real-time comparison.

### Message Contracts

One of the main goals of the project was to make the service boundaries explicit through schemas rather than ad hoc JSON passing.

The pipeline currently uses three core protobuf messages:

**`RawTick`**
The exchange-facing event. Includes raw payload bytes, optional price fields, and shared metadata such as exchange, symbol, sequence ID, exchange timestamp, and ingestor receive timestamp.

**`NormalizedTick`**
The common downstream event used by the detector. Includes canonical symbol, bid, ask, exchange attribution, raw-topic traceability, and normalized timestamps.

**`Signal`**
The detector output. Includes symbol, buy exchange, sell exchange, spread, fee-adjusted profit, opportunity timestamp, and a latency field intended to represent end-to-end timing.

Using Protobuf instead of JSON kept payloads smaller and gave the system a more disciplined contract-first structure.

### The Most Interesting Engineering Problem

The hardest part of the project was not connecting Kafka or parsing WebSocket JSON. It was making a live multi-service pipeline behave like a truly live system.

Early testing was confusing because the detector often appeared to "work" while not publishing meaningful signals. The root cause turned out not to be detector math, but development workflow:

- reused Kafka topics still contained old records from prior runs
- short-lived Compose runs created startup churn
- normalized messages preserved their original ingestor timestamps
- the detector correctly rejected stale quotes, which made it look inactive

To debug this, I added detector-level visibility around quote freshness, active exchange count, and signal suppression reasons. I also added configurable Kafka start offsets and moved from one-off short-lived runs toward long-lived Compose services.

The most important fix was isolating live validation into clean `.live` topics:

- `raw.ticks.coinbase.live`
- `raw.ticks.kraken.live`
- `normalized.ticks.live`
- `arbitrage.signals.live`

That removed old topic history from the test path and let the pipeline process only fresh live data. Once the system was running against isolated live topics, the detector began emitting real signals between Coinbase and Kraken.

That debugging process ended up being the most valuable part of the project from a distributed-systems perspective: the code path was often fine, but correctness depended heavily on offsets, message history, freshness windows, and the behavior of long-lived services.

### Current Status

Arbiter currently supports:

- live Coinbase ingestion
- live Kraken ingestion
- raw-to-normalized Kafka flow
- cross-exchange detector logic
- emitted arbitrage-style signals
- a simple signal consumer for real-time inspection

Planned next steps include:

- smarter signal deduping and throttling
- more realistic fee modeling
- a WebSocket or REST signal API
- persisted signal storage
- health and metrics endpoints
- eventual deployment to Kubernetes

### Why I Built It

I wanted to build something that combined streaming systems, message contracts, concurrency, and low-latency event processing in a concrete way. Market data is a good fit for that because it forces attention on timing, ordering, schema design, and service isolation.

The result is not a production trading engine, and it is not meant to be. It is a distributed event-driven system for ingesting, transforming, and analyzing live market data in real time, and that makes it a strong vehicle for exploring the kinds of backend and infrastructure problems I'm most interested in.
