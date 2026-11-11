# Multi-Agent Banking Assistant with RAG — Case Study

Nov 11, 2026 · @Abhi

## 1. Executive summary

NeoBank AI Assist is a multi-agent customer-service assistant for a retail bank, built in Python with FastAPI and Retrieval-Augmented Generation (RAG). One orchestrator routes each customer message to a specialist agent (Accounts, Loans, Knowledge, Fraud), and every policy answer is grounded in the bank's own documents with cited sources.

The case study is designed for teaching: the full system runs offline on a laptop with no paid API keys, using a TF-IDF retriever and a rule-based LLM stub, and uses Google Gemini (gemini-2.5-flash) as its LLM as soon as a Gemini API key is set; if Gemini is unavailable, the offline stub answers instead.

### Business problem

A mid-sized Indian retail bank (fictional "NeoBank") handles about 40,000 support contacts a month. Most are repetitive: balance checks, recent transactions, EMI questions, KYC document lists, card fee queries and "I don't recognise this transaction" reports.

The bank faces four problems:

- **Slow answers**: average first response time is around 6 minutes in chat and longer in peak hours.
- **Inconsistent policy answers**: agents quote outdated fee schedules or KYC rules.
- **Single-bot limits**: an earlier single-prompt chatbot hallucinated policy details and could not safely call account systems.
- **Compliance risk**: answers must be traceable to an approved source, and personal data (PAN, Aadhaar, card numbers) must never leak into logs or LLM prompts.

### Proposed solution in one line

Split the work across small, testable agents, ground every policy answer in retrieved bank documents, and expose everything through a secured, documented FastAPI service.

## 2. Objectives, scope and stakeholders

The project aims to resolve at least 60% of routine queries without a human, with every policy answer citing its source document.

### Objectives

1. Answer balance, transaction and EMI questions from live (mock) account data through tools, never from the LLM's memory.
2. Answer policy questions (KYC, loans, cards, fraud) only from retrieved bank documents, with citations.
3. Detect suspicious-transaction reports and escalate them with a case ID.
4. Mask PII before any text reaches logs or the LLM.
5. Expose a versioned REST API with OpenAPI docs that web, mobile and WhatsApp channels can share.

### In scope

- Text chat for retail customers (savings accounts, personal and home loans, debit and credit cards).
- Four specialist agents plus an orchestrator and router.
- RAG over four policy documents, with re-indexing through an admin endpoint.
- API-key authentication, session memory, audit trace per request.

### Out of scope

- Money movement (transfers, payments): the assistant can explain, not execute.
- Voice channels, multi-language support and real core-banking integration (mocked with JSON).
- Production-grade identity (OAuth2/OIDC) — noted as future work.

### Stakeholders

| Stakeholder | Interest | What they need from the system |
| --- | --- | --- |
| Retail customers | Fast, correct answers | 24x7 chat, plain language, cited sources |
| Contact-centre team | Fewer repetitive tickets | Clean escalations with case IDs and context |
| Compliance officer | Traceable, approved answers | Source citations, PII masking, audit trace |
| IT / platform team | Maintainable service | FastAPI, tests, Docker, config by env vars |
| Students (this course) | Learn agentic RAG | Runnable code, manual, exercises |

## 3. Requirements

The system has 10 functional and 8 non-functional requirements; each functional requirement maps to one agent or endpoint so it can be tested on its own.

### Functional requirements

| ID | Requirement | Owner component |
| --- | --- | --- |
| FR-01 | Classify each message into an intent: account, loan, knowledge, fraud, greeting or unknown | Router Agent |
| FR-02 | Return the current balance and last N transactions for an authenticated customer | Account Agent |
| FR-03 | Calculate loan EMI from principal, annual rate and tenure | Loan Agent + `/loans/emi` |
| FR-04 | Check basic loan eligibility (income, credit score, existing EMIs) | Loan Agent |
| FR-05 | Answer policy questions only from retrieved document chunks, with citations | Knowledge Agent + RAG |
| FR-06 | Detect fraud keywords and unusual transactions; raise a case ID | Fraud Agent |
| FR-07 | Keep short conversation memory per session (last 10 turns) | Session memory |
| FR-08 | Re-index policy documents on demand | `/rag/ingest` |
| FR-09 | Return an agent trace (which agents ran, timings) with each answer | Orchestrator |
| FR-10 | Fall back politely and offer a human handoff when confidence is low | Orchestrator |

### Non-functional requirements

| ID | Category | Target |
| --- | --- | --- |
| NFR-01 | Latency | p95 < 2 s with the offline stack; < 6 s with a hosted LLM |
| NFR-02 | Security | API-key header on every `/api/v1` route; secrets only from env vars |
| NFR-03 | Privacy | PAN, Aadhaar, card, phone and email masked before logging or LLM calls |
| NFR-04 | Grounding | Every Knowledge answer cites at least one source; no answer if retrieval score < threshold |
| NFR-05 | Testability | Unit and API tests with pytest; runs without network |
| NFR-06 | Portability | Python 3.10+; single Docker image |
| NFR-07 | Observability | Structured logs and per-request trace IDs |
| NFR-08 | Extensibility | New agent added by one class + one registry line |

## 4. System architecture

The system is layered: channels call one FastAPI service, an orchestrator routes each message to one of four specialist agents, and those agents reach data only through tools (core banking) or the RAG pipeline.

&#91;embedded content: NeoBank AI Assist architecture · 4 layers, 4 specialist agents\]

Read top to bottom: a request passes authentication and validation, the orchestrator consults the router and session memory, then exactly one specialist answers using the data layer beneath it.

### Technology stack

| Layer | Technology | Why |
| --- | --- | --- |
| Language | Python 3.10+ | Standard for AI/ML, rich libraries |
| API | FastAPI 0.110, Uvicorn | Async, automatic OpenAPI docs, Pydantic validation |
| Validation / config | Pydantic v2, pydantic-settings | Typed schemas, settings from env vars |
| Retrieval | scikit-learn TF-IDF, NumPy cosine search | Runs offline by default; Gemini embeddings (gemini-embedding-001) for semantic search |
| LLM | Google Gemini via the google-genai SDK; offline extractive stub as fallback; OpenAI/Anthropic optional | Same interface, chosen by `LLM_PROVIDER` |
| Testing | pytest, FastAPI TestClient | Fast, no network needed |
| Packaging | Docker (python:3.11-slim) | One image for any environment |

### Request lifecycle

1. Client sends `POST /api/v1/chat` with `X-API-Key`.
2. FastAPI validates the body against `ChatRequest`.
3. Orchestrator masks PII and asks the Router for an intent.
4. The matching agent runs; tool agents call `bank_db`, knowledge agents call the retriever and LLM.
5. Orchestrator applies the handoff rule, masks the answer, saves memory and returns `ChatResponse` with sources and trace.

## 5. Agent specifications

Each agent is one Python class with a `run(ctx) -> AgentResult` method; the Orchestrator owns routing, guardrails and memory, so agents stay small and testable.

| Agent | File | Triggers (intent) | Tools / data used | Output | Uses LLM? |
| --- | --- | --- | --- | --- | --- |
| Router | `agents/router.py` | Every message | Weighted keyword rules; LLM fallback when a real provider is set | intent + confidence | Only as fallback |
| Orchestrator | `agents/orchestrator.py` | Every message | Router, agent registry, PII masker, session memory | `ChatResponse` with trace | No |
| Account Agent | `agents/account_agent.py` | `account` | `bank_db.get_balance`, `get_transactions` | Balance or last 5 transactions | No (tools only) |
| Loan Agent | `agents/loan_agent.py` | `loan` | EMI formula, eligibility rules (FOIR 50%, score 700, income Rs 25,000), RAG on `loan_policy` | EMI, eligibility verdict or policy answer | For policy questions |
| Knowledge Agent | `agents/knowledge_agent.py` | `knowledge` (default) | Retriever (top-k 3, min score 0.08) | Grounded answer + sources | Yes |
| Fraud Agent | `agents/fraud_agent.py` | `fraud` (highest priority) | Transaction rules (international POS, debit > 40% of balance), RAG on `fraud_guidelines` | Case ID, flagged list, next steps | For policy text |

### Routing rules

1. Mask PII in the incoming message.
2. Score each intent by keyword matches; ties break in the order fraud > account > loan > knowledge > greeting, so a fraud report is never missed.
3. No keyword match: a short follow-up ("and for 15 years?") inherits the last intent from session memory; otherwise an LLM classifies it (hosted mode) or it defaults to `knowledge`.
4. Run the agent, time it, and add the step to the trace.
5. Confidence below 0.35 appends a human-handoff offer.
6. Mask PII again in the answer, save the turn to memory, return the response.

### Design choices worth discussing in class

- **Tools before generation**: balances and EMIs come from code, never from the LLM, so numbers cannot be hallucinated.
- **Specialists can reuse each other**: Loan and Fraud agents call the Knowledge Agent with a document filter instead of duplicating RAG logic.
- **Deterministic router first**: keyword routing is fast, free and auditable; the LLM is only a fallback.

## 6. RAG design

The RAG pipeline turns four policy documents into 18 section-aware chunks at startup and answers only when a retrieved chunk scores at least 0.08 cosine similarity.

### Knowledge base

| Document | Covers | Sections |
| --- | --- | --- |
| `kyc_policy.md` | OVDs, Video KYC, Re-KYC periods, minimum-KYC limits | 5 |
| `loan_policy.md` | Personal and home loans, eligibility, foreclosure, documents | 5 |
| `card_fees.md` | Debit/credit card fees, ATM charges, forex markup | 4 |
| `fraud_guidelines.md` | Reporting, customer liability, timelines, safety tips | 4 |

### Pipeline stages

1. **Load**: read every `.md` / `.txt` file in `data/docs/`.
2. **Chunk** (`rag/chunker.py`): split on `##` headings, then a sentence-aware sliding window of 500 characters with 80 characters of overlap. The section heading is prefixed to each chunk so a query like "ATM charges" matches the "ATM withdrawals" section.
3. **Embed** (`rag/embeddings.py`): TF-IDF over unigrams and bigrams with a light stemmer (default, offline) or `all-MiniLM-L6-v2` sentence embeddings, or Gemini gemini-embedding-001 vectors (768 dimensions) with separate document and query task types when EMBEDDING\_BACKEND=gemini.
4. **Index** (`rag/vector_store.py`): L2-normalised vectors in a NumPy matrix; search is one matrix product (cosine similarity).
5. **Retrieve** (`rag/retriever.py`): top 6 candidates, optional document filter (e.g. only `loan_policy` for the Loan Agent), drop scores below 0.08, keep top 3.
6. **Generate** (`agents/knowledge_agent.py`): numbered context + question go to Gemini (gemini-2.5-flash, temperature 0.1) as a system instruction plus user content, with a strict system prompt: answer only from context, cite \[n\], never ask for OTP/PIN/CVV.
7. **Cite**: the response returns document name, chunk ID, score and a snippet for each source.

### Grounding guardrails

- No chunk above the threshold → the agent says it cannot find the answer and offers a human, instead of guessing.
- Confidence is derived from the top retrieval score (0.5 + score, capped at 0.95).
- The offline "mock" LLM is extractive: it chooses the best-matching chunk (rare words weigh more) and returns its most relevant sentences, so it cannot invent facts. It also takes over automatically whenever a Gemini call fails (bad key, quota, network), so the API keeps answering.

### Production upgrade path

Switch on Gemini embeddings (and raise the score threshold to about 0.55, tuned with /rag/search), swap the NumPy store for FAISS, Chroma or pgvector, add a cross-encoder re-ranker, and version documents so each answer records the policy version it cited.

## 7. API specification

The service exposes 8 REST endpoints; everything under `/api/v1` requires the `X-API-Key` header, and FastAPI publishes live OpenAPI docs at `/docs`.

| Method | Path | Purpose | Request | Response |
| --- | --- | --- | --- | --- |
| GET | `/health` | Liveness, LLM provider, indexed chunk count | none | `{status, llm_provider, indexed_chunks}` |
| POST | `/api/v1/chat` | Main multi-agent entry point | `ChatRequest` | `ChatResponse` |
| GET | `/api/v1/agents` | List registered agents | none | `[{intent, name, description}]` |
| GET | `/api/v1/accounts/{customer_id}/balance` | Masked account number + balance | path param | `{account_no, account_type, balance, currency}` |
| GET | `/api/v1/accounts/{customer_id}/transactions?limit=5` | Recent transactions (1–50) | path + query | `[{id, date, description, amount}]` |
| POST | `/api/v1/loans/emi` | EMI calculator | `EMIRequest` | `{emi, total_payment, total_interest}` |
| POST | `/api/v1/rag/search` | Inspect retrieval (debugging, teaching) | `{query, top_k}` | `[Source]` |
| POST | `/api/v1/rag/ingest` | Rebuild the index after document changes | none | `{status, documents, chunks}` |

### Chat request and response

```json
POST /api/v1/chat
X-API-Key: dev-secret-key

{
  "session_id": "sess-001",
  "customer_id": "CUST1001",
  "message": "What are the ATM withdrawal charges?"
}
```

```json
{
  "session_id": "sess-001",
  "intent": "knowledge",
  "agent": "knowledge_agent",
  "answer": "Five free transactions per month at NeoBank ATMs and three free transactions at other banks' ATMs in metro cities. Beyond the free limit, each financial transaction costs Rs 21 plus GST. [1]",
  "confidence": 0.79,
  "sources": [
    {"document": "card_fees", "chunk_id": "card_fees#1.0", "score": 0.291, "snippet": "ATM withdrawals: Five free transactions per month..."},
    {"document": "loan_policy", "chunk_id": "loan_policy#3.0", "score": 0.111, "snippet": "Prepayment and foreclosure: There is no foreclosure charge..."}
  ],
  "trace": [
    {"agent": "router", "action": "intent=knowledge (0.95)", "duration_ms": 0.79},
    {"agent": "knowledge_agent", "action": "run", "duration_ms": 1.08}
  ],
  "escalated": false,
  "case_id": null
}
```

### Validation and errors

| Code | When |
| --- | --- |
| 401 | Wrong `X-API-Key` |
| 422 | Missing header or invalid body (e.g. empty message, message over 2,000 characters, tenure over 360 months) |
| 404 | Unknown `customer_id` on account endpoints |

Every response carries `X-Request-ID` and `X-Process-Time-ms` headers for tracing.

## 8. Data model, security and compliance

Customer data stays behind tool functions and is masked twice (on the way in and on the way out), so raw PII never reaches logs or an LLM prompt.

### Core data objects

| Object | Key fields | Where defined |
| --- | --- | --- |
| Customer (mock CBS) | name, account\_no, account\_type, balance, monthly\_income, credit\_score, existing\_emi, transactions\[\] | `data/customers.json` |
| Transaction | id, date, description, amount (negative = debit) | `data/customers.json` |
| Chunk | chunk\_id (`doc#section.window`), document, section, text, metadata.title | `rag/chunker.py` |
| AgentContext | session\_id, customer\_id, message (masked), history | `agents/base.py` |
| AgentResult | answer, confidence, sources, escalated, case\_id, data | `agents/base.py` |
| ChatRequest / ChatResponse | see API specification | `schemas.py` |

Two test customers are provided: `CUST1001` (Ananya, strong profile, one suspicious Singapore transaction) and `CUST1002` (Rohit, credit score 682, so eligibility fails).

### Security controls

| Control | Implementation | File |
| --- | --- | --- |
| Authentication | `X-API-Key` header checked by a FastAPI dependency on the whole `/api/v1` router | `services/security.py` |
| Secrets | All keys from environment / `.env`; nothing hard-coded except the dev default | `config.py` |
| PII masking | Regex redaction of PAN, Aadhaar, card numbers, Indian mobile numbers, email | `services/security.py` |
| Account masking | Only the last 4 digits of account numbers are returned | `services/bank_db.py` |
| Input limits | Pydantic: message 1–2,000 chars, EMI tenure ≤ 360 months, rate < 50% | `schemas.py` |
| Prompt safety | System prompt forbids answers outside context and requests for OTP/PIN/CVV | `agents/knowledge_agent.py` |
| Traceability | Request ID header, per-agent trace, cited chunk IDs | `main.py`, `orchestrator.py` |

### Compliance mapping (illustrative)

- **RBI KYC Master Direction**: KYC answers come only from the approved KYC policy document.
- **RBI customer-liability framework**: the Fraud Agent quotes the 3-day zero-liability rule and raises a case immediately.
- **DPDP Act 2023 (data minimisation)**: PII is redacted before storage or third-party LLM calls.
- **Model risk / explainability**: every answer carries its sources, confidence and agent trace for audit.

These mappings are teaching examples, not legal advice; a real deployment needs review by the bank's compliance team.

## 9. Testing, deployment, results and future work

The offline build passes all 18 automated tests and answers each sample scenario below correctly in under 10 ms of server time.

### Automated tests (pytest)

| File | Tests | What they prove |
| --- | --- | --- |
| `tests/test_rag.py` | 3 | Chunker keeps sections; KYC and ATM queries retrieve the right document |
| `tests/test_agents.py` | 5 | EMI formula (Rs 5,00,000 at 10.5% for 60 months = Rs 10,746.95), query parsing, router intents, PII masking, loan follow-up memory |
| `tests/test_api.py` | 6 | Health, auth (401/422), balance, KYC with sources, fraud escalation, EMI endpoint |
| `tests/test_gemini.py` | 4 | Gemini gets the system instruction and model name; a failed Gemini call falls back offline; no key means mock; embeddings use document vs query task types |

The Gemini tests use a fake client, so they run without a key. A live call with an invalid key was also checked: Google's API rejected only the key, and the chat still answered through the fallback.

### Scenario results (offline mode)

| Customer message | Routed to | Outcome |
| --- | --- | --- |
| What is my balance? | Account | Rs 84,250.75, account shown as XXXXXXXXXX5678 |
| Which documents do I need for KYC? | Knowledge | OVD list + PAN/Form 60, cited `kyc_policy` |
| What are the ATM withdrawal charges? | Knowledge | 5 free own-ATM, 3 other-bank, then Rs 21 + GST, cited `card_fees` |
| Calculate EMI for 10 lakh at 9% for 20 years | Loan | EMI Rs 8,997.26 |
| and for 15 years? | Loan (follow-up) | EMI Rs 10,142.67, reusing amount and rate |
| Am I eligible for a personal loan of 2 lakh? (CUST1002) | Loan | Not eligible: credit score 682 < 700 |
| Can I foreclose my personal loan? | Loan → RAG | After 12 EMIs, 3% charge, cited `loan_policy` |
| I don't recognise a transaction from Singapore | Fraud | Case FRD-… raised, TXN9004 flagged, zero-liability rule quoted |
| What is the weather today? | Knowledge | No evidence above threshold → polite refusal + human handoff |

### How to evaluate with a real LLM

Build a 50-question gold set (question, expected agent, expected source document, key facts) and measure routing accuracy, retrieval hit rate at k=3, answer faithfulness (every fact present in a cited chunk) and p95 latency. Re-run it whenever prompts, documents or models change.

### Deployment

- Local: `uvicorn app.main:app --reload`, Swagger at `/docs`.
- Container: `docker build -t neobank-ai-assist .` then `docker run -p 8000:8000 --env-file .env neobank-ai-assist`.
- Production shape: 2+ replicas behind an API gateway, Redis for session memory, a managed vector database, secrets in a vault, logs to a SIEM.

### Limitations

- Keyword routing misses unusual phrasing in offline mode.
- In-memory session store and index are lost on restart and not shared across replicas.
- API-key auth identifies the channel, not the customer.

### Future work

1. OAuth2/OIDC with customer-level authorisation (a customer can read only their own account).
2. LLM-based planner that can call several agents for one message ("balance and EMI options").
3. Hindi and Marathi support with multilingual embeddings.
4. Human-in-the-loop dashboard for fraud cases and low-confidence answers.
5. Automated evaluation in CI with a gold question set.
