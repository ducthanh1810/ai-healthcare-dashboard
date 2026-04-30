# ⚡ Technical Decisions — Trade-off Analysis

> Every architecture choice is a trade-off. Here's why we chose what we chose, and what we considered.

---

## Decision 1: Multi-LLM vs Single-LLM

### Context

The AI agent system needs to generate hospital reports every morning at 7:15 AM. Missing a report is unacceptable — staff rely on it for daily planning.

### Options Considered

| Option | Pros | Cons |
|---|---|---|
| **Single LLM (OpenRouter only)** | Simple setup, one API key | Rate limits at peak hours, single point of failure |
| **Multi-LLM with manual switch** | Reliability | Operator intervention needed |
| **Multi-LLM with auto-fallback** ✅ | Transparent failover, zero downtime | More complex setup, need to handle model differences |

### Decision

**Multi-LLM with automatic fallback chain** (4 providers).

### Rationale

- Hospital operations are **24/7** — we can't wait for a provider to come back online
- Different providers have different pricing → route cheap tasks to cheap providers
- Local Ollama as last resort means we're never fully dependent on cloud

### Result

| Metric | Before | After |
|---|---|---|
| Failed morning reports | 3/week | **0/month** |
| Failover time | Manual (minutes) | **< 2 seconds** |
| Monthly cost | $80 | **$35** (smart routing) |

---

## Decision 2: Dual Database (MySQL + Oracle HIS)

### Context

Hospital data lives in **Oracle 11g HIS** — the hospital's core information system. We need to build dashboards on top of this data, but:
- Oracle HIS is a **shared critical system** — any disruption affects the entire hospital
- We have **read-only access** — no write permissions (nor should we want them)
- Oracle queries for financial reports are **slow** (5-15 seconds)

### Options Considered

| Option | Pros | Cons |
|---|---|---|
| **Oracle only** | Single data source, real-time | Slow queries, risk to HIS, no app state storage |
| **MySQL only (full sync)** | Fast queries, full control | Stale data, complex sync, storage duplication |
| **Dual DB (Oracle read + MySQL app)** ✅ | Clean separation, fast dashboard, real-time when needed | Two DB adapters, sync logic needed |

### Decision

**MySQL 8** for application state (users, configs, logs, schedules) + **Oracle 11g** read-only for HIS data.

### Sync Strategy

```
Oracle HIS ──[nightly sync]──> MySQL (billing cache)
                                    ↓
                              Redis (query cache)
                                    ↓
                              Dashboard (< 1s load)
```

- **Daily sync** (01:00): 1-month billing window → MySQL
- **Weekly sync** (Sunday 02:00): 3-month billing window → MySQL
- **Real-time**: Critical queries still go directly to Oracle
- **Cache warmup**: Every 30 minutes during business hours

### Result

- Dashboard loads **< 1 second** for 90% of reports
- Zero disruptions to Oracle HIS
- App state fully independent of hospital core system

---

## Decision 3: Multi-Agent (Router + Specialists) vs Single Agent

### Context

We have 40+ tools across finance, HR, clinical, booking, and analytics domains. One agent can't handle all of them effectively.

### Options Considered

| Option | Pros | Cons |
|---|---|---|
| **Single agent, all tools** | Simple, one entry point | Context overflow, wrong tools, slow |
| **Hardcoded routing (if/else)** | Fast, predictable | Brittle, can't handle ambiguous queries |
| **LLM Router + Specialist Agents** ✅ | Focused context, right-sized models, extensible | Router overhead, more complex orchestration |

### Decision

**10 specialized agents** with an LLM-powered Router and Reflection layer.

### Key Design Choices

1. **Router uses structured output** (Pydantic) — not free-text parsing
2. **Three-layer fallback** — LLM → text parse → keyword regex (never crashes)
3. **Confidence threshold** — below 0.4, ask for clarification instead of guessing
4. **Right-sized models** — simple tasks get cheap models, complex get expensive
5. **Focused toolkits** — each agent has only 2-20 relevant tools

### Result

| Metric | Single Agent | Multi-Agent |
|---|---|---|
| Routing accuracy | ~60% | **~92%** |
| Response time | 8-15s | **3-6s** |
| Monthly LLM cost | $80 | **$35** |
| Adding new domain | Modify monolith | Add new specialist agent |

---

## Decision 4: MCP Protocol for Tool Communication

### Context

40+ tools across 12 files, each with different calling conventions, error formats, and response structures. Adding new tools requires modifying agent code.

### Options Considered

| Option | Pros | Cons |
|---|---|---|
| **Direct function calls** | Simple, fast | Tight coupling, inconsistent interfaces |
| **REST API wrappers** | Standardized HTTP | Extra network hop, still manual registration |
| **MCP Protocol** ✅ | Standard schema, auto-discovery, tool server pattern | Newer protocol, ecosystem still growing |
| **Hybrid (REST + MCP)** ✅ | Best of both worlds | More complex setup |

### Decision

**Hybrid approach**: Local REST tools (high priority) + MCP tools (supplemental, auto-discovered).

### Rationale

- **Local tools** are battle-tested and fast — they call our own FastAPI backend
- **MCP tools** allow extensibility without code changes — register a tool on the server, agents discover it automatically
- **Priority system** — local tools always win on name collision
- **Future-proof** — as MCP ecosystem grows, we can leverage community tools

### Result

- New tools can be added by registering on MCP server — zero agent code changes
- Standard schema makes tools easy to test independently
- Mixed reliability: critical tools are local (fast, reliable), experimental tools are MCP

---

## Decision 5: Snapshot + Cache Strategy

### Context

Oracle HIS queries for department-level financial reports take **5-15 seconds**. Running these on every dashboard load is unacceptable for user experience and Oracle load.

### Options Considered

| Option | Pros | Cons |
|---|---|---|
| **Real-time Oracle queries** | Always fresh data | 5-15s load, high Oracle load |
| **Full MySQL mirror** | Fast queries | Data staleness, sync complexity, storage |
| **Daily snapshot + cache warmup** ✅ | Fast dashboard, predictable Oracle load | End-of-day data, not real-time |
| **Materialized views (Oracle)** | Oracle-native, fast | Requires Oracle admin access (we don't have) |

### Decision

**Daily financial snapshot** at 23:30 + **Redis cache warmup** every 30 minutes during business hours.

### How It Works

```
23:30 — Daily Snapshot Job
├── Query Oracle HIS for today's financial data
├── Aggregate by department, service type, insurance
├── Store in MySQL snapshot tables
└── Clear stale cache entries

Every 30 min (7:00-17:00) — Cache Warmup
├── Pre-compute common dashboard queries
├── Store results in Redis (TTL: 35 min)
└── Dashboard reads from cache first
```

### Trade-off

- ❌ Financial data is **T-0** (today's snapshot from last night) — not real-time
- ✅ Dashboard loads in **< 1 second**
- ✅ Oracle HIS load reduced by ~90%
- ✅ Predictable query patterns (no surprise heavy queries during business hours)

### Mitigation

- For cases needing real-time data, specific endpoints bypass cache and query Oracle directly
- Cache can be manually invalidated via admin API
- Anomaly detection still runs against live Oracle data (scheduled, not on-demand)

---

## Summary

| Decision | Pattern | Key Benefit |
|---|---|---|
| Multi-LLM | Fallback chain | Zero downtime |
| Dual Database | Read-only Oracle + MySQL app state | Performance + Safety |
| Multi-Agent | Router + Specialists + Reflection | Accuracy + Cost |
| MCP Protocol | Hybrid local + MCP tools | Extensibility |
| Snapshot + Cache | Pre-compute + warmup | Speed (< 1s) |
