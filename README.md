## Crypto RSI Streaming - Quickstart & Runbook

This repository is a complete, local, real‑time analytics stack that ingests historical trades from `trades_data.csv`, computes 14‑period RSI per token in Rust, and streams results to a Next.js dashboard.

### Stack
- Redpanda (Kafka‑compatible) broker + Console UI
- Ingestion service: Node.js + `kafkajs` reading `trades_data.csv`
- RSI service: Rust + `rdkafka` computing RSI per token
- Web dashboard: Next.js + TypeScript + Recharts consuming RSI stream

---

## Prerequisites
- Docker Desktop
- Node.js 18+
- Rust toolchain (only if building locally; Docker image is provided)

---

## 1) Start Infrastructure

```
docker-compose up -d
```

Services:
- Redpanda: `localhost:9092` (internal), `localhost:29092` (outside)
- Redpanda Console: `http://localhost:18080`

Create topics (idempotent):

```
powershell -ExecutionPolicy Bypass -File .\scripts\create_topics.ps1
```

---

## 2) Ingest Historical Trades

From repository root:

```
cd ingest
npm install
npm start
```

Environment (defaults shown):
- `KAFKA_BROKERS=localhost:29092`
- `TOPIC=trade-data`
- `CSV_PATH=../trades_data.csv`

The ingestor publishes one JSON message per CSV row with key = `token_address` so multiple tokens are processed concurrently.

---

## 3) Start RSI Microservice (Rust)

The docker‑compose already builds and runs it. To run locally instead:

```
cd rust-rsi
KAFKA_BROKERS=localhost:29092 \
TRADE_TOPIC=trade-data \
RSI_TOPIC=rsi-data \
GROUP_ID=rust-rsi \
RSI_PERIOD=14 \
cargo run --release
```

This service keeps per‑token price buffers and publishes RSI points to `rsi-data`.

---

## 4) Run the Web Dashboard (Next.js)

From repository root:

```
cd web
npm install
npm run dev
```

Open `http://localhost:3000`.

API endpoints:
- `GET /api/tokens` – parses CSV to list unique token addresses
- `GET /api/history?token=...&limit=400` – reads recent RSI from Kafka (from beginning)
- `GET /api/stream?token=...` – Server‑Sent Events (SSE) streaming live RSI (filters by `token` if provided)

---

## Operational Notes
- If port 3000 is in use, stop existing Next.js instances or run `npm run dev -- -p 3001`.
- If PowerShell blocks npm scripts, run: `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`.
- Redpanda Console at `http://localhost:18080` can be used to inspect topics `trade-data` and `rsi-data`.

---

## End‑to‑End Run (Clean Start)
1. `docker-compose up -d`
2. `powershell -ExecutionPolicy Bypass -File .\scripts\create_topics.ps1`
3. `cd ingest && npm install && npm start` (wait for "Published N records")
4. Ensure `rust-rsi` container is up (docker-compose) or start locally
5. `cd web && npm install && npm run dev`, then open `http://localhost:3000`

You should see a token selector with 5 tokens and real‑time charts for Price and RSI.

---

## Troubleshooting
- Next.js already running: kill the process listening on 3000 or change port.
- No RSI data: verify ingestor published to `trade-data` and `rust-rsi` is running.
- Topic not found: re‑run the topic creation script.

