# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

Early stage. The repo currently holds only `README.md` and `requirements.txt`. The README's Quickstart describes a `docker compose` setup (Redpanda, a `topics-init` service, Redpanda Console on :8080), but no compose file has been committed yet. Check what actually exists before relying on the README's commands. Per the roadmap, the WebSocket ingestor is the next piece to build.

## Commands

```bash
python3.12 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

docker compose up -d              # Redpanda + topic creation + console (once compose file exists)
docker compose logs topics-init   # verify topics were created

pytest                            # run tests
pytest path/to/test_file.py::test_name   # single test
```

Kafka address: `localhost:19092` from host code, `redpanda:9092` from services inside Compose.

## Architecture

Real-time pipeline that rebuilds Coinbase Level 2 order books from the public WebSocket feed. Python 3.12 + asyncio throughout. Redpanda (Kafka API, via `aiokafka`) is the central bus: each stage reads from and writes back to topics, not to each other.

```
Coinbase WS (level2, market_trades, heartbeats)
  → Ingestor → coinbase.l2.raw / coinbase.trades.raw
      → Archive writer → MinIO (Parquet)  → Replay tool
      → Book engine → book.features
          → Anomaly detector → anomalies
      → ClickHouse → Grafana
```

| Topic | Partitions | Retention | Contents |
|---|---|---|---|
| `coinbase.l2.raw` | 6 | 7 days | Raw Level 2 messages |
| `coinbase.trades.raw` | 6 | 7 days | Raw trade messages |
| `book.features` | 6 | 3 days | Derived book features |
| `anomalies` | 1 | default | Scored alerts |

Key design invariants:
- **Messages are keyed by `product_id`.** All events for a product land in one partition, which keeps them in order for the book engine.
- **Raw events are archived verbatim.** The Parquet archive must be enough to rebuild any book exactly as it stood at any past moment. Replay is meant to be deterministic and covered by golden tests, so don't transform raw messages before archiving them.
- **The book engine applies a snapshot, then updates, one in-memory book per product.** It must detect sequence gaps and resync. Books are also validated against Coinbase's REST product book.
- **Features:** spread, mid, depth, imbalance, microprice.
- **Anomalies:** liquidity withdrawals, price jumps, spoofing-like behavior, and data-quality issues (crossed books, sequence gaps, stale feeds).
- **Supporting stack:** Prometheus for pipeline metrics (lag, throughput, latency), Dagster for jobs, dbt (`dbt-clickhouse`) for models, GitHub Actions for CI.
