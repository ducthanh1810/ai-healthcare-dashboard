# 🤖 AI Agent Design Documentation

> How 10 specialized AI agents power hospital analytics, automated reporting, and clinical decision support.

---

## 1. Design Philosophy

> **"Route, don't bloat."** — Specialized agents with focused toolkits beat one god-agent.

### Problems with the "One Agent" Approach

Our first version used a single ReAct agent with all 40+ tools loaded. It failed because:

1. **Context window overflow** — Tool descriptions consumed most tokens, leaving little room for reasoning
2. **Wrong tool selection** — Agent confused financial tools with HR tools
3. **Slow responses** — Every query required processing all tool options (8-15 seconds)
4. **Single point of failure** — One LLM provider down = entire AI offline

### Solution: Three-Layer Architecture

```
Router → Specialist → Reflection
```

Each layer has a single responsibility:
- **Router**: Classify intent, route to the right agent
- **Specialist**: Execute domain-specific analysis with focused tools
- **Reflection**: Verify quality before responding to user

---

## 2. Router Agent

### Intent Classification

The Router uses LLM structured output (Pydantic schema) for precise classification into 12 intent categories:

| Intent | Description | Example Query |
|---|---|---|
| `clinical` | Patient lookup, lab results, clinical alerts | "Tra cứu bệnh nhân Nguyễn Văn A" |
| `booking` | Appointment create/cancel | "Đặt lịch khám nội khoa ngày mai" |
| `analysis` | Financial analysis, KPIs | "Phân tích doanh thu tháng 4" |
| `deep_analysis` | Root cause, strategic recommendations | "Tại sao doanh thu giảm?" |
| `strategic` | Planning, improvement proposals | "Đề xuất cải thiện KPI quý 3" |
| `hospital_situation` | Operational overview, benchmarks | "Tình hình BV tuần này?" |
| `national_health` | National health data, disease trends | "So sánh với benchmark quốc gia" |
| `hr_dispatch` | Workload analysis, staff reallocation | "Phân bổ nhân sự khoa nội" |
| `hr_training` | Doctor KPIs, skill gaps, training | "Đề xuất đào tạo cho khoa ngoại" |
| `report` | Quick hospital overview | "Báo cáo tổng quan" |
| `reminder` | Schedule reminders | "Nhắc uống thuốc lúc 8h sáng" |
| `general` | Greeting, help, general Q&A | "Bot này làm được gì?" |

### Three-Layer Fallback

```
Layer 1: LLM Structured Output (Pydantic schema)
    → Precise, includes confidence score
    ↓ (parse error or LLM timeout)
Layer 2: Text Response Parsing
    → Extract intent keyword from free text
    ↓ (LLM completely down)
Layer 3: Keyword Regex Matching
    → Pattern match Vietnamese/English keywords
    → Always works, no API needed
```

### Confidence-Based Routing

- **≥ 0.6**: High confidence → route directly
- **0.4 – 0.6**: Medium confidence → route with escalation flag
- **< 0.4**: Low confidence → ask user to clarify instead of guessing wrong

---

## 3. Specialist Agents

### Agent Registry

| Agent | Complexity | Tools | Max Iters | Use Case |
|---|---|---|---|---|
| **FinancialAnalyst** | Complex | 20+ | 10 | Revenue, insurance, forecasting, anomalies |
| **HospitalAnalyst** | Complex | 4 | 6 | Bed occupancy, operational benchmarks |
| **NationalHealthExpert** | Complex | 4 | 5 | National data comparison, disease trends |
| **StrategicPlanner** | Complex | 4 | 8 | Strategic recommendations, historical analysis |
| **ClinicalAgent** | Complex | 4 | 5 | Patient lookup, medical guidelines (RAG) |
| **HRDispatchAgent** | Complex | 3 | 6 | Workload analysis, staff reallocation |
| **TrainingAdvisor** | Complex | 3 | 6 | Doctor KPIs, training recommendations |
| **BookingAgent** | Simple | 2 | 3 | Create/cancel appointments |
| **ReminderAgent** | Simple | 1 | 3 | Schedule reminders |
| **GeneralAssistant** | Simple | 2 | 3 | Help, SOP search, knowledge base |

### Right-Sizing Strategy

**Complex agents** (GPT-4 class):
- Tasks requiring multi-step reasoning
- Tools that return large datasets needing interpretation
- Higher `max_iters` for deeper analysis
- Parallel tool calls enabled for data gathering

**Simple agents** (GPT-3.5 class):
- Straightforward tasks (book appointment, set reminder)
- Few tools with clear inputs/outputs
- Low `max_iters` — finish fast
- Sequential tool calls

**Cost impact**: ~60% reduction by routing simple queries to cheap models.

### Agent Factory Pattern

Agents are lazily created and cached. Key features:

- **Lazy initialization**: Agent only created on first request
- **Singleton per type**: Same agent instance reused across requests
- **Force-new**: Can recreate agent with fresh model config (API key rotation)
- **Persistent memory**: MySQL-backed conversation memory for chat sessions
- **MCP-aware**: Hybrid toolkit combining local tools + MCP-discovered tools

---

## 4. Tool System

### Tool Categories

| Category | Tools | Source | Priority |
|---|---|---|---|
| **Analytics** | Revenue metrics, trend analysis, bed performance | Backend REST API | High |
| **Financial** | Insurance, executive summary, period comparison | Backend REST API | High |
| **Diagnostic** | Revenue drivers, anomaly detection, benchmarking | Local algorithms | High |
| **Predictive** | Revenue forecast, scenario simulation | Local algorithms | Medium |
| **HR** | Workload analysis, reallocation, KPIs, training | Backend REST API | High |
| **Clinical** | Patient lookup, dashboard summary | Backend REST API | High |
| **RAG** | Medical guidelines, hospital SOPs, drug info | ChromaDB vector search | Medium |
| **National** | National benchmarks, disease trends | Static + computed data | Low |
| **Operations** | Appointments, reminders | Backend REST API | High |

### MCP Integration

Tools are accessed via two mechanisms:

1. **Local functions** — Direct REST calls to FastAPI backend (prioritized)
2. **MCP tools** — Discovered at runtime from MCP Tool Server (supplemental)

The hybrid toolkit builder:
- Registers all local functions first (high priority)
- Discovers MCP tools by category
- Caps total tools to prevent context overflow
- Local tools always take precedence on name collision

---

## 5. Reflection Layer

### Quality Evaluation Pipeline

Before any response reaches the user, 5 automated checks run:

| Check | What it Catches | Severity |
|---|---|---|
| **Empty/Short** | Agent returned < 20 chars | 🔴 Critical |
| **All Tools Failed** | Every API call errored, agent ignored it | 🔴 Critical |
| **Ungrounded Medical** | Specific dosages/treatments without RAG source | 🔴 Critical |
| **Repetition** | Same sentence repeated 3+ times (LLM loop) | 🟡 Warning |
| **Generic Non-answer** | "I can't help" when tools are available | 🔴 Critical |

### Scoring & Retry Logic

```
Quality Score = 1.0 - (issue_count × 0.2)

If score < 0.4 OR critical issue found:
    → Generate improved prompt with retry hints
    → Re-run specialist agent (max 1 retry)
    → If still fails → return with disclaimer

If score ≥ 0.4:
    → Send response to user
```

### Medical Safety

For clinical intent, the Reflection layer specifically checks:
- Dosage patterns (e.g., "500mg", "liều 2 lần/ngày")
- Contraindication mentions
- Whether RAG tools were actually used and returned results

If medical claims are present without RAG backing → **blocks response and retries with RAG-first instruction**.

---

## 6. Multi-LLM Strategy

### Provider Chain

| Order | Provider | Model Type | Strengths | Weakness |
|---|---|---|---|---|
| 1 | **OpenRouter** | DeepSeek-V4 | Best cost/performance | Rate limits |
| 2 | **Google Gemini** | Gemini Flash | Free tier, good quality | Quota limits |
| 3 | **Groq** | Various | Fastest inference | Limited context |
| 4 | **Ollama** | Local models | Always available | Requires server GPU |

### Failover Mechanism

- Automatic detection of 429 (rate limit) and timeout errors
- Seamless switch to next provider in < 2 seconds
- `force_new=True` flag enables API key rotation without system restart
- Per-agent model selection: complex agents get best model, simple agents get cheapest

### Two Model Tiers

- **`get_simple_model()`** — For Router, Booking, Reminder, General (cost-optimized)
- **`get_complex_model()`** — For Financial, Clinical, Strategic, HR (quality-optimized)

---

## 7. RAG Pipeline

### Knowledge Sources

| Source | Content | Update Frequency |
|---|---|---|
| Hospital SOPs | Standard operating procedures | Manual ingestion |
| Medical Guidelines | Clinical practice guidelines | Manual ingestion |
| Drug Database | Medication information | Periodic update |
| KPI Dictionary | KPI definitions and targets | Semi-annual |

### Search Strategy

**Hybrid Search** = Dense vectors + Sparse keywords

- Dense: ChromaDB embedding similarity (semantic understanding)
- Sparse: BM25-style keyword matching (exact term matching)
- Combined: Weighted fusion for best of both worlds

### Integration with Agents

RAG tools are registered as standard agent tools:
- `search_medical_guidelines(query)` → returns relevant guideline chunks
- `search_hospital_sop(query)` → returns SOP sections
- `search_drug_info(drug_name)` → returns drug information
- `get_knowledge_stats()` → returns knowledge base statistics

---

## 8. Performance Metrics

| Metric | Before (v1) | After (v2) |
|---|---|---|
| Routing accuracy | ~60% | **~92%** |
| Average response time | 8-15s | **3-6s** |
| Failed morning reports | 3/week | **0/month** |
| Monthly LLM cost | ~$80 | **~$35** |
| Agent types | 1 (monolith) | **10 (specialized)** |
| Tool organization | Flat (40+) | **Categorized (2-20 per agent)** |
