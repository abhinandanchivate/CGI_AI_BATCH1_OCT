# Banking AI Assistant — Complete Understanding Workflow

Nov 6, 2026 · @Abhi

## How to use this guide

Read the 17 files in the order below and you will understand the whole project in about 3 hours. The project is a chat API for a bank. FastAPI receives a message, a LangGraph workflow decides what to do, LangChain tools do the banking work, Gemini understands and phrases the language, and LangSmith records every step.

The code is layered so each file only uses the files above it. Read from the bottom layer (settings and data) up to the top layer (the API), then run the tests and demo to see it all move.

| Step | File | Why read it now | Time |
| --- | --- | --- | --- |
| 1 | `README.md` | The goal, architecture and run commands | 15 min |
| 2 | `requirements.txt`, `.env.example` | What is installed and what you can configure | 5 min |
| 3 | `app/config.py` | Every setting and business rule in one place | 10 min |
| 4 | `app/data.py` | The fake bank: customers, transactions, policies | 10 min |
| 5 | `app/tools.py` | The banking operations, as LangChain tools | 25 min |
| 6 | `app/llm.py` | How Gemini classifies intent and writes replies | 25 min |
| 7 | `app/graph.py` | The LangGraph workflow that ties it together | 40 min |
| 8 | `app/main.py` | The FastAPI endpoints that call the graph | 20 min |
| 9 | `tests/test_api.py` | Each feature proved by a test | 15 min |
| 10 | `demo.py` | A full conversation against the running server | 10 min |
| 11 | `ASSIGNMENT.md` | What to build next | 10 min |

`app/__init__.py` and `tests/__init__.py` are empty. They only mark the folders as Python packages so `from app import ...` works.

## Project map

The files form five layers, and each file imports only from the layers below it. Read the picture top-down to follow a request, or bottom-up to follow the reading order.

&#91;embedded content: project map · 10 files, 2 outside services\]

`main.py` also imports `config.py` (for `/health`) and `data.py` (to check the customer exists). `requirements.txt` lists the packages every box needs. `README.md` and `ASSIGNMENT.md` are documents only, and the two `__init__.py` files are empty.

## File-by-file walkthrough

Each file has one job. For every file: what it does, the parts to study, and the question to be able to answer after reading it.

### requirements.txt

Lists the packages: `fastapi` and `uvicorn` (web server), `langchain` and `langchain-core` (tools, messages), `langgraph` (workflow), `langsmith` (tracing), `langchain-google-genai` (Gemini), `python-dotenv` (reads `.env`), `pytest` and `httpx` (testing).

**Check yourself:** which package would you remove to drop Gemini, and what would still work?

### .env.example

A template for your `.env` file, copied once and never committed. It has three groups: the Gemini settings (`LLM_MODE`, `GOOGLE_API_KEY`, `GEMINI_MODEL`), the LangSmith settings (`LANGSMITH_TRACING`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT`), and the bank rules in rupees.

**Check yourself:** what happens if `GOOGLE_API_KEY` is empty?

### app/config.py

Calls `load_dotenv()` and turns every environment variable into a Python constant. Key lines:

- `USE_GEMINI = LLM_MODE == "gemini" and bool(GOOGLE_API_KEY)` decides whether real Gemini is used. Without a key it prints a warning and falls back to mock mode.
- `LLM_MODEL` is only a label (`gemini:gemini-3.5-flash` or `mock`) shown by `/health`.
- `APPROVAL_THRESHOLD` (Rs 50,000), `DAILY_TRANSFER_LIMIT` (Rs 1,00,000), `LOAN_INTEREST_RATE` (10.5%), `MIN_CREDIT_SCORE` (700) and `MAX_FOIR` (50%) are the bank rules.
- `LANGSMITH_ENABLED` only reports the flag. Tracing itself is switched on by LangChain reading the env vars directly.

**Check yourself:** where would you change the approval limit to Rs 25,000, without editing code?

### app/data.py

An in-memory fake core banking system. It resets every time the server restarts.

- `CUSTOMERS`: two customers. C1001 Asha Patil (balance Rs 1,85,000, score 782, income Rs 1,20,000, beneficiaries SB2001 and SB3001). C1002 Rahul Deshmukh (balance Rs 42,000, score 655).
- `TRANSACTIONS`: recent entries per customer, dated relative to today.
- `TRANSFERRED_TODAY`: running total of today's transfers, used for the daily limit.
- `POLICY_DOCS`: six short policy texts (FD rates, KYC, charges, lost card, fraud, personal loan) used by the FAQ.

**Check yourself:** why does C1002 fail the loan check?

### app/tools.py

Six banking operations, each a LangChain tool made with `@tool`. The docstring becomes the tool description, and each call shows as a run in LangSmith. Nodes call them with `.invoke({...})`.

| Tool | Reads or writes | What it does |
| --- | --- | --- |
| `get_balance` | Reads | Returns account number, type and balance |
| `get_transactions` | Reads | Returns the latest 5 transactions |
| `validate_transfer` | Reads | Checks amount, beneficiary, balance and daily limit; says if approval is needed |
| `execute_transfer` | Writes | Debits sender, credits receiver if internal, adds transactions, returns a reference |
| `check_loan_eligibility` | Reads | Computes EMI, FOIR, eligibility reasons and the maximum loan |
| `search_policies` | Reads | Scores policy docs by matching words and returns the top 2 |

The EMI uses the standard formula, with r as the monthly rate and n the months:

```latex
EMI = P \cdot \frac{r(1+r)^n}{(1+r)^n - 1}
```

FOIR is (existing EMI + new EMI) divided by monthly income. Above 50% means not eligible.

**Check yourself:** why is validation a separate tool from execution?

### app/llm.py

The only file that talks to Gemini. It has four parts.

1. `IntentResult`: a Pydantic model with `intent` (one of 7 values), `amount`, `to_account` and `tenure_months`. Gemini must return exactly this shape.
2. Two prompts. `CLASSIFY_PROMPT` lists the intents and tells Gemini to convert "1 lakh" to 100000 and years to months. `ANSWER_PROMPT` tells it to answer only from the tool result, in rupees, without asking for OTP or PIN.
3. `get_llm()` creates `ChatGoogleGenerativeAI` once and caches it, or returns `None` in mock mode.
4. `classify()` and `answer()`, both decorated with `@traceable` so they appear by name in LangSmith. `classify` uses `llm.with_structured_output(IntentResult)`. In mock mode it uses regular expressions (`_mock_classify`, `_parse_amount`) instead. `answer` returns the pre-written `fallback` text in mock mode.

**Check yourself:** what does Gemini decide, and what does it never decide?

### app/graph.py

The heart of the project. It defines the state, eight nodes, two routing functions and the compiled graph.

- `BankState` is the shared data passed between nodes (see the State lifecycle section).
- `guardrail` uses three regular expressions: prompt injection, OTP/PIN/CVV/password sharing, and other customer ids. If any match it sets `blocked=True` and replies.
- `classify` stores Gemini's `IntentResult` in `state["intent"]`.
- Six worker nodes each call tools and return a reply: `balance_node`, `transactions_node`, `transfer_node`, `loan_node`, `faq_node`, `smalltalk_node`.
- `transfer_node` calls `interrupt()` for amounts over the approval limit. The graph stops and saves its state. When resumed, the node runs again from the top and `interrupt()` returns the approval decision, so nothing may change data before that line.
- `after_guardrail` and `route_intent` are the routing functions used by conditional edges.
- `build_graph()` wires it all and compiles with `MemorySaver`, which saves state per `thread_id`.
- Running `python -m app.graph` prints the graph as a Mermaid diagram.

**Check yourself:** what would go wrong if `execute_transfer` were called before `interrupt()`?

### app/main.py

The FastAPI layer. It validates input with Pydantic models (`ChatRequest`, `ApprovalRequest`, `ChatResponse`) and calls the graph.

| Endpoint | What it does |
| --- | --- |
| `GET /health` | Returns status, active model and tracing flag |
| `POST /chat` | Checks the customer exists, makes a `thread_id` if none, refuses if a transfer is pending, runs `graph.invoke`, returns the reply or the pending approval |
| `POST /approve` | Resumes a paused graph with `Command(resume={"approved": ...})` |
| `GET /history/{thread_id}` | Returns all messages saved for that thread |

`_respond()` reads `graph.get_state()`. If `snap.next` is not empty the graph is paused, so it returns `pending_approval` with the interrupt payload. Otherwise it returns the reply and intent. The `metadata` and `tags` passed in the config appear on the LangSmith trace.

**Check yourself:** why does `/chat` return HTTP 409 sometimes?

### tests/test\_api.py

Eleven tests using FastAPI's `TestClient`, so no server is needed. The first lines force `LLM_MODE=mock`, so tests never call Gemini and need no key. Run with `pytest -q`.

### demo.py

A script that talks to a running server through `httpx`. It walks through greeting, balance, transactions, a small transfer, a Rs 60,000 transfer (asking you to approve it), a loan check, an FD question and an OTP warning.

### README.md and ASSIGNMENT.md

`README.md` is the setup guide: architecture, run steps, Gemini wiring, API table and troubleshooting. `ASSIGNMENT.md` is the 100-mark extension task, with a rubric and viva questions.

## End-to-end request traces

Three requests cover every path through the code: a normal read, a transfer that needs approval, and a blocked message.

### Trace 1: a Rs 60,000 transfer with approval

This is the richest path, so it is drawn. Solid arrows are calls, dashed arrows are returns, and the shaded band is the time the graph waits for a person.

&#91;embedded content: Rs 60,000 transfer, two HTTP calls, one pause\]

The node runs twice. On the second run `interrupt()` returns the approval decision instead of pausing, so validation repeats and only then does `execute_transfer` change the balance. If the customer rejects, the node replies that the transfer is cancelled and no money moves.

### Trace 2: a balance question

1. `main.chat()` checks that C1001 exists, creates a `thread_id`, resets `intent`, `reply` and `blocked`, and calls `graph.invoke()`.
2. `graph.guardrail()` finds no injection, OTP request or other customer id, so `after_guardrail` returns `classify`.
3. `graph.classify()` calls `llm.classify()`, and Gemini returns the intent `balance`.
4. `route_intent` sends the state to `balance_node`, which calls `tools.get_balance` to read `data.CUSTOMERS`.
5. `llm.answer()` asks Gemini to phrase the result. In mock mode the prepared sentence is used instead.
6. `_say()` stores the reply in `reply` and `messages`. `main._respond()` sees that `snap.next` is empty and returns `status: completed`.

### Trace 3: a request to share an OTP

1. `main.chat()` calls `graph.invoke()` as usual.
2. `graph.guardrail()` matches the `SECRETS` pattern plus the word share. It sets `blocked` to True and writes the safety reply.
3. `after_guardrail` returns `end`, so no classify, no tools and no Gemini call happen.
4. `main._respond()` returns the warning with an empty intent.

## State lifecycle

`BankState` is one dictionary that every node reads and returns pieces of. LangGraph merges what each node returns into it, and `MemorySaver` stores a copy after every node under the request's `thread_id`.

| Field | Set by | Read by | Merge rule |
| --- | --- | --- | --- |
| `messages` | `main.py` (user message), every worker node via `_say()` (bot reply) | `/history` endpoint | Appended (`add_messages`), so the chat history grows each turn |
| `customer_id` | `main.py` on each `/chat` | Every tool call, `guardrail` | Overwritten |
| `question` | `main.py` on each `/chat` | `guardrail`, `classify`, worker nodes | Overwritten |
| `blocked` | `main.py` resets to False; `guardrail` sets it | `after_guardrail` router | Overwritten |
| `intent` | `main.py` resets to None; `classify` fills it | `route_intent`, `transfer_node`, `loan_node`, `smalltalk_node`, `_respond()` | Overwritten |
| `reply` | Worker nodes and `guardrail` | `_respond()` in `main.py` | Overwritten |

Two points to remember:

- **Memory is per thread.** Send the same `thread_id` and `messages` keeps growing. Send none and `main.py` creates a new thread with an empty history.
- **Memory is in RAM.** `MemorySaver` loses everything, including a pending approval, when the server restarts. The assignment bonus asks you to replace it with SQLite or Postgres.

## Gemini and LangSmith

Gemini is called at most twice per request, both times from `app/llm.py`. LangSmith needs no code: it records everything once three env vars are set.

### Where Gemini is used

| Call | Function | Input | Output | Used by nodes |
| --- | --- | --- | --- | --- |
| 1. Understand | `llm.classify()` | `CLASSIFY_PROMPT` + customer message | `IntentResult` (intent, amount, to\_account, tenure\_months) | `classify` |
| 2. Phrase | `llm.answer()` | `ANSWER_PROMPT` + message + tool result | Friendly reply text | `balance`, `transactions`, `loan`, `faq` |

Gemini is not called for guardrail checks, transfer messages or small talk. Those replies are fixed text in `graph.py`, so money movement wording stays exact. A blocked message makes zero Gemini calls.

### What a LangSmith trace shows

Set `LANGSMITH_TRACING=true`, `LANGSMITH_API_KEY` and `LANGSMITH_PROJECT`, then restart the server. For "What is my balance?" the trace tree looks like this:

```
LangGraph                      tags: banking-api, metadata: customer_id=C1001
├── guardrail
├── classify
│   └── classify_intent        (@traceable in llm.py)
│       └── ChatGoogleGenerativeAI   prompt, JSON output, tokens, latency
├── balance
│   ├── get_balance            (tool: input and output)
│   └── generate_answer        (@traceable in llm.py)
│       └── ChatGoogleGenerativeAI
```

Use the trace to answer three debugging questions: Did Gemini pick the right intent? Did the tool get the right arguments? Which step was slow? A paused transfer appears as a run that ends at `transfer`, followed by a second run when `/approve` resumes it.

## Config and test maps

Every setting changes one behaviour, and every feature is proved by at least one of the 11 tests.

### What each setting changes

| Setting in `.env` | Default | Read in | Changes |
| --- | --- | --- | --- |
| `LLM_MODE` | gemini | `config.py` → `llm.get_llm()` | Gemini or offline regex mode |
| `GOOGLE_API_KEY` | none | `config.py` → `llm.get_llm()` | Missing key forces mock mode |
| `GEMINI_MODEL` | gemini-3.5-flash | `llm.get_llm()` | Which Gemini model answers |
| `GEMINI_TEMPERATURE` | 0 | `llm.get_llm()` | Randomness of Gemini output |
| `APPROVAL_THRESHOLD` | 50000 | `tools.validate_transfer` | Amount above which `interrupt()` pauses |
| `DAILY_TRANSFER_LIMIT` | 100000 | `tools.validate_transfer` | Total allowed per customer per day |
| `LOAN_INTEREST_RATE` | 10.5 | `tools.check_loan_eligibility` | EMI and maximum loan |
| `MIN_CREDIT_SCORE` | 700 | `tools.check_loan_eligibility` | Loan rejection by score |
| `MAX_FOIR` | 0.5 | `tools.check_loan_eligibility` | Loan rejection by EMI burden |
| `LANGSMITH_TRACING` and keys | false | LangChain itself | Sends traces to LangSmith |

### Which test proves which feature

| Test in `tests/test_api.py` | Feature | Path through the code |
| --- | --- | --- |
| `test_health` | API is up, mock mode active | `main.health` |
| `test_balance` | Balance enquiry | guardrail → classify → `balance_node` → `get_balance` |
| `test_transactions` | Mini statement | → `transactions_node` → `get_transactions` |
| `test_small_transfer_no_approval` | Transfer under Rs 50,000 | → `transfer_node` → validate → execute |
| `test_large_transfer_needs_approval_then_approve` | Human approval | → `interrupt()` → `/approve` → `Command(resume)` → execute |
| `test_daily_limit_then_rejection` | Daily limit, then a rejected approval | → `validate_transfer` fails; then resume with approved=false |
| `test_unregistered_beneficiary` | Beneficiary check | → `validate_transfer` fails |
| `test_loan_eligible_and_not` | Loan rules for C1001 and C1002 | → `loan_node` → `check_loan_eligibility` |
| `test_faq` | Policy question | → `faq_node` → `search_policies` |
| `test_guardrails` | OTP warning and privacy block | `guardrail` → END |
| `test_memory_across_turns` | Chat memory | Same `thread_id` → `/history` returns 4 messages |

## Hands-on learning path

Four short sessions take you from running the code to changing it. Tick each task when done.

### Session 1: run it (45 min)

- [ ] Create the virtual environment and `pip install -r requirements.txt`
- [ ] Copy `.env.example` to `.env`, leave the key empty, run `pytest -q` and see 11 passed
- [ ] Start `uvicorn app.main:app --reload` and open `/docs`; call `/health` and confirm `mock`
- [ ] Run `python demo.py` and approve the Rs 60,000 transfer
- [ ] Run `python -m app.graph` and paste the output into mermaid.live

### Session 2: add Gemini and LangSmith (45 min)

- [ ] Add `GOOGLE_API_KEY`, restart, and confirm `/health` shows the Gemini model
- [ ] Send "can you move fifteen thousand rupees to SB2001" and compare with mock mode, which cannot read words as numbers
- [ ] Turn on LangSmith and find the `classify_intent` run; read the exact prompt and JSON Gemini returned
- [ ] Find a paused transfer in LangSmith and its resumed run

### Session 3: trace by hand (45 min)

- [ ] Put a breakpoint or `print` in `guardrail`, `classify` and `transfer_node`; send one transfer and note the order
- [ ] Call `/chat` twice on a pending thread and explain the 409
- [ ] Change `APPROVAL_THRESHOLD` to 1000 in `.env`, restart, and watch a small transfer pause
- [ ] Use `/history/{thread_id}` after five messages and explain the count

### Session 4: change it (60 min)

- [ ] Add a policy to `POLICY_DOCS` about cheque books and ask about it
- [ ] Add a new intent `branch_hours`: update `IntentResult`, `CLASSIFY_PROMPT`, `_mock_classify`, a new node, the routing map, and a test
- [ ] Make the OTP guardrail also catch "UPI PIN"
- [ ] Start the assignment in `ASSIGNMENT.md`

### Self-check questions

1. Name the six files a balance request passes through, in order.
2. Why does the code route with conditional edges instead of letting Gemini call tools freely?
3. What does `interrupt()` return the first time, and the second time `transfer_node` runs?
4. Which two fields reset at the start of every `/chat`, and why?
5. If Gemini returns `amount: null` for a transfer, which function catches it and what does the user see?
6. What would you need to change to keep chat history after a restart?
