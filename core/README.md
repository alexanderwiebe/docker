# core stack

The foundational self-hosted services that power the local AI briefing pipeline.

## Services

### RSSHub (`port 1200`)
Converts non-RSS sources into RSS feeds. Here it's used to expose Twitter/X list
timelines as RSS, which the briefing script polls directly. Requires Twitter auth
credentials via environment variables (see `.env.example`).

Backed by Redis for response caching so repeated fetches don't hit rate limits.

### Redis (`internal`)
Cache layer for RSSHub. No external port — only reachable by other services in
the compose network. Data is persisted to `/mnt/docker/core/redis`.

### FreshRSS (`port 8080`)
Web-based RSS reader and aggregator. Optional — useful for browsing raw feeds
manually before they're processed by the briefing pipeline. Data and extensions
are persisted to `/mnt/docker/core/freshrss`.

## How it fits together

```
Twitter/X lists
      │
      ▼
   RSSHub :1200  ←──  Redis (cache)
      │
      ▼
 briefing.py  (~/ai-briefing)
      │
      ▼
 Claude CLI  →  Telegram
```

## Setup

1. Copy `.env.example` to `.env` and fill in your Twitter credentials.
2. `docker compose up -d`

## Environment variables

See `.env.example` for required variables. Never commit `.env`.
