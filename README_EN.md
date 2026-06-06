[🇺🇦 Читати українською](README_UK.md)

# XSignalsBot

**An intelligent, fully automated cryptocurrency signal analysis and trade execution platform.**

XSignalsBot is a microservices-based system that ingests real-time market data and social sentiment signals, evaluates them through a suite of quantitative trading strategies, and autonomously executes trades on a cryptocurrency exchange. The platform also delivers actionable notifications to users via a Telegram bot, complete with position management, risk controls, and an administrative panel.

---

## Core Features

- **Real-Time Market Screening** — Continuously monitors live order-book trades, candlestick data, liquidation events, and ticker anomalies across multiple cryptocurrency pairs via exchange WebSocket feeds.
- **Multi-Timeframe Strategy Engine** — Runs 12+ independent technical analysis strategies (Supertrend, Ichimoku, Bollinger Squeeze, VWAP, Donchian Breakout, and more) across several timeframes to generate high-confidence trading signals.
- **AI-Powered Signal Analysis** — Integrates an AI routing layer to enrich and contextualize generated signals before they reach the decision stage.
- **Automated Trade Execution** — Places, modifies, and closes orders on Bybit (V5 API) with configurable take-profit, stop-loss, trailing stops, and dynamic position sizing.
- **Risk Management Module** — Enforces per-trade and per-portfolio risk limits, calculates optimal position sizes, and manages concurrent deal exposure.
- **Telegram Bot Interface** — Provides subscribers with real-time signal alerts, portfolio status, manual trade controls, and an integrated payment flow for subscription management.
- **Social Signal Ingestion** — An agent service monitors curated Telegram channels for analyst forecasts and community sentiment, feeding that data into the signal pipeline.
- **Admin Dashboard** — A protected REST API with JWT-based authentication, role-based access, user management, signal history, and deal tracking.
- **Background Task Processing** — Scheduled and event-driven tasks (historical data aggregation, periodic deal reconciliation, signal re-evaluation) run on a distributed async worker pool.
- **CI/CD Pipeline** — Automated build and deployment through GitHub Actions with a self-hosted runner and Docker Compose.

---

## System Architecture

XSignalsBot follows an **event-driven microservices architecture** where every service communicates asynchronously through a central message broker. Data flows through a multi-stage pipeline: raw market data is first screened, then analyzed by strategies, then orchestrated by the backend, and finally executed by the trade bot.

```
┌──────────────────────────────────────────────────────────────────┐
│                        EXTERNAL SOURCES                          │
│          Bybit WebSocket/REST API  ·  Telegram Channels          │
└────────────┬──────────────────────────────────┬──────────────────┘
             │                                  │
    ┌────────▼────────┐                ┌────────▼────────┐
    │   XScreeners    │                │   Agent Bot     │
    │ (Market Scanner)│                │ (Social Signal  │
    │                 │                │   Collector)    │
    └────────┬────────┘                └────────┬────────┘
             │ data_stream                      │ signals /
             │                                  │ channel_status
    ┌────────▼────────┐                         │
    │  XStrategies    │                         │
    │ (Strategy Engine │──────────┐             │
    │  & Signal Gen)  │ signals / │             │
    └─────────────────┘ upsert_   │             │
                          sources │             │
             ┌────────────▼───────▼─────────────▼──┐
             │           Redis (Cache Layer)        │
             │         RabbitMQ (Message Broker)    │
             └────────────────┬────────────────────┘
                              │
              ┌───────────────▼────────────────┐
              │         Backend API            │
              │  (Core Orchestrator, Auth,     │
              │   Signal Processing, AI)       │
              └──┬──────────┬──────────┬───────┘
                 │          │          │
      get_signals│  crypto_ │  update_ │
        _{group} │  signal  │  chats   │
                 │          │          │
     ┌───────────▼───┐  ┌──▼──────────▼──┐
     │   Trade Bot   │  │  Telegram Bot  │
     │ (Execution &  │  │ (User Alerts & │
     │ Risk Mgmt)    │  │  Interface)    │
     └───────┬───────┘  └────────────────┘
             │
     ┌───────▼───────┐
     │ Bybit Exchange │
     │ (V5 Trading    │
     │  API)          │
     └────────────────┘
```

### Service Breakdown

| Service | Responsibility | Communicates With |
|---|---|---|
| **XScreeners** | Connects to exchange WebSocket and REST streams, detects market anomalies and volume spikes, caches snapshots in Redis | Publishes raw events → **XStrategies** |
| **XStrategies** | Consumes screener data, runs multi-timeframe technical analysis via 12+ strategies, produces rated trading signals | Subscribes from **XScreeners**; publishes signals → **Backend** |
| **Agent Bot** | Telethon-based service that monitors Telegram channels for social/analyst crypto signals and channel status | Publishes signals & channel data → **Backend** directly |
| **Backend API** | Central orchestrator — signal processing, AI enrichment, JWT auth, user management, order lifecycle coordination | Receives from **XStrategies** & **Agent Bot**; dispatches → **Trade Bot** & **Telegram** |
| **Trade Bot** | Receives processed orders, calculates position sizes, places and manages orders on Bybit (demo & live modes) | Receives from **Backend**; publishes execution results → **Backend** |
| **Telegram Bot** | aiogram-powered user interface — signal alerts, portfolio overview, subscription payments | Receives notifications from **Backend** |
| **TaskIQ Workers** | Distributed async workers for scheduled jobs (data sync, deal updates, periodic recalculations) | Runs within **Backend** context |
| **Nginx** | Reverse proxy routing external traffic to the backend API | Fronts **Backend API** |

---

## Tech Stack

| Category | Technology |
|---|---|
| **Language** | Python 3.10+ |
| **Web Framework** | FastAPI, Uvicorn |
| **ORM / Database** | SQLAlchemy 2.0 (async), PostgreSQL 17, Alembic (migrations) |
| **Message Broker** | RabbitMQ 4.x (via FastStream, aio-pika) |
| **Cache** | Redis 7.2 |
| **Task Queue** | Taskiq (with scheduler and worker processes) |
| **Telegram Bot** | aiogram 3.x |
| **Channel Monitor** | Telethon (Telegram MTProto client) |
| **Exchange API** | pybit (Bybit V5 REST & WebSocket) |
| **Technical Analysis** | pandas, pandas-ta, numpy |
| **AI Integration** | Google Generative AI, Groq |
| **Reverse Proxy** | Nginx (Alpine) |
| **Containerization** | Docker, Docker Compose |
| **CI/CD** | GitHub Actions (self-hosted runner) |
| **Monitoring** | Sentry SDK, RedisInsight, pgAdmin |

---

## Project Structure

```
XSignalsBot/
├── backend/          # Core FastAPI service (REST API, auth, orchestrators)
├── telegram/         # Telegram bot (aiogram user interface)
├── agent_bot/        # Social signal collector (Telethon channel monitor)
├── xscreaners/       # Real-time market data scanner (Bybit WebSocket)
├── xstrategies/      # Multi-timeframe strategy & signal engine
├── trade_bot/        # Trade execution & risk management (Bybit V5 API)
├── nginx/            # Reverse proxy configuration
├── alembic/          # Database migration scripts
├── .github/          # CI/CD pipeline (GitHub Actions)
└── docker-compose.yml
```

---

## Getting Started

> **Note:** This is a private portfolio project. The instructions below outline the general setup for reference purposes.

1. **Prerequisites** — Docker & Docker Compose installed on the host machine.
2. **Environment** — Each service requires its own `.env` file with credentials for the exchange API, Telegram bot tokens, database connection, and message broker.
3. **Launch** — The entire stack is started with a single Docker Compose command, which provisions all services, databases, and infrastructure components in the correct dependency order.
4. **Database Migrations** — Schema changes are managed via Alembic and applied automatically on service startup.
