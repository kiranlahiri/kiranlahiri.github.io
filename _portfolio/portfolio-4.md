---
layout: single
title: "Arbiter"
collection: portfolio
date: 2026-04-22
author_profile: false
excerpt: "Arbiter is a distributed streaming system for monitoring cross-exchange arbitrage signals in near real time."
external_url: https://github.com/kiranlahiri/arbiter
---

<p>
  <a class="btn" href="{{ page.external_url }}" target="_blank" rel="noopener">GitHub: kiranlahiri/arbiter</a>
</p>

<iframe width="560" height="315"
  src="https://www.youtube.com/embed/xwwSDpZmQ9g"
  title="Arbiter demo" frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen>
</iframe>

Arbiter is a distributed streaming system for monitoring cross-exchange arbitrage signals in near real time. The project ingests live BTC-USD market data from Coinbase and Kraken, publishes exchange-specific events into Kafka using Protobuf, normalizes them into a shared schema, detects spread opportunities across venues, persists emitted signals to Postgres, and serves them through a live API and browser dashboard.

Built in Go with Kafka, Docker Compose, Railway Postgres, and public exchange WebSocket feeds, Arbiter is designed as an event-driven backend system that emphasizes service boundaries, message contracts, and timing-aware signal generation rather than direct trade execution.

---

## Arbiter: Building A Distributed Market-Data Arbitrage Pipeline

### What It Does

Arbiter watches live BTC-USD quotes from multiple exchanges and looks for moments where the best bid on one venue exceeds the best ask on another.

The pipeline currently does six things:

1. **Live ingestion** — separate ingestor services connect to the Coinbase and Kraken public WebSocket feeds, subscribe to ticker updates, and publish raw market-data events into Kafka. Each raw message includes exchange metadata, sequence IDs, raw payload bytes, and the timestamp when the ingestor received the tick.

2. **Normalization** — normalizer services consume raw exchange events and map them into a shared `NormalizedTick` schema so downstream services do not need exchange-specific parsing logic. This stage also canonicalizes symbols, for example converting Kraken's `BTC/USD` into `BTC-USD`.

3. **Detection** — a detector service consumes normalized ticks, maintains the latest quote per symbol per exchange in memory, filters out stale quotes, measures quote gap and freshness, and computes simple cross-exchange spreads of the form:

   ```
   sell bid - buy ask
   ```

   If a spread passes the configured freshness and threshold checks, the detector emits a `Signal` message into Kafka.

4. **Persistence** — a signal-writer service consumes those signal messages and stores them in Postgres. This turns the pipeline from a transient stream into a durable backend system that can serve history, not just terminal output.

5. **API and WebSocket delivery** — a Go API service reads persisted signals from Postgres and exposes:
   - `GET /health`
   - `GET /signals`
   - `WS /ws/signals`

   The WebSocket layer pushes new signals live as they are persisted.

6. **Dashboard** — the browser dashboard loads recent signal history over REST, receives live updates over WebSocket, and presents the signal stream as a monitoring surface rather than a raw terminal feed.

The result is a working event-driven platform that ingests live market data, transforms it through multiple services, persists its outputs, and exposes them through a public-facing interface.

![Arbiter dashboard showing live signal feed](/images/Screenshot from 2026-04-27 13-46-00.png)

### Tech Stack

| Layer | Technology |
|---|---|
| Language | Go |
| Messaging | Kafka |
| Serialization | Protobuf |
| Exchange feeds | Coinbase WebSocket API, Kraken WebSocket API |
| Kafka client | segmentio/kafka-go |
| WebSocket client/server | gorilla/websocket |
| Persistence | PostgreSQL |
| Local orchestration | Docker Compose |
| Deployment | Docker Compose on a cloud VM |
| Managed database | Railway Postgres |

### Architecture

The deployed architecture is:

```
exchange feeds -> ingestors -> Kafka -> normalizers -> detector -> signals topic -> signal-writer -> Postgres -> REST/WebSocket API -> dashboard
```

This shape matters because each stage has a clear responsibility and can be reasoned about independently.

**Ingestors**

There is one ingestor per exchange. That separation matters because exchange connections fail independently. If Coinbase disconnects, Kraken can continue publishing.

Each ingestor:

- opens a persistent WebSocket connection
- subscribes to a ticker channel
- parses exchange-specific JSON messages
- wraps them in a `RawTick` protobuf message
- publishes the event into Kafka

This stage is intentionally thin. Ingestors understand exchange-specific wire formats, but they do not make downstream assumptions about analysis or arbitrage logic.

**Normalizers**

The normalizers are separate Kafka consumer/producer stages. They read raw exchange messages, convert them into a shared `NormalizedTick` schema, and publish into a common normalized topic.

This ended up being a valuable boundary. It keeps exchange-specific parsing isolated at the edges while allowing every downstream service to work from the same contract.

**Detector**

The detector is the core analytical service.

For each symbol, it keeps the latest quote from each exchange in memory, including:

- exchange
- bid
- ask
- ingestor receive timestamp
- sequence ID
- source metadata

On every new normalized tick, it recomputes eligible exchange pairs and emits a signal if a spread is found within the configured freshness window.

This stage is where most of the interesting real-time behavior shows up: stale-quote filtering, quote freshness windows, quote-gap checks, repeated-signal throttling, and event timing vs. wall-clock timing.

**Persistence and Delivery**

One of the biggest improvements to the project was moving beyond a terminal-only consumer.

Signals are now written into Postgres by a dedicated writer service, which gives the system durable storage, queryable history, a stable API backend, and a foundation for a live dashboard.

The API layer then serves both recent history over REST and new persisted signals over WebSocket. That turned Arbiter from a pipeline demo into a real product surface.

### Message Contracts

One of the main goals of the project was to make service boundaries explicit through schemas rather than ad hoc JSON passing.

The pipeline uses three core protobuf messages:

**`RawTick`**
The exchange-facing event. Includes raw payload bytes, optional price fields, and shared metadata such as exchange, symbol, sequence ID, exchange timestamp, and ingestor receive timestamp.

**`NormalizedTick`**
The common downstream event used by the detector. Includes canonical symbol, bid, ask, exchange attribution, raw-topic traceability, and normalized timestamps.

**`Signal`**
The detector output. Includes symbol, buy exchange, sell exchange, spread, fee-adjusted profit, opportunity timestamp, quote freshness metadata, and supporting trace information.

Using Protobuf instead of JSON kept payloads smaller and gave the system a more disciplined contract-first structure.

### The Most Interesting Engineering Problems

The hardest part of the project was not parsing WebSocket JSON or wiring up Kafka. It was getting a live multi-service system to behave like a truly live system.

Early on, the detector often appeared inactive even when the code path was correct. The real issues were operational:

- reused Kafka topics still contained old records from prior runs
- startup churn made validation noisy
- normalized messages preserved original ingestor timestamps
- the detector correctly rejected stale quotes, which made it look broken

To debug that, I added visibility around quote freshness, active exchange count, and signal suppression reasons. I also introduced isolated `.live` topics for clean validation runs so the detector could operate only on fresh data rather than historical backlog.

Later, during production deployment, I hit a more subtle bug:

- the signal writer deduped inserts by `(kafka_topic, kafka_partition, kafka_offset)`
- a fresh Kafka cluster restarted offsets from low values
- Postgres treated those reused offsets as duplicates from an earlier deployment using the same topic names
- the writer appeared healthy but silently inserted nothing because of `ON CONFLICT DO NOTHING`

The fix was to deploy production with distinct `.prod` topic names and to recognize that Kafka offsets are only unique within a single cluster, not globally across redeployments.

That bug ended up being one of the strongest lessons from the project: distributed systems often fail at the seams between individually reasonable components.

### Current Status

Arbiter currently supports:

- live Coinbase ingestion
- live Kraken ingestion
- raw and normalized Kafka flow
- real-time detector logic with quote freshness filtering
- durable signal persistence in Postgres
- REST signal API
- WebSocket live stream
- browser dashboard
- public deployment on a cloud VM with managed Postgres

### Why I Built It

I wanted to build something that combined streaming systems, concurrency, message contracts, timing-sensitive event processing, and real deployment concerns in one coherent project.

Market data was a good fit because it forces attention on event ordering, freshness, schema design, backpressure and backlog, long-running service behavior, and the differences between "working code" and "live correctness."

The result is not a profitable arbitrage bot, and it is not meant to be. It is a distributed event-driven system for monitoring and serving live cross-exchange arbitrage-style signals, and that made it a strong vehicle for the kinds of backend and infrastructure problems I wanted to explore.
